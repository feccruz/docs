---
lodash: true
---

## Ionic Framework + Firebase Tutorial

<div class="package" style="text-align: center;">
  <blockquote>
    <a href="@@base_url@@/auth0-ionic/master/create-package?path=examples/firebase-sample" class="btn btn-lg btn-success btn-package" style="text-transform: uppercase; color: white">
      <span style="display: block">Download a sample project</span>
    </a>
  </blockquote>
</div>

### 1. Setting up the callback URL in Auth0

<div class="setup-callback">
<p>Go to the <a href="@@uiAppSettingsURL@@" target="_new">Application Settings</a> section in the Auth0 dashboard and make sure that <b>Allowed Callback URLs</b> contains the following value:</p>

<pre><code>https://@@account.namespace@@/mobile</pre></code>

<p>Also, if you are testing your application with `ionic serve`, make sure to add your the following to Allowed Callback URL</p>

<pre><code>file://*
http://localhost:8100
</code></pre>


</div>

### 2. Configure Auth0 to work with Firebase

Now, you need to add your Firebase account information to Auth0. Once the user logs in to the App, Auth0 will use this configuration to get the Firebase token. 

In order to configure it, go to [Application Settings](@@uiAppSettingsURL@@) and go to the Addons section. In there, just turn on the Firebase Addon and enter your Firebase secret.

![Firebase secret](http://d.pr/i/1ejf7+)


### 3. Adding the needed dependencies

Add the following dependencies to the `bower.json` and run `bower install`:

````json
"dependencies" : {
  // Auth0 Dependencies
  "auth0-angular": "4.*",
  "a0-angular-storage": ">= 0.0.6",
  "angular-jwt": ">= 0.0.4",
  // Firebase dependencies
  "angularfire": "~0.9.2"
},
```

### 4. Add the references to the scripts in the `index.html`

````html
<!-- Auth0 Lock -->
<script src="lib/auth0-lock/build/auth0-lock.js"></script>
<!-- auth0-angular -->
<script src="lib/auth0-angular/build/auth0-angular.js"></script>
<!-- angular storage -->
<script src="lib/a0-angular-storage/dist/angular-storage.js"></script>
<!-- angular-jwt -->
<script src="lib/angular-jwt/dist/angular-jwt.js"></script>
<!-- Firebase dependencies -->
<script src="lib/firebase/firebase.js"></script>
<script src="lib/angularfire/dist/angularfire.js"></script>
```

### 5. Add `InAppBrowser` plugin

You must install the `InAppBrowser` plugin from Cordova to be able to show the Login popup. For that, just run the following command:

````bash
ionic plugin add org.apache.cordova.inappbrowser
```

and then add the following configuration to the `config.xml` file:

````xml
<feature name="InAppBrowser">
  <param name="ios-package" value="CDVInAppBrowser" />
  <param name="android-package" value="org.apache.cordova.inappbrowser.InAppBrowser" />
</feature>
```

### 6. Add the module dependency and configure the service

Add the `auth0`, `angular-storage`, `angular-jwt` and `firebase` module dependencies to your angular app definition and configure `auth0` by calling the `init` method of the `authProvider`

````js
// app.js
angular.module('starter', ['ionic',
  'starter.controllers',
  'starter.services',
  'auth0',
  'angular-storage',
  'angular-jwt',
  'firebase'])
.config(function($stateProvider, $urlRouterProvider, authProvider, $httpProvider,
  jwtInterceptorProvider) {


  $stateProvider
  // This is the state where you'll show the login
  .state('login', {
    url: '/login',
    templateUrl: 'templates/login.html',
    controller: 'LoginCtrl',
  })
  // Your app states
  .state('dashboard', {
    url: '/dashboard',
    templateUrl: 'templates/dashboard.html',
    data: {
      // This tells Auth0 that this state requires the user to be logged in.
      // If the user isn't logged in and he tries to access this state
      // he'll be redirected to the login page
      requiresLogin: true
    }
  })

  ...

  authProvider.init({
    domain: '<%= account.namespace %>',
    clientID: '<%= account.clientId %>',
    loginState: 'login'
  });

  ...

})
.run(function(auth) {
  // This hooks all auth events to check everything as soon as the app starts
  auth.hookEvents();
});
```


### 7. Let's implement the login

Now we're ready to implement the Login. We can inject the `auth` service in any controller and just call `signin` method to show the Login / SignUp popup.
In this case, we'll add the call in the `login` method of the `LoginCtrl` controller. 
On login success, we'll save the user profile, token and [refresh token](@@base_url@@/refresh-token) into `localStorage`. We'll also use the [Delegation endpoint](https://auth0.com/docs/auth-api#delegated) to get the Firebase token.

````js
// LoginCtrl.js
function LoginCtrl(store, $scope, $location, auth) {
  $scope.login = function() {
    auth.signin({
      authParams: {
        scope: 'openid offline_access',
        device: 'Mobile device'
      }
    }, function(profile, token, accessToken, state, refreshToken) {
      store.set('profile', profile);
      store.set('token', idToken);
      store.set('refreshToken', refreshToken);
      auth.getToken({
        api: 'firebase'
      }).then(function(delegation) {
        store.set('firebaseToken', delegation.id_token);
        $state.go('dashboard');
      }, function(error) {
        console.log("There was an error logging in", error);
      })
    }, function() {
      // Error callback
    });
  }
}
```

````html
<!-- login.tpl.html -->
<!-- ... -->
<input type="submit" ng-click="login()" />
<!-- ... -->
```

> Note: there are multiple ways of implementing login. What you see above is the Login Widget, but if you want to have your own UI you can change the `<script src="//cdn.auth0.com/js/auth0-lock-6.js">` for `<script src="//cdn.auth0.com/w2/auth0-2.1.js">`. For more details [check the GitHub repo](https://github.com/auth0/auth0-angular#with-your-own-ui).

### 8. Adding a logout button

You can just call the `signout` method of Auth0 to log the user out. You should also remove the information saved into `localStorage`:

````js
$scope.logout = function() {
  auth.signout();
  store.remove('token');
  store.remove('profile');
  store.remove('refreshToken');
  $state.go('login');
}
```

````html
<input type="submit" ng-click="logout()" value="Log out" />
```
### 9. Configuring secure calls to Firebase

Now that we have the Firebase token, we can just use it with AngularFire.

````js
var friendsRef = new Firebase("https://<your-account>.firebaseio.com/<your collection>");
// Here we're using the Firebase Token we stored after login
friendsRef.authWithCustomToken(store.get('firebaseToken'), function(error, auth) {
  if (error) {
    // There was an error logging in, redirect the user to login page
    $state.go('login');
  }
});

var friendsSync = $firebase(friendsRef);
var friends = friendsSync.$asArray();
friends.$add({name: 'Hey John'});
```

### 10. Showing user information

After the user has logged in, we can get the `profile` property from the `auth` service which has all the user information:

````html
<span>His name is {{auth.profile.nickname}}</span>
```

````js
// UserInfoCtrl.js
function UserInfoCtrl($scope, auth) {
  $scope.auth = auth;
}
```

You can [click here](@@base_url@@/user-profile) to find out all of the available properties from the user's profile. Please note that some of this depend on the social provider being used.

### 11. Keeping the user logged in after page refreshes

We already saved the user profile and tokens into `localStorage`. We just need to fetch them on page refresh and let `auth0-angular` know that the user is already authenticated.

````js
angular.module('myApp', ['auth0', 'angular-storage', 'angular-jwt'])
.run(function($rootScope, auth, store, jwtHelper, $location) {
  // This events gets triggered on refresh or URL change
  $rootScope.$on('$locationChangeStart', function() {
    if (!auth.isAuthenticated) {
      var token = store.get('token');
      if (token) {
        if (!jwtHelper.isTokenExpired(token)) {
          auth.authenticate(store.get('profile'), token);
        } else {
          // Use the refresh token we had
          auth.refreshIdToken(refreshToken).then(function(idToken) {
            store.set('token', idToken);
            auth.authenticate(store.get('profile'), token);
          });
        }
      }
    }
  });
});
```


### 11. Sit back and relax

Now it's time to sit back, relax and open a beer. You've implemented Login and Signup with Auth0 and Ionic.


### Troubleshooting

#### Command failed with exit code 65 when running ionic build

This means that the `InAppBrowser` plugin wasn't installed successfully by Cordova. Try any of the followings to fix this.

* Reinstall the `InAppBrowser` plugin

````bash
ionic plugin remove org.apache.cordova.inappbrowser
ionic plugin add org.apache.cordova.inappbrowser
```
* Remove the platform and re add it

````bash
ionic platform remove ios
ionic platform add ios
```

* Copy the contents from the plugin to the platform plugins

````bash
cp plugins/org.apache.cordova.inappbrowser/src/ios/* platforms/ios/[yourAppName]/Plugins/org.apache.cordova.inappbrowser/
```

#### Get a blank page with an OK after signin

This means that the `InAppBrowser` plugin wasn't installed successfully by Cordova. See the previous section to learn how to solve this.