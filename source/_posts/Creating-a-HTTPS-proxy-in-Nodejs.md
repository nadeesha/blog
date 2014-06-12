title: 'Creating a HTTPS proxy in Node.js'
date: 2013-10-07 14:35:15
tags: 
- node.js 
- devops
---
Recently, I needed to mock a dev box to act as the production environment to do some debugging on a Node.js express web app. The problem was that the calls that are received by the said dev box are all HTTPS, (hence received on port 443) and in the local environment, Node is not configured to run on HTTPS, and not on port 80 or 443.

Quick and Dirty solution is to, alter the Node server to start on port 443, after giving a fake HTTPS key and a certificate.

```Javascript
function HttpServer(handlers) {
  // this is a fake key and a certificate
  var options = {
    key: fs.readFileSync('cert/key.pem'),
    cert: fs.readFileSync('cert/key-cert.pem')
  }

  this.handlers = handlers;
  this.server = https.createServer(options, this.handleRequest_.bind(this));
}

HttpServer.prototype.start = function(port) {
  this.port = port;
  this.server.listen(443, 'example.com'); // 443 because of HTTPS
  util.puts('Server started successfully...');
};
```

and of course, doing this would require you start node as `su`, thereby rendering your already set environment variable useless - if you're into that kind of a thing for setting up your `NODE_ENV` and `PORT`.

```bash
sudo node app.js
```

I honestly, can't live with this approach, because we're doing modifications to the app code and running Node on sudo, which is well.. ugly.

**Doing it the right way**
What we really need is a middleman to listen to 443 as a HTTPS proxy and redirect that traffic to a blissfully oblivious Node server running on 8080 or whatever, with no HTTPS configuration or whatever. To accomplish this, we'll use Nodejitsu's excellent [node-http-proxy](https://github.com/nodejitsu/node-http-proxy).

First, we'll need a HTTPS key and a certificate. You can create a [self-signed HTTPS certificate](https://devcenter.heroku.com/articles/ssl-certificate-self) and use it for this purpose.

```Javascript
var options = {
  https: {
    key: fs.readFileSync('key.pem'),
    cert: fs.readFileSync('key-cert.pem')
  }
};
```

Then we'll need to have the target server ready. My Node.js app runs on `localhost:8080`, so this will do:

```Javascript
var proxy = new httpProxy.HttpProxy({
  target: {
    host: '127.0.0.1',
    port: 8080
  }
});
```

Finally of course, we'll create the proxy server and put everything together.

```Javascript
https.createServer(options.https, function(req, res) {
    console.log('Proxying https request at %s', new Date());
    proxy.proxyRequest(req, res);
  }).listen(443, function(err) {
    if (err)
      console.log('Error creating https proxy ');
	console.log('Created https proxy. Forwarding requests from %s to %s:%s', '443', proxy.target.host, proxy.target.port);
});
```

Of course, you can `console.log` the `req` object for enhanced debugging. Since we're running the node on 443, we will need to start the proxy with sudo. Otherwise, you'll encounter an `EACCESS` error.

```bash
sudo node proxy.js
```

I have created a gist of this example that supports both HTTP and HTTPS [right here](https://gist.github.com/nadeeshacabral/6863947).