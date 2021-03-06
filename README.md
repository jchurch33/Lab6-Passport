# Lab6-Authentication
CS260 Authentication Lab

Authentication is a critical part of almost any application.  Passport allows you to use an authenticaion framework for local database authentication as well as authentication through google, twitter or facebook.  In this lab, you will set up local authentication for your application.  

The project is organized into the following directory structure:
* ./: Contains the base application files and supporting folders. This is the project root.
* ./node_modules: Created when the NPMs listed above are installed in the system.
* ./controllers: Contains the Express route controllers that provide the interaction between routes and changes to the MongoDB database.
* ./models: Contains the Mongoose model definitions for objects in the database.
* ./views: Contains the HTML templates that will be rendered by EJS.

When the user registers, you will create a document in the mongo database that includes the password.  When the user logs in later, you will check to make sure the password is correct.

#1. Create an express project
<pre>
express auth
cd auth
mkdir models
mkdir controllers
</pre>

#2. Create a schema for your user document.  
Notice that the schema defined implements a unique username as well as email, color, and hashed_password fields. The final line creates the model in Mongoose.  Put this file in models/users_model.js
```javascript
var mongoose = require('mongoose'),
    Schema = mongoose.Schema;
var UserSchema = new Schema({
    username: { type: String, unique: true },
    email: String,
    color: String,
    hashed_password: String
});
mongoose.model('User', UserSchema);
```
#2. Now implement the Express webserver for the application. 
When you create a project, the setup code is in app.js. The general flow of this code is to first require the necessary modules, connect to the MongoDB database, configure the Express server, and begin listening.
The line "require('./models/users_model.js');" ensures that the User model is loaded for Mongoose:

The Express configuration code uses the connect-mongo library to register the MongoDB connection as the persistent store for the authenticated sessions. Notice that the connect-mongo store is passed an object with session set to the express-session module instance. Also notice that the db value in the mongoStore instance is set to the mongoose.connection.db database that is already connected.  The session will be passed back to the browser in a cookie and will be sent to the server each time another HTTP request is made.  You can change the port from 3000 to something else if you want to.

```javascript
var express = require('express');
var path = require('path');
var favicon = require('serve-favicon');
var logger = require('morgan');
var cookieParser = require('cookie-parser');
var bodyParser = require('body-parser');
var expressSession = require('express-session');
var mongoStore = require('connect-mongo')({session: expressSession});
var mongoose = require('mongoose');
require('./models/users_model.js');
var conn = mongoose.connect('mongodb://localhost/myapp', { useMongoClient: true });

var routes = require('./routes/index');
var users = require('./routes/users');

var app = express();

app.use(expressSession({
  secret: 'SECRET',
  cookie: {maxAge:2628000000},
  resave: true,
  saveUninitialized: true,
  store: new mongoStore({
      mongooseConnection:mongoose.connection
    })
  }));

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.engine('.html', require('ejs').__express);
app.set('view engine', 'html');


// uncomment after placing your favicon in /public
//app.use(favicon(__dirname + '/public/favicon.ico'));
app.use(logger('dev'));
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

app.use('/', routes);
app.use('/users', users);

// catch 404 and forward to error handler
app.use(function(req, res, next) {
    var err = new Error('Not Found');
    err.status = 404;
    next(err);
});

// error handlers

// development error handler
// will print stacktrace
if (app.get('env') === 'development') {
    app.use(function(err, req, res, next) {
        res.status(err.status || 500);
        res.render('error', {
            message: err.message,
            error: err
        });
    });
}

// production error handler
// no stacktraces leaked to user
app.use(function(err, req, res, next) {
    res.status(err.status || 500);
    res.render('error', {
        message: err.message,
        error: {}
    });
});


module.exports = app;
```
#3. Now add the routes/index.js file. 
This file implements the routes necessary to support signup, login, editing, and user deletion. This code also implements static routes that support loading the static files.

Notice that req.session is used frequently throughout the routes code. This is the session created when expressSession() middleware was added in the previous section. Notice the code is called to clean up the existing session data when the user logs out or is deleted "req.session.destroy(function(){});"

The code attaches text strings to the session.msg variable so that they can be added to the template. (This is just for example purpose so that you can see the status of requests on the webpages.)

Notice that the express server will serve static files from the "public" directory.  It will create new ejs content from the views directory.  When the "/" route is accessed, the code will check the cookie to see if there is already a session for the user.  If so, it renders the "views/index.html" file, otherwise it redirects them to the login page at "views/login.html".

```javascript
var express = require('express');
var router = express.Router();
var expressSession = require('express-session');

var users = require('../controllers/users_controller');
console.log("before / Route");
router.get('/', function(req, res){
    console.log("/ Route");
//    console.log(req);
    console.log(req.session);
    if (req.session.user) {
      console.log("/ Route if user");
      res.render('index', {username: req.session.username,
                           msg:req.session.msg,
                           color:req.session.color});
    } else {
      console.log("/ Route else user");
      req.session.msg = 'Access denied!';
      res.redirect('/login');
    }
});
router.get('/user', function(req, res){
    console.log("/user Route");
    if (req.session.user) {
      res.render('user', {msg:req.session.msg});
    } else {
      req.session.msg = 'Access denied!';
      res.redirect('/login');
    }
});
router.get('/signup', function(req, res){
    console.log("/signup Route");
    if(req.session.user){
      res.redirect('/');
    }
    res.render('signup', {msg:req.session.msg});
});
router.get('/login',  function(req, res){
    console.log("/login Route");
    if(req.session.user){
      res.redirect('/');
    }
    res.render('login', {msg:req.session.msg});
});
router.get('/logout', function(req, res){
    console.log("/logout Route");
    req.session.destroy(function(){
      res.redirect('/login');
    });
  });
router.post('/signup', users.signup);
router.post('/user/update', users.updateUser);
router.post('/user/delete', users.deleteUser);
router.post('/login', users.login);
router.get('/user/profile', users.getUserProfile);


module.exports = router;
```

#4. Now you need to implement the route code to support interaction with the MongoDB model. 
Put it in "controllers/users_controller.js".
```javascript
var crypto = require('crypto');
var mongoose = require('mongoose'),
    User = mongoose.model('User');
function hashPW(pwd){
  return crypto.createHash('sha256').update(pwd).
         digest('base64').toString();
}
exports.signup = function(req, res){
  console.log("Begin exports.signup");
  var user = new User({username:req.body.username});
  console.log("after new user exports.signup");
  user.set('hashed_password', hashPW(req.body.password));
  console.log("after hashing user exports.signup");
  user.set('email', req.body.email);
  console.log("after email user exports.signup");
  user.save(function(err) {
    console.log("In exports.signup");
    console.log(err);
    if (err){
      res.session.error = err;
      res.redirect('/signup');
    } else {
      req.session.user = user.id;
      req.session.username = user.username;
      req.session.msg = 'Authenticated as ' + user.username;
      res.redirect('/');
    }
  });
};
exports.login = function(req, res){
  User.findOne({ username: req.body.username })
  .exec(function(err, user) {
    if (!user){
      err = 'User Not Found.';
    } else if (user.hashed_password ===
               hashPW(req.body.password.toString())) {
      req.session.regenerate(function(){
        console.log("login");
        console.log(user);
        req.session.user = user.id;
        req.session.username = user.username;
        req.session.msg = 'Authenticated as ' + user.username;
        req.session.color = user.color;
        res.redirect('/');
      });
    }else{
      err = 'Authentication failed.';
    }
    if(err){
      req.session.regenerate(function(){
        req.session.msg = err;
        res.redirect('/login');
      });
    }
  });
};
exports.getUserProfile = function(req, res) {
  User.findOne({ _id: req.session.user })
  .exec(function(err, user) {
    if (!user){
      res.json(404, {err: 'User Not Found.'});
    } else {
      res.json(user);
    }
  });
};
exports.updateUser = function(req, res){
  User.findOne({ _id: req.session.user })
  .exec(function(err, user) {
    user.set('email', req.body.email);
    user.set('color', req.body.color);
    user.save(function(err) {
      if (err){
        res.sessor.error = err;
      } else {
        req.session.msg = 'User Updated.';
        req.session.color = req.body.color;
      }
      res.redirect('/user');
    });
  });
};
exports.deleteUser = function(req, res){
  User.findOne({ _id: req.session.user })
  .exec(function(err, user) {
    if(user){
      user.remove(function(err){
        if (err){
          req.session.msg = err;
        }
        req.session.destroy(function(){
          res.redirect('/login');
        });
      });
    } else{
      req.session.msg = "User Not Found!";
      req.session.destroy(function(){
        res.redirect('/login');
      });
    }
  });
};

```
The logic for the "signup" route first creates a new User object and then adds the email address and hashed password, using the hashPW() function defined in the same file. Then the Mongoose save() method is called on the object to store it in the database. On error, the user is redirected back to the signup page.

If the user saves successfully, the ID created by MongoDB is added as the req.session.user property, and the username is added as the req.session.username. The request is then directed to the index page.  The req.session information is passed back and forth using cookies.

The logic for the "login" route finds the user by username, then it compares the stored hashed password with a hash of the password sent in the request. If the passwords match, the user session is regenerated using the regenerate() method. Notice that req.session.user and req.session.username are set in the regenerated session.

The "getUserProfile" route finds the user by using the user id that is stored in req.session.user. If the user is found, a JSON representation of the user object is returned in the request. If the user is not found, a 404 error is sent.

The "updateUser" route finds the user and then sets the values from the req.body.email and req.body.color properties that will come from the update form in the body of the POST request. Then the save() method is called on the User object, and the request is redirected back to the /user route to display the changed results.

The deleteUser route finds the user in the MongoDB database and then calls the remove() method on the User object to remove the user from the database. Also notice that req.session.destroy() is called to remove the session because the user no longer exists.

#5. Now that the routes are set up and configured, you are ready to implement the views that are rendered by the routes. 
These views are intentionally very basic to allow you to see the interaction between the EJS render engine and the AngularJS support functionality. The following sections discuss the index, signup, login, and user views.

These views are implemented as EJS templates. This approach uses the EJS render engine because it is very similar to HTML, so you do not need to learn a new template language, such as Jade. However, you can use any template engine that is supported by Express to produce the same results.

Create views/index.html
```
<!doctype html>
<html ng-app="myApp"> 
    
<head>
    <title>User Login and Sessions</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/style.css" />
</head>
        
<body>  
    <div ng-controller="myController">
        <h2 id="welcomeText">Welcome. You are Logged In as
            <%= username %>
        </h2>
        <a id="logoutLink" href="/logout">logout</a>
        <a id="editLink" href="/user">Edit User</a>
        <p>Place Your Code Here
            <p>
    </div>
    <hr>
    <span id="indexMsg">
        <%= msg %>
    </span>
    <hr>
    <span id="colorText">
    Color
        <%= color %>
      </span>
    <script src="http://code.angularjs.org/1.2.9/angular.min.js"></script>
    <script src="/javascripts/my_app.js"></script>
</body>
      
</html>
```
Then create views/login.html
```
<!doctype html>
<html>
    
<head>
    <title>User Login and Sessions</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/style.css" />
</head>
        
<body>  
    <div class="form-container">
        <p class="form-header">Login</p>
        <form method="POST" action="/login">
            <label>Username:</label>
            <input type="text" id="username" name="username"><br>
            <label>Password:</label>
            <input type="password" id="password" name="password"><br>
            <input type="submit" id="login" value="Login">
        </form>
    </div>
    <a id="signUpLink" href="/signup">Sign Up</a>
    <hr>
    <span id="loginMsg">
    <%= msg %>
    </span>
</body>
        
</html>
```
Then create views/signup.html
```
<!doctype html>
<html>
    
<head>
    <title>User Login and Sessions</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/style.css" />
</head>
        
<body>  
    <div class="form-container">
        <p class="form-header">Sign Up</p>
        <form method="POST" action="/signup">
            <label>Username:</label>
            <input type="text" id="username" name="username"><br>
            <label>Password:</label>
            <input type="password" id="password" name="password"><br>
            <label>Email:</label>
            <input type="email" id="email" name="email"><br>
            <input type="submit" id="register" value="Register">
        </form>
    </div>
    <hr>
    <span id="signupMsg">
    <%= msg %>
    </span>
</body> 

</html>

```
Then create views/user.html
```
<!doctype html>
<html ng-app="myApp">
    
<head>
    <title>User Login and Sessions</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/style.css" />
</head>
        
<body>  
    <div class="form-container" ng-controller="myController">
        <p id="sectionTitle" class="form-header">User Profile</p>
        <form method="POST" action="/user/update">
            <label>Username:</label>
            <input type="text" id="username" name="username" ng-model="user.username" disabled><br>
            <label>Email:</label>
            <input type="email" id="email" name="email" ng-model="user.email"><br>          
            <label>Favorite Color:</label>
            <input type="text" id="color" name="color" ng-model="user.color"><br>   
            <input type="submit" id="save" value="Save">
        </form>
    </div>
    <form method="POST" action="/user/delete">
        <input type="submit" id="delete" value="Delete User">
    </form>
    <hr>
    <span id="userMsg">
    <%= msg %>
    </span>
    <script src="http://code.angularjs.org/1.2.9/angular.min.js"></script>
    <script src="/javascripts/my_app.js"></script>
    <a id="homeLink" href="/">Home</a>
</body>

</html>

```
Then create views/error.html

```
<!doctype html>
<html>

<head>
    <title>User Login and Sessions</title>
    <link rel="stylesheet" type="text/css" href="/stylesheets/style.css" />
</head>

<body>
<h1> Error </h1>
</body>

</html>


```

And you will need the angular controller "public/javascripts/my_app.js"
```js
angular.module('myApp', []).
  controller('myController', ['$scope', '$http',
                              function($scope, $http) {
    $http.get('/user/profile')
        .success(function(data, status, headers, config) {
      $scope.user = data;
      $scope.error = "";
    }).
    error(function(data, status, headers, config) {
      $scope.user = {};
      $scope.error = data;
    });
  }]);
```

Test your application to make sure you can create a new user, change the colors and see them preserved during the session, then logout to destroy the session. 

You will find that you need to install several packages:
<pre>
npm install 
npm install express-session
npm install connect-mongo
npm install ejs
</pre>
Passoff:

You should test your server to make sure it works correctly. You should have utilized google classroom to get started. Your submission to learningsuite should contain:


	- The URL of the working application on your EC2 node (or other host). 



<strong>Behavior</strong> |	<strong>Points</strong>
--- | ---
You can create a new user who is auto-logged-in | 25
When you logout, you can login using that created user. | 25
User color changes persist through authentication cycles. | 25
When you logout the session is destroyed and non-authenticated users cannot access the site. | 15
Your code is included in your submission, your application works with the test driver, and your page looks really good. This is subjective, so wow us. | 10
