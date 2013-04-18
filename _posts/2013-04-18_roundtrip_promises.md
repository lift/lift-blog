tags:	x-dpp
title:	Angular JS, Lift 3, and Streaming Promises

## Simple AngularJS ##

Lift has always had the best server-push technology around. Why? It's secure, it
deals well with spotty connections, it respects the limited number of HTTP connections
between the client and the server, and so much more.

[Angular JS](http://angularjs.org/) is a very exciting UI package that makes
building dynamic single-page applications a snap because there's a 2-way
binding between the model and the UI so that changes in the model are correctly
reflected in the UI. And the whole binding is declarative so that once you
use a model item in the UI, that part of the UI is always updated when the
model changes.

## Round Trips ##

The default way to build Angular JS apps is to associate certain
events in the client with REST calls to the server and when the REST
calls are satisfied, the model is updated and UI is redrawn. Sounds reasonable.

But there are issues:

* There's an HTTP connection open during the REST call and given the limited number of available HTTP connections from the browser to a given host, having multiple REST calls open at the same time is not optimal.
* REST works well for simple, quick request/response cycles, but if the computation takes a long time, there's an HTTP connection open for a long time and that sometimes mucks with proxies which will often time HTTP requests out after 90 seconds to 3 minutes.
* REST does not do streaming well
* The developer has to secure each REST call because REST endpoints are always open, so there has to be access control logic built into each call

With Lift 3, we introduce the idea of Round Trips: a client-side request that's sent to the server where the client receives a Streaming Promise, the server performs its computations and when the results are ready, they get pushed to the client… and if there are multiple results, the multiple results are pushed to the client as the results become available.

Lift's round trips have the following advantages over simple REST:

* They consume far fewer HTTP connections
* They use Lift's Comet server-push technology so that results are streamed to the client even if the results are computed while the client is temporarily disconnected and streamed in order
* The requests use Lift's Ajax facilities which include advanced retry and de-duping mechanisms
* Lift-json does a lot of the JSON <-> Scala-land serialization automatically
* Results are streamed so that results are pushed from the server to the client as the computations are completed on the server

## Simple Round Trip ##


Let's look at implementing a simple Round Trip for loading and saving
some String data. On the server-side we have code that looks like:

      // Save the text… we get a JSON blog and manually decode it
      // If an exception is thrown during the save, the client automatically
      // gets a Failure
      def doSave(info: JValue): JValue = {
         for {
           JString(path) <- info \ "path"
           JString(text) <- info \ "text"
         } {
           // save the text
         }
         JNull // a no-op
      }

      // Load the file
      def doLoad(fileName: String): String = {
        // load the named file, turn it into a String and return it
      }

    // Associate the server functions with client-side functions
    JScript(
    JsCrVar("serverFuncs", sess.buildRoundtrip(List[RoundTripInfo](
    "save" -> doSave _, "load" -> doLoad _)))

And to invoke the code from the client:

    // Save the file
    $scope.save = function() {
      serverFuncs.save({path: $scope.curFile, text: $scope.curText});
    }

    // load the named file and update the model
    // when the file arrives
    $scope.load = function(fn) {
      serverFuncs.load(fn).then(function(v) {
      $scope.$apply(function() {
        $scope.curFile = fn;
        $scope.curText = v;
       });
    });
    }

So the lines of code here are very similar to the lines of code for a REST implementation.

But we have access control because the client-side functions are actually associated by
GUID to the server-side functions so there's no worry about CSRF or protecting the REST
resource based on user permissions because the only functions that the client could
possibly call are the functions that are presented to it.

We also don't have to explicitly set up error handling because an exception on the
server-side will turn into a failure on the client.

## But it Streams ##

What happens when you want to stream data from the server to the client?

Lift's Round Trips have a simple answer, use Streams. Lift will automatically
detect if the return type of a `buildRoundTrip` function is a `Stream[T]`
and "do the right thing"™. So, if our server-side function is:

	  def thing(s: String): Stream[String] = {
	    var x = 0
	    (s + x) #:: {
	      Thread.sleep(1000);
	      x += 1;
	      s + x
	    } #:: {
	      Thread.sleep(1000);
	      x += 1;
	      s + x
	    } #:: Stream.empty[String]
	  }
	
And our client looks like:

    $scope.doThing = function(fn) {
      $scope.data = [];
      $scope.done = false;
      serverFuncs.thing(fn).then(function(v) {
      $scope.$apply(function() {
        $scope.data.push(v);
       }).done(function() {
       $scope.$apply(function() {$scope.done = true;})
       });
    });
    }

So, even thought the data comes in with 1 second gaps in between, the data gets pushed to
the client as it become available on the server.

## It's infinitely flexible ##

Sometimes a Stream with Exceptions to denote failure isn't the right answer. For
example, if you have an Iteratee library and would like to wrap a Round Trip around
an Iteratee, you need to signal new data, done, and failure. the `RoundTripHandlerFunc`
is the answer for you.

Here's the raw code:

	  private def thing(q: String, onChange: RoundTripHandlerFunc) {
	    if (q.trim.length < 3) {
	      onChange.failure("request too short!")
	    } else {
	      def doIt() {
	        if (ran.nextFloat() < 0.05f) {
	          onChange.done()
	        } else if (ran.nextFloat() < 0.02f) {
	          onChange.failure("Hey, I'm a failure")
	        } else {

	          onChange.send(/* some data */)
	            
	          // delay without consuming a thread
	          Schedule(doIt _, /* some timeout */)
	        }
	      }
	
	      doIt()
	    }
	  }

Okay, so the above code functions like the `Stream` example above, but gives us
explicit control over sending data, done, and failure.

You can make an implicit conversion from any form of streaming library to the
`RoundTripHandlerFunc` style call and have something like `def myFunc(in: MyData): Iteratee[T]` serviced via Round Trips.

Also note that the input types of our functions have been `String` and `JValue`, but
they can be anything as long as there's a way to convert from `JValue` to that
type via Lift-json.

## Wrap up ##

This describes how Lift 3's Round Trip feature lets you build really excellent AngularJS apps
because you can easily stream data from the client to the server, you don't have to worry
about converting data types, you don't have to worry about access control, and you
don't have to worry about connection starvation. In fact, you don't worry about plumbing,
you worry about the business logic of your app.

