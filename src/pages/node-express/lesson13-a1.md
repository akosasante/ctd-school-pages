---
layout: "../../layouts/genericMarkdownFile.astro"
title: Node/Express Lesson 13 Coding Assignment
description: Node/Express Lesson 13 Coding Assignment
---

# Node/Express Lesson 13 Coding Assignment

In this lesson, you use the passport and passport-local packages to handle user authentication, from within a server-side rendered application.

## First Steps

You continue to work with the same repository as the previous lesson, but you create a new branch
called lesson13.

The user records are stored in the Mongo database, just as for the Jobs API lesson. You have
already copied the models directory tree from the Jobs API lesson into the jobs-ejs repository.
The user model is used unchanged. You configure passport to use that model.

To begin, you will need the following views:

```
views/index.ejs
views/logon.ejs
views/register.ejs
```

The index.ejs view just shows links to login or register. The logon collects the email and
password from the user. The register collects the name, email, and password for a new user.
We want to use the partials as well. We want to modify the header partial to give
the name of the logged on user, and to add a logoff button if a user is logged on.

views/index.ejs:

```
<%- include("partials/head.ejs") %>
<%- include("partials/header.ejs") %>
    <% if (user) { %>
    <a href="/secretWord">Click this link to view/change the secret word.</a>
    <% } else { %>
    <a href="/session/logon">Click this link to logon.</a>
    <a href="/session/register">Click this link to register.</a>
    <% } %>
<%- include("partials/footer.ejs") %>
```

views/logon.ejs:

```
<%- include("partials/head.ejs") %>
<%- include("partials/header.ejs") %>
    <form method="POST">
      <div>
      <label name="email">Enter your email:</label>
      <input name="email">
      </div>
      <div>
      <lable name="password">Enter your password:</label>
      <input type=password name="password>
      </div>
      <div>
      <button>Logon</button>
      <a href="/"><button type="button">Cancel</button></a>
      </div>
      </form>
<%- include("partials/footer.ejs") %>
```

views/register.js

```
<%- include("partials/head.ejs") %>
<%- include("partials/header.ejs") %>
    <form method="POST">
          <div>
      <label name="name">Enter your name:</label>
      <input name="name">
      </div>
      <div>
      <label name="email">Enter your email:</label>
      <input name="email">
      </div>
      <div>
      <lable name="password">Enter your password:</label>
      <input type=password name="password">
      </div>
      <div>
      <lable name="password1">Confirm your password:</label>
      <input type=password name="password1">
      </div>
      <div>
      <button>Register</button>
      <a href="/"><button type="button">Cancel</button></a>
      </div>
      </form>
<%- include("partials/footer.ejs") %>
```

Revised views/header.ejs:

```
<h1>The Jobs EJS Application</h1>
<% if (user) { %>
   <p>User <%= user.name %> is logged on.</p>
   <form method="POST" action="/sessions/logoff">
   <button>Logoff</button>
   </form>
<% } %>
<% if (errors) {
    errors.forEach((err) => { %>
      <div>
        Error: <%= err %>
      </div>
    <% })
  } %>
  <% if (info) {
    info.forEach((msg) => { %>
      <div>
        Info: <%= msg %>
      </div>
    <% })
  } %>
<hr>
```

These changes won't suffice to do anything in the application, until routes are added to match.
We need to follow best practices, with separate route and controller files.

## Router and Controller

Create a file routes/sessionRoutes.js, as follows:

```
const express = require("express");
//const passport = require("passport");
const router = express.Router();

const {
  logonShow,
  registerShow,
  registerDo,
  logoff,
} = require("../controllers/sessionController");

router.route("/register").get(registerShow).post(registerDo);
router
  .route("/logon")
  .get(logonShow)
  .post(
    //    passport.authenticate("local", {
    //      successRedirect: "/",
    //      failureRedirect: "/sessions/logon",
    //      failureFlash: true,
    //    }
    //    )
    (req, res) => {
      res.send("Not yet implemented.");
    },
  );
router.route("/logoff").post(logoff);

module.exports = router;
```

Ignore the passport lines for the moment. This just sets up the routes. We need to create
a corresponding file controllers/sessionController.js. Here we use the User model. However,
the file you copied makes some references to the JWT library. You must edit models/User.js
to remove those references in order for User.js to load. We aren't using JWTs in this project.

```
const User = require("../models/User");
const parse_v = require("../util/parse_v_error");

const registerShow = (req, res) => {
  res.render("register");
};

const registerDo = async (req, res, next) => {
  if (req.body.password != req.body.password1) {
    req.flash("error", "The passwords entered do not match.");
    res.render("register");
  }
  try {
    await User.create(req.body);
  } catch (e) {
    if (e.constructor.name === "ValidationError") {
      parse_v(e, req);
    } else if (e.name === "MongoServerError" && e.code === 11000) {
      req.flash("error", "That email address is already registered.");
    } else {
      return next(e);
    }
    return res.render("register");
  }
  res.redirect("/");
};

const logoff = (req, res) => {
  req.session.destroy(function (err) {
    if (err) {
      console.log(err);
    }
    res.redirect("/");
  });
};

const logonShow = (req, res) => {
  if (req.user) {
    return res.redirect("/");
  }
  res.render("logon", {
    errors: req.flash("error"),
    info: req.flash("info"),
  });
};

module.exports = {
  registerShow,
  registerDo,
  logoff,
  logonShow,
};
```

The creation of the user entry in Mongo is just the same as it was for the Jobs API.

If there is a validation error
when creating a user record, we need to parse the validation error to return the
issues to the user, and we do that in the file util/parse_v_error.js:

```
const parse_v = (e, req) => {
  const keys = Object.keys(e.errors);
  keys.forEach((key) => {
    req.flash("error", key + ": " + e.errors[key].properties.message);
  });
};

module.exports = parse_v;
```

We need some middleware to load res.locals as needed. Create middleware/storeLocals.js:

```
const storeLocals = (req, res, next) => {
  if (req.user) {
    res.locals.user = req.user;
  } else {
    res.locals.user = null;
  }
  res.locals.info = req.flash("info");
  res.locals.errors = req.flash("error");
  next();
};

module.exports = storeLocals;
```

Now, we need a couple of app.use statements. Add these lines right after the connect-flash line:

```
app.use(require("./middleware/storeLocals"));
app.get("/", (req, res) => {
  res.render("index");
});
app.use("/session", require("./routes/sessionRoutes"));
```

We are now using the database. So, we need to connect to it at startup. You need a file,
db/connect.js. Check that it looks like the following:

```
const mongoose = require("mongoose");

const connectDB = (url) => {
  return mongoose.connect(url, {});
};

module.exports = connectDB;
```

Then add this line to app.js, just before the listen line:

```
    await require("./db/connect")(process.env.MONGO_URI);
```

Then try the application out, starting at the / URL. You can try each of the new views. But
you still can't logon. The logon operation is commented out, because Passport is not set up.

## Configuring Passport

To use Passport, you have to tell it how to authenticate users, retrieving them from the database.
Create a file passport/passport_init.js, as follows:

```
const passport = require("passport");
const LocalStrategy = require("passport-local").Strategy;
const User = require("../models/User");

const passport_init = () => {
  passport.use(
    "local",
    new LocalStrategy(
      { usernameField: "email", passwordField: "password" },
      async (email, password, done) => {
      try {
        const user = await User.findOne({ email: email });
        if (!user) {
          return done(null, false, { message: "Incorrect credentials." });
        }
        const result = await user.comparePassword(password);
        if (result) {
          return done(null, user);
        } else {
          return done(null, false, { message: "Incorrect credentials." });
        }
      } catch (e) {
        return done(e);
      }
    })
  );
  passport.serializeUser(async function (user, done) {
    done(null, user.id);
  });

  passport.deserializeUser(async function (id, done) {
    try {
      const user = await User.findById(id);
      if (!user) {
        return done(new Error("user not found"));
      }
      return done(null, user);
    } catch (e) {
      done(e);
    }
  });
};

module.exports = passport_init;
```

You can now add the following lines to app.js, right after the app.use for session (Passport
relies on session):

```
const passport = require("passport");
const passport_init = require("./passport/passport_init");
passport_init();
app.use(passport.initialize());
app.use(passport.session());
```

Finally, you can now uncomment the lines having to do with Passport in routes/sessionRoutes.js,
so that the require statement for Passport is included, and so that the route for logon looks like

```
router
  .route("/logon")
  .get(logonShow)
  .post(
    passport.authenticate("local", {
    successRedirect: "/",
    failureRedirect: "/sessions/logon",
    failureFlash: true})
  );
```

After that, try logon for one of the accounts you have created. You should see that you
are logged in and can access the secretWord page. You should also see appropriate
error messages for bad logon credentials. Also, test logoff.

## Protecting a Route

To protect a route, you need some middleware, as follows.

middleware/auth.js:

```
const authMiddleware = (req, res, next) => {
  if (!req.user) {
    req.flash("error", "You can't access that page before logon.");
    res.redirect("/");
  } else {
    next();
  }
};

module.exports = authMiddleware
```

We want to protect the route for the secretWord. The best practice is to put that
code into a router, as follows.

routes/secretWord.js:

```
const express = require("express");
const router = express.Router();

router.get("/", (req, res) => {
  if (!req.session.secretWord) {
    req.session.secretWord = "syzygy";
  }
  res.render("secretWord", { secretWord: req.session.secretWord });
});
router.post("/", (req, res) => {
  if (req.body.secretWord.toUpperCase()[0] == "P") {
    req.flash("error", "That word won't work!");
    req.flash("error", "You can't use words that start with p.");
  } else {
    req.session.secretWord = req.body.secretWord;
    req.flash("info", "The secret word was changed.");
  }
  res.redirect("/secretWord");
});

module.exports=router;
```

Then replace the app.get and app.post statements for /secretWord in app.js with these lines.

```
const secretWordRouter = require('./routes/secretWord');
app.use("/secretWord", secretWordRouter);
```

Then try out the secretWord page to make sure it still works. Turning on protection is simple.
You add the authentication middleware to the route as follows:

```
const auth = require('./middleware/auth');
app.use("/secretWord", auth, secretWordRouter);
```

That causes the authentication middleware to run before the secretWordRouter, and it redirects
if any requests are made for those routes before logon. Try it out: login and verify that
you can see and change the secretWord. Then logoff and try to go to the /secretWord URL.

## Fixing the Security

Passport is using the session cookie to determine if the user is logged in. This creates a security vulnerability called cross site request forgery (CSRF). We will demonstrate this.

To see this, clone **[this repository](https://github.com/Code-the-Dream-School/csrf-attack)** into a separate directory, outside jobs-ejs. Then, within the directory you cloned, do an "npm install" and a "node app". This will start another express application listening on port 4000 of your local machine. This is the attacking code. It could be running anywhere on the Internet -- that has nothing to do with the attack.

You should have two browser tabs open, one for localhost:3000, and one for localhost:4000. The one at localhost:4000 just shows a button that says Click Me! Don't click it yet. Use the jobs-ejs
application in the 3000 tab to set the secret string to some value. Then log off.
Then click the button in the 4000 tab. Then log back on in the 3000 tab and check the value of the secret string. So far so good -- it still has the value you set.

Now, without logging off of jobs-ejs , click the button in the 4000 tab. Then refresh the /secretWord
page in jobs-ejs. Hey, what happened! (By the way, this attack would
succeed even if you closed the 3000 tab entirely.)

You see, the other application sends a request to your application in the context of your browser -- and that request automatically includes the cookie. So, the application thinks the request comes from a logged on user, and honors it. If the application, as a result of a form post, makes database changes, or even transfers money, the attacker could do that as well.

So, how to fix this? This is the purpose of the host-csrf package you installed at the start
of the project. Follow the instructions **[here](https://www.npmjs.com/package/host-csrf)** to integrate the package with your application. You will need to change app.js as well as **each of the forms** in your ejs files. You can use process.env.SESSION_SECRET as your cookie-parser secret. Note that the app.use for the csrf middleware must come after the cookie parser middleware and after the body parser middleware, but before any of the routes. You will see a message logged to the console that the CSRF protection is not secure. That is because you are using HTTP, not HTTPS, so the package is less secure in this case, but you would be using HTTPS in production. As you will see, it stops the attack.

Retest, first to see that your application still works, and second, to see that the attack no longer works. (A moral: Always log off of sensitive applications before you surf, in case the sensitive application is vulnerable in this way. Also note that it does not help to close the application, as the cookie is still present in the browser. You have to log off to clear the cookie. Even restarting the browser does not
suffice.)

Enabling CSRF protection in the project is an _important_ part of this lesson -- don't omit it!
By the way, the CSRF attack only works when the credential is in a cookie. It doesn't work
if you use JWTs in the authorization header.

## Submitting Your Work

As usual, add and commit your changes and push the lesson13 branch to your github. Then
create the pull request and incude the link in your homework submission.
