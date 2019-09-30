# Intoduction
In this guide we'll be creating a small discord bot dashboard for managing guild settings, nothing too fancy.

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
