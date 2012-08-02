---
layout: post
title: Backbone.js on WebSockets
author: myrlund
categories: HTML5
---

_[Originally published on Comoyo's developer blog.](http://comoyo.github.com/blog/2012/08/01/backbone-on-websockets/)_

When we arrived at Comoyo for our summer internship, we were given a pretty exciting task: build a messaging web app, using any technologies at hand, but _without using a backend_. Well, not completely true – we already had a UNIX-socket backend that we were to hook into, so the problem soon became building a self-contained HTML app able to talk to it _without any backend of its own_.

The natural platform choice became WebSocket-heavy HTML5.

After [implementing a WebSocket handler in our Netty backend](http://comoyo.github.com/blog/2012/07/30/integrating-websockets-in-netty/) we started thinking about our client-side technology stack.
Apart from using WebSockets for server communication, we decided to use localStorage for client-side storage and cache, and Backbone.js for the actual front-end.
As Backbone.js depends on [Underscore.js](http://underscorejs.org/), we'll just as well use it throughout the application.
To speed up development, we decided to go for [CoffeeScript](http://coffeescript.org/) instead of pure JavaScript.

Other people have integrated Backbone with web sockets before us.
However, they all seem to be using Node.js with Socket.io, so we thought we'd share our slightly different approach.

_**Heads up!** This article focuses on integrating Backbone with an asynchronous web socket protocol, and assumes some level of comprehension of how Backbone works. Please check out [the Backbone documentation](http://backbonejs.org/) and [its examples](http://backbonejs.org/#examples) for a quick introduction before checking back._

A pretty simple layered architecture emerged.

![Our layered architecture](/images/posts/backbone-on-websockets-layers.png "Our layered architecture")

Before we take a stroll through the layers, let's talk about the glue: _the event dispatch_.

## The event dispatch

To let the components talk to each other in a nice and loose fashion, we needed some sort of event dispatch system. Luckily, it turns out Backbone supplies it for absolutely free through its `Backbone.Events` class:

{% highlight coffeescript %}
window.dispatch = _.clone(Backbone.Events)
{% endhighlight %}

Using the event dispatch is easy: you subscribe to and trigger events with `dispatch.on(eventName, callback)` and `dispatch.trigger(eventName, payload)` respectively.

# The communication layer

With event dispatch in place, we naturally started with the bottom layer: the communication layer.

A very simple `Communicator` class wound up looking like this:

{% highlight coffeescript linenos %}
class Communicator

  # The messages used in the protocol look like:
  #   {"com.comoyo.CommandName": {"key": value}}
  # 
  # ...so we keep the namespace handy.
  commandNamespace: 'com.comoyo.'
  
  # Set up the web socket and listen for incoming messages
  constructor: (@server) ->
    @webSocket = new WebSocket(@server)
    @webSocket.onmessage = @handleMessage
    @webSocket.onopen = -> dispatch.trigger('WebSocketOpen')

  handleMessage: (message) =>
  
    # The message string is the message object's data property
    strMessage = message.data
  
    # Now, parse that string
    jsonMessage = JSON.parse(strMessage)
  
    # Grab the command name, ie. the first root key (using Underscore.js)
    fullCommandName = _.keys(jsonData)[0]
    commandName = _.last(fullCommandName.split('.'))
  
    # Trigger the command in event dispatch, passing the payload
    dispatch.trigger(commandName, jsonMessage[fullCommandName])

  sendMessage: (commandName, messageData) =>
  
    # Full command name with namespace
    fullCommandName = @commandNamespace + commandName
  
    # Build the message
    jsonMessage = {}
    jsonMessage[fullCommandName] = messageData
  
    # Serialize the object into a JSON string...
    strMessage = JSON.stringify(jsonMessage)
  
    # ...and send it!
    @webSocket.send(strMessage)
{% endhighlight %}

The `Communicator` simply encapsulates the web sockets, wrapping them in a couple of simple messaging methods.
This should make it easy as a breeze to swap our beloved WebSockets with [SSE](http://www.w3.org/TR/eventsource/) or [other long polling techniques](http://en.wikipedia.org/wiki/Push_technology), in case we want to provide working fallbacks in old browsers, for instance.

## The protocol controllers

With our communicator in place, we can set up controllers handling the various parts of communication with the backend protocol.
The controllers communicate through the `Communicator`: they listen to incoming messages through the event dispatch and send messages with `Communicator.sendMessage` calls.

A large part of the protocol we implemented relies upon sequences of messages. The typical pattern consists of the following steps:

1. We subscribe to a resource
2. We're notified of a changed resource
3. We request the changed resource
4. We receive the resource

Let's take a look at our login controller for an example of how to implement this sequential protocol.

A simplified version of out login process should clarify the pattern. It consists of only two simple sequential steps:

1. Client registration
2. Account login

To tie these together, we could build a controller looking like this:

{% highlight coffeescript linenos %}
class LoginController
  
  # Let the initializer listen for relevant events,
  # delegating them to its respective event handlers.
  initialize: ->
    dispatch.on('WebSocketOpen',
                          @sendClientRegistrationCommand, this)
    dispatch.on('ClientRegistrationResponse', 
                          @handleClientRegistrationResponse, this)
    dispatch.on('AccountLoginResponse', 
                          @handleAccountLoginResponse, this)
  
  # The client registration command is a simple message 
  # with some metadata to initiate the connection
  sendClientRegistrationCommand: ->
    payload =
      clientInformation:
        clientMedium: 'web'
    communicator.sendMessage('ClientRegistrationCommand', payload)
  
  # Our client registration response handler should store
  # required metadata and send an account login command
  handleClientRegistrationResponse: (data) ->
    # Store client registration data somewhere handy
    
    # Send account login command
    payload =
      userInformation:
        username: "myrlund"
        password: "ihazpassword"
    
    communicator.sendMessage('AccountLoginCommand', payload)
  
  # Handle the response to our AccountLoginCommand
  handleAccountLoginResponse: (data) ->
    # Store session keys and user data somewhere handy
    
    # Check response data to see if login was successful
    if data.loggedIn
      beHappy()
    else
      tryAgain()
{% endhighlight %}

Simply put, the various commands listen to responses and fire the next step as soon as the response is handled.
_It's easy to implement, and the message sequences are easily understood from the listener declarations in the controller initializer._

# The storage layer

We want a persistence layer capable of two things: _storing data_ received from the backend controllers, and _talking to the front-end_ part of our app.
Since we've already looked at setting up the controller layer, let's start with the former: storing data from the backend controllers.

## Storing data

In case you're not familiar with localStorage, fear not: you don't need to be. It's as simple a key-value store as they come.

{% highlight javascript %}
localStorage.setItem('ourKey', 'someValue')
localStorage.getItem('ourKey') // => 'someValue'
{% endhighlight %}

Next, we run into a small problem in that localStorage doesn't support storing objects. It is, however, pretty good at strings.
A simple solution is to serialize our objects into the store.
To make it so, we encapsulate the localStorage in an event-driven storage object:

{% highlight coffeescript %}
class Store
  
  # We keep an in-memory cache to speed up reading of data
  data: {}
  
  # Set this.store to localStorage and load cache
  constructor: (@schemas) ->
    @store = localStorage
    @loadCache()
  
  # We'll call addItems from the backend controllers.
  # 
  # items: object, indexed on unique id.
  #   ex. {"1": {content: "Foo"}, "2": {content: "Bar"}}
  addItems: (schema, items) ->
    
    # Add or overwrite existing items
    _.extend(@data[schema], items)
    
    # Write cache to store
    @save()
  
  # Iterates over keys in cache, serializing
  save: ->
    for key in _.keys(@data)
      @store.setItem(key, JSON.stringify(@data[key]))
  
  # Populates cache with stored data
  loadCache: ->
    for schema in @schemas
      @data[schema] = @fetch(schema) || {}
  
  # Fetches object from store
  fetch: (schema) ->
    JSON.parse(@store.getItem(schema))
  
{% endhighlight %}

## Talking to Backbone

Although Backbone is designed for AJAX REST APIs out of the box, it supports any kind of backend through [an extremely simple synchronization interface](http://backbonejs.org/#Sync).
One simply sets `Backbone.sync` to a function that in some way can handle the basic CRUD operations -- _create_, _read_, _update_ and _delete_.

Let's add a `sync` method to our store, along with some helper methods.

_**Note:** We're using a read-only API, so we don't really handle writing to the store. It should, however, be easy enough to implement by triggering a change event resulting in an appropriate backend call._

{% highlight coffeescript %}
class Store
  
  # ...
  
  # Attaches to Backbone.sync
  # 
  # method:  either "create", "read", "update" or "delete"
  # model:   the model instance or model class in question
  # options: carries callback functions
  sync: (method, model, options) =>
    resp = false
    schemaName = @getSchemaName(model)
    
    # Switch over the possible methods
    switch method
    
      when "create"
        # In our case, we never create models directly
        console.log "This shouldn't happen."
        
      when "read"
        # Read one or all models, depending on whether id is set
        resp = if model.id then
          @find(schema, model.id)
        else
          @findAll(schema)
        
        unless resp
          return options.error("Not found.")
        
      when "destroy"
        # Perform a fake destroy
        resp = true
    
    # Fire the appropriate callback
    if resp
      options.success(resp)
    else
      options.error("Unknown error.")
  
  # Simple getters for one or all models in a schema
  find: (schema, id) ->
    @data[schema] && @data[schema][id]
  findAll: (schema) ->
    _.values(@data[schema]) || []
  
  # Models either have a schema name attached to themselves
  # or through their collections
  getSchemaName: (model) ->
    if model.schemaName
      model.schemaName
    else if model.collection && model.collection.schemaName
      model.collection.schemaName

# Export and attach to Backbone.sync
this.store = new Store(['contacts'])
Backbone.sync = this.store.sync
{% endhighlight %}

Overriding the Backbone.sync allows Backbone to talk to our store, but web sockets are a two way street, and we still don't have any way of telling our Backbone collections of incoming data.

So, to the actual talking _to_ Backbone part...
A simple way to allow collections to subscribe to changes to schemas is to trigger an event when adding items from the backend.
Let's add to our `addItems` method.

{% highlight diff %}
  addItems: (schema, items) ->
    
    # Add or overwrite existing items
    _.extend(@data[schema], items)
    
    # Write cache to store
    @save()
+   
+   # Fire a notification passing the changed ids
+   payload = {}
+   payload[schema] = _.keys(items)
+   dispatch.trigger("store:change:#{schema}", payload)
{% endhighlight %}

Here is an example of a Backbone collection integrating with this event mechanism:

{% highlight coffeescript %}
class ContactCollection extends Backbone.Collection
  
  model: Contact
  
  # We don't use URLs in our protocol, but 
  # Backbone requires that we set it...
  url: ''
  
  # We'll need to define a schema name for use
  # in the Backbone.sync method of our store
  schemaName: "contacts"
  
  initialize: ->
    # Bind to the store's appropriate change event
    dispatch.on("store:change:#{@schemaName}", 
                @updateContacts, this)
  
  # Called whenever new data is inserted into 
  # the data store.
  updateContacts: (data) ->
    
    # The contacts property of the passed data is
    # an array of ids of the changed contacts
    contactIds = data[@schemaName]
    
    for contactId in contactIds
      # Check if the contact exists
      if conversation = @get(contactId)
        # If it exists, simply _set_ its updated properties
        conversation.set(store.find(@schemaName, contactId))
      else
        # Elsewise, create it and add it
        contactData = store.find(@schemaName, contactId)
        @add(new Contact(contactData))
{% endhighlight %}

# That's it!

That should cover integrating Backbone with an arbitrary web socket protocol.

Note that this approach is especially well suited to our particular use case, and there are probably both easier and better ways to integrate with other protocols.
However, our approach should be generic enough to be fitted to any sensible situation.

# Issues

On our journey, we ran into some issues that you might do well to keep in mind if you're trying to do something similar to us.

## LocalStorage is completely unencrypted

Without a backend rendering our HTML, we don't have any safe place to store user credentials. We're also handling sensitive data, which shouldn't be left plainly stored in any computer's web cache.

We took some simple measures to secure our users' data:

1. When a user logs out, we clear the entire localStorage.
2. In the login form, we present an option for whether the computer in use is a public computer. If so, persist as little as possible.

We've been looking into some recently matured client-side crypto libraries, and the possibilities of encrypting the store.
However, there is no safe place to store the encryption key client-side, requiring us to get it from the server for every page reload, in turn requiring authentication.
[This stackoverflow thread](http://stackoverflow.com/questions/2642043/html5-web-db-security) pretty much sums it up.

# Resources

1. **[Introducing WebSockets](http://www.html5rocks.com/en/tutorials/websockets/basics/)** by HTML5 rocks -- a great intro to using WebSockets in HTML5
2. **[backbone-localstorage.js](http://documentcloud.github.com/backbone/docs/backbone-localstorage.html)** by documentcloud -- a simple adapter for using Backbone with localStorage
3. **[Understanding Backbone](https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md)** by Kim Joar Bekkelund -- a great tutorial on Backbone-ifying a typical jQuery app

