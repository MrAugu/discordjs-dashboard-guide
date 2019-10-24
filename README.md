# Intoduction
In this guide we'll be creating a small discord bot dashboard for managing guild settings, nothing too fancy. This is currently being worked on.

**If you want to contribute to this guide open a pull request.**
# Software
We'll be using `mongoose` to interact with our MongoDB database, we are also going to use `express` (+ a little middleware) to set up the web server. To handle logins we'll be using `passport`. As a viewing engine we'll use `ejs` which will convert html + server side scripts into full html pages that can be display by the browser. The tutorial is 99% compatible with both `Stable` and `Dev` version of Discord.Js.

# #1 Setting up the Webserver
We begin by importing a few of the needed modules, express, discord.js (you can get those on npm), path and url respectively (those 2 are native modules, they help us to locate the templates we'll render).
```js
const Discord = require("discord.js");
const express = require("express");
const url = require("url");
const path = require("path");
```
Now we initialize the express application, and bind it to the port 8000.
```js
const app = express(); // You need to place this at the beggining.
app.listen(8000, null, null, () => console.log("The web server is up and running!")); // You need to place this at the really end of your file.
```
Now that our web server is up and running, we can start to configure it, we'll start by locating our template files.
```js
 const filesDirectory = path.resolve(`${process.cwd()}${path.sep}dashboard`);
 const templatesDirectory = path.resolve(`${filesDirectory}${path.sep}templates`);
 ```
 We can begin to implement our passport.
 ```js
 const passport = require("passport"); // The general passport module.
 const passportDiscord = require("passport-discord"); // This will allow us to solve the output from OAuth.
 const Strategy = passportDiscord.Strategy; // This will solve the actual output.
```
In order to handle logins we need to set the `serializeUser` and `deserializeUser` callbacks inside passport.
```js
passport.serializeUser((user, done) => {
    done(null, user);
});

passport.deserializeUser((obj, done) => {
    done(null, obj);
});
```
Now we are going to create a new strategy for passport.
```js
const strategy = new Strategy({
    clientID: "Your bot's user id goes here.",
    clientSecret: "Your client secret goes here.",
    callbackURL: "http://localhost:8000/callback", // The url that will handle callbacks.
    scope: ["identify", "guilds"] // Get tag and profile picture + servers user is in.
  },
  (accessToken, refreshToken, profile, done) => {
    process.nextTick(() => done(null, profile));
});

passport.use(strategy);
```
And we load the passport into the webserver.
```js
  app.use(passport.initialize());
  app.use(passport.session());
```
We are going to set the view engine.
```js
const ejs = require("ejs"); // You can get this from npm as well.
app.engine("html", ejs.renderFile);
app.set("view engine", "html");
```
To handle the forms that use will input data in, we need to use another middleware called `body-parser`.
```js
var bodyParser = require("body-parser");
  app.use(bodyParser.json());
  app.use(bodyParser.urlencoded({
    extended: true
  }));
```
To keep user logged in between pages, we need to use `memorystore` and `express-session` (you can get both from npm).
```js
const session = require("express-session");
const MemoryStore = require("memorystore");
const mStore = MemoryStore(session); // We initialize memorystore with express-session.
```
Now we need to configure them.
```js
app.use(session({
    store: new mStore({ checkPeriod: 86400000 }), // we refresh the store each day
    secret: "A_RANDOM_STRING_FOR_SECURITY_PURPOSES_OH_MY",
    resave: false,
    saveUninitialized: false
}));
```
As our web-server is set up, we can begin to setup the routes that handle logins.

# #2 Login, Callback and Logout Routes
**1.Login Route**

Inside this route a returning URL is being set and user gets redirected to the appropriate discord auth page.
```js
app.get("/login", (req, res, next) => {
  if (req.session.backURL) { // We check if there a return URL has been set prior redirecting/accesing.
  /* Return URL is the url that user will be redirected to after login. */
    req.session.backURL = req.session.backURL;
  } else { // If there is no return URL we simply set it to index page.
    req.session.backURL = "/";
  }
  // Now that we have configured the returning URL, we can let passport redirect user to appropriate auth page.
  next();
}, passport.authenticate("discord"));
```
**2. Callback Route**

In here we take data from discord, process it and redirect user accordingly.
```js
Here passport takes data returned from discord and we redirect user accordingly.
app.get("/callback", passport.authenticate("discord", { failureRedirect: "/" }), (req, res) => { // Passport collects data that discord has returned and if user aborted auhorization it redirects to '/'
  session.us = req.user;
  if (req.session.backURL) { // If there is a returning url we redirect user to it.
    const url = req.session.backURL;
    req.session.backURL = null; // We change returning url to null for little more performance.
    res.redirect(url);
  } else { // If there still isn't we won't leave user alone and stuck so well redirect it to index page.
    res.redirect("/");
  }
});
```
**3. Logout Route**

In here we logout the user, simple as that.
```js
app.get("/logout", function (req, res) {
  req.session.destroy(() => { // We destroy session
    req.logout(); // Inside callback we logout user
    res.redirect("/"); // And to make sure he isnt on any pages that require authorization, we redirect it to main page.
  });
});
```
# #3 Handy Functions
We are going to create some functions that will become handy.

**1. Authenticate Function**

This lets us check if user is logged in and if not redirect him, in a convenient way.
```js
const authenticate = (req, res, next) => {
  if (req.isAuthenticated()) return next(); // If the user is logged in, we skip execution of the rest of the code in this function and let the code for te route run.
  req.session.backURL = req.url; // If execution reached this point, means that user is not logged in and we can set the return url to the current url.
  res.redirect("/login"); // And we redirect it to our login handler that will do the job.
};
```
**2. Render Function**

This lets us render templates in a convenient way.

```js
const render = (req, res, template, data = {}) => {
  const baseData = {
    bot: client, // Your discord client.
    path: req.path, // Current path of the url
    user: req.isAuthenticated() ? req.user : null // If user is authenticated, we pass user, otherwise null.
  };
  
  const mergedData = Object.assign(baseData, data); // We merge the base data with data provided to function.
  const templatePath = path.resolve(`${templateDirectory}${path.sep}${template}`); // We resolve the template.
  
  res.render(templatePath, mergedData); // We render the template.
};
```
# #4 Setting Up Assets Folder
First let's define what paths we are going to use.

In the root folder, let's first create a new folder called `dashboard`. Inside this folder let's create another 2 folders, first is `assets` and the other one is `templates`.

**Setting Up Aseets Folder**

In order to have the `/assets/..` route working without us needing to hardcode everything we set another static route:
```js
app.use("/assets", express.static(path.resolve(`${filesDirectory}${path.sep}assets`)));
```

# #5 First Route
The index page is the first page user is going to see.

**1. Adding a new route to our express application.**

This makes server process a request and run code when user is accesing this route.
```js
app.get("/", (req, res) => { // We set the route so server handles request by running this code.
  // "/" domain is the root of the domain, for example gooogle.com is a "/" and google.com/somewhat is "/somewhat"
  renderTemplate( // We call render template function we defined earlier this guide.
     res, // We pass in the Response object that was returned by express.
     req, // We pass in the Request object that was returned by express as well.
     "index.js" // We pass in the template we want to render. The path of the template will result in /templates/index.ejs
  );
});
```

**2. Creating block templates.**

In our templates folder we create a new file called `index.ejs` **must end with `.ejs`**, the ejs view engine extension.

Inside our .ejs fine we can execute server-side javascript using the EJS's specific Tags and also front end javascript within the `<script>` tags and also include HTML and CSS for our front-end.

You can read more about EJS template engine in [EJS's Documentation](https://ejs.co/#docs).

To make our code more cleaner and because we'll be repeating the code most likely, we are going to create a `header.ejs` and `footer.ejs` files that we'll be including in other files.

So.. let's create our files, in our `dashboard/templates` folder, we'll be creating another folder called `blocks`, inside it we create 2 files called `header.ejs` and `footer.ejs`.

In our `header.ejs` we'll be having our `<html>` and `<head>` tag and everything in between and also a navigation bar, inside the `footer.ejs` we'll be having endings for our tags, the cross-file (that are more or less required on every page) `<script>` tags and possibly an actual footer.

*a) Creating `header.ejs`*
```ejs
<!DOCTYPE html>
<html>
  <head>
  <title><%= title %></title> <!-- We'll be setting the title of the page to a title varable that we'll pass in inside each template. -->
  <link rel="stylesheet" type="text/css" href="assets/style.css"> <!-- Linking our CSS file (we are about to create) from the assets folder. -->
  </head>
  <body>
```

*b) Creating `footer.ejs`*
```ejs
  </boody>
</html>
```
Now we are going to create a `style.css` file inside `dashboard/assets` folder.

**3. Creating our `index.ejs` template.**

```ejs
<!-- Including our template blocks: -->
<%- include("blocks/header.ejs", { bot, path, user, title: "Home" }); %> <!-- We include the header and we pass in the optios, the bot, path and user are automatically passed into main file by renderTemplate function  -->

<%- include("blocks/footer.ejs"); %> <!-- We include the footer and we do not pass any parameters as the footer template does not require any. -->
```
<img src="https://media1.tenor.com/images/93253f6c6f029c3e056281164084c209/tenor.gif?itemid=12050318" />
