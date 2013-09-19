# Getting Started with Ruby and WebSockets

In this post, we'll be covering an introduduction to WebSockets, and
implementing a basic WebSockets chat app using [EM-WebSocket][].

WebSockets is one of the cooler new features in the HTML5 spec, and allows
clients and servers to communicate without using AJAX requests, or HTTP
long-polling([Comet][]).

## What are WebSockets?

WebSockets are, according to the [specification][]:

>  an API that enables Web pages to use the WebSocket protocol (defined by the
>  IETF) for two-way communication with a remote host.

Essentially, WebSockets can replace existing HTTP long-polling solutions.
WebSockets allow for a single, long-held TCP connection to be established
between the client and server. This allows for full-duplex, bi-directional
messaging between both sides with very little overhead (and, as a result, very
little latency).

The WebSocket specification defines two new URI scemes, **ws:** and **wss:**,
for unencrypted and encrypted WebSocket connections respectively.

## Browser Support

The current WebSockets spec is supported by the following browsers:

- Chrome 16
- Internet Explorer 10
- Opera 12
- Safari 6

## Getting Started

To get started on building our [Em-WebSocket][] chat app, let's build a very
basic Sinatra app running within EventMachine. We're going to need to use
[thin][], since it supports EventMachine:

```ruby
# Gemfile
source 'https://rubygems.org'

gem 'thin'
gem 'sinatra'
gem 'em-websocket'
```

```ruby
# app.rb
require 'thin'
require 'sinatra/base'
require 'em-websocket'

EventMachine.run do
  class App < Sinatra::Base
    get '/' do
      erb :index
    end
  end

  # our WebSockets server logic will go here

  App.run! :port => 3000
end
```

```html
# views/index.erb
<!doctype html>
<html>
  <head>
  </head>
  <body>
    <div class="container">
      <h1>WebSockets Chat App</h1>
      <div id="chat-log"></div>
      <div id="form">
        <input type="text" id="message">
        <button id="disconnect">Disconnect</button>
      </div>
    </div>

    <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
    <script>
      // where our WebSockets logic will go later
    </script>
  </body>
</html>
```

Run it with:

    bundle exec ruby app.rb

And now we have a good start for our chat application. Next, we'll get the
server side of things set up.

## WebSockets Server

First, let's get the server set up to run within EventMachine, and handle
a WebSocket connection. This is made really simple by `em-websocket`, and we
don't even need to handle the headers for the WebSocket handshake:

```ruby
EventMachine.run do
  # ... [previous Sinatra stuff]

  @clients = []

  EM::WebSocket.start(:host => '0.0.0.0', :port => '3001') do |ws|
    ws.onopen do |handshake|
      @clients << ws
      ws.send "Connected to #{handshake.path}."
    end

    ws.onclose do
      ws.send "Closed."
      @clients.delete ws
    end

    ws.onmessage do |msg|
      puts "Received Message: #{msg}"
      @clients.each do |socket|
        socket.send msg
      end
    end
  end

  # ... [run Sinatra server]
end
```

This will take care of the WebSocket handshake with clients, and take care of
the chat backend. Now, let's set up the client!

## WebSockets Client

This will set up our clients to be able to interact with the WebSocket server we
just set up. The server will take any messages the client sends, and
re-broadcast it to all clients.

First, let's write basic functions to display messages on the page:

```javascript
function addMessage(msg) {
  $("#chat-log").append("<p>" + msg + "</p>");
}
```

Now, let's set up a connection with our WebSocket server, and set it up to
connect as soon as the browser's ready:

```javascript
var socket, host;
host = "ws://localhost:3001";

function connect() {
  try {
    socket = new WebSocket(host);

    addMessage("Socket State: " + socket.readyState);

    socket.onopen = function() {
      addMessage("Socket Status: " + socket.readyState + " (open)");
    }

    socket.onclose = function() {
      addMessage("Socket Status: " + socket.readyState + " (closed)");
    }

    socket.onmessage = function(msg) {
      addMessage("Received: " + msg.data);
    }
  } catch(exception) {
    addMessage("Error: " + exception);
  }
}

$(function() {
  connect();
});
```

Now we can set up the logic to send messages to the server whenever the user
presses the return key (keycode 13). This will send the message to the server,
and let the user know that the message has been sent:

```javascript
function send() {
  var text = $("#message").val();
  if (text == '') {
    addMessage("Please Enter a Message");
    return;
  }

  try {
    socket.send(text);
    addMessage("Sent: " + text)
  } catch(exception) {
    addMessage("Failed To Send")
  }

  $("#message").val('');
}

$('#message').keypress(function(event) {
  if (event.keyCode == '13') { send(); }
});

```

And, last but not least, let's let our users disconnect from the server if they
want to:

```javascript
$("#disconnect").click(function() {
  socket.close()
});
```

And this wraps up our client side. It will now open a WebSocket connection to
our server, send/receive messages, and disconnect if the user wants to.

## Conclusion

Now we have a working chat service. To start it, we run the same command as
before:

    bundle exec ruby app.rb

Now you can browse to `http://localhost:3000/`, and see the chat system in
action. Try opening it in multiple browsers to test it out! If you'd like to
check out the full source code, you can find it on [GitHub][source].

For some further reading on WebSockets, you may wish to consult [the
spec][specification]. Additionally, there's some good info on the [MDN][], but
that site isn't 100% complete yet.

[Em-WebSocket]: https://github.com/igrigorik/em-websocket
[Comet]: http://en.wikipedia.org/wiki/Comet_(programming)
[specification]: http://dev.w3.org/html5/websockets/
[thin]: http://code.macournoyer.com/thin/
[MDN]: https://developer.mozilla.org/en-US/docs/WebSockets
[source]: https://github.com/stewart/eventmachine-websockets-demo
