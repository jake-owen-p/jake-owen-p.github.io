# Record and Upload to S3

A few months ago I was working on a project where we had to record a video using a webcam and send the link to an email address.

Googling around there wasn't much information when it came integrating webcams, React and S3 - so i've detailed a (semi) step by step guide to kick you off.

What i'll be covering:
- Recording a video in React using a webcam
- Accessing AWS resources from a website
- Uploading an object to S3
- Getting a public link to the video

## Recording a Webcam Video with React

The list of requirements I had was:
- Record video & audio through a webcam
- Get real time feedback on the video as you record
- Have a playback option after recording

[React Video Recorder](https://www.npmjs.com/package/react-video-recorder) is a highly configurable, lightweight skin over [styled component's](https://www.npmjs.com/package/styled-components) Video component. It provides a toolbox of utilities to tailor the recording and [blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) to your requirements such as MimeType, playback options and image overlays. These are all documented on [their github](https://github.com/fbaiodias/react-video-recorder/blob/master/src/video-recorder.js#L75).

### The React Component

```javascript
<VideoRecorder 
    onRecordingComplete={(videoBlob) => {
        // do stuff with blob
    }}
/>
```

## Creating your AWS Resources

### Creating the S3 bucket

From within the [AWS S3 Console](https://console.aws.amazon.com/s3/) you can create your bucket. To make the video link public and viewable outside of your AWS account you must turn off the default option `Block all public access`.

## Amazon Cognito 

When using the AWS SDK within a browser it is preffered to use Amazon Cognito Identity Pools as it provides temporary credentials where you can specify what resources users can access. Users can access your AWS resources either as guests (unauthenticated) or authenticated users. For the purpose of this article, we will be focussing on guests, but if you wish to read more on authenciting users (which is highly advised) you can [here](https://docs.aws.amazon.com/cognito/latest/developerguide/switching-identities.html).

You can create your Amazon Cognito Identity Pool from the [console](https://eu-west-1.console.aws.amazon.com/cognito/federated). When creating the identity pool, tick the box `Enable access to unauthenticated identities` to allow users to access the resouces without logging in.

The following page will allow you to chose which resources you wish to give the users access to through IAM policies. By default, users have access to Cognito services & analytics, so we need to allow access to S3. Under the each role, click edit on the policy document and add the following:

Make sure to replace `<bucketname>` with the name of your bucket.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cognito-sync:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
          "Effect": "Allow",
          "Action": [
                "s3:PutObject",
            ],
          "Resource": [
                "arn:aws:s3:::<bucketname>/*"
            ]
        }
    ]
}
```

The only thing we have added to the existing document is the ability to add objects to a specific S3 bucket. It is good practice to only give the minimum access required to limit any damage done by potential attackers. You can also add this permission through the [IAM Console](https://console.aws.amazon.com/iam/home) after creating your identity pool.

## Uploading to S3

### Install AWS SDK
The [AWS SDK](https://www.npmjs.com/package/aws-sdk) will give you access to your S3 bucket.

### Authentication 

Now we can add in our identity pool. From the [Amazon Cognito Identity Pool console](https://eu-west-1.console.aws.amazon.com/cognito/federated) click in the pool you've just created, then into `Sample Code`. This will provide you a snippet similar to below, with a unique identifer for your identity pool.

```javascript
AWS.config.update({
    credentials: new AWS.CognitoIdentityCredentials({
        IdentityPoolId: 'eu-west-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
    })
})
```

### Configuring S3

The AWS SDK defaults to the latest version of the S3 client. Whilst most of the time this will be your preference, it can cause issues down the line if the API changes. To avoid this, you can globally lock the S3 API version to a specific version.

```javascript
AWS.config.apiVersions = {
  s3: '2006-03-01'
};
```

To access your S3 bucket you create an instance of it that will allow you to perform CRUD operations using the user's credentials from the identity pool. 

```javascript
var s3 = new AWS.S3({
    params: { 
        Bucket: 'your-bucket-name' 
    }
});
```

### Uploading to S3

```javascript
s3.putObject({
    Key: "video.mp4",
    Body: videoBlob,
    'ContentType': 'video/mp4',
    ACL: 'public-read'
    }, (err) => {
        if(err){
            // On Error
        } else {
            // On Success
        }
    }
)
```

There are a few properties to break down here:
- Key: The name of the file to be saved to S3. This key also respects file structure using forward slashes (`/`), so to save it to a specific folder you can use `folder1/folder2/video.mp4`.
- Body: The content to upload.
- Content-Type: This tells amazon what format the content is. It is required for playback when accesing the video through the public URL S3 provides.
- ACL: Amazon's access control list (ACL) specifies the permissions set on an objects. Just like creating the S3 bucket, objects within the bucket default to non public. To allow the URL to be accessed publically this needs to be set to `public-read`.

## Final React Component

```javascript
<VideoRecorder 
    onRecordingComplete={ (videoBlob) => {
        s3.putObject({
            Key: "video.mp4",
            Body: videoBlob,
            'ContentType': 'video/mp4',
            ACL: 'public-read'
            }, (err) => {
                if(err){
                    // On Error
                } else {
                    // On Success
                }
            }
        )
    }}
/>
```

Once added, your S3 bucket will contain your video with a URL that follows the format `https://<bucketname>.<region>.amazonaws.com/<videoname>` which can be shared publically.

### Points to Consider.

1) If you are planning on using this on a real site, the domain must have an SSL certificate. This is by design on browsers as its dangerous to stream webcam content over HTTP.
2) If you only want to send the public video to a single user, and do not want the video to be easily discoverable, i'd advise at a minimum you add a [UUID](https://www.npmjs.com/package/uuid) to the name.
