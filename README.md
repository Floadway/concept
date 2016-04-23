

# Floodway

Floodway is the result of several years of back-end work done by me, it is the solution to some of the things that have always bothered me while writing software.
The concept repository holds the basic structure and explaination to the framework. That said, let's get started.


## What is Floodway supposed to be?

Floodway is an architecture building up on node.js (for now; framework should be implementable in other languages) that is supposed to: 

* Speed up developement of **Real-Time** applications
* Remove **boilerplate** code from Applications
* Allow for easy integration with multiple protocols such as **HTTP, Websockets, GCM , TCP etc.**
* Allow for **easy scaling** across multiple machines and even applications

Achieving these goals is not really easy to do, this is the reason why all the theoretical work is split up from the practical implementation. 

## General structure

Floodway is supposed to be scalable right from the start. Scaling can be a pain and time-consuming, therefore it is Floodway's goal to automatically do most of the work for you.

Floodway is split up into multiple servers. 

* Interface Servers 
  * Handle connections on the several Interfaces (HTTP,Websockets,GCM,TCP etc.)
* Eventbusses
  * Handle communication between the processing servers
* Processing Servers
  * Are slaves to interface servers; Process requests
  
**Example**

1. User connects to the interface server via a Websocket
2. User performs a request
3. Interface Server picks a processing Server based on load. 
4. Processing server processes the request
5. Result gets passed back to the interface server
6. Client receives the result

### Why split up anways?

This architecture has multiple benefits.

* Work-heavy applications can easily add more processing slaves to the cluster without needing to setup routing to each interface
* Each Interface can have a dedicated interface server
* Possible to easily utilize services like AWS as result of run-time cluster size changing.


## Setup 

The setup of Floodway is supposed to be as easy as possbile. That's why it will ship with a CLI that sets up everything on its own

The CLI will then start an instance of the specified type, and make sure it's kept running. 

A single CLI can also manage multiple services on the same machine. The CLI is either instructed via parameters or a clusterConfig.json file.

## Inter-cluster communication

This aspect is handled via WebSocket connections, as they have low overhead, are stable and easily encryptable. 

## Developement 

The goal of Floodway is to simplify the coding experience. The core idea is that each interface can follow a uniform request scheme:

**Request**

```js
  {
    "messageType": "request",
    "requestId": "1203vjqioiqwjoijqw",  // Unique ID per request.
    "namespace": "user",
    "action": "login",
    "parameters": {
      "username": "foobar",
      "password": "nevertransmitpasswordsasplaintext"
    }
  }
```

This is a basic request. It is the job of the interface servers to build such requests. The request is then passed to a processing server alonside with a **sessionID** and the **ID** of the interface server to handle replies.


A request also has the following attributes:

* It can have multiple results, meaning the result can be updated. This only applies if the interface supports updates AND the request is specified as ```updatable```
* It can **fail**, meaning the operation did not work. This can only happen once.  Once a request is failed it also no longer supports sending a result. A failure returns an error in form of a camelCasedString. The error has to be defined when declaring the action.


** Actions **

Are essentially endpoints. They are defined in the application code running on the cluster and have to belong to a namespace.

```js
  // Declaring a namespace and some actions
  
  floodway.registerNamespace({
    name: "user" // Has to be unique
    
    middleware: {
      doSomethingFancy: {
        possibleErrors: ["nothingFanyToDo"]
        params: {} // Optional requires a set of of parameters validated before runnning this middleware
        process: function(request){} // Has a fail(string) and a next(obj) method. The next method can be passed a set of new params
      }
    }
    
    actions: {
      
      
      login: {
        updatable: false,
        middleware: ["doSomethingFancy"]
        description: "A method description is required. This might be a pain, but it is great to have good documentation",
        possibleErrors: ["wrongCredentials","internalError","accountNotActivated"] // Errors have to be defined here to ensure proper documentation and easier integration on the front-end
        params: { // Validation follows the validation spec of the Floodway validator. The params object is of type "object" with mode set to "strict"
          username: { type: "string", minLength: 4, trim: true, maxLength: 32 },
          password: { type: "string", minLength: 6, trim: true, maxLength: 64 },
          rememberMe: { type: "boolean", default: false } // Defaults to false if not processable
        }
        process: function(request){ } // The request object contains a session instance, parameters, as well as a fail(string) and result(object) method
      }
      
      logout: {...}
    }
  
  })
  
```
Floodway has a custom validator built in. You can find more information about it in the validator repository. It can validate json trees recursively and also perform some actions on the data. It also has support for custom validaton functions.

Before an action is ran the validator checks the provided parameters to make sure they match the specified spec. 

An action can also require a middleware to be ran beforehand. A middleware can either fail, preventing the action from being executed or it can call the ```next()``` method.
The params passed to the ```next()``` method are then used as new parameters for the request. Also validation is re-ran on those.

**An action can also require a middleware from another namespace. This can be achieved by using the dot-notation (eg. ```"user.authenticate"``` )**


