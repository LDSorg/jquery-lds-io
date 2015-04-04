angular-lds-io
==============

Angular services for working with LDS.org API data.

* Login
* API
* Caching

Install & Usage
===============

```bash
bower install --save angular-lds-io
```

```html
<script src="bower_components/angular-lds-io/lds-io.min.js"></script>
```

```javascript
angular.module('myApp', [
  'ngRoute',
  'lds.io',
  'myApp.login'
]).run([
    '$rootScope'
  , 'MyAppLogin'
  , 'LdsApi'
  , 'LdsApiSession'
  , function ($rootScope, MyAppLogin, LdsApi, LdsApiSession) {

  return LdsApi.init({
    appId: 'TEST_ID_9e78b54c44a8746a5727c972'
  , appVersion: '1.0.0'
  , invokeLogin: MyAppLogin.invokeLogin
  }).then(function (LdsApiConfig) {
    return LdsApiSession.backgroundLogin().then(function () {

      // <body class="fade" ng-class="{ in: rootReady }">
      $rootScope.rootReady = true;

      // <div ng-if="rootDeveloperMode" class="alert alert-info">...</div>
      $rootScope.rootDeveloperMode = LdsApiConfig.developerMode;

    });
  });
}]);
```

Example `MyAppLogin.invokeLogin`
```javascript
// poor man's login modal
function invokeLogin() {
  // <div ng-if="rootShowLoginModal" ng-controller="LoginController as L"
  //   class="fade" ng-class="{ in: rootShowLoginModalFull }">...</div>
  $rootScope.rootShowLoginModal = true;
  $rootScope.rootLoginDeferred = $q.defer();
  $timeout(function () {
    $rootScope.rootShowLoginModalFull = true;
  }, 50);

  // 

  return $rootScope.rootLoginDeferred.promise;
}
```

API
===

All APIs return a promise unless otherwise noted

LdsApi
------

* LdsApi.init(opts)                                       // typically just appId, appVersion, and invokeLogin

LdsApiSession
-------------

* `LdsApiSession.backgroundLogin()`                         // attempts login via oauth3 iframe
* `LdsApiSession.login()`                                   // must be attached to a click handler
* `LdsApiSession.logout()`             // opens iframe with logout
* `LdsApiSession.onLogin($scope, fn)`  // fires when switching from logged out to logged in
* `LdsApiSession.onLogout($scope, fn)` // fires when switching from logged in to logged out
* `LdsApiSession.checkSession()`       // resolves session or rejects nothing
* `LdsApiSession.requireSession()`     // calls `invokeLogin()` if `checkSession()` is rejected

LdsApiRequest
-------------

As long as data is not older than `LdsApiConfig.uselessWait`, it will be presented immediately.
However, if it is older than `LdsApiConfig.refreshWait` it will refresh in the background. 

* `LdsApiRequest.profile()`                                 // logged-in user's info
* `LdsApiRequest.stake(p.homeStakeId)`                      // returns ward member data
* `LdsApiRequest.stakePhotos(p.homeStakeId)`                // returns photo metadata
* `LdsApiRequest.ward(p.homeStakeId, p.homeWardId)`         // returns ward member data
* `LdsApiRequest.wardPhotos(p.homeStakeId, p.homeWardId)`   // returns photo metadata
* `LdsApiRequest.photoUrl(metadata)` (non-promise)          // constructs ward member photo url
* `LdsApiRequest.guessGender(member)` (non-promise)         // returns a guess based on organizations (priest, laural, etc)
* TODO `.leadership(ward)` (non-promise)              // pluck important leadership callings from ward data
* TODO `.stakeLeadership(stake)` (non-promise)        // pluck important leadership callings from stake data

Note: all api requests can accept `{ expire: true }` to force them to ignore the cache

LdsApiStorage
-------------

These use `localStorage`, but all return promises. It would be extremely easy to swap this out for an indexeddb version. Please pull-request with a new file `lds-io-storage-indexeddb.js` if you do so.

All keys are prefixed with `io.lds.`.

* `LdsApiStorage.get(key)` // Done't JSON.parse(), this is done for you
* `LdsApiStorage.set(key, obj)` // Don't JSON.stringify(), this is done for you
* `LdsApiStorage.remove(key)`
* `LdsApiStorage.clear(key)` // this will only remove `io.lds.` prefixed keys, it DOES NOT `localStorage.clear()`

Note: If a `get` retrieves `undefined`, `"undefined"`, `null` or `"null"`, or if the parse fails, the promise will be rejected.

LdsApiCache
--------

This is a layer atop `LdsApiStorage`, and used by `LdsApiRequest`. You may find it useful for your own applications as it guarantees that any existing data will be returned and stale data will be refreshed in the background.

* `LdsApiCache.read(id, x, fetch, opts)`
  * `id` is the id that should be used
  * `x` is an obsolete parameter and will be removed in 1.0.0 (no code change will be necessary on your end)
  * `fetch()` must be a promise. It will be called if the data is not in cache or needs to be refreshed
    * (this was abstracted out so that the code can easily be shared between node, jQuery, angular, etc)
  * `opts` such as `{ expire: true }`
* `LdsApiCache.destroy()` // expires all caches and calls `LdsApiStorage.clear()`

Note: there is also an internal `init()`

LdsApiConfig
------------

This is just helper class. It is called by `LdsApi.init(opts)`

* `LdsApiConfig.init(opts)` // Called by `LdsApi.init(opts)`
  * `invokeLogin()`         // shows UI for login and returns a promise of when login has completed
  * `providerUri`           // misnomer this will point to the oauth3 provider ('https://ldsconnect.org'), but currently points to 'https://lds.io' because that part isn't implemented
  * `appUri` // $window.location.protocol + '//' + $window.location.host + $window.location.pathname
  * `appId`
  * `appVersion` // might be used to clear cache on version change
  * `apiPrefix` // defaults to `/api/ldsio`
  * `logoutIframe` // `/logout.html`
  * `refreshWait` // (in milliseconds) defaults to 15 minutes
    * will not attempt to check for new data for 15 minutes
  * `uselessWait` // (in milliseconds) defaults to Inifinity
    * if data is this old, it will not be retrieved from cache
