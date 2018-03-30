# The Node Beginner Book: Handling File Uploads

## 13. Handling file uploads

Let's tackle our final use case. Our plan was to allow users to upload
an image file, and display the uploaded image in the browser.

Back in the 90's this would have qualified as a business model for an
IPO, today it must suffice to teach us two things: how to install
external Node.js libraries, and how to make use of them in our own code.

The external module we are going to use is *node-formidable* by Felix
Geisendoerfer. It nicely abstracts away all the nasty details of parsing
incoming file data. At the end of the day, handling incoming files is
"only" about handling POST data - but the devil really *is* in the
details here, and using a ready-made solution makes a lot of sense in
this case.

In order to make use of Felix' code, the according Node.js module needs
to be installed. Node.js ships with its own package manager, dubbed
*NPM*. It allows us to install external Node.js modules in a very
convenient fashion. On a working local Node.js installation, it boils down
to issuing

    npm install formidable

via the command line. On Glitch, we just need to add it to the 'dependencies' section in our `package.json` file.

The *formidable* module is now available to our own code - all we need
to do is requiring it just like one of the built-in modules we used
earlier:

    var formidable = require("formidable");

The metaphor formidable uses is that of a form being submitted via HTTP
POST, making it parseable in Node.js. All we need to do is create a new
*IncomingForm*, which is an abstraction of this submitted form, and
which can then be used to parse the *request* object of our HTTP server
for the fields and files that were submitted through this form.

The example code from the node-formidable project page shows how the
different parts play together:

    var formidable = require('formidable'),
        http = require('http'),
        sys = require('sys');
    http.createServer(function(req, res) {
      if (req.url == '/upload' && req.method.toLowerCase() == 'post') {
        // parse a file upload
        var form = new formidable.IncomingForm();
        form.parse(req, function(error, fields, files) {
          res.writeHead(200, {'content-type': 'text/plain'});
          res.write('received upload:\n\n');
          res.end(sys.inspect({fields: fields, files: files}));
        });
        return;
      }
      // show a file upload form
      res.writeHead(200, {'content-type': 'text/html'});
      res.end(
        '<form action="/upload" enctype="multipart/form-data" '+
        'method="post">'+
        '<input type="text" name="title"><br>'+
        '<input type="file" name="upload" multiple="multiple"><br>'+
        '<input type="submit" value="Upload">'+
        '</form>'
      );
    }).listen(8888);

If we put this code into a file and execute it through *node*, we are
able to submit a simple form, including a file upload, and see how the
*files* object, which is passed to the callback defined in the
*form.parse* call, is structured:

    received upload:
    { fields: { title: 'Hello World' },
      files:
       { upload:
          { size: 1558,
            path: '/tmp/1c747974a27a6292743669e91f29350b',
            name: 'us-flag.png',
            type: 'image/png',
            lastModifiedDate: Tue, 21 Jun 2011 07:02:41 GMT,
            _writeStream: [Object],
            length: [Getter],
            filename: [Getter],
            mime: [Getter] } } }

In order to make our use case happen, what we need to do is to include
the form-parsing logic of formidable into our code structure, plus we
will need to find out how to serve the content of the uploaded file
(which is saved into the */tmp* folder) to a requesting browser.

Let's tackle the latter one first: if there is an image file on our
local hardrive, how do we serve it to a requesting browser?

We are obviously going to read the contents of this file into our
Node.js server, and unsurprisingly, there is a module for that - it's
called *fs*.

Let's add another request handler for the URL */show*, which will
hardcodingly display the contents of the file */tmp/test.png*. It of
course makes a lot of sense to save a real png image file to this
location first.

We'd need to modify *requestHandlers.js* as follows:

    var querystring = require("querystring"),
        fs = require("fs");
    function start(response, postData) {
      console.log("Request handler 'start' was called.");
      var body = '<html>'+
        '<head>'+
        '<meta http-equiv="Content-Type" '+
        'content="text/html; charset=UTF-8" />'+
        '</head>'+
        '<body>'+
        '<form action="/upload" method="post">'+
        '<textarea name="text" rows="20" cols="60"></textarea>'+
        '<input type="submit" value="Submit text" />'+
        '</form>'+
        '</body>'+
        '</html>';
      response.writeHead(200, {"Content-Type": "text/html"});
      response.write(body);
      response.end();
    }
    function upload(response, postData) {
      console.log("Request handler 'upload' was called.");
      response.writeHead(200, {"Content-Type": "text/plain"});
      response.write("You've sent the text: "+
      querystring.parse(postData).text);
      response.end();
    }
    function show(response) {
      console.log("Request handler 'show' was called.");
      response.writeHead(200, {"Content-Type": "image/png"});
      fs.createReadStream("/tmp/test.png").pipe(response);
    }
    exports.start = start;
    exports.upload = upload;
    exports.show = show;

We also need to map this new request handler to the URL */show* in file
*index.js*\:

    var server = require("./server");
    var router = require("./router");
    var requestHandlers = require("./requestHandlers");
    var handle = {};
    handle["/"] = requestHandlers.start;
    handle["/start"] = requestHandlers.start;
    handle["/upload"] = requestHandlers.upload;
    handle["/show"] = requestHandlers.show;
    server.start(router.route, handle);

Fine. All we need to do now is: 

* add a file upload element to the form which is served at */start*,
* integrate node-formidable into the *upload* request handler, in order
  to save the uploaded file to */tmp/test.png*,
* embed the uploaded image into the HTML output of the */upload* URL.

Step 1 is simple. We need to add an encoding type of *multipart/form-data* to our HTML form, remove the textarea, add a file upload input field, and change the submit button text to "Upload file". We've done just that in file *requestHandlers.js*\:

    var querystring = require("querystring"),
        fs = require("fs");
    function start(response, postData) {
      console.log("Request handler 'start' was called.");
      var body = '<html>'+
        '<head>'+
        '<meta http-equiv="Content-Type" '+
        'content="text/html; charset=UTF-8" />'+
        '</head>'+
        '<body>'+
        '<form action="/upload" enctype="multipart/form-data" '+
        'method="post">'+
        '<input type="file" name="upload">'+
        '<input type="submit" value="Upload file" />'+
        '</form>'+
        '</body>'+
        '</html>';
      response.writeHead(200, {"Content-Type": "text/html"});
      response.write(body);
      response.end();
    }
    function upload(response, postData) {
      console.log("Request handler 'upload' was called.");
      response.writeHead(200, {"Content-Type": "text/plain"});
      response.write("You've sent the text: "+
      querystring.parse(postData).text);
      response.end();
    }
    function show(response) {
      console.log("Request handler 'show' was called.");
      response.writeHead(200, {"Content-Type": "image/png"});
      fs.createReadStream("/tmp/test.png").pipe(response);
    }
    exports.start = start;
    exports.upload = upload;
    exports.show = show;

Great. The next step is a bit more complex of course. The first problem
is: we want to handle the file upload in our *upload* request handler,
and there, we will need to pass the *request* object to the *form.parse*
call of node-formidable.

But all we have is the *response* object and the *postData* array. Sad
panda. Looks like we will have to pass the *request* object all the way
from the server to the router to the request handler. There may be more
elegant solutions, but this approach should do the job for now.

And while we are at it, let's remove the whole *postData* stuff in our
server and request handlers - we won't need it for handling the file
upload, and it even raises a problem: we already "consumed" the *data*
events of the *request* object in the server, which means that
*form.parse*, which also needs to consume those events, wouldn't
receive any more data from them (because Node.js doesn't buffer any
data).

Let's start with *server.js* - we remove the postData handling and the
*request.setEncoding* line (which is going to be handled by
node-formidable itself), and we pass *request* to the router instead:

    var http = require("http");
    var url = require("url");
    function start(route, handle) {
      function onRequest(request, response) {
        var pathname = url.parse(request.url).pathname;
        console.log("Request for " + pathname + " received.");
        route(handle, pathname, response, request);
      }
      http.createServer(onRequest).listen(8888);
      console.log("Server has started.");
    }
    exports.start = start;

Next comes *router.js* - we don't need to pass *postData* on anymore,
and instead pass *request*\:

    function route(handle, pathname, response, request) {
      console.log("About to route a request for " + pathname);
      if (typeof handle[pathname] === 'function') {
        handle[pathname](response, request);
      } else {
        console.log("No request handler found for " + pathname);
        response.writeHead(404, {"Content-Type": "text/html"});
        response.write("404 Not found");
        response.end();
      }
    }
    exports.route = route;

Now, the *request* object can be used in our *upload* request handler
function. node-formidable will handle the details of saving the uploaded
file to a local file within */tmp*, but we need to make sure that this
file is renamed to */tmp/test.png* ourselves. Yes, we keep things really
simple and assume that only PNG images will be uploaded.

There is a bit of extra-complexity in the rename logic: the Windows
implementation of node doesn't like it when you try to rename a file
onto the position of an existing file, which is why we need to delete
the file in case of an error. This doesn't matter in HyperDev, but just so you know for your own purposes.

We've put the pieces of managing the uploaded file and renaming it
together in file *requestHandlers.js*\:

    var querystring = require("querystring"),
        fs = require("fs"),
        formidable = require("formidable");
    function start(response) {
      console.log("Request handler 'start' was called.");
      var body = '<html>'+
        '<head>'+
        '<meta http-equiv="Content-Type" '+
        'content="text/html; charset=UTF-8" />'+
        '</head>'+
        '<body>'+
        '<form action="/upload" enctype="multipart/form-data" '+
        'method="post">'+
        '<input type="file" name="upload" multiple="multiple">'+
        '<input type="submit" value="Upload file" />'+
        '</form>'+
        '</body>'+
        '</html>';
      response.writeHead(200, {"Content-Type": "text/html"});
      response.write(body);
      response.end();
    }
    function upload(response, request) {
      console.log("Request handler 'upload' was called.");
      var form = new formidable.IncomingForm();
      console.log("about to parse");
      form.parse(request, function(error, fields, files) {
        console.log("parsing done");
        /* Possible error on Windows systems:
           tried to rename to an already existing file */
        fs.rename(files.upload.path, "/tmp/test.png", function(error) {
          if (error) {
            fs.unlink("/tmp/test.png");
            fs.rename(files.upload.path, "/tmp/test.png");
          }
        });
        response.writeHead(200, {"Content-Type": "text/html"});
        response.write("received image:<br/>");
        response.write("<img src='/show' />");
        response.end();
      });
    }
    function show(response) {
      console.log("Request handler 'show' was called.");
      response.writeHead(200, {"Content-Type": "image/png"});
      fs.createReadStream("/tmp/test.png").pipe(response);
    }
    exports.start = start;
    exports.upload = upload;
    exports.show = show;

And that's it - the complete use case will be available. Select a local PNG image from your hardrive, upload it to the server, and have it displayed in the web page.

# Conclusion and outlook

Congratulations, our mission is accomplished! We wrote a simple yet
full-fledged Node.js web application. We talked about server-side
JavaScript, functional programming, blocking and non-blocking
operations, callbacks, events, custom, internal and external modules,
and a lot more.

# License
The Node Beginner Book (C) Manuel Kiessling
[Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License](https://creativecommons.org/licenses/by-nc-sa/3.0/). Some small text changes have been made to the original to make sense on Glitch.
