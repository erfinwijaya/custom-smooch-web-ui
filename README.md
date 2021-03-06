# Creating your own UI for the Smooch Web Messenger

**This guide shows you how to create your own UI for the Smooch Web Messenger.**

The Smooch Web messenger comes with a rich prebuilt user interface with the option to configure the interface using the [Smooch dashboard](https://docs.smooch.io/guide/web-messenger/#styling-the-conversation-interface) or the [REST API](https://docs.smooch.io/rest/#web-messenger-integration).

If needed you can completely replace Smooch's default Web Messenger UI with your own interface.

Although you can replace the default UI, note that this means rewriting support for all the [Smooch message types](https://docs.smooch.io/guide/structured-messages/), that you want to support in your custom UI.

## Overview

The Web messenger can be initialized without displaying it's default UI. You can then make use of the SDK's messaging APIs to send messages, and it's callback event interface to receive messages.

This guide covers:
- [initializing the SDK](#initialize-the-sdk)
- [fetching the initial conversation and user data](#fetch-the-initial-data)
- [sending appUser messages](#send-messages)
- [receiving new messages](#receive-messages)

It will help to have the [SDK documentation](https://github.com/smooch/smooch-web) on hand while following this guide.

## Starter code

Note that the complete code is available at the end of this guide.

We start with a basic web site with the Smooch SDK loaded:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>
<body>
  <script>
  var appId = '<your-app-id>';

  !function(e,n,t,r){
    function o(){try{var e;if((e="string"==typeof this.response?JSON.parse(this.response):this.response).url){var t=n.getElementsByTagName("script")[0],r=n.createElement("script");r.async=!0,r.src=e.url,t.parentNode.insertBefore(r,t)}}catch(e){}}var s,p,a,i=[],c=[];e[t]={init:function(){s=arguments;var e={then:function(n){return c.push({type:"t",next:n}),e},catch:function(n){return c.push({type:"c",next:n}),e}};return e},on:function(){i.push(arguments)},render:function(){p=arguments},destroy:function(){a=arguments}},e.__onWebMessengerHostReady__=function(n){if(delete e.__onWebMessengerHostReady__,e[t]=n,s)for(var r=n.init.apply(n,s),o=0;o<c.length;o++){var u=c[o];r="t"===u.type?r.then(u.next):r.catch(u.next)}p&&n.render.apply(n,p),a&&n.destroy.apply(n,a);for(o=0;o<i.length;o++)n.on.apply(n,i[o])};var u=new XMLHttpRequest;u.addEventListener("load",o),u.open("GET","https://"+r+".webloader.smooch.io/",!0),u.responseType="json",u.send()
  }(window,document,"Smooch", appId);
  </script>
</body>
</html>
```

## Initialize the SDK

To initialize the SDK and access it's core messaging APIs, without displaying the UI you must enable embedded mode and provide an invisible container element on render (_note that in future versions the SDK will ship with a non-rendering mode, but for now it must be rendered in an invisible element_).

- See the documentation for [embedded mode](https://github.com/smooch/smooch-web#embedded-mode)

First, let's define the html element where the default UI will be hidden:

```html
<div id="no-display" style="display:none;"></div>
```

Next, we'll initialize the Smooch SDK and render the Web Messenger in our invisible element:

```javascript
Smooch.init({ appId: appId, embedded: true });
Smooch.render(document.getElementById('no-display'));
```

## Fetch the initial data

The SDK exposes methods to fetch the appUser object and conversation state. We use these methods to determine the initial state of our UI.

- See the documentation for [`Smooch.getConversation`](https://github.com/smooch/smooch-web#getconversation). This method gives you access to things like the unread messages count, conversation history, and more.

- See the documentation for [`Smooch.getUser`](https://github.com/smooch/smooch-web#getuser). This method gives you access to the complete user record that Smooch stores.

OK, let's write HTML to 1) display the conversation, and 2) display our user ID:

```html
<p>User ID: <span id="user-id"></span></p>
<ul id="conversation"></ul>
```

Let's define a function in our JavaScript that we can call to display a message in our custom UI, and another function to display the user ID:

```javascript
function displayMessage(message) {
  var conversationElement = document.getElementById('conversation');
  var messageElement = document.createElement('li');
  messageElement.innerText = message.name + ' says "' + message.text + '"';
  conversationElement.appendChild(messageElement);
}

function displayUserId(user) {
  document.getElementById('user-id').innerText = user ? user._id : 'None';
}
```

Now, we can display the initial conversation state and some user info after we init the SDK. Replace the `Smooch.init` call with this:

```javascript
Smooch.init({ appId: appId, embedded: true }).then(function() {
  // displays initial messages
  var conversation = Smooch.getConversation();
  conversation.messages.forEach(displayMessage);

  // display user ID if user currently exists
  var user = Smooch.getUser();
  displayUserId(user);
});
```

## Send messages

Smooch exposes an SDK method to send messages as a user.

- See the documentation for [`Smooch.sendMessage`](https://github.com/smooch/smooch-web#sendmessage). This method allows you to send plain text or structured messages.

We can create an input element in our HTML to accept plain text user messages:

```html
<input type="text" id="text-input" placeholder="text">
```

Now, we'll call the `Smooch.sendMessage` function each time the enter key is pressed while our input element is active. Add the following after you Smooch.init call:

```javascript

Smooch.init({ appId: appId, embedded: true }).then(function () {
  var inputElement = document.getElementById('text-input');
  inputElement.onkeyup = function(e) {
    if (e.key === 'Enter') {
      Smooch.sendMessage(inputElement.value).then(function() {
          inputElement.value = '';

          // update the user ID
          displayUserId(Smooch.getUser());
      });
    }
  }
  
  //...
});
```

Note that this code is being run after `Smooch.init` has finished. To confirm that messages are sent, open the Logs for the app are using in the [Smooch dashboard](https://app.smooch.io/).

## Receive messages

In order to update the UI with the content of new messages, we can use Smooch's event interface `on`. We're interested in handling two events `message:received` for incoming messages and `message:sent` for outbound messages.

- See the documentation for [events](https://github.com/smooch/smooch-web#events)

We're going to call our `displayMessage` function when message events occur. Somewhere in your JavaScript, below where the SDK is loaded, add:

```javascript
Smooch.init({ appId: appId, embedded: true }).then(() => {
    Smooch.on('message:sent', displayMessage);
    Smooch.on('message:received', displayMessage);

    // ...
});
```

Note again that we're making sure `Smooch.init` completes successfully before creating these event subscriptions.

## Wrap up

You've created your own UI for the Smooch Web SDK. It should look something like this:

<img width="313" alt="custom_ui" src="https://user-images.githubusercontent.com/2235885/34859709-f9ac4e32-f725-11e7-8297-de84d7837bd1.png">

Now you might want to consider how you'll represent more complex messages and activites, such as:
- [structured messages](https://docs.smooch.io/guide/structured-messages/)
- [conversation extensions](https://docs.smooch.io/guide/conversation-extensions/)
- [conversation activity](https://docs.smooch.io/rest/#conversation-activity)

You can find the complete code for this guide here: [index.html](index.html).
