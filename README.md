node-discover
=============

Automatic and decentralized discovery and monitoring of nodejs instances with built in support for a variable number of master processes, service advertising and channel messaging.

Version 0.1.0

Probably has bugs.

Why?
====

So, you have a whole bunch of node processes running but you have no way within each process to determine where the other processes are or what they can do. This module aims to make discovery of new processes as simple as possible. Additionally, what if you want one process to be in charge of a cluster of processes? This module also has automatic master process selection.

Compatibility
=============

This module uses broadcast and multicast features from node's dgram module. All required features of the dgram module are implemented in the following versions of node

	- v0.4.x
	- v0.6.9+

Example
=======

Be sure to look in the examples folder, especially at the [distributed event emitter](https://github.com/skeggse/node-discover/blob/master/examples/deventemitter.js)

```js
var Discover = require('node-discover');

var d = new Discover();

d.on("promotion", function () {
	/**
	 * Launch things this master process should do.
	 * 
	 * For example:
	 *	- Monitior your redis servers and handle failover by issuing slaveof
	 *    commands then notify other node instances to use the new master
	 *	- Make sure there are a certain number of nodes in the cluster and 
	 *    launch new ones if there are not enough
	 *	- whatever
	 */
	console.log("I was promoted to a master.");
});

d.on("demotion", function () {
	/**
	 * End all master specific functions or whatever you might like.
	 */
	console.log("I was demoted from being a master.");
});

d.on("added", function (obj) {
	console.log("A new node has been added.");
});

d.on("removed", function (obj) {
	console.log("A node has been removed.");
});

d.on("master", function (obj) {
	/**
	 * A new master process has been selected
	 * 
	 * Things we might want to do:
	 * 	- Review what the new master is advertising use its services
	 *	- Kill all connections to the old master
	 */
	console.log("A new master is in control");
});
```

Installing
==========

### npm

	npm install node-discover

### git

	git clone git://github.com/wankdanker/node-discover.git


API
===

Constructor
-----------

```js
new Discover({
	helloInterval: 1000, // How often to broadcast a hello packet in milliseconds
	checkInterval: 2000, // How often to to check for missing nodes in milliseconds
	nodeTimeout: 2000, // Consider a node dead if not seen in this many milliseconds
	masterTimeout: 2000, // Consider a master node dead if not seen in this many milliseconds
	address: '0.0.0.0', // Address to bind to
	port: 12345, // Port on which to bind and communicate with other node-discover processes
	broadcast: '255.255.255.255', // Broadcast address if using broadcast
	multicast: null, // Multicast address if using multicast (don't use multicast, use broadcast)
	mulitcastTTL: 1, // Multicast TTL for when using multicast
	key: null, // Encryption key if your broadcast packets should be encrypted (null means no encryption)
	mastersRequired: 1, // The count of master processes that should always be available
	weight: Math.random() // A number used to determine the preference for a specific process to become master. Higher numbers win.
});
```

Attributes
----------

* nodes

Methods
-------

### promote()

Promote the instance to master.

This causes the old master to demote.

```js
var Discover = require('node-discover');
var d = new Discover();

d.promote();
```

### demote(permanent=false)

Demote the instance from being a master. Optionally pass true to demote to specify that this node should not automatically become master again.

This causes another node to become master

```js
var Discover = require('node-discover');
var d = new Discover();

// different usages
d.demote(); // this node is still eligible to become a master node.
d.demote(true); // this node is no longer eligible to become a master node.
```

### join(channel, messageCallback)

Join a channel on which to receive messages/objects

```js
var Discover = require('node-discover');
var d = new Discover();

// pass the channel and the callback function for handling received data from that channel
var success = d.join("config-updates", function (data) {
	if (data.redisMaster) {
		// connect to the new redis master
	}
});

if (!success) {
	// could not join that channel; probably because it is reserved
}
```
	
#### Reserved channels

* promotion
* demotion
* added
* removed
* master
* hello

### leave(channel)

Leave a channel

```js
var Discover = require('node-discover');
var d = new Discover();

// pass the channel which we want to leave
var success = d.leave("config-updates");

if (!success) {
	// could leave channel; who cares?
}
```

### send(channel, objectToSend)

Send a message/object on a specific channel

```js
var Discover = require('node-discover');
var d = new Discover();

var success = d.send("config-updates", {redisMaster : "10.0.1.4"});

if (!succes) {
	// could not send on that channel; probably because it is reserved
}
```

### advertise(objectToAdvertise)

Advertise an object or message with each hello packet; this is completely arbitrary. make this object/message whatever you applies to your application that you want your nodes to know about the other nodes.

```js
var Discover = require('node-discover');
var d = new Discover();

// any of these invocations
d.advertise({
	localServices : [
		{type: 'http', port: '9911', description: 'my awesome http server'},
		{type: 'smtp', port: '25', description: 'smtp server'}
	]
});

d.advertise("i love nodejs");

d.advertise({something: "something"});
```
		
### start()

Start broadcasting hello packets and checking for missing nodes (start is called automatically in the constructor)

```js
var Discover = require('node-discover');
var d = new Discover();

d.start();
```

### stop()

Stop broadcasting hello packets and checking for missing nodes

```js
var Discover = require('node-discover');
var d = new Discover();

d.stop();
```

### eachNode(fn) 

For each node execute fn, passing fn the node fn(node)

```js
var Discover = require('node-discover');
var d = new Discover();

d.eachNode(function (node) {
	if (node.advertisement === "i love nodejs") {
		console.log("nodejs loves this node too");
	}
});
```

Events
------

Each event is passed the `Node Object` for which the event is occuring.


### promotion

Triggered when the node has been promoted to a master.

* Could happen by calling the promote() method
* Could happen by the current master instance being demoted and this instance automatically being promoted
* Could happen by the current master instance dying and this instance automatically being promoted

### demotion

Triggered when the node is no longer a master.

* Could happen by calling the demote() method
* Could happen by another node promoting itself to master

### added

Triggered when a new node is discovered.

### removed

Triggered when a new node is not heard from within `nodeTimeout`.

### master

Triggered when a new master has been selected.


Node Object
-----------

```js
{ 
	isMaster: true,
	isMasterEligible: true,
	advertisement: null,
	lastSeen: 1317323922551,
	address: '10.0.0.1',
	port: 12345,
	id: '31d39c91d4dfd7cdaa56738de8240bc4',
	hostName: 'myMachine'
}
```

TODO
====

* Large packets are not tested. The current version does not handle recombining split messages.
* Discover assumes the broadcast address to be `255.255.255.255`.
* Unique ID algorithm not terribly robust.
* Local address assumed to be `127.0.0.1`.
* Missing node check may not be sufficiently optimized.

LICENSE
=======

> (MIT License)

> Copyright &copy; 2011 Dan VerWeire dverweire@gmail.com

> Copyright &copy; 2013 Eli Skeggs skeggse@gmail.com

> Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

> The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.