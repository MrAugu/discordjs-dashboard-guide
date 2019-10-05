# Intoduction
In this guide we'll be creating a small discord bot dashboard for managing guild settings, nothing too fancy. This is currently being worked on.
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
```js
app.get("/logout", function (req, res) {
  req.session.destroy(() => { // We destroy session
    req.logout(); // Inside callback we logout user
    res.redirect("/"); // And to make sure he isnt on any pages that require authorization, we redirect it to main page.
  });
});
```
