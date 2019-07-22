---
layout: post
title:  "Building a Website With Node.js, Express, and Pug"
date:   2019-07-19 18:19:25 +0800
permalink: "/building-a-website-with-nodejs-express-and-pug"
sitemap: false
description: An example (in Java) of asymmetric encryption/decryption using ECC key pairs and the ECDH key agreement protocol.
tags: node express pug
# permalink: /:categories/:year/:month
---

In this project, we will be building a static grocery list website using Node.js, Express, and Pug.

The first step is to [download](https://nodejs.org/en/download/) and install Node.js. Node.js is an asynchronous, event-driven JavaScript runtime used to build scalable network applications. We will be using Node.js as the 'server' from which we serve our static website.

Once you have installed Node.js, open up the terminal. Verify that Node.js has been installed by running `node -v` in the terminal. If Node.js has been installed correctly, you should see a version number show up on the terminal. For this project I will be using Node.js version `12.4.0`.

Create a new project folder and navigate to that folder by running the following commands in the terminal:

```
mkdir myProject
cd myProject
```

Now that you've navigated to your project folder, create a new Node.js project by running the following command in the terminal:

```
npm init
```

The npm command-line utility should now walk you through creating a new `package.json` file. You may choose to edit each prompt as it appears on your screen, or choose to skip a prompt by pressing the return key. After going through all the prompts, a `package.json` file should be created in your project folder.

At this point, there should only be one file in your project folder -- the `package.json` file. Now we need to install Express.js. Express.js

TODO

Next, we need to install Express.js as a dependency. Express.js is a web application framework for Node.js and can be used to listen for any input/connection requests from clients. Install Express.js by running `npm install express` in your terminal. Once Express.js has been installed correctly, it should now appear in your `package.json` file, under the `dependencies` field.

```json
"dependencies": {
	"express": "^4.17.1"
}
```

Once we begin developing our web application, changes we make to our application will remain hidden until we restart the server. To support faster development we can install nodemon as a dependency, which is a tool that automatically restarts the Node.js server when file changes in the project folder are detected. Install nodemon by running `npm install --save-dev nodemon` in your terminal. Once nodemon has been installed correctly, it should now appear in your `package.json` file, under the `devDependencies` field.
```json
"devDependencies": {
	"nodemon": "^1.19.1"
}
```

With Express.js installed, we can now start our web application for the first time. In your project folder, create a new file called `server.js`. In `server.js`, add the following code:

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
	res.send('Hello World!');
});

const PORT = 8080;
app.listen(PORT, () => {
	console.log(`App is listening on Port ${PORT}!`);
});
```
In the terminal, run `node server.js`. If everything is working properly, you should see a message in the terminal with the text `App is listening on Port 8080!` This means our server is now running and if we navigate to `localhost:8080` in our web browser, we should see the text `Hello World!` come up on the screen!

Let's break down the code in `server.js`.

- First, we need to import the Express.js module that we installed earlier on. We do this with the `require()` function on Line 1.
- Afterwards, we need to initialize the Express.js module, and we do this by calling the function `express()` and putting the Express.js application inside the `app` variable.
- The `app.listen()` function at the bottom of `server.js` tells the Express.js application to listen for connections on the specified port. In this case, we have set our port to be `8080`.
- The `app.get()` function in the middle of `server.js` tells the Express.js application to listen for `GET` requests at the `/` address, which is simply `localhost:8080`. Upon getting a request, the Express.js application sends back a response via `res.send()`. In this case, we are simply sending back some text, `Hello World!`

You can terminate the server connection by pressing `control` + `c` in the terminal.

Now let's add some code that will allow us to use the nodemon module that we installed earlier. Open up the `package.json` file and edit the `scripts` field to look like this:

```javascript
"scripts": {
	"start": "nodemon server.js"
}
```

Now we can start our server using `npm start` (instead of `node server.js`). Additionally, nodemon should automatically run whenever we start our server.

Admittedly, there isn't much to our web application at the moment. It would be nice to be able to display a more complex web page when we navigate to `localhost:8080` instead of just having `Hello World!`. We can do this by installing a template engine on our Node.js service. Template engines allow us to use static template files in our application. At runtime, the template engine replaces variables in a template file with actual values, and transforms the template into a HTML file sent to the client.

We will be using Pug as our template engine. To install pug, run `npm install pug --save` from your project folder. You can verify that Pug has been installed correctly by checking the `dependencies` field in the `package.json` file.

With Pug installed, the first thing we need to do is to import the module in our Node.js service. To do this, add the following lines of code after `const app = express()`:

```javascript
const path = require('path');
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'pug');
```

The code above tells the Express.js application to look in the `views` folder for template files and to use Pug as the template engine.

Before we can tell our Node.js service to render these template files, we need to first create a template file we can use. If you haven't done so already, create `views` folder your project folder. Inside the `views` folder, create a new file called `index.pug` with the following contents:

```
html
	head
		title My Website
	body
		h2 Hello World!
		p by Dave
```

You might observe that the format of a Pug file is very similar to that of a regular HTML file. The difference is that there are no closing tags in Pug and, as a result, whitespace is extremely important in Pug. You can read more about Pug syntax [here](https://pugjs.org/language/attributes.html).

Even without seeing the web page that this Pug file renders to, we can already guess that the web page will look like. To get our Node.js service to actually serve this Pug file, all we need to do is to make a small edit to our `app.get()` function in our `server.js` file to look like this:

```javascript
app.get('/', (req, res) => {
	res.render('index.pug');
});
```

Now, when a user navigates to `localhost:8080`, instead of sending them a String, we now send them the rendered `index.pug` file.

#### Further Reading

Armed with Node.js, Express.js, and Pug, we can now create any kind of static website imaginable. Anything that we can do in a regular HTML file -- we can also do in Pug. Here are a two useful tips when working with Pug.

```
//- This is how you add comments!
//- You can add attributes to tags like so:

div(class='my-div')
	h1(style='color: blue' id='my-div-header') I am Blue!
	p(style='color: red') I am Red!
```

```
//- You can import CSS like this:

html
	head
		link(
			rel='stylesheet'
			href='my-styles.css'
		)
		link(
			rel='stylesheet'
			href='https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css'
			crossorigin='anonymous'
		)
	body
```

Once you've familiarized yourself with Pug and created a few other web pages, you might want to serve these web pages along with `index.pug`. To do this, we simply need to add more `app.get()` functions pointing to different addresses. For instance, if we wanted to serve the file `about.pug` at `localhost:8080/about`, we would add this block of code to `server.js`:

```javascript
app.get('/about', (req, res) => {
	res.render('about.pug');
});
```

Now, when a user navigates to `localhost:8080/about`, our Express.js application will send them the rendered `about.pug` file.
