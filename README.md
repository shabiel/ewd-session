# ewd-session: Session Management based on ewd-document-store DocumentNodes
 
Rob Tweed <rtweed@mgateway.com>  
24 February 2016, M/Gateway Developments Ltd [http://www.mgateway.com](http://www.mgateway.com)  

Twitter: @rtweed

Google Group for discussions, support, advice etc: [http://groups.google.co.uk/group/enterprise-web-developer-community](http://groups.google.co.uk/group/enterprise-web-developer-community)


## ewd-session

This module may be used to provide session management.  Sessions make use of the persistent JavaScript objects and 
fine-grained document database provided by ewd-document-store.  For more information on ewd-document-store:

[https://github.com/robtweed/ewd-document-store](https://github.com/robtweed/ewd-document-store)

You'll usually use ewd-session within an ewd-qoper8 worker process.  For more informaation on ewd-qoper8:

[https://github.com/robtweed/ewd-qoper8](https://github.com/robtweed/ewd-qoper8)


## Installing

       npm install ewd-session
	   
## Using ewd-session

If you are using ewd-session within an ewd-qoper8 worker process, you must first have connected the worker to a
database that is supported by ewd-document-store, eg:

- InterSystems Cache, using ewd-qoper8-cache: [https://github.com/robtweed/ewd-qoper8-cache](https://github.com/robtweed/ewd-qoper8-cache)
- InterSystems GlobalsDB using ewd-qoper8-gtm [https://github.com/robtweed/ewd-qoper8-gtm](https://github.com/robtweed/ewd-qoper8-gtm)
- GT.M (module not yet implemented)

Specifically, ewd-session expects the worker process to have instantiated a DocumentStore object, which is what these
database-specific modules will have done when invoked within the worker's 'start' event handler, eg:

      this.on('start', function() {
        var connectCacheTo = require('ewd-qoper8-cache');
        connectCacheTo(this);
        // this.documentStore provides access to the DocumentStore object
      });


In order to create and use Sessions, you must load the ewd-session module into your ewd-qoper8 worker module:

      var sessions = require('ewd-session');

This provides you with the following constructors and methods:

- init:               initialises the session module environment
- create:             method for creating a new Session: returns a Session object (including Session.token - a unique token)
- authenticate:       method for authenticating a Session Token and returning the corresponding Session object if valid
- garbageCollector:   timer for use within a ewd-qoper8 worker process, for garbage-collecting expired Sessions
- uuid:               method for generating a UUID
- symbolTable         specialist method: creates methods for maintaining the connected MUMPS process's symbol table (aka local variables)


### Initialising ewd-sessions

You must first initialise ewd-sessions.  This provides it with access to the DocumentStore object, eg:

    sessions.init(this.documentStore);

By default, Sessions are maintained in a Document named '%zewdSession'.  You can change this by using the init() function's second 
argument, eg:

    sessions.init(this.documentStore, 'mySessionDocument');


### Session Garbage Collection

It is important to enable Session Garbage collection, otherwise your Document Store will accumulate expired session data.

If you are using ewd-qoper8, you should start an instance of the Session Garbage collector timer in your worker module.
Do this within the worker's on('DocumentStoreStarted') handler:

    this.on('DocumentStoreStarted', function() {
      sessions.garbageCollector(this);
    });

By default, every 5 minutes, this timer will check the Document Store for sessions and delete any expired ones.  You can adjust the 
checking frequency using the 2nd optional argument which specifies the delay time in seconds:

      sessions.garbageCollector(this, 1200); // check every 20 minutes


### Creating a New Session

To create a new Session:

      var applicationName = 'vista';
      var sessionTimeout = 300;  // in seconds; defaults to 3600 = 1 hour
      var session = sessions.create(application, sessionTimeout);

You'll be returned a Session object which provides access to the underlying DocumentNode storing the session's data.

A Token Object will also have been created.  This provides a pointer from the session's unique token to the Session.  You
don't directly access the Token Object.

Normally you'll send the Session token value (session.token) back to the client / browser.  It is used for subsequent, secure
access to the Session the next time the user or client requires access to their Session.

### The Session Object

#### Properties

- exists         (boolean; read only)
- token          (string: read only) The unique token UUID that was created for this session
- expired        (boolean: read only) Invoking this automatically deletes the Session from the Document Store if it was found to be expired
- authenticated  (boolean: read/write) Set this to true when the user successfully logs in (if relevant)
- expiryTime     (integer: read/write) The number of seconds before the Session will expire if inactive
- data           (DocumentNode Object) Provides access to the Session's persistent data

#### Methods

- delete()   deletes the Session and its associated Token object.  Normally you can let sessions just expire naturally, so 
use of this method is usually unnecessary.


### Accessing an Existing Session using a Token

You'll normally send a Session token as part of a message to your worker module.  In the case of Express HTTP requests, the
token will normally be conveyed as the Authorization header value.  In WebSocket messages, you can allocate an appropriate
message property to hold it.

To access the associated Session, use the authenticate method:

      var results = sessions.authenticate(token);

      where token is the value of the Session Token

results is an object of the structure:

     {
       error: errorMessage,
       session: SessionObject
     }


results.session will contain the associated Session object if:

- the token value was not null or a null string; and
- the token was found in the Token Document in the Document Store; and
- a corresponding Session Document was found in the Document Store; and
- the corresponding Session had not expired

Otherwise results.error will be defined and will be a string value that explains the reason why a Session object could not be returned.

By default, a successful invocation of the sessions.authenticate() method will update the associated Session's expiry time.  In certain
circumstances you might not want this to happen, for example if you want to log in the user.  In such a situation, if the login
credentials aren't correct, you'll want to keep the clock ticking in terms of the remaining time you allow before the user can
log in.

In such a situation, add true as a third argument:

      var results = sessions.authenticate(token, true);

## Examples

You'll find two examples in the /examples path of this module repository:

- cache-standalone.js: stanadlone example using Cache that can be run as a script file
- cache-express.js:    Example using Express, ewd-qoper8 and Cache.  It will load cache-module1.js as the worker module. This allows you
 to invoke a GET request: /qoper8/initiate which starts a Session and returns a token.  You can then login using a POST request
of /qoper8/login with a payload: {"username": "rob", "password": "secret"}  


## License

 Copyright (c) 2016 M/Gateway Developments Ltd,                           
 Reigate, Surrey UK.                                                      
 All rights reserved.                                                     
                                                                           
  http://www.mgateway.com                                                  
  Email: rtweed@mgateway.com                                               
                                                                           
                                                                           
  Licensed under the Apache License, Version 2.0 (the "License");          
  you may not use this file except in compliance with the License.         
  You may obtain a copy of the License at                                  
                                                                           
      http://www.apache.org/licenses/LICENSE-2.0                           
                                                                           
  Unless required by applicable law or agreed to in writing, software      
  distributed under the License is distributed on an "AS IS" BASIS,        
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 
  See the License for the specific language governing permissions and      
   limitations under the License.      
