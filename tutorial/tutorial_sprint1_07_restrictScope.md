[< Tutorial Sprint 1](tutorial_sprint1.md)
# Tutorial - Limit access to the find() operation to users with the proper permission

### Overview
Continue securing our external API, this time implementing [permission scopes](../develop/develop_permissions_scopes.md) for our `find` and `update` routes.

In our permission scheme, an action permission is granted in the context of a scope of users.  We want to ensure that any request for Approvals, only return approvals generated by users within the assigned scope.


### Let's write some tests

Back in [step 3](tutorial_sprint1_03_fixtureResponse.md) we created a unit test to verify we only got back the entries for the users in our scope.  (looks like we already wrote our Unit Test for this step then.)

However, we hard coded our `PARequestController.find()` to manualy remove our unwanted entry.  So for this go around, we are now going to remove that hard coded response, and then get our system to properly filter based upon scopes:
```javascript
// api/controllers/PARequestController.js

module.exports = {

    _config: {
        model: "parequest", // all lowercase model name
        actions: false,
        shortcuts: false,
        rest: true
    },

    create:function(req, res) {
        // this is not allowed:
        res.forbidden();
    },

    destroy:function(req, res) {
        // this is not allowed
        res.forbidden();
    }

};
```

Now run our tests:
```sh
$ npm test

> opstool-process-approval@0.0.0 test /sails/node_modules/opstool-process-approval
> make test

  ․․․․․․․․․․

  9 passing (13s)
  1 failing

  1) PARequestController should return data on a request: :
     Uncaught AssertionError:  --> should only get 3 of our test entries back.: expected [ Array(4) ] to have a length of 3 but got 4
      at Function.assert.lengthOf (node_modules/chai/lib/chai/interface/assert.js:1033:37)
      at Test.<anonymous> (test/controllers/PARequestController.js:86:24)
      at Test.assert (node_modules/ad-utils/node_modules/supertest/lib/test.js:156:6)
      at Server.assert (node_modules/ad-utils/node_modules/supertest/lib/test.js:127:12)
      at net.js:1419:10



make: *** [test] Error 1
npm ERR! Test failed.  See above for more details.
```

And now we are back to our test actually catching the reality of our current route: it is not filtering the data based upon scopes.


### Add a policy to limit our results

In Sails, you are expected to modify the behavior of routes through the use of [policies](http://sailsjs.org/documentation/concepts/policies).

Policies are applied to the actions of a Controller before those actions are called.

In our default setup, the AppDev framework already applies a set of policies to our routes/actions that handles authentication and user verification.  

Now we are going to create an additional policy to limit our route according to the scope of people a user is allowed to work with.

To begin with, we will edit our `[plugin]/config/policies.js` with the following:
```javascript
// [plugin]/config/policies.js
var limitScope = function(req, res, next){

    Permissions.limitRouteToScope(req, res, next, {
        actionKey:'process.approval.tool.view',
        field:'userID'
    })
}

var scopedStack = ADCore.policy.serviceStack([ limitScope ]);

module.exports = {

    'opstool-process-approval/PARequestController': {
       find: scopedStack,
       findOne: scopedStack,
       update: scopedStack
    }

};
```



#### Analyzing what we have done:

```javascript
var limitScope = function(req, res, next){

    Permissions.limitRouteToScope(req, res, next, {
        actionKey:'process.approval.tool.view',
        field:'userID'
    })
}
```
is a new policy for limiting the route according to the scope of the currently authenticated user.  `limitScope` is created in the same format of an [express middleware function](http://expressjs.com/guide/using-middleware.html) (meaning it has a `req`, `res`, and `next` parameters).  

In our new policy, we simply call one of our existing AppDev provided `Permissions` routines to limit a route to a user's scope.  The `.limitRouteToScope(req, res, next, options)` takes the following options values:

| key |  description |
|------------|--------------|
| actionKey | Which of the action keys do you want to assess the scopes for (in this case: `process.approval.tool.view`) |
| field | in the model being used, which field has the value to compare against the allowed Users |
| userField | (optional) which of the User Object fields correspond to the value in `field`?  Default value: guid |
| error | (optional) An error response to return if you don't want the default 'forbidden' response.|


In our design, our PARequest model has a `userID` field that contains the `guid` values of the user who created the request.

Since we are storing the `guid` field, we don't have to specify another field to link to in the User data.  However, if we designed the field to track the User.id, or User.username values, then we would specify:
```javascript
Permissions.limitRouteToScope(req, res, next, {
    actionKey:'process.approval.tool.view',
    field:'userID',
    userField:'id'   // or 'username'
})
```

Also, in some cases you might not like the idea that someone without permission could randomly request information from your web service and receive a 403 response.  This response implies the information they requested existed, they just don't have permission to access it.  Instead, you might rather take a "we can neither confirm nor deny any such involvement" and not admit there is such a resource if they don't have permission.  So specifying a 'not found' error to be returned instead might be what you want:
```javascript
Permissions.limitRouteToScope(req, res, next, {
    actionKey:'process.approval.tool.view',
    field:'userID',
    userField:'guid',
    error:{
      code:404,
      message:'Not Found'
    }
})
```

Next we get our default AppDev stack of policies for authenticating a service request, and add our new policy to the end:
```javascript
var scopedStack = ADCore.policy.serviceStack([ limitScope ]);
```

This `scopedStack` represents an array of policies (usually the string names of policy files to load) to be run in a given order.  If for some reason you wanted to register your policy before any of our default ones:
```javascript
var scopedStack = ADCore.policy.serviceStack().unshift(limitScope);
```


After this, we now apply this new set of policies to the Controller actions that need them.  In this case it is the `PARequestController`'s `.find`, `.findOne`, and `.update` actions.  (remember we disabled the `.create` and `.destroy` actions back in [step 5](tutorial_sprint1_05_lockdownAPI.md)):
```javascript
module.exports = {

    'opstool-process-approval/PARequestController': {
       find: scopedStack,
       findOne: scopedStack,
       update: scopedStack
    }

};
```
>NOTE: when you create one of our plugins, our controller's show up in Sails' `api/controllers` directory under a directory named after our plugin.  So we have to reference the controller as `[pluginName]/[ControllerName]`


### Verify our Tests work now:

<!-- [[TODO: show how we updated our scope fixtures here.]] -->


So run our tests:
```sh
$ npm test

> opstool-process-approval@0.0.0 test /sails/node_modules/opstool-process-approval
> make test

  ․․․․․․․․․․

  10 passing (14s)

```


### Enable the Live System to return data

OK, we have our test environment running, but our live environment doesn't actually have any data in our DB to return.  If you wan to try to request this data from the browser and test things out, you will need to manually insert some example data into your MySQL DB.

<!-- [[TODO: make sure our test data corresponds to User info in the real life system. ]] -->

---
[< step 6 : limit access to the find() operation](tutorial_sprint1_06_restrictAccess.md)
[step 8 : create the server side Message Queue >](tutorial_sprint1_08_messageQueue.md)