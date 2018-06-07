# hang-outs mock app

Rails app generated with [lewagon/rails-templates](https://github.com/lewagon/rails-templates), created by the [Le Wagon coding bootcamp](https://www.lewagon.com) team.

## Table of Contents

* [Getting started](#getting-started)
* [Following users](#following-users)
* [Mutual friendship](#mutual-friendship)
* [Live chat](#live-chat)


## Getting started

After cloning the app, create the db, run the migrations and seed.

```bash
$ rails db:create db:migrate db:seed
```

## Following users

We want to set up a system where users can follow other users, i.e., any user can be a follower and a followee.

We want to be able to call `@user.followers` to see the list of users who are following the `@user` and `@user.following` to see the list of users the `@user` is following.

Assuming we already have a User model, let's create the a Join Table conneting followers and followees.

```bash
$ rails g model Follow
````
Add this to the migration file

```ruby
class CreateFollows < ActiveRecord::Migration[5.2]
  def change
    create_table 'follows' do |t|
      t.integer 'following_id', null: false
      t.integer 'follower_id', null: false

      t.timestamps null: false
    end

    add_index :follows, :following_id
    add_index :follows, :follower_id
    add_index :follows, [:following_id, :follower_id], unique: true
  end
end
```
Run `rails db:migrate` and update your models with the following:

```ruby
class User < ApplicationRecord
  has_many :follower_relationships, foreign_key: :following_id, class_name: 'Follow'
  has_many :followers, through: :follower_relationships, source: :follower

  has_many :following_relationships, foreign_key: :follower_id, class_name: 'Follow'
  has_many :following, through: :following_relationships, source: :following
end
```

And then add some custom methods to the User model to have a simpler way of following and unfollowing users and to check if a user is following another:

```ruby
  def follow(user_id)
    following_relationships.create(following_id: user_id)
  end

  def unfollow(user_id)
    following_relationships.find_by(following_id: user_id).destroy
  end

  def is_following?(user_id)
    relationship = Follow.find_by(follower_id: id, following_id: user_id)
    return true if relationship
  end
```

Let's add the routes for the actions in our UsersController:

```ruby
resources :users, only: [:index] do
    member do
      post :follow
      post :unfollow
    end
 end
 ```
 And the controller should look like this:

 ```ruby
 class UsersController < ApplicationController
  skip_before_action :authenticate_user!, only: :index
  before_action :set_user, only: [:follow, :unfollow]

  def index
    @users = User.all.where.not(id: current_user.id)
  end

  def follow
    if current_user.follow(@user.id)
      respond_to do |format|
        format.html { redirect_to root_path }
        format.js
      end
    end
  end

  def unfollow
    if current_user.unfollow(@user.id)
      respond_to do |format|
        format.html { redirect_to root_path }
        format.js { render action: :follow }
      end
    end
  end

  private

  def set_user
    @user = User.find(params[:id])
  end
end
```

Finally, let's setup our views to render the list of users with the follow/unfollow button which is updated though AJAX:

```erb
<!-- users/_user.html.erb -->
<li data-user=<%= user.id %>>
  <%= user.full_name %>
  <% if current_user.is_following?(user.id) %>
    <%= link_to "Unfollow", unfollow_user_path(user.id), method: :post, remote: true %>
  <% else %>
    <%= link_to "Follow", follow_user_path(user.id), method: :post, remote: true %>
  <% end %>
</li>
```
And lastly set up the JS response for the follow/unfollow actions:

```javascript
// users/follow.js.erb
function renderUser(userHTML) {
  const user = document.querySelector("[data-user='<%= @user.id %>']");
  user.outerHTML = userHTML;
}

renderUser('<%= j render @user %>');
```

## Mutual friendship

We want to create a system that allows a user to send a friend request to another user. The latter user can either accept or reject the request. If they accept, a mutual friendship is created.

First, we’ll create FriendRequest model.

```bash
$ rails g model FriendRequest user:references friend:references
```
And update the migration file accordingly:
```ruby
class CreateFriendRequests < ActiveRecord::Migration[5.2]
  def change
    create_table :friend_requests do |t|
      t.references :user, foreign_key: true
      t.integer :friend_id, null: false

      t.timestamps
    end

    add_index :friend_requests, :friend_id
    add_index :friend_requests, [:user_id, :friend_id], unique: true
  end
end
```
Then we add the `has_many :through` association between users:
```ruby
# app/models/user.rb:
class User < ActiveRecord::Base
  has_many :friend_requests, dependent: :destroy
  has_many :pending_friends, through: :friend_requests, source: :friend
end
```
Because the FriendRequest model has self-referential association, we have to specify the class name:
```ruby
class FriendRequest < ActiveRecord::Base
  belongs_to :user
  belongs_to :friend, class_name: 'User'
end
```
Now let's setup the FriendRequestsController:
```ruby
# app/controllers/friend_requests_controller.rb
class FriendRequestsController < ApplicationController
  before_action :set_friend_request, except: [:index, :create]
  before_action :set_user, except: [:index]

  def index
    @incoming = FriendRequest.where(friend: current_user)
    @outgoing = current_user.friend_requests
  end

  def create
    friend = User.find(params[:friend_id])
    @friend_request = current_user.friend_requests.new(friend: friend)

    if @friend_request.save
      respond_to do |format|
        format.html { redirect_to root_path }
        format.js
      end
    end
  end

  def destroy
    @friend_request.destroy
    respond_to do |format|
      format.html { redirect_to root_path }
      format.js { render action: :create }
    end
  end

  private

  def set_friend_request
    @friend_request = FriendRequest.find(params[:id])
  end

  def set_user
    @user = User.find(params[:user_id] || params[:friend_id])
  end
end
```
Add the following to `routes.rb`
```ruby
resources :friend_requests, only: [:index, :create, :destroy]
```
And setup our Users index to Send/Cancel friend requests with AJAX requests:
```erb
<!-- users/_user.html.erb -->
<li data-user=<%= user.id %>>
  <%= user.full_name %>
  <% if request = current_user.find_friend_request(user.id) %>
    <%= link_to "Cancel Friend Request", friend_request_path(request, user_id: user.id), method: :delete, remote: true %>
  <% else %>
    <%= link_to "Send Friend Request", friend_requests_path(friend_id: user.id), method: :post, remote: true %>
  <% end %>
</li>
```
And also setup our JS response:
```javascript
// friend_requests/create.js.erb
function renderUser(userHTML) {
  const user = document.querySelector("[data-user='<%= @user.id %>']");
  user.outerHTML = userHTML;
}

renderUser('<%= j render @user %>');
```
Now let’s move on to another join model: Friendship. We’ll be using this association to get and destroy friends.
```bash
$ rails g model Friendship user:references friend:references
```
Update the migration
```ruby
def change
    create_table :friendships do |t|
      t.references :user, index: true, foreign_key: true
      t.references :friend, index: true

      t.timestamps
    end

    add_foreign_key :friendships, :users, column: :friend_id
    add_index :friendships, [:user_id, :friend_id], unique: true
  end
```
Update the User model:
```ruby
class User < ApplicationRecord
  # [...]
  has_many :friendships, dependent: :destroy
  has_many :friends, through: :friendships
 end
 ```
 And the Friendships model should look like this:
 ```ruby
 class Friendship < ApplicationRecord
  after_create :create_inverse_relationship, unless: :has_inverse_relationship?
  after_destroy :destroy_inverse_relationship

  belongs_to :user
  belongs_to :friend, class_name: 'User'

  private

  def create_inverse_relationship
    friend.friendships.create(friend: user)
  end

  def destroy_inverse_relationship
    friendship = friend.friendships.find_by(friend: user)
    friendship.destroy if friendship
  end

  def has_inverse_relationship?
    self.class.exists?(inverse_friendship_options)
  end

  def inverse_friendship_options
    { friend_id: user_id, user_id: friend_id }
  end
end
```
You can find a more comprehensive implementation [here](https://dankim.io/mutual-friendship-rails/).

## Live chat

Let's build a live chat between users with Rails ActionCable. We'll neet a Chat and Message models.

```bash
$ rails g model Chat sender_id:integer recipient_id:integer
$ rails g model Message body:text user:references chat:references
$ rails db:migrate
```

Update your models:
```ruby
class Chat < ApplicationRecord
  belongs_to :sender, foreign_key: :sender_id, class_name: 'User'
  belongs_to :recipient, foreign_key: :recipient_id, class_name: 'User'

  has_many :messages, dependent: :destroy

  validates_uniqueness_of :sender_id, scope: :recipient_id

  scope :involving, -> (user) {
    where("chats.sender_id = ? OR chats.recipient_id = ?", user.id, user.id)
  }

  scope :between, -> (user_A, user_B) {
    where("(chats.sender_id = ? AND chats.recipient_id = ?) OR (chats.sender_id = ? AND chats.recipient_id = ?)", user_A, user_B, user_B, user_A)
  }
end
```
The scopes will allow us to call `Chat.involving(current_user)` to find the chats where the current_user is participating and `Chat.between(current_user, another_user)` to find the chats between these two users.

NOTE that due to our `validates_uniqueness_of` there will only be one chat between any two users, but the `between` scope will still return an ActiveRecordCollection [array like object].

```ruby
class Message < ApplicationRecord
  belongs_to :user
  belongs_to :chat

  validates_presence_of :body, :chat_id, :user_id

  def message_time
    created_at.strftime("%d %b, %Y")
  end
end
````
We add a helper method `message_time` to format how we will display the time in our views.

Next create the controllers for Chats and Messages:
```bash
$ rails g controller Chats
$ rails g controller Messages
```

```ruby
class ChatsController < ApplicationController
  before_action :authenticate_user!

  def index
    @chats = Chat.involving(current_user)
  end

  def show
    @chat = Chat.find(params[:id])
    @other_user = current_user == @chat.sender ? @chat.recipient : @chat.sender
    @messages = @chat.messages.order(created_at: :asc).last(20)
    @message = Message.new
  end

  def create
    @chat = Chat.between(params[:sender_id], params[:recipient_id]).first_or_create!(chat_params)

    redirect_to @chat
  end

  private

  def chat_params
    params.permit(:sender_id, :recipient_id)
  end
end
````
The trick in the `chats#create` is the ActiveRecord method `find_or_create`, which returns the first record it finds or creates a new one if no record exists.

```ruby
class MessagesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_chat

  def create
    @message = @chat.messages.new(message_params)
    @messages = @chat.messages.order(created_at: :desc)
    if @message.save
      redirect_to chat_messages_path(@chat)
    end
  end

  private

  def set_chat
    @chat = Chat.find(params[:chat_id])
  end

  def message_params
    params.require(:message).permit(:body, :user_id)
  end
end
```
NOTE that I have deliberately left out any authorization scheme, which means that our private chats are available for any user o eavesdrop. You should set up an authorization system to protect these actions, for example with `Pundit` policies - a user can only access chats where they are either a sender or a recipient.

Don't forget to nest the routes:
```ruby
resources :chats, only: [:index, :create] do
   resources :messages, only: [:index, :create]
end
```
#### Action Cable
Now let's configure ActionCable, which integrates integrates WebSockets in Rails and allows you to write real time features with server-side logic in Ruby and client-side in JavaScript.

Mount the ActionCable server in your `routes.rb` file:
```ruby
# config/routes.rb
mount ActionCable.server => '/cable'
```
Edit your `development.rb` environment file and add the following:
```ruby
# config/environments/development.rb

Rails.application.configure do
# ...
config.action_cable.url = "ws://localhost:3000/cable"
end
```

First let's generate a channel for our chat which will create the file `app/channels/chat_channel.rb`

```bash
rails g channel chat
```
We need to make sure that Action Cable only broadcasts to authenticated users. Inside the `app/channels/application_cable` folder that was generated when you created your application, you can find a `connection.rb` file that is responsible for WebSocket authentication. Make the following changes:
```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
      logger.add_tags "ActionCable", "User #{current_user.id}"
    end

    protected

    def find_verified_user
      if current_user = env['warden'].user
        current_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```
Now it’s time to handle our `chat_channel.rb`:
```ruby
class ChatChannel < ApplicationCable::Channel
  def subscribed
    Chat.involving(current_user).each do |chat|
      stream_from "chat_#{chat.id}"
    end
  end

  # Called when message-form contents are received by the server
  def send_message(payload)
    message = Message.new(user: current_user, chat_id: payload["id"], body: payload["message"])

    ActionCable.server.broadcast "chat_#{payload['id']}", message: render(message) if message.save
  end

  def unsubscribed
    stop_all_streams
  end

  private

  def render(message)
    ApplicationController.render(
          partial: 'messages/message',
          locals: {message: message}
      )
  end
end
```
#### JavaScript

Now it's time to plug in everything together with JavaScript and render our new messages in real time.

First let's add the actioncable js so we can use it inside our packs components:
```bash
$ yarn add actioncable
```
And create the necessary files and folders:
```bash
$ mkdir app/javascrip/client
$ touch app/javascrip/client/cable.js
$ touch app/javascript/client/chat.js
$ mkdir app/javascrip/components
$ touch app/javascrip/components/message-form.js
$ touch app/javascrip/components/messages.js
```
And modify accordingly:
```javascript
// app/javascrip/client/cable.js
import cable from "actioncable";

let consumer;

function createChannel(...args) {
  if (!consumer) {
    consumer = cable.createConsumer();
  }

  return consumer.subscriptions.create(...args);
}

export default createChannel;
```
```javascript
// app/javascript/client/chat.js
import createChannel from 'client/cable';

let callback; // declaring a variable that will hold a function later

const chat = createChannel('ChatChannel', {
  received({ message }) {
    if (callback) callback.call(null, message);
  }
});

// Sending a message: "perform" method calls a respective Ruby method
// defined in chat_channel.rb. That's your bridge between JS and Ruby!
function sendMessage(message, chatId) {
  chat.perform("send_message", { message: message, id: chatId });
}

// Getting a message: this callback will be invoked once we receive
// something over ChatChannel
function setCallback(fn) {
  callback = fn;
}

export { sendMessage, setCallback };
```
Finally, let's add our js to send messages that will be picked up by our ChatChannel in Rail's ActionCable which will be responsible for creating a new Message instance and distributing it to the respective channel's subscribers:
```javascript
// app/javascript/components/message-form.js

// we need to import sendMessage from our client/chat.js
import { sendMessage } from "../client/chat";

function submitMessage(inputMessage, inputChatId) {
  // Invokes sendMessage, that, in turn, invokes Ruby send_message method
  // that will create a Message instance with ActiveRecord
  sendMessage(inputMessage.value, inputChatId.value);

  // eslint-disable-next-line
  inputMessage.value = "";
  inputMessage.focus();
}

const form = document.getElementById('new_message');

if (form) {
  const inputMessage = form.querySelector('.message-form--input');
  const inputChatId = form.querySelector('#message_chat_id');
  const submit = form.querySelector('.message-form--submit');

  // You can send a message with cmd/ctrl+enter
  inputMessage.addEventListener('keydown', event => {
    if (event.keyCode === 13 && event.metaKey) {
      event.preventDefault();
      submitMessage(inputMessage, inputChatId);
    }
  });

  // Or by cicking a button
  submit.addEventListener('click', event => {
    event.preventDefault();
    submitMessage(inputMessage, inputChatId);
  });
}
```
And finally the callback function that receives ActionaCable's response from the backend and is responsbile for appending it to the page and scrolling our chat window:
```javascript
// app/javascript/components/messages.js

import { setCallback } from 'client/chat';

function scrollToBottom(element) {
  // eslint-disable-next-line
  console.log(element.scrollTop)
  console.log(element.scrollHeight)
  element.scrollTop = element.scrollHeight;
}

const messages = document.querySelector('.messages');

if (messages) {
  const content = messages.querySelector(".messages--content");

  scrollToBottom(content);

  // Telling `chat.js` to call this piece of code whenever a new message is received
  // over ActionCable
  setCallback(message => {
    content.insertAdjacentHTML("beforeend", message);

    scrollToBottom(content);
  });
}
```
And don't forget two import these two files in your application's packs entry point:
```javascript
// app/javascript/packs/application.js

import '../components/message-form.js';
import '../components/messages.js';
```

NOTE that the Javascript part may have to be modified and adapted to the way you organize and name your HTML and CSS components.

For a more detailed explanation please visit the links below. The live chat was inspired by and adapted from:
https://evilmartians.com/chronicles/evil-front-part-3
https://code4startup.com/
https://gorails.com/
