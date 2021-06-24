
# 17_direct_messages

## 要件

- グループチャット機能の実装
- ２人〜何人までも参加できる要件にする(グループチャットであるということ)
- ユーザーの詳細画面に「メッセージ」というボタンを配置し、それを押すとそのユーザーとのチャットルームに遷移するようにする
- チャットルームの一覧画面にはチャットルームの情報を表示する。内容は以下の通り
  - チャットルームに参加しているユーザーのうち、自分以外のユーザーを１人ランダムで抽出し、そのユーザーのアバターを表示する
  - チャットルームに参加しているユーザーのうち、自分以外のユーザーの名前を`,`で区切って表示させる
  - 最新のメッセージ
  - 要はLINEの様なイメージ
- チャットルーム一覧画面にグループチャットを開始できるボタンを追加する
  - ボタンを押すとモーダルが表示され、そのモーダルにはフォロワーの一覧が出てくる様にする
  - グループチャットを開始したいユーザーのチェックボックスをチェックして「作成」を押すとそのユーザーが参加しているチャットルームが作成される
- 余裕があれば通知機能を実装する
  - 自分が参加しているチャットにメッセージがあった時
    - XXX(リンク)からメッセージが来ています
    - 通知そのものに対してはXXXへのリンクを張る
- メッセージに対する既読管理ができるとより本格的になる
- 未読状態が10分以上続く場合は、「メッセージを見逃しています」などのメールをユーザーに送る機能をつければほぼ完璧なプロダクト
- ActionCableを使ってリアルタイムにチャットの投稿、編集、削除が他のユーザーのブラウザにも反映されるようにする

## tables

Chatrooms

|id|name|
|-|-|
|1|1:4|
|2|1:10:15:18|

「誰がどのチャットに参加しているか」は下記のChatroomUsersテーブルで表現できるが、それだけだとSQLがかなり複雑になるため、参加しているユーザーのidを`:`で繋げた形で非正規形にデータを持たせるようにする。上の例だとユーザーidが1,4のユーザーがチャットルーム1に所属、ユーザーidが1,10,15,18のユーザーがチャットルーム２に所属していることを表現。

ChatroomUsers

|id|user_id|chatroom_id|
|-|-|-|
|1|1|1|
|2|2|1|
|3|3|1|
|4|1|2|
|5|10|2|

Messages

|id|user_id|chatroom_id|body|
|-|-|-|-|
|1|1|1|nice to meet you|
|2|2|1|nice to meet you, too|

## commit

### DataBase作成

```ruby
# chatrooms

name: string , null: false

$ be rails g model Chatroom name:string
```

```ruby
# chatroom_users

user :references
chatroom :references
last_read_at :datetime

add_index :chatroom_users, [:user_id, :chatroom_id], unique: true

$ be rails g model Chatroom_user user:references chatroom:references last_read_at:datetime
```

```ruby
# messages

user :references
chatroom :references
body :text, null: false

$ be rails g model Message user:references chatroom:references body:text
```

### associations

```ruby
# chatroom

class Chatroom < ApplicationRecord
  has_many :chatroom_users, dependent: :destroy
  has_many :users, through: :chatroom_users
  has_many :messages, dependent: :destroy
end
```

```ruby
# chatroom_users

class ChatroomUser < ApplicationRecord
  belongs_to :user
  belongs_to :chatroom
end
```

```ruby
# messages

class Message < ApplicationRecord
  belongs_to :chatroom
  belongs_to :user
end
```

```ruby
# users

class User < ApplicationRecord
  has_many :chatroom_users, dependent: :destroy
  has_many :chatrooms, through: :chatroom_users
  has_many :messages, dependent: :destroy
end
```

## commit

### routing (のちに使わなくなるが備考として)

chatroomsに対するルーティングを定義する

```ruby
resources :chatrooms, only: %i[create show]
```

### Chatrooms_contoller (のちに使わなくなるが備考として)

```ruby
class ChatroomsController < ApplicationController
  def create
    user = User.find(params[:user_id])
    chatroom = current_user.start_chat_with!(user)
    redirect_to chatroom_path(chatroom)
  end
end
```

### User model (のちに使わなくなるが備考として)

```ruby
class User < ApplicationRecord

  def start_chat_with!(user)
    chatroom = Chatroom.new
    chatroom.users = [self, user]
    chatroom.save!
    chatroom
  end
end
```

current_userがAさんとチャットを始める場合、userにはAさんの情報が格納される。次に`start_chat_with!(user)`メソッドにて、Chatroomのインスタンスを生成し、そのインスタンスに紐づくusersにcurrent_userとAさんを格納し、そのデータベースに保存する。そして新規に作成したchatroomのページへ遷移させる。

## commit

### routing

```ruby
resources :chatrooms, onyl: %i[create, show], shallow: true do
  resources :messages
end
```

生成されるルーティング...


```
chatroom_messages GET    /chatrooms/:chatroom_id/messages(.:format)    messages#index
                  POST   /chatrooms/:chatroom_id/messages(.:format)    messages#create
new_chatroom_message GET    /chatrooms/:chatroom_id/messages/new(.:format)    messages#new
        chatrooms POST   /chatrooms(.:format)                            chatrooms#create
          chatroom GET    /chatrooms/:id(.:format)                    chatrooms#show
```

### messages_controller

次にmessages_controllerを作成し、以下の様にする

```ruby
class MessagesController < ApplicationController
  def create
    @message = current_user.messages.create(message_params)
  end

  def edit
    @message = current_user.messages.find(params[:id])
  end

  def update
    @message = current_user.messages.find(params[:id])
    @message.update(message_update_params)
  end

  def destroy
    @message = current_user.messages.find(params[:id])
    @message.destroy!
  end

  private
  def message_params
    params.require(:message).permit(:body).merge(:chatroom_id: params[:chatroom_id])
  end

  def message_update_params
    params.require(:message).permit(:body
  end
end
```

これらの操作ができるのは、current_userのみなので`current_user.messages`と所有者を絞った上で検索させる。


### validation of body:message

models/message内でバリデーションを定義する`validates :body, presence: true, length: { maximum: 1000 }`

### models/chatroom

以下の通り、クラスメソッドを定義する

```ruby
class Chatroom < ApplicationRecord

  def self.chatroom_for_users(users)
    user_ids = users.map(&:id).sort
    name = user_ids.join(":").to_s

    if chatroom = where(name: name).first
      chatroom
    else
      chatroom = new(name: name)
      chatroom.users = users
      chatroom.save
      chatroom
    end
  end
end
```

current_userがAさんとチャットする場合、すでにcurrent_user.idとA.idの組み合わせを持つnameを持つレコードがあれば、そのレコードをchatroomに格納。なければ、そのレコードを新規に作成する。

引数であるusersは、`chatroom_path(Chatroom.chatroom_for_users([current_user, user]))`で表現させる

### draper導入

`decorator`とは、デザインパターン(オブジェクト指向言語で開発を行うときに、先達がまとめた「よく出会う問題とそれに対する良い設計」のこと)の１つであり、既存のオブジェクトを新しい`Decorator`オブジェクトでラップすることで既存のメソッドやクラスの中身を直接触ることなく、その外側から機能を追加したり書き換えたりできる機能のこと。

viewで使いたいメソッドはModelに書くのではなく、Decoratorファイルに書くということ。Modelの肥大化防止にもつながる

[Decoratorやそのgem`draper`について学んだことはこちらにまとめました。](https://github.com/koteharyu/TIL/blob/main/insta_clone/17_direct_messages/decorator.md)

では、早速`draper`を導入する。なお、gemのインストールは済んだものとする。

`$ bundle exec rails g decorator Message` コマンドを実行することで以下のファイルが生成される

```ruby
# app/decorators/message_decorator.rb

class MessageDecorator < ApplicationDecorator
  delegate_all
  def created_at
    helpers.content_tag :span, class: 'time' do
      object.created_at.strftime('%Y-%m-%d %H:%M:%S')
    end
  end
end
```

この様にデコることで、見辛かったcreated_atが返すデータではなく、2021-6-12 10:15:34の様に見慣れた形式で表示できる

`delegate_all`...これを記述することで、XXXモデルで定義した全てのメソッドがdecorator内でも使える様になる。これがつまり、既存のオブジェクトを新しいDecoratorオブジェクトでラップすることで既存の関数やクラスの中身を直接触ることなく、その外側から機能を追加したり書き換えたりする。」という部分のこと。

`object`...デコレートしているモデルを参照するためのメソッド。XXXモデルの`last_name`メソッドを利用することを明示する。

`model`...`object`メソッドのエイリアス。

`decorate`メソッド...デコレーター層のメソッドを使う時に必要なメソッド

### chatrooms/show

```
.container
  .row
    .col-sm-8.offset-sm-2
      .card
        .card-header
          h2 チャットルーム
        .card-body
          ul.list-unstyled#message-box
            = render @chatroom.messages.include(:user)
          = render 'messages/form', chatroom: @chatroom, message: @message ||= Message.new
```

### messages/_form

```ruby
= form_with model: [chatroom, message], remote: true do |f|
  = render 'shared/error_messages', object: message
  .form-group
    = f.label :body
    = f.text_field :body, class: 'form-control mb-3 input-message-body'
    - if message.new_record?
      = f.submit "投稿", class: 'btn btn-raised btn-primary'
    - else
      = f.submit "更新", class: 'btn btn-raised btn-primary'
```

`new_record?`メソッド...ActiveRecordが持っているメソッド。`objectが保存されていない時だけtrue`を返す。

`persisted?`メソッド...`objectが保存されていない、かつ今までに削除されていない時true`を返す。

### messages/_message

```ruby
li.media.mb-3.border-bottom.p-2 id="message-#{message.id}"
  div
    div
      = link_to message.user.name, user_path(message.user)
    = image_tag message.user.avatar_url, class: 'mr-3 rounded-circle', size: '50x50'
  .media-body
    = simple_format(message.body)
    div.text-right
      = message.decorate.created_at  # decoratorの出番！
    div.text-right
      - if current_user&.onw?(message)
        = link_to message_path(message), class: 'mr-3', method: :delete, data: { confirm: '本当に削除しますか?' }, remote: true do
          = icon 'far', 'trash-alt', class: 'fa-sm'
        = link_to edit_message_path(message), remote: true do
          = icon 'far', 'edit', class: 'fa-sm'
```

### messages/_modal_form

```
.modal#message-edit-modal
  .modal-dialog
    .modal-content
      .modal-header
        h5.modal-title メッセージ編集
        button.close aria-label="Close" data-dismiss="modal" type="button"
          span aria-hidden="true" x
      .modal-body
        = render 'form', chatroom: nil, message: @message
```

### messages/creata.js

```
- if @message.errors.present?
  | alert("#{@message.errors.full_messages.join("\n")});
- else
  | $('#message-box').append("#{j render('messages/message', message: @message)}");
  | $('.input-message-body').val("");
```

### messages/destroy.js

```
| $('#message-#{@message.id}').remove();
```

### messages/edit.js

```
| $("#modal-container").html("#{escape_javascript(render 'modal_form', message: @message)}");
| $("#message-edit-modal").modal('show');
```

### messages/update.js

```
- if @message.errors.present?
  | alert("#{@message.errors.full_messages.join("\n")}");
- else
  | $('#message-#{@message.id}').replaceWith("#{j render('message/message', message: @message)}");
  | $('#message-edit-modal').modal('hide');
```

### users/_message_button

```ruby
= link_to "メッセージ", chatroom_path(Chatroom.chatroom_for_users([current_user, user])), class: 'btn btn-raised btn-warning'
```

### users/show

```ruby

- if current_user && current_user&.id != @user.id
  .text-center
    = render 'message_button', user: @user
```

current_user(ログイン済)が存在しないとチャットルームは作成できない

### ja.ymlへの追加

```ruby
# config/locales/ja.yml

ja:
  activerecord:
    models:
      message: 'メッセージ'
    attributes:
      message:
        body: 'メッセージ'
```

## commit メッセージ機能追加

### chatrooms_controller

```ruby
class ChatroomsController < ApplicationController
  before_action :require_login, only: %i[index create show]

  def index
    @chatrooms = current_user.chatrooms.page(params[:page]).order(created: :desc)
  end

  def create
    users = User.where(id: params.dig(:chatroom, :user_ids)) + [current_user]
    @chatroom = Chatroom.chatroom_for_users(users)
    @messages = @chatroom.messages.order(created_at: :desc).limit(100).reverse
    @chatroom_user = current_user.chatroom_users.find_by(chatroom_id: @chatroom.id)
    redirect_to chatroom_path(@chatroom)
  end

  def show
    @chatroom = current_user.chatrooms.find(params[:id])
  end
end
```

`dig`...ネスとしたハッシュから安全に値を取り出すためのメソッド。もし値がなければ`nil`を返す。[参考](https://qiita.com/jnchito/items/02ba8aad634a6bd8a2f6#dig)

### models/chatroom

```ruby
class Chatroom < ApplicationRecord

  def self.chatroom_for_users(users)
    user_ids = users.map(&:id).sort
    name = user_ids.join(':').to_s
    unless chatroom = find_by(name: name)
      chatroom = new(name: name)
      chatroom.users = users
      chatroom.save
  end
end
```

### chatrooms/_chatroom.html

```ruby
.chatroom.mb-3.d-flex
  = image_tag chatroom.users.reject { |user| user == current_user }.sample.avatar.url, size: '40x40', class: 'rounded-circle mr-1'
  div
    p.font-weight-bold = raw(chatroom.users.reject { |user| user == current_user }.map{ |user| (link_to(user.name, user_path(user)))}.join(','))
    p.fond-weight-light = link_to chatroom.messages.last&.body&.truncate(30) || 'まだメッセージがありません', chatroom_path(chatroom)
hr
```

`truncate`メソッド...テキストを与えた引数文字分に省略するメソッド　

`raw`メソッド...ActionView::Helpers::OutputSafetyHelperクラスで用意されているメソッドであり、引数で渡した文字列をエスケープ処理せずに出力させるメソッド。`String.html_safe`メソッドとほぼ同じ挙動になる。

### chatrooms/index

```html
.conianer
  .text-center.mb-3
    = link_to "チャットルームを作る", "#", class: 'btn btn-primary btn-raised', data: { toggle: 'modal', target: '#create-chatroom-modal'}
  .row
    .col-md-6.col-12.offset-md-3.mb-3
      .card
        .card-body
          = render @chatrooms
    .col-md-6.col-12.offset-md-3.mb-3
      = paginate @chatrooms
#create-chatroom-modal.modal.fade tabindex="-1"
  = form_with model: Chatroom.new, local: true do |f|
    .modal-dialog
      .modal-content
        .modal-header
          h5.modal-title 誰とのチャットルームを作りますか？
          button.close aria-label="Close" data-dismiss="modal" type="button"
            span aria-hidden="true" x
        .modal-body
          ul.list-unstyled

            = f.collection_check_boxes :user_ids, current_user.following, :id, :name do |b|
              .mb-3
                = b.label do
                  = b.check_box class: 'mr-1'
                  = image_tag b.object.avatar.url, size: '40x40', class: 'rounded-circle mr-1'
                  = b.text
        .modal-footer.text-center
          = f.submit "作成", class: 'btn btn-raised btn-primary'
```

### shared/_header

```ruby
li.nav-item
  = link_to chatrooms_path, class: 'nav-link' do
    = icon 'far', 'comments', class: 'fa-lg'
```

### users/_message_button

```ruby
= form_with url: chatrooms_path, local: true do |f|
  = f.hidden_field 'chatroom[:user_ids][]', value: user.id
  = f.submit 'メッセージ', class: 'btn btn-raised btn-warning'
```

### routing

```ruby
resources :chatrooms, only: %i[index create show], shallow: true do
  resources :messages
end
```

## commit

デコレーターを使ってリファクタリング

### ChatroomDecorator

```ruby
class ChatroomDecorator < ApplicationDecorator
  degelate_all
  def message_text
    messages.last&.body&.truncate(30) || 'まだメッセージはありません'
  end
end
```

### chatrooms/_chatroom.html

```html
# before
p.font-weight-light = link_to chatroom.messages.last&.body&.truncate(30) || 'まだメッセージがありません', chatroom_path(chatroom)

p.font-weight-light = link_to chatroom.messages.last&.body&.truncate(30) || 'まだメッセージがありません', chatroom_path(chatroom)

# after

p.font-weight-light = link_to chatroom.decorate.message_text, chatroom_path(chatroom)
```

## commit

参加者一覧機能

### chatrooms/show

```html
      .card
        .card-header
          h2 チャットルーム
          = link_to "参加者一覧", "#", class: 'btn btn-raised btn-primary', data: { toogle: 'modal', target: '#chatroom-users'}
          ////
#chatroom-users.modal.fade tabindex="-1"
  .modal-dialog
    .modal-content
      .modal-header
        h5.modal-title 参加者
        button.close aria-label="Close" data-dismiss="mdoal" type="button"
          span aria-hidden="true" x
      .modal-body
        = render @chatroom.users
      .modal-footer.text-center
```

## commit

N+1問題解消

### chatrooms_controller

```ruby
def index
  @chatrooms = current_user.chatrooms.includes(:users, messages: :user).page(params[:page]).order(created_at: :desc)
end

def show
  @chatroom = current_user.chatrooms.includes(:users).find(params[:id])
end
```

## commit

viewで使うロジックをmodelに寄せる && デコり

### models/chatroom

```ruby
def users_excluding(user)
  users.reject { |u| u == user }
end
```

```ruby
# decorators/chatroom_decorator.rb

class ChatroomDecorator < ApplicationDecorator
  delegate_all
  def message_text
    messages.last&.body&.truncate(30) || 'まだメッセージがありません'
  end
end
```

### chatrooms/_chatroom

```html
.chatroom.mb-3.d-flex
  = image_tag chatroom.users_excluding(current_user).sample.avatar.url, size: '40x40', class: 'rounded-circle mr-1'
  div
    p.font-weight-bold = raw(chatroom.users_excluding(current_user).map { |user| (link_to(user.name, user_path(user)))}.join(','))
    p.font-weight-light = link_to chatroom.decorate.message_text, chatroom_path(chatroom)
```

## commit

不正なパラメータが送られてきた時の対処

### chatrooms_controller

```ruby
before_action :requrie_user_ids, only: %i[create]

private
def require_user_ids
  redirect_back(fallback_location: root_path, danger: 'パラメータが不正です') if params.dig(:chatroom, :user_ids).reject(&:blank?).blank?
end
```

### chatrooms/index

```html
.container
  .text-center.mb-3
    = link_to 'チャットルームを作る', "#", class: 'btn btn-primary btn-raised', data: { toggle: 'modal', target: '#create-chatroom-modal'}
  .row
    .col-md-6.col-12.offset-md-3.mb-3
      .card
        .card-body
          = render(@chatrooms) || 'まだありません'
    .col-md-6.col-12.offset-md-3.mb-3
      = paginate @chatrooms

#create-chatroom-modal.modal.fade tabindex="-1"
  = form_with model: Chatroom.new, local: true do |f|
    - if current_user.following.present?
      .modal-dialog
        .modal-content
          .modal-header
            h5.modal-title 誰とのチャットルームを作りますか？
            button.close aria-label="Close" data-dismiss="modal" type="button"
              span aria-hidden="true" x
          .modal-body
            ul.list-unstyled
              = f.collection_check_boxes :user_ids, current_user.following, :id, :name do |b|
                .mb-3
                  = b.label do
                    = b.check_box class: 'mr-1'
                    = image_tag b.object.avatar.url, size: '40x40', class: 'rounded-circle mr-1'
                    = b.text
          .modal_footer.text-center
            = f.submit "作成", class: 'btn btn-primary btn-raised'
    - else
      .modal-dialog
        .modal-content
          .modal-header
            h5.modal-title フォロワーがいません
            button.close aria-label="Close" data-dismiss="modal" type="button"
              span aria-hidden="true" x
          .modal-body
            h5.h5 おすすめユーザー
            = link_to 'ユーザー一覧', users_path, class: 'btn btn-raised btn-primary'
```

## commit

ActionCable

### ActionCable

[この記事がわかりやすかったです。](https://techtechmedia.com/action-cable-rails6/)

`ActionCable`とは...`WebSocket通信`を使用することで、サーバー側とクライアント側が常に監視している状態を作り出し、リアルタイム通信を可能にした技術。

### 導入

`$ be rails g channel chartroom`コマンドを実行し、chatroomチャネルを生成する

```javascript
# javascripts/application.js

#一番下に追記
//= require cable
//= require cable/chatroom.js
```

```js
# javascripts/cable/chatroom.js

$(function () {
  if ($("#chatroom").length > 0) {
    const chatroomId = $("#chatroom").data("chatroomId")
    const currentUserId = $("#chatroom").data("currentUserId")
    App.chatrooms = App.cable.subscriptions.create({ channel: "ChatroomChannel", chatroom_id: chatroomId }, {
      connected: function () {
        console.log("connected")
      },
      disconnected: function () {
        console.log("disconnected")
      },
      received: function (data) {
        switch (data.type) {
          case "create":
            $('#message-box').append(data.html);
            if ($(`#message-${data.message.id}`).data("senderId") != currentUserId) {
              // 自分の投稿じゃない場合は編集・削除ボタンを非表示にする
              $(`#message-${data.message.id}`).find('.crud-area').hide()
            }
            $('.input-message-body').val('');
            break;
          case "update":
            $(`#message-${data.message.id}`).replaceWith(data.html);
            if ($(`#message-${data.message.id}`).data("senderId") != currentUserId) {
              // 自分の投稿じゃない場合は編集・削除ボタンを非表示にする
              $(`#message-${data.message.id}`).find('.crud-area').hide()
            }
            $("#message-edit-modal").modal('hide');
            break;
          case "delete":
            $(`#message-${data.message.id}`).remove();
            break;
        }
      },
    });
  }
})
```

### application_cable/connection.rb

```ruby
# application_cable/connection.rb

module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if (verified_user = User.find_by(id: cookies.signed['user.id']))
        verified_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

### channel/chatroom_channel.rb

```ruby
# channel/chatroom_channel.rb

class ChatroomChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chatroom_#{params[:chatroom_id]}"
  end

  def unstubscribed
  end
end
```

### chatrooms_controller

```ruby
# chatrooms_controller

def index
  @chatrooms = current_user.chatrooms.includes(:user, message: :user).page(params[:page]).order(created_ad: :desc)
end
```

### messages_controller

```ruby
# messages_controller

def create
  @message = current_user.messages.create(message_params)
  ActionCable.server.broadcast(
    "chatroom_#{@message.chatroom_id}",
    { type: :create, html: (render_to_string partial: 'message', local: { message: @message }, layout: false), message: @message.as_json }
  )
  head :ok
end

def update
  @message = current_user.messages.find(params[:id])
  @message.update(message_update_params)
  ActionCable.server.broadcast(
    "chatroom_#{@message.chatroom_id}",
    type: :update, html: (render_to_string partial: 'message', locals: { message: @message }, layout: false), message: @message.as_json
  )
  head :ok
end

def destroy
  @message = current_user.messages.find(params[:id])
  @message.destroy!

  ActionCable.server.broadcast(
    "chatroom_#{@message.chatroom_id}",
    { type: :delete, html: (render_to_string partial: 'message', local: { message: @message }, layout: false), message: @message.as_json }
  )
  head :ok
end
```

### user_sessions_controller

```ruby
# user_sessions_controller

def create
  @user = login(params[:email], params[:password])
  if @user
    # ActionCableでユーザーを特定するために必要
    cookies.signed["user.id"] = current_user.id
    redirect_back_or_to posts_path, success: 'ログインしました'
  else
    flash.now[:danger] = "ログインに失敗しました"
    render :new
  end
end

def destroy
  logout
  cookies.delete("user.id")
  redirect_to login_path, success: "ログアウトしました"
end
```

### chatrooms/show

```html
///
      .card#chatroom darta-chatroom-id="#{@chatroom.id}" data-current-user-id="#{current_user.id}"
        .card-header
          h2 ///

          ul.list-unstyled#message-box
            = render @chatroom.messages # ←変更ポイント
```

### messages/_message

```html
li.medis.mb-3.border-bottom.p-2 id="message-#{message.id}" data-sender-id="#{message.user.id}"
////
    div.text-right.crud-area
      - if current_user&.own?(message)
        = link_to message_path(message), class: 'mr-3 delete-button', method: :delete, data: { confirm: "本当に削除しますか？"}, remote: true do
          = icon 'far', 'trush-alt', class: 'fa-sm'
```

## commit

ActionCable導入により、不要になったファイルの削除

- messages/create.js
- messages/destroy.js
- messages/update.js

## commit

blank対策

### MessagesController

```ruby
def create
  @message = current_user.messages.build(message_params)
  if @message.save
    ActionCable.server.broadcast(
      "chatroom_#{@message.chatroom_id}",
      { type: :create, html: (render_to_string partial: 'message', locals: {message: @message}, layout: false), message: @message.as_json }
    )
    head :ok
  else
    head :bad_request
  end
end

def update
  @message = current_user.messages.find(params[:id])
  if @message.update(message_update_params)
    ActionCable.server.broadcast(
      "chatroom_#{@message.chatroom_id}",
      { type: :update, html: (render_to_string partial: 'message', locals: { message: @message }, layout: false), message: @message.as_json　}
    )
    head :ok
  else
    head :bad_request
  end
end
```

`render_to_string`...renderした結果を文字列として取得する場合に使用するメソッド。

### messages/_form

```html
  = form_with model: [chatroom, message], remote: true do |f|
    = render 'shared/error_messages', object: message
    = f.label :body
    = f.text_field :body, class: 'form_control mb-3 input-message-body', required: true
    - if message.new_record?
      = f.submit "投稿", class: 'btn btn-raised btn-primary'
    - else
      = f.submit "更新", class: 'btn btn-primary btn-raised'
```

`required: true`...空の送信を防ぐことができるオプション
