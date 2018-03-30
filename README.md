# The Node Beginner Book: Request Routing

## 6. What's needed to "route" requests?

We need to be able to feed the requested URL and possible additional GET
and POST parameters into our router, and based on these the router then
needs to be able to decide which code to execute (this "code to
execute" is the third part of our application: a collection of request
handlers that do the actual work when a request is received).

So, we need to look into the HTTP requests and extract the requested URL
as well as the GET/POST parameters from them. It could be argued if that
should be part of the router or part of the server (or even a module of
its own), but let's just agree on making it part of our HTTP server for
now.

All the information we need is available through the *request* object
which is passed as the first parameter to our callback function
*onRequest()*. But to interpret this information, we need some
additional Node.js modules, namely *url* and *querystring*.

The *url* module provides methods which allow us to extract the
different parts of a URL (like e.g. the requested path and query
string), and *querystring* can in turn be used to parse the query string
for request parameters:

                                   url.parse(string).query
                                               |
               url.parse(string).pathname      |
                           |                   |
                           |                   |
                         ------ -------------------
    http://localhost:8888/start?foo=bar&hello=world
                                    ---       -----
                                     |          |
                                     |          |
            querystring.parse(string)["foo"]    |
                                                |
                       querystring.parse(string)["hello"]

We can, of course, also use *querystring* to parse the body of a POST
request for parameters, as we will see later.

Let's now add to our *onRequest()* function the logic needed to find
out which URL path the browser requested:

    var http = require("http");
    var url = require("url");
    function start() {
      function onRequest(request, response) {
    	var pathname = url.parse(request.url).pathname;
    	console.log("Request for " + pathname + " received.");
    	response.writeHead(200, {"Content-Type": "text/plain"});
    	response.write("Hello World");
    	response.end();
      }
      http.createServer(onRequest).listen(8888);
      console.log("Server has started.");
    }
    exports.start = start;

Fine. Our application can now distinguish requests based on the URL path
requested - this allows us to map requests to our request handlers based
on the URL path using our (yet to be written) router.

In the context of our application, it simply means that we will be able
to have requests for the */start* and */upload* URLs handled by
different parts of our code. We will see how everything fits together
soon.

Ok, it's time to actually write our router. We've created a new file called *router.js*, with the following content:

    function route(pathname) {
      console.log("About to route a request for " + pathname);
    }
    exports.route = route;

Of course, this code basically does nothing, but that's ok for now.
Let's first see how to wire together this router with our server before
putting more logic into the router.

Our HTTP server needs to know about and make use of our router. We could
hard-wire this dependency into the server, but because we learned the
hard way from our experience with other programming languages, we are
going to loosely couple server and router by injecting this dependency
(you may want to read [Martin Fowler's excellent post on Dependency
Injection](http://martinfowler.com/articles/injection.html) for background information).

Let's first extend our server's *start()* function in order to enable
us to pass the route function to be used by parameter:

    var http = require("http");
    var url = require("url");
    function start(route) {
      function onRequest(request, response) {
        var pathname = url.parse(request.url).pathname;
        console.log("Request for " + pathname + " received.");
        route(pathname);
        response.writeHead(200, {"Content-Type": "text/plain"});
        response.write("Hello World");
        response.end();
      }
      http.createServer(onRequest).listen(8888);
      console.log("Server has started.");
    }
    exports.start = start;

And let's extend our *index.js* accordingly, that is, injecting the
route function of our router into the server:

    var server = require("./server");
    var router = require("./router");
    server.start(router.route);

Again, we are passing a function, which by now isn't any news for us.

If we click `'Show'` and request the URL /foo, you can now see from the application's output that our HTTP server makes use of our router and passes it the requested pathname - see the following in `'Logs'`:
    
    Request for /foo received.
    About to route a request for /foo

(I omitted the rather annoying output for the /favicon.ico request).

<a href="https://glitch.com/edit/#!/remix/NodeBeginner7/2bcba06c-14e7-4642-b68b-bb72f4f49fc4" target="_blank">>> Go to the next part</a>.


# License
The Node Beginner Book (C) Manuel Kiessling
[Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License](https://creativecommons.org/licenses/by-nc-sa/3.0/). Some small text changes have been made to the original to make sense on Glitch.
