# lambduh
Composable modules intended for use with AWS Lambda functions

## Philosophy

[Lambduh](https://github.com/lambduh) is a grouping of tasks that help me quickly compose Lambda functions.

These modules are tiny! Seriously, you could just as easily write them yourself vs. learn whatever API they present. I just don't want to re-write them every time.

New Repos/Contributors/etc. are welcome - reach out with anything as an Issue.

## Modules

- [`lambduh-transform-s3-event`](https://github.com/lambduh/lambduh-transform-s3-event) - Transforms S3 Event JSON into a flattened object with bucket and key
- [`lambduh-validate`](https://github.com/lambduh/lambduh-validate) - Validates fields according to your will
- [`lambduh-execute`](https://github.com/lambduh/lambduh-execute) - Executes any shell string or bash file
- [`lambduh-get-s3-object`](https://github.com/lambduh/lambduh-get-s3-object) - Download any file from S3 to a local filepath
- [`lambduh-download-file`](https://github.com/lambduh/lambduh-download-file) - Download any file based on URL
- [`lambduh-put-s3-object`](https://github.com/lambduh/lambduh-put-s3-object) - Upload any local file to S3
- [`lambduh-list-s3-objects`](https://github.com/lambduh/lambduh-list-s3-objects) - List any S3 keys based on Bucket, prefix, and regex
- [`lambduh-gulp`](https://github.com/lambduh/lambduh-gulp) - Gulp tasks to make your lambda workflow all hunky-dory


## Example Lambda functions

- [`gif-to-mp4`](https://github.com/russmatney/lambda-gif-to-mp4) - Converts a .gif uploaded to S3 into an .mp4 and re-uploads to the same bucket. Uses ffmpeg as a binary.
- [`create-timelapse`](https://github.com/russmatney/lambda-create-timelapse) - Orchestrates the creation of a timelapse based on an arbitrary number of .jpg or .gifs passed via URL, with music based on an mp3 url, then uploads to Vimeo. Utilizes the following 4 lambda functions:
  - [`file-to-png`](https://github.com/russmatney/lambda-file-to-png) - Converts a passed url to a png (should be .gif or .jpg)
  - [`pngs-to-mp4`](https://github.com/russmatney/lambda-pngs-to-mp4) - Converts passed png keys into an mp4
  - [`mp4s-to-timelapse`](https://github.com/russmatney/lambda-mp4s-to-timelapse) - Concatenates mp4s into a single mp4 file, and adds an mp3
  - [`upload-to-vimeo`](https://github.com/russmatney/lambda-upload-to-vimeo) - Uploads a file in S3 to Vimeo, and sets some metadata


## Usage

Lambduh modules are promise-based. In general, any used fields should be explicitly passed in, and usually those fields are included on the returned object. The API is not yet uniform, but I've been productive with these pieces as they exist so far.

I'm very much open to suggestions – at this point, I think keeping the modules isolated and flexible (rather than forcing them into a standard) will help them to be the most useful and easy to understand.

### A composed lambda function

```javascript
var Q = require('q');
var lambduhModule = require('lambduh-module-name');
var transformS3Event = require('lambduh-transform-s3-event');
var validate = require('lambduh-validate');
var download = require('lambduh-get-s3-object');
var upload = require('lambduh-put-s3-object');

//your lambda function
exports.handler = function(s3event, context) {

  lambduhModule({
    data: somethingSpecific
  })
  
  .then(function(event) {
    return transformS3Event(event, s3event); //where `event` is an S3 event
  })
  
  .then(function(event) {
    return validate(event, {
      srcKey: { // requires options.srcKey to exist and meet the following criteria:
        endsWith: "\\.gif", //only operate on `.gif`s
        endsWithout: "_\\d+\\.gif", //exlude files with a `_300.gif` convention
        startsWith: "events/" //only operate on the bucket's "events/" folder
      }
    })
  })
  
  .then(function(event) {
    //download the file from s3
    options = {
      srcBucket: "source-bucket",
      srcKey: "path/to/key.png",
      downloadFilepath: "/tmp/file.png"
    }
    return download(event, options);
  })
  
  .then(function(event) {
    var def = Q.defer();
    //manipulate the file, say, with ffmpeg and/or imagemagick
    def.resolve(event);
    return def.promise;
  })

  .then(function(event) {
    //upload the new file to a bucket/key on S3
    options = {
      dstBucket: "destination-bucket",
      dstKey: "path/to/s3/upload/key.txt",
      uploadFilepath: "/tmp/path/to/local/manipulated/file.txt"
    }
    return upload(event, options);
  })

  .then(function(event) {
    context.done()
  })
  
  .fail(function(err) {
    console.log("derp");
    console.log(err);
    context.done(null, err); //to be handled however you prefer
  })
}
```
