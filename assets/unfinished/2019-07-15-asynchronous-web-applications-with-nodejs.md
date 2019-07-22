---
layout: post
title:  "Asynchronous Web Applications with Node.js"
date:   2019-07-15 18:19:25 +0800
permalink: "/async-nodejs"
sitemap: false
description: Building asynchronous web applications with Node.js, MySQL, jQuery, and Ajax.
tags: nodejs ajax jquery mysql
# permalink: /:categories/:year/:month
---

In this project, we will be building an asychronous web application using Node.js, Express.js, Pug, jQuery, Ajax, and MySQL. The project will be split into three sections: in the first section, we build a static website using Node.js, Express.js, and Pug; in the second section, we add asynchronous, dynamic functionality to the website using jQuery and Ajax; and in the third and last section, we allow the website to read and write data asynchronously to a MySQL database.

**Table of Contents**

1. [Creating a Website using Node, Express, and Pug](#creating-a-website-using-node-express-and-pug)
2. [Adding Asynchronous Dynamic Functionality with jQuery and Ajax](#adding-asynchronous-dynamic-functionality-with-jquery-and-ajax)
3. [Connecting MySQL to a Dynamic Node Website](#connecting-mysql-to-a-dynamic-node-website)



## Creating a Website Using Node, Express, and Pug

In this project, we will be using Node.js to serve our web application. If you don't already have Node.js installed, you can download the installer [here](https://nodejs.org/en/download/). Once you have installed Node.js, we are now ready to begin creating our web application.

First, create a new project folder (the name doesn't matter) and navigate to the folder in your terminal. In your terminal, run `npm init`. The npm command-line utility should now walk you through creating a new `package.json` file. You may choose to edit each prompt or skip a prompt by pressing the return key. After the utility has covered each prompt, a `package.json` file should be created in your project folder.

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

## Adding Asynchronous Dynamic Functionality with jQuery and Ajax

With jQuery, we can add dynamic elements to our website. For instance, let's add a button to our website that, when pressed, logs a message to our console.

To start, we need to import jQuery into our website. There are multiple ways of doing this, but the easiest way is to simply include the jQuery CDN in our `index.pug` file. Let's also add a button below our `p` tag.

```
html
	head
		title My Website
	body
		h2 Hello World!
		p by Dave
		button(id='my-button' type='button') Press Me
		script(
            src='https://code.jquery.com/jquery-3.4.1.js'
            integrity='sha256-WpOohJOqMqqyKL9FccASB9O0KwACQJpFTUBLTYOVvVU='
            crossorigin='anonymous'
		)
```

Observe that including the jQuery in our `index.pug` file is no different from how we would include it in a normal HTML file. Now that jQuery is included in our website, let's add some JavaScript that detects when the button is pressed -- add the following block of code below the code that includes the jQuery CDN.

```javascript
script.
	$(document).ready(() => {
		$('#my-button').click(() => {
			console.log('Pressed!');
		});
	});
```

Make sure that `script.` lines up vertically with the `script()` used to include the jQuery CDN; whitespace is important in Pug!

The code above tells jQuery to add a `click` event handler to whichever HTML element has an `id` of `my-button`. In our case, the button has this `id`. Whenever the button is clicked, the event handler function logs the message `Pressed!` to the console. Enclosing the event handler in `$(document).ready()` tells jQuery to wait for all the elements in the DOM to finish loading before attaching the event handler.

> To view the client-side console, right click on the website (in the browser) and click on `Inspect`. Then, in the top-right corner of the screen, switch from the `elements` tab to the `console` tab.

Success! Our website is now pseudo-dynamic. Now, let's add some code that will create additional visible HTML elements on every click. First, we need to define an area in the Pug file where these additional elements should go. The best way to do this is to use a `div`. Add this code below (and inline with) the `button` tag, but *before* the `script.` tag:

```
	div(id='add-elements-here')
```

Now, let's edit our event handler so that we add a line of text inside this `div` whenever the button is pressed:

```javascript

```

TODO jquery go to server side using AJAX
	but not that useful because data resets every time server resets


### Working with MySQL

The next step in building our web application is to connect it to a database. There are many different kinds of databases and database services we can choose from, but in this project we will use the popular MySQL relational database management system.

First, we need to download and install both [MySQL Server](https://dev.mysql.com/downloads/mysql/) and [MySQL Workbench](https://dev.mysql.com/downloads/workbench/). You should be able to start, stop, and configure your MySQL installation through System Preferences. Once the MySQL Server is running on your local machine, start up MySQL Workbench and create a new Schema called `main` with the `utf8` and `utf8-bin` options selected.

> **Note**: if this is your first time setting up MySQL, you may be prompted to create a new root user account.

Once your Schema has been created, double-click on your Schema on the left-hand pane to use the Schema. Next, right-click on the `Tables` tab in the left-hand pane and click on `Create Table`.  Name the table `post`, and add three columns to the table.

- The name of the first column should be `id`. Make sure to check the `PK`, `NN`, and `AI` boxes for this particular column.
- The se


First, we need to install the MySQL module in our Node.js service. Do this by running `npm install mysql` from your project folder. Again, if the module has been installed correctly, you should be able to see it in the `package.json` file, under the `dependencies` field.


npm install body-parser
npm install pug --save
npm install mysql
in mysql workbench, execute ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'.

npm install mysql-events
var MySQLEvents = require('mysql-events');
