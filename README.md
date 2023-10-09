# Intermediate-JS-Notes Week 2

#File and FileReader:
A File object inherits from Blob and is extended with filesystem-related capabilities.

There are two ways to obtain it.

First, there’s a constructor, similar to Blob:

new File(fileParts, fileName, [options])
fileParts – is an array of Blob/BufferSource/String values.
fileName – file name string.
options – optional object:
lastModified – the timestamp (integer date) of last modification.
Second, more often we get a file from <input type="file"> or drag’n’drop or other browser interfaces. In that case, the file gets this information from OS.

As File inherits from Blob, File objects have the same properties, plus:

name – the file name,
lastModified – the timestamp of last modification.
That’s how we can get a File object from <input type="file">:

<input type="file" onchange="showFile(this)">

<script>
function showFile(input) {
  let file = input.files[0];

  alert(`File name: ${file.name}`); // e.g my.png
  alert(`Last modified: ${file.lastModified}`); // e.g 1552830408824
}
</script>
Please note:
The input may select multiple files, so input.files is an array-like object with them. Here we have only one file, so we just take input.files[0].

FileReader
FileReader is an object with the sole purpose of reading data from Blob (and hence File too) objects.

It delivers the data using events, as reading from disk may take time.

The constructor:

let reader = new FileReader(); // no arguments
The main methods:

readAsArrayBuffer(blob) – read the data in binary format ArrayBuffer.
readAsText(blob, [encoding]) – read the data as a text string with the given encoding (utf-8 by default).
readAsDataURL(blob) – read the binary data and encode it as base64 data url.
abort() – cancel the operation.
The choice of read* method depends on which format we prefer, how we’re going to use the data.

readAsArrayBuffer – for binary files, to do low-level binary operations. For high-level operations, like slicing, File inherits from Blob, so we can call them directly, without reading.
readAsText – for text files, when we’d like to get a string.
readAsDataURL – when we’d like to use this data in src for img or another tag. There’s an alternative to reading a file for that, as discussed in chapter Blob: URL.createObjectURL(file).

#As the reading proceeds, there are events:

loadstart – loading started.
progress – occurs during reading.
load – no errors, reading complete.
abort – abort() called.
error – error has occurred.
loadend – reading finished with either success or failure.
When the reading is finished, we can access the result as:

reader.result is the result (if successful)
reader.error is the error (if failed).
The most widely used events are for sure load and error.

Here’s an example of reading a file:

<input type="file" onchange="readFile(this)">

<script>
function readFile(input) {
  let file = input.files[0];
  let reader = new FileReader();

  reader.readAsText(file);
  reader.onload = function() {
    console.log(reader.result);
  };

  reader.onerror = function() {
    console.log(reader.error);
  };
}
</script>

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Summary
File objects inherit from Blob.

In addition to Blob methods and properties, File objects also have name and lastModified properties, plus the internal ability to read from filesystem. We usually get File objects from user input, like <input> or Drag’n’Drop events (ondragend).

FileReader objects can read from a file or a blob, in one of three formats:

String (readAsText).
ArrayBuffer (readAsArrayBuffer).
Data url, base-64 encoded (readAsDataURL).
In many cases though, we don’t have to read the file contents. Just as we did with blobs, we can create a short url with URL.createObjectURL(file) and assign it to <a> or <img>. This way the file can be downloaded or shown up as an image, as a part of canvas etc.

And if we’re going to send a File over a network, that’s also easy: network API like XMLHttpRequest or fetch natively accepts File objects.

#Files and PostRequest
-----------------------


Fetch: Cross-Origin Requests
If we send a fetch request to another web-site, it will probably fail.

For instance, let’s try fetching http://example.com:

try {
  await fetch('http://example.com');
} catch(err) {
  alert(err); // Failed to fetch
}
Fetch fails, as expected.

The core concept here is origin – a domain/port/protocol triplet.

Cross-origin requests – those sent to another domain (even a subdomain) or protocol or port – require special headers from the remote side.

That policy is called “CORS”: Cross-Origin Resource Sharing.

Why is CORS needed? A brief history
CORS exists to protect the internet from evil hackers.

Seriously. Let’s make a very brief historical digression.

For many years a script from one site could not access the content of another site.

That simple, yet powerful rule was a foundation of the internet security. E.g. an evil script from website hacker.com could not access the user’s mailbox at website gmail.com. People felt safe.

JavaScript also did not have any special methods to perform network requests at that time. It was a toy language to decorate a web page.

But web developers demanded more power. A variety of tricks were invented to work around the limitation and make requests to other websites.

Using forms
One way to communicate with another server was to submit a <form> there. People submitted it into <iframe>, just to stay on the current page, like this:

<!-- form target -->
<iframe name="iframe"></iframe>

<!-- a form could be dynamically generated and submitted by JavaScript -->
<form target="iframe" method="POST" action="http://another.com/…">
  ...
</form>
So, it was possible to make a GET/POST request to another site, even without networking methods, as forms can send data anywhere. But as it’s forbidden to access the content of an <iframe> from another site, it wasn’t possible to read the response.

To be precise, there were actually tricks for that, they required special scripts at both the iframe and the page. So the communication with the iframe was technically possible. Right now there’s no point to go into details, let these dinosaurs rest in peace.

Using scripts
Another trick was to use a script tag. A script could have any src, with any domain, like <script src="http://another.com/…">. It’s possible to execute a script from any website.

If a website, e.g. another.com intended to expose data for this kind of access, then a so-called “JSONP (JSON with padding)” protocol was used.

Here’s how it worked.

Let’s say we, at our site, need to get the data from http://another.com, such as the weather:

First, in advance, we declare a global function to accept the data, e.g. gotWeather.

// 1. Declare the function to process the weather data
function gotWeather({ temperature, humidity }) {
  alert(`temperature: ${temperature}, humidity: ${humidity}`);
}
Then we make a <script> tag with src="http://another.com/weather.json?callback=gotWeather", using the name of our function as the callback URL-parameter.

let script = document.createElement('script');
script.src = `http://another.com/weather.json?callback=gotWeather`;
document.body.append(script);
The remote server another.com dynamically generates a script that calls gotWeather(...) with the data it wants us to receive.

// The expected answer from the server looks like this:
gotWeather({
  temperature: 25,
  humidity: 78
});
When the remote script loads and executes, gotWeather runs, and, as it’s our function, we have the data.

That works, and doesn’t violate security, because both sides agreed to pass the data this way. And, when both sides agree, it’s definitely not a hack. There are still services that provide such access, as it works even for very old browsers.

After a while, networking methods appeared in browser JavaScript.

At first, cross-origin requests were forbidden. But as a result of long discussions, cross-origin requests were allowed, but with any new capabilities requiring an explicit allowance by the server, expressed in special headers.

Safe requests
There are two types of cross-origin requests:

Safe requests.
All the others.
Safe Requests are simpler to make, so let’s start with them.

A request is safe if it satisfies two conditions:

Safe method: GET, POST or HEAD
Safe headers – the only allowed custom headers are:
Accept,
Accept-Language,
Content-Language,
Content-Type with the value application/x-www-form-urlencoded, multipart/form-data or text/plain.
Any other request is considered “unsafe”. For instance, a request with PUT method or with an API-Key HTTP-header does not fit the limitations.

The essential difference is that a safe request can be made with a <form> or a <script>, without any special methods.

So, even a very old server should be ready to accept a safe request.

Contrary to that, requests with non-standard headers or e.g. method DELETE can’t be created this way. For a long time JavaScript was unable to do such requests. So an old server may assume that such requests come from a privileged source, “because a webpage is unable to send them”.

When we try to make a unsafe request, the browser sends a special “preflight” request that asks the server – does it agree to accept such cross-origin requests, or not?

And, unless the server explicitly confirms that with headers, an unsafe request is not sent.

Now we’ll go into details.

CORS for safe requests
If a request is cross-origin, the browser always adds the Origin header to it.

For instance, if we request https://anywhere.com/request from https://javascript.info/page, the headers will look like:

GET /request
Host: anywhere.com
Origin: https://javascript.info
...
As you can see, the Origin header contains exactly the origin (domain/protocol/port), without a path.

The server can inspect the Origin and, if it agrees to accept such a request, add a special header Access-Control-Allow-Origin to the response. That header should contain the allowed origin (in our case https://javascript.info), or a star *. Then the response is successful, otherwise it’s an error.

The browser plays the role of a trusted mediator here:

It ensures that the correct Origin is sent with a cross-origin request.
It checks for permitting Access-Control-Allow-Origin in the response, if it exists, then JavaScript is allowed to access the response, otherwise it fails with an error.

Here’s an example of a permissive server response:

200 OK
Content-Type:text/html; charset=UTF-8
Access-Control-Allow-Origin: https://javascript.info
Response headers
For cross-origin request, by default JavaScript may only access so-called “safe” response headers:

#Cache-Control
#Content-Language
#Content-Length
#Content-Type
#Expires
#Last-Modified
#Pragma

Accessing any other response header causes an error.

To grant JavaScript access to any other response header, the server must send the Access-Control-Expose-Headers header. It contains a comma-separated list of unsafe header names that should be made accessible.

For example:

200 OK
Content-Type:text/html; charset=UTF-8
Content-Length: 12345
Content-Encoding: gzip
API-Key: 2c9de507f2c54aa1
Access-Control-Allow-Origin: https://javascript.info
Access-Control-Expose-Headers: Content-Encoding,API-Key
With such an Access-Control-Expose-Headers header, the script is allowed to read the Content-Encoding and API-Key headers of the response.

“Unsafe” requests
------------------
We can use any HTTP-method: not just GET/POST, but also PATCH, DELETE and others.

Some time ago no one could even imagine that a webpage could make such requests. So there may still exist webservices that treat a non-standard method as a signal: “That’s not a browser”. They can take it into account when checking access rights.

So, to avoid misunderstandings, any “unsafe” request – that couldn’t be done in the old times, the browser does not make such requests right away. First, it sends a preliminary, so-called “preflight” request, to ask for permission.

A preflight request uses the method OPTIONS, no body and three headers:

Access-Control-Request-Method header has the method of the unsafe request.
Access-Control-Request-Headers header provides a comma-separated list of its unsafe HTTP-headers.
Origin header tells from where the request came. (such as https://javascript.info)
If the server agrees to serve the requests, then it should respond with empty body, status 200 and headers:

Access-Control-Allow-Origin must be either * or the requesting origin, such as https://javascript.info, to allow it.
Access-Control-Allow-Methods must have the allowed method.
Access-Control-Allow-Headers must have a list of allowed headers.
Additionally, the header Access-Control-Max-Age may specify a number of seconds to cache the permissions. So the browser won’t have to send a preflight for subsequent requests that satisfy given permissions.

Let’s see how it works step-by-step on the example of a cross-origin PATCH request (this method is often used to update data):

let response = await fetch('https://site.com/service.json', {
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/json',
    'API-Key': 'secret'
  }
});

There are three reasons why the request is unsafe (one is enough):

Method PATCH
Content-Type is not one of: application/x-www-form-urlencoded, multipart/form-data, text/plain.
“Unsafe” API-Key header.
Step 1 (preflight request)
Prior to sending such a request, the browser, on its own, sends a preflight request that looks like this:

OPTIONS /service.json
Host: site.com
Origin: https://javascript.info
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: Content-Type,API-Key
Method: OPTIONS.
The path – exactly the same as the main request: /service.json.
Cross-origin special headers:
Origin – the source origin.
Access-Control-Request-Method – requested method.
Access-Control-Request-Headers – a comma-separated list of “unsafe” headers.
Step 2 (preflight response)
The server should respond with status 200 and the headers:

Access-Control-Allow-Origin: https://javascript.info
Access-Control-Allow-Methods: PATCH
Access-Control-Allow-Headers: Content-Type,API-Key.
That allows future communication, otherwise an error is triggered.

If the server expects other methods and headers in the future, it makes sense to allow them in advance by adding them to the list.

For example, this response also allows PUT, DELETE and additional headers:

200 OK
Access-Control-Allow-Origin: https://javascript.info
Access-Control-Allow-Methods: PUT,PATCH,DELETE
Access-Control-Allow-Headers: API-Key,Content-Type,If-Modified-Since,Cache-Control
Access-Control-Max-Age: 86400
Now the browser can see that PATCH is in Access-Control-Allow-Methods and Content-Type,API-Key are in the list Access-Control-Allow-Headers, so it sends out the main request.

If there’s the header Access-Control-Max-Age with a number of seconds, then the preflight permissions are cached for the given time. The response above will be cached for 86400 seconds (one day). Within this timeframe, subsequent requests will not cause a preflight. Assuming that they fit the cached allowances, they will be sent directly.

Step 3 (actual request)
When the preflight is successful, the browser now makes the main request. The process here is the same as for safe requests.

The main request has the Origin header (because it’s cross-origin):

PATCH /service.json
Host: site.com
Content-Type: application/json
API-Key: secret
Origin: https://javascript.info
Step 4 (actual response)
The server should not forget to add Access-Control-Allow-Origin to the main response. A successful preflight does not relieve from that:

Access-Control-Allow-Origin: https://javascript.info
Then JavaScript is able to read the main server response.

Please note:
Preflight request occurs “behind the scenes”, it’s invisible to JavaScript.

JavaScript only gets the response to the main request or an error if there’s no server permission.

Credentials
A cross-origin request initiated by JavaScript code by default does not bring any credentials (cookies or HTTP authentication).

That’s uncommon for HTTP-requests. Usually, a request to http://site.com is accompanied by all cookies from that domain. Cross-origin requests made by JavaScript methods on the other hand are an exception.

For example, fetch('http://another.com') does not send any cookies, even those (!) that belong to another.com domain.

Why?

That’s because a request with credentials is much more powerful than without them. If allowed, it grants JavaScript the full power to act on behalf of the user and access sensitive information using their credentials.

Does the server really trust the script that much? Then it must explicitly allow requests with credentials with an additional header.

To send credentials in fetch, we need to add the option credentials: "include", like this:

fetch('http://another.com', {
  credentials: "include"
});
Now fetch sends cookies originating from another.com with request to that site.

If the server agrees to accept the request with credentials, it should add a header Access-Control-Allow-Credentials: true to the response, in addition to Access-Control-Allow-Origin.

For example:

200 OK
Access-Control-Allow-Origin: https://javascript.info
Access-Control-Allow-Credentials: true
Please note: Access-Control-Allow-Origin is prohibited from using a star * for requests with credentials. Like shown above, it must provide the exact origin there. That’s an additional safety measure, to ensure that the server really knows who it trusts to make such requests.



#---------------------------------
FormData-----------------------------------------------------------------------------------------------------------------
This chapter is about sending HTML forms: with or without files, with additional fields and so on.

FormData objects can help with that. As you might have guessed, it’s the object to represent HTML form data.

The constructor is:

let formData = new FormData([form]);
If HTML form element is provided, it automatically captures its fields.

The special thing about FormData is that network methods, such as fetch, can accept a FormData object as a body. It’s encoded and sent out with Content-Type: multipart/form-data.

From the server point of view, that looks like a usual form submission.

Sending a simple form
Let’s send a simple form first.

As you can see, that’s almost one-liner:

<form id="formElem">
  <input type="text" name="name" value="John">
  <input type="text" name="surname" value="Smith">
  <input type="submit">
</form>

<script>
  formElem.onsubmit = async (e) => {
    e.preventDefault();

    let response = await fetch('/article/formdata/post/user', {
      method: 'POST',
      body: new FormData(formElem)
    });

    let result = await response.json();

    alert(result.message);
  };
</script>

In this example, the server code is not presented, as it’s beyond our scope. The server accepts the POST request and replies “User saved”.

FormData Methods-----------------------------------------------------------------------------------------------------
We can modify fields in FormData with methods:

formData.append(name, value) – add a form field with the given name and value,
formData.append(name, blob, fileName) – add a field as if it were <input type="file">, the third argument fileName sets file name (not form field name), as it were a name of the file in user’s filesystem,
formData.delete(name) – remove the field with the given name,
formData.get(name) – get the value of the field with the given name,
formData.has(name) – if there exists a field with the given name, returns true, otherwise false
A form is technically allowed to have many fields with the same name, so multiple calls to append add more same-named fields.

There’s also method set, with the same syntax as append. The difference is that .set removes all fields with the given name, and then appends a new field. So it makes sure there’s only one field with such name, the rest is just like append:

formData.set(name, value),
formData.set(name, blob, fileName).
Also we can iterate over formData fields using for..of loop:

let formData = new FormData();
formData.append('key1', 'value1');
formData.append('key2', 'value2');

// List key/value pairs
for(let [name, value] of formData) {
  alert(`${name} = ${value}`); // key1 = value1, then key2 = value2
}
Sending a form with a file------------------------------------------------------------------------------------------------------
The form is always sent as Content-Type: multipart/form-data, this encoding allows to send files. So, <input type="file"> fields are sent also, similar to a usual form submission.

Here’s an example with such form:

<form id="formElem">
  <input type="text" name="firstName" value="John">
  Picture: <input type="file" name="picture" accept="image/*">
  <input type="submit">
</form>

<script>
  formElem.onsubmit = async (e) => {
    e.preventDefault();

    let response = await fetch('/article/formdata/post/user-avatar', {
      method: 'POST',
      body: new FormData(formElem)
    });

    let result = await response.json();

    alert(result.message);
  };
</script>

Sending a form with Blob data
As we’ve seen in the chapter Fetch, it’s easy to send dynamically generated binary data e.g. an image, as Blob. We can supply it directly as fetch parameter body.

In practice though, it’s often convenient to send an image not separately, but as a part of the form, with additional fields, such as “name” and other metadata.

Also, servers are usually more suited to accept multipart-encoded forms, rather than raw binary data.

This example submits an image from <canvas>, along with some other fields, as a form, using FormData:

<body style="margin:0">
  <canvas id="canvasElem" width="100" height="80" style="border:1px solid"></canvas>

  <input type="button" value="Submit" onclick="submit()">

  <script>
    canvasElem.onmousemove = function(e) {
      let ctx = canvasElem.getContext('2d');
      ctx.lineTo(e.clientX, e.clientY);
      ctx.stroke();
    };

    async function submit() {
      let imageBlob = await new Promise(resolve => canvasElem.toBlob(resolve, 'image/png'));

      let formData = new FormData();
      formData.append("firstName", "John");
      formData.append("image", imageBlob, "image.png");

      let response = await fetch('/article/formdata/post/image-form', {
        method: 'POST',
        body: formData
      });
      let result = await response.json();
      alert(result.message);
    }

  </script>
</body>

Please note how the image Blob is added:

formData.append("image", imageBlob, "image.png");
That’s same as if there were <input type="file" name="image"> in the form, and the visitor submitted a file named "image.png" (3rd argument) with the data imageBlob (2nd argument) from their filesystem.

The server reads form data and the file, as if it were a regular form submission.

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------


# Week 3: Intro to Node JS  Day 1 – Introduction to Node JS  What is Node JS? 

Google chrome's V8 – JavaScript engine  

	Listen to network traffic  
	Access files on your local 
 	Listen to http 
	Send back files 
	Access databases 
 
Node.js is an open-source, cross-platform JavaScript runtime environment that allows developers to execute JavaScript code outside of a web browser. 
It is built on the V8 JavaScript engine developed by Google and is designed to be efficient and lightweight, making it well-suited for building scalable and high-performance network applications. 

Key features and characteristics of Node.js include: 

Non-blocking I/O: Node.js is designed around an event-driven, non-blocking I/O model, which means that it can handle a large number of simultaneous connections and perform tasks asynchronously. This makes it particularly suitable for building real-time applications like chat applications or online gaming. 

Server-side scripting: Node.js is often used for server-side scripting, enabling developers to build web servers and APIs using JavaScript. This allows for a consistent language and codebase between the server and the client, which can simplify development and maintenance. 

Package ecosystem: Node.js has a vibrant ecosystem of open-source packages and modules available through the Node Package Manager (NPM). NPM makes it easy to manage dependencies and integrate third-party libraries into your projects. 

Community and support: Node.js has a large and active community of developers, which means there is a wealth of documentation, tutorials, and support available for those working with Node.js. 

Cross-platform: Node.js is compatible with various operating systems, including Windows, macOS, and various Linux distributions, making it a versatile choice for building applications that can run on different platforms. 

Node.js is commonly used for building web servers, APIs, real-time applications, and microservices. It has gained popularity in recent years due to its performance, scalability, and the ability to use JavaScript for both client-side and server-side development, making it a powerful tool for full-stack developers. 

# Code for creating node.js sever 


const http = require('http'); 
const hostname = '127.0.0.1'; 
const port = 3000; 

const server = http.createServer((req, res) => { 

  res.statusCode = 200; 
  res.setHeader('Content-Type', 'text/plain'); 
  res.end('Hello World\n'); 
}); 

server.listen(port, hostname, () => { 
  console.log(`Server running at http://${hostname}:${port}/`); 
}); 

 

## Answers to Activity 3 

What is Node JS? 

=Node.js is an open source server environment 

How is Node JS initiated on a computer? 

=Through the command line interface 

Why do we use Node JS? 

=Node JS is asynchronous 

What can Node JS do? 

=Node JS can send dynamic content  
Node JS contains some tasks that can be executed on certain events eg someone trying to access a port on the server 

What is a module in Node JS the same as in JavaScript? 

=Libraries.  

What is NPM? 

=Node JS Package Manager 

What is contained  in a Node JS Package? 

=A package in Node.js contains all the files you need for a module 


