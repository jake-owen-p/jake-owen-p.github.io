# Record and Upload to S3

When I googled 'How to record a video with react and upload to S3' I expected there to be a whole host of options. But alas, I struggled finding an article anywhere. I was barraged with react-native posts (didn't want it on phone) and I couldn't find anything for uploading videos to S3.

So, after hacking around I created something (simple) that worked. Heres my attempt at an article:

## Recording a Video with React

The list of requirements I had was:
- Record video through webcam
- Record audio
- Be able to watch the video as you recorded (this was the pain point)
- Have a playback option

[React Video Recorder](https://www.npmjs.com/package/react-video-recorder) ticked these boxes. 

### The React Component

```html
<VideoRecorder 
    onRecordingComplete={(videoBlob) => {
    }}
/>
```

### Video Blob Properties

### Examples of adding class names and other properties

## Uploading to S3

### Install AWS SDK
The [AWS SDK](https://www.npmjs.com/package/aws-sdk) will give you access to your S3 bucket.

### Setting the AWS Config

```javascript
AWS.config.update({
    region: 'eu-west-1',
    credentials: new AWS.CognitoIdentityCredentials({
        IdentityPoolId: 'eu-west-1:19a6a80e-a0d3-4e14-b190-2436be30f5d4'
    })
})
```
#### Give description of identity pools in AWS

### Targetting the S3 Bucket
```javascript
var s3 = new AWS.S3({
    apiVersion: "2006-03-01",
    params: { Bucket: 'video-holder-scribble' }
});
```

### Uploading to S3

```javascript
s3.putObject({
    Key: "video.mp4",
    Body: videoBlob,
    'ContentType': 'video/mp4',
    'ACL': 'public-read'
    }, (err) => {
    if(err){
        // Do error'y stuff
    }
})
```

If you want to create a public link to the video, S3 provides two options on click: 
- The video can be installed to the user's device
- The video can be viewed through the browser

