# lambduh
Composable modules intended for use with AWS Lambda functions

#Philosophy

AWS Lambda is pretty sweet, albeit still a bit quirky.

I just spent a week building out my first Lambda function,
and some tasks can be DRYed up. 

[Lambduh](https://github.com/lambduh) is a grouping of these composable modules.

These modules are tiny! Seriously, you could just as easily write them yourself vs. learn whatever API they present. I just don't want to re-write them every time.

New Repos/Contributors/etc. are welcome - reach out with anything as an Issue.

Please Unit Test the crap out of your modules/PRs! These modules should be tiny and thus all scenarios should be easily covered!

#Modules

- [`lambduh-transform-s3-event`](https://github.com/lambduh/lambduh-transform-s3-event) - Transforms S3 Event JSON into a flattened object with attached bucket and key
- [`lambduh-validate`](https://github.com/lambduh/lambduh-validate) - Validates fields according to your will
- [`lambduh-execute`](https://github.com/lambduh/lambduh-execute) - Executes any shell string, with an option for showing the logs or not
- [`lambduh-get-s3-object`](https://github.com/lambduh/lambduh-get-s3-object) - Download any file from S3 to a local filepath
- [`lambduh-put-s3-object`](https://github.com/lambduh/lambduh-put-s3-object) - Upload any local file to S3

#Usage - `options` object flow

Lambduh modules are promise-based, and rely on passing an `options` object. Every module receives an inputted object, manipulates it as necessary, then resolves it. 

In this way, modules should be constructed like middleware:

```javascript
var Q = require('q');
module.exports = function(settingsOrData) {
  return function(options) {
    if (!options) options = {};
    var def = Q.defer();
    
    //do work
    if (allIsDandy) {
      def.resolve(options);
    } else {
      def.reject(new Error("Not all was dandy :-/"));
    }
    
    return def.promise;
  }
}
```

#Composed lambda functions

This structure allows you to compose Lambda functions with modules:

```javascript
var Q = require('q');
var lambduhModule = require('lambduh-module-name');
var transformS3Event = require('lambduh-transform-s3-event');
var validate = require('lambduh-validate');
var download = require('lambduh-get-s3-object');
var upload = require('lambduh-put-s3-object');

//your lambda function
exports.handler = function(event, context) {
  var promises = [];
  
  promises.push(lambduhModule({
    data: somethingSpecific
  }))
  
  promises.push(transformS3Event(event)) //where `event` is an S3 event
  
  promises.push(validate({
    srcKey: {
      endsWith: "\\.gif",
      endsWithout: "_\\d+\\.gif",
      startsWith: "events/"
    }
  })) //only operate `.gif` files in the buckets "events" folder, excluding files ending with `_300.gif`
  
  promises.push(execute({
    shell: "cp /var/task/ffmpeg /tmp/.; chmod 755 /tmp/ffmpeg"
  }))
  
  promises.push(function(options) {
    options.downloadFilepath = "/tmp/path/to/local/file.txt"
    return download()(options);
  })
  
  promises.push(function(options) {
    var def = Q.defer();
    //manipulate the file, say, with ffmpeg and/or imagemagick
    def.resolve(options);
    return def.promise;
  })

  promises.push(function(options) {
    options.dstBucket = "destination-bucket"
    options.dstKey = "path/to/s3/upload/key.txt"
    options.uploadFilepath = "/tmp/path/to/local/manipulated/file.txt"
    return upload()(options);
  })

  promises.push(function(options) {
    context.done()
  })
  
  promises.reduce(Q.when, Q())
    .fail(function(err) {
      console.log("derp");
      console.log(err);
      context.done(null, err);
    })
}
```
