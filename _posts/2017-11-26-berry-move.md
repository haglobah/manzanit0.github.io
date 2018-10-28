---
layout: post
title: "A first attempt to fight Salesforce deployments"
author: Javier Garcia
description: "A quick sketch of an application I want to develop to ease Salesforce deployments."
category: salesforce
apprenticeship: false
tags: salesforce, nodejs, automation, deployment
---

In this article and some to follow I'm going to cover the design and development of Berry-Move. A **small application to ease the deployment of metadata between Salesforce instances**.

### Change sets are horrible!

Whoever has been working with Salesforce for quite some time knows that deploying changes between sandboxes or the production environment is incredibly exhausting with change sets... So after a year or so trying different solutions in the market (Ant Migration tool, Copado, jsforce npm package...) I decided to make my own tool.

What I decided is to take one of the tools that I had tried earlier, `js-metadata-tools`, which is an extension of the famous `jsforce`, wrap it in an Express server and give it a simple yet powerfull interface to make deployments a breeze.

### The approach

After thinking for a while, I decided to:

* Make a very simple API that just calls the `js-metadata-tools` API. No complications.
* Give it a Salesforce-friendly interface by using `Lightning Design System`.
* Go for a SPA approach using `Vue.js`.

My idea basically revolved around having an absurdly simple interface that allowed us to connect to two orgs, origin and destination, load all the metadata, select the files we wanted to deploy and _bam_! Deploy it. Again, no complicated stuff.

Today I'll just talk about the creation of application's initial backend which covers the creation of the Express server, the elaboration of the endpoints which make use of `jsforce` and the architecture chosen for it.

### Starting with the Express server

As a starting point, I just took the Express snippet from the official docs:

```javascript
var express = require('express')
var app = express()

app.get('/', function (req, res) {
  res.send('hello world')
})
```

Now, if you've ever worked with Express you'll know that it's very easy and quick to spin up an application with some basic functionality but, if we don't organize the code well, it's also very easy and quick for it to get messy. So, I started to break stuff down by creating separated files for the routes, the endpoints and the "_business logic_".

When the application isn't going to be very big I usually go for the "_endpoints-helpers_" architecture. I create a directory for my endpoints and another one for some helper files which will conglomerate the business logic. In this case, we'll have a `metadata.endpoint.js` and a `metadata.helper.js` file.

Furthermore, I also like having a `routes.js` file which contains all the different routes to my endpoints. So, let's say tomorrow I need a new endpoint for users, then I'll create a `user.endpoint.js` with a `user.helper.js` and reference the endpoint in our `routes.js` file.

As I've already said... don't extrapolate this architecture for all your apps. If you're goint to build something complex, it might end up being better to go for an [onion](https://solidgeargroup.com/clean-architecture-in-nodejs) one or any of the more scalable ones.

Let's see the code:

###### app.js
```javascript
const express = require('express'),
    path = require('path'),
    bodyParser = require('body-parser'); //Parse JSON in body request

// Express configuration.
const app = express();
app.use(bodyParser.json());

// Routes configuration.
require('./endpoints/routes.js')(app);

 // Init app.
app.listen(3000, () => console.log('Berry-Move listening on port ' + 3000 + '!\n'));

module.exports = { app };
```

###### router.js
```javascript
const app = require('express');

const metadata = require('./metadata.endpoint.js');

const routes = (app) => {
    app.use('/metadata', metadata);
};

module.exports = routes;
```

###### metadata.endpoint.js
```javascript
const express = require('express'),
    router = express.Router();

const retrieve = (req, res) => {
    // TODO
}

const deploy = (req, res) => {
    // TODO
}

router.post('/retrieve', retrieve);
router.post('/deploy', deploy);

module.exports = router;
```

This is aproximately the backbone of the application. In a nutshell:

* We will have an `app.js` file which will contain the Express code to start the server.
* A `routes.js` file which will break down all the routes of the endpoints, at the moment one, but maybe more in the future.
* an `endpoints` folder with a single file for each set of functionality endpoint.
* an `helpers` folder with a file for each module of logic.

### Elaborating the metadata endpoint

Taking into account how the Salesforce Metadata API is designed, the best way to implement a complete migration endpoint is to create a retrieval endpoint, a deployment endpoint and then a third one that uses the other two. For this we'll need the credentials for origin and destination instances, of course.

The approach I decided to take was to make the retrieval method download the metadata to a local path given a string which contains the names as stated in the API. Then, the make the deployment method simply pickup the zip file downloaded by the previous and deploy it to the destination org.

Regarding the authentication, I addressed it in the easiest way I could, that is, read the user and password from the request header, copy it to an `options` object which `jsforce` admits and just pass it as a parameter.

This is how the helper would look:

````javascript
'use strict';

const { promisify } = require('util');
const fs = require('fs');
const readFile = promisify(fs.readFile);
const writeFile = promisify(fs.writeFile);
const unzip = require('unzip');

const jsforce = require('jsforce-metadata-tools');

const retrieve = async (path, metadata, options) => {
    const result = await jsforce.retrieveByTypes(metadata, options); 
    writeFile(path, new Buffer(result.zipFile, 'base64'));
    return result;
};

const deploy = async (path, options) => {
    const data = await readFile(path);
    return jsforce.deployFromZipStream(data, options);
};

module.exports = { deploy, retrieve };
````

* The `metadata` variable contains the name in a pattern like this: `ApexClass:*;ApexPages:page1,page2;`, just like the API admits.
* The `path` variable is were we'll place the zip file with all the retrieved metadata.

Now, making use of that code, the implementation of the endpoint would be like this:

###### retrieve method
``` javascript
const retrieve = (req, res) => {
    const options = {
        username: req.body.username,
        password: req.body.password
    };

    metadataHelper.retrieve('./package.zip', res.body.metadata, options)
        .then(result => res.status(200).send(result))
        .catch(err => res.status(500).send(err));
};
```

###### deploy method
``` javascript
const deploy = (req, res) => {
    const options = {
        username: req.body.username,
        password: req.body.password
    };

    metadataHelper.deploy('./package.zip', options)
        .then(result => res.status(200).send(result))
        .catch(err => res.status(500).send(err));
};
```

With all this, we have most of our backend logic finished! To tie it up with the frontend we will need another endpoint to log into the Salesforce instances, but apart from that - this is all the magic.

### Executing our app!
If we want to try and deploy some Apex code, we'll just call `npm start` from the console and call the endpoints just like this:

1. http://localhost:3000/metadata/retrieve

###### body of the request
````
{
	"username": "<username>",
	"password": "<password+securitytoken>",
	"loginUrl": "http://login.salesforce.com",
	"metadata": "ApexClass:AccountTriggerHandler, EmailService"
}
````

2. http://localhost:3000/metadata/deploy

### Github Repository
If you want to see the full code covered in this article, checkout the [`chapter1`](https://github.com/Manzanit0/berry-move/tree/blog-chapter1) branch. The latest version of the app will be in [`master`](https://github.com/Manzanit0/berry-move).

Happy coding! :)

