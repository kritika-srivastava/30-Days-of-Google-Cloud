# Perform Foundational Infrastructure Tasks in Google Cloud

_Note: It is highly recommended that you __try__ to complete the objectives either twice or thrice before visiting the solutions._

## Task 1: Create a bucket

1. Go to Navigation menu > Cloud Storage > Browser. Click CREATE BUCKET.
2. Enter a unique name for your bucket(eg-your GC Project-ID). Select region > Region > us-east1 leaving rest settings as default and you are good to go.
3. Click on create.

## Task 2: Create a Pub/Sub topic

Type the following command in cloud shell replacing [TOPIC-NAME] with a name of your choice.

```
gcloud pubsub topics create [TOPIC-NAME]              
```
Eg-
```
gcloud pubsub topics create Something             
```

## Task 3: Create the thumbnail Cloud Function

1. Navigation menu > **COMPUTE** >  Cloud Functions > Create Function

2. Use the following configurations:

   **Name:** SomethingCF
   **Region:** us-east1
   **Trigger type:** Cloud Storage
   **Event type:** Finalize/Create
   **Bucket:** BROWSE > Select the qwiklabs bucket

3. Leave the Remaining as default > Save >Next

4. Set the following :
   **Runtime:** Node.js 14
   **Entry point:** thumbnail
5. Add the code appropriately:
- index.js 

In line 16 of index.js replace the text REPLACE_WITH_YOUR_TOPIC ID with the Topic ID you created in task 2.
```
/* globals exports, require */
//jshint strict: false
//jshint esversion: 6
"use strict";
const crc32 = require("fast-crc32c");
const { Storage } = require('@google-cloud/storage');
const gcs = new Storage();
const { PubSub } = require('@google-cloud/pubsub');
const imagemagick = require("imagemagick-stream");
exports.thumbnail = (event, context) => {
  const fileName = event.name;
  const bucketName = event.bucket;
  const size = "64x64"
  const bucket = gcs.bucket(bucketName);
  const topicName = "Something";                    // Replace with the topic name you used
  const pubsub = new PubSub();
  if ( fileName.search("64x64_thumbnail") == -1 ){
    // doesn't have a thumbnail, get the filename extension
    var filename_split = fileName.split('.');
    var filename_ext = filename_split[filename_split.length - 1];
    var filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length );
    if (filename_ext.toLowerCase() == 'png' || filename_ext.toLowerCase() == 'jpg'){
      // only support png and jpg at this point
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      let newFilename = filename_without_ext + size + '_thumbnail.' + filename_ext;
      let gcsNewObject = bucket.file(newFilename);
      let srcStream = gcsObject.createReadStream();
      let dstStream = gcsNewObject.createWriteStream();
      let resize = imagemagick().resize(size).quality(90);
      srcStream.pipe(resize).pipe(dstStream);
      return new Promise((resolve, reject) => {
        dstStream
          .on("error", (err) => {
            console.log(`Error: ${err}`);
            reject(err);
          })
          .on("finish", () => {
            console.log(`Success: ${fileName} → ${newFilename}`);
              // set the content-type
              gcsNewObject.setMetadata(
              {
                contentType: 'image/'+ filename_ext.toLowerCase()
              }, function(err, apiResponse) {});
              pubsub
                .topic(topicName)
                .publisher()
                .publish(Buffer.from(newFilename))
                .then(messageId => {
                  console.log(`Message ${messageId} published.`);
                })
                .catch(err => {
                  console.error('ERROR:', err);
                });
          });
      });
    }
    else {
      console.log(`gs://${bucketName}/${fileName} is not an image I can handle`);
    }
  }
  else {
    console.log(`gs://${bucketName}/${fileName} already has a thumbnail`);
  }
};
```
- package.json
```
{
  "name": "thumbnails",
  "version": "1.0.0",
  "description": "Create Thumbnail of uploaded image",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "@google-cloud/functions-framework": "^1.1.1",
    "@google-cloud/pubsub": "^2.0.0",
    "@google-cloud/storage": "^5.0.0",
    "fast-crc32c": "1.0.4",
    "imagemagick-stream": "4.1.1"
  },
  "devDependencies": {},
  "engines": {
    "node": ">=4.3.2"
  }
}
```
6. Click on Deploy.
7. Download the image from the given [URL](https://storage.googleapis.com/cloud-training/gsp315/map.jpg).
8. Go to Navigation menu > **STORAGE** > Storage > Select your bucket > Upload the image
9. Refresh bucket
10. Now you should see a thumbnail that is created.

## Task 4: Remove the previous cloud engineer

1. Go to Navigation menu > IAM & Admin
2. Among the list in permissions search for the "**Username 2**"  > Edit > Delete Role

Kudos! You have completed your lab successfully 🌟.