title: 'Setting up multiple worker processes for your node server'
date: 2013-11-25 15:19:25
tags:
- node.js
- express.js
---
Node runs on a single thread. And many implementations won't need anything more than this, if they just stick to the optimum use case for node - which is handling non-CPU intensive stuff. But suppose you want to push node beyond the default singleton server it is.

Here's how you do it.

If you're creating an express app, you would have these lines somewhere in your startup node file.

```js
var app = express();
app.listen(8080);
```

This starts up your app on port `8080`, as a single threaded server. In order to implement the node app cluster, you will need the cluster package first.

```js
var cluster = require('cluster');
```

When you start up a node app cluster, the master process forks itself to make more instances of itself. So, when your startup js file runs, you will need to, 
1. make sure that it's the master process running, (because the child processes will also run the same startup file) 
2. fork the process accordingly
3. and if it's the child, start up node on that process listening to your port.

In a typical `app.js` this translates to:

```js
if (cluster.isMaster) {
	//  let's make three child processes
	for (var i = 0; i < 3; i++) {
		cluster.fork();
	}
} else {
	app.listen(8080);
}
```

This is the most basic implementation. But we can always improve this:

* Detect the number of `n` cores the OS has access to and fork the master `n` times.
* On child process exiting (an unhandled exception - maybe?), start up another child in it's place.

With that in mind, this would be a slightly improved version.

```js
var	cluster = require('cluster'),
	process = require('process'),
	cpus = require('os').cpus().length;

// express initialization goes here

if (cluster.isMaster) {
	console.log('Detected %s cores', cpus);

	// fork workers.
	for (var i = 0; i < cpus; i++) {
		cluster.fork();
	}

	//  on death, let's start up a new child process
	cluster.on('exit', function(worker, code, signal) {
		console.log('worker %s died on code: %s, signal: %s', 
			worker.process.pid, code, signal);
			
		cluster.fork();
	});
} else {
	var workerId = cluster.worker.id;

	// opening port
	var port = process.env.PORT || 8080;
	app.listen(port);
	console.log('Woker %s running on port %s', workerId, port);
}
```

