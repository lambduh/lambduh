# lambduh
Composable modules intended for use with AWS Lambda functions

#Intent

AWS Lambda is pretty sweet, albeit still a bit quirky.

I just spent a week building out my first Lambda function,
and some tasks can be DRYed up. 

[Lambduh](https://github.com/lambduh) is a grouping of these composable modules.

New Repos/Contributors/etc. are welcome - reach out with anything as an Issue.

Please Unit Test the crap out of your modules/PRs! These modules should be tiny and thus all scenarios should be easily covered!

#Modules

- [`lambduh-transform-s3-event`](https://github.com/lambduh/lambduh-transform-s3-event) - Transforms S3 Event JSON into a flattened object with attached bucket and key

#Usage

Lambduh modules are promise-based, and rely on passing an `options` object. Every module receives an inputted object, manipulates it as necessary, then resolves it. 

In this way, modules should be constructed like middleware:

```javascript
var Q = require('q');
module.exports = function(settingsOrData) {
  return function(options) {
    if (!options) { options = {} }
    return Q.promise(function(resolve, reject, notify) {

      options.do = "work"
      
      if (allIsDandy) {
        resolve(options)
      } else {
        reject(new Error("Not all was dandy :-/"))
      }
    }
  }
}
```

This allows you to compose Lambda functions with modules:

```javascript
var Q = require('q');
var lambduhModule = require('lambduh-module-name');
var transformS3Event = require('lambduh-transform-s3-event');

//your lambda function
exports.handler = function(event, context) {
  var promises = [];
  
  promises.push(lambduhModule({
    data: somethingSpecific
  }))
  promises.push(transformS3Event(event)) //where `event` is an S3 event
  
  promises.push(function(options) {
    console.log(options);
    context.done()
  })
  
  promises.reduce(Q.when, Q())
    .fail(function(err) {
      console.log("derp");
      console.log(err);
      context.done(null, err);
    });
}
```
