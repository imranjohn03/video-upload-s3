
Live streaming upload to S3 by javascript.

Just file upload with form element just-upload.html

File upload after streaming in index.html

The user will need to grant access to the Camera and Microphone using the getUserMedia API. Using this stream, RecordRTC will be able to start the video recording.We will use following libraries.


```html
<script src="https://www.WebRTC-Experiment.com/RecordRTC.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/RecordRTC/5.5.5/RecordRTC.js"></script>
```
### 
The following example shows how to implement a basic start/stop recorder using RecordRTC :

```html
	<button id="btn-start-recording">
            Start Recording
        </button>
		
        <button disabled="disabled" id="btn-stop-recording">
            Stop Recording
        </button>
		
        <button id="cancel-button">
            Cancel Upload
        </button>
```

```javascript
document.getElementById('btn-start-recording').addEventListener("click", function(){
        // Disable start recording button
        this.disabled = true;

        // Request access to the media devices
        navigator.mediaDevices.getUserMedia({
            audio: true, 
            video: true
        }).then(function(stream) {
            // Display a live preview on the video element of the page
            setSrcObject(stream, video);

            // Start to display the preview on the video element
            // and mute the video to disable the echo issue !
            video.play();
            video.muted = true;

            // Initialize the recorder
            recorder = new RecordRTCPromisesHandler(stream, {
                mimeType: 'video/webm',
                bitsPerSecond: 128000
            });

            // Start recording the video
            recorder.startRecording().then(function() {
                console.info('Recording video ...');
            }).catch(function(error) {
                console.log(error);
                console.error('Cannot start video recording: ', error);
            });

            // release stream on stopRecording
            recorder.stream = stream;

            // Enable stop recording button
            document.getElementById('btn-stop-recording').disabled = false;
        }).catch(function(error) {
                   console.log(error);
            console.error("Cannot access media devices: ", error);
        });
    }, false);
	
	
   // When the user clicks on Stop video recording
    document.getElementById('btn-stop-recording').addEventListener("click", function(){
        this.disabled = true;

        recorder.stopRecording().then(function() {
            console.info('stopRecording success');

            // Retrieve recorded video as blob and display in the preview element
            var blob = recorder.getBlob();
            video.src = URL.createObjectURL(blob);
            video.play();

            // Unmute video on preview
            video.muted = false;

            // Stop the device streaming
            recorder.stream.stop();

            // Enable record button again !
            document.getElementById('btn-start-recording').disabled = false;
        }).catch(function(error) {
            console.error('stopRecording failure', error);
        });
    }, false);
	
```
### 
Now we have to upload the video/audio file to s3

###  
1) In the Amazon S3 console, create an Amazon S3 bucket that you will use to store the photos in the album. Make sure you have both Read and Write permissions on Objects.

2) In the Amazon Cognito console, create an Amazon Cognito identity pool using Federated Identities with access enabled for unauthenticated users in the same region as the Amazon S3 bucket. You need to include the identity pool ID in the code to obtain credentials for the browser script.

3) In the IAM console, find the IAM role created by Amazon Cognito for unauthenticated users. Add the following policy to grant read and write permissions to an Amazon S3 bucket.

```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
            "s3:*"
         ],
         "Resource": [
            "arn:aws:s3:::BUCKET_NAME/*"
         ]
      }
   ]
}
```
###  
Before the browser script can access the Amazon S3 bucket, you must first set up its CORS configuration as follows.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>POST</AllowedMethod>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>PUT</AllowedMethod>
        <AllowedMethod>DELETE</AllowedMethod>
        <AllowedMethod>HEAD</AllowedMethod>
        <AllowedHeader>*</AllowedHeader>
    </CORSRule>
</CORSConfiguration>

```

###  
Obtain the credentials needed to configure the SDK by calling the CognitoIdentityCredentials method, providing the Amazon Cognito identity pool ID. Next, create an AWS.S3 service object.

```html
 <script src="https://sdk.amazonaws.com/js/aws-sdk-2.283.1.min.js"></script> // Include in html head
```

```javascript
AWS.config.region = 'us-east-2'; // 1. Enter your region

 AWS.config.credentials = new AWS.CognitoIdentityCredentials({
       IdentityPoolId: 'IDENTITY_POOL_ID' // 2. Enter your identity pool
 });

AWS.config.credentials.get(function(err) {
	if (err) alert(err);
	console.log(AWS.config.credentials);
});

var bucketName = 'BUCKET_NAME/*'; // Enter your bucket name
var bucket = new AWS.S3({
	params: {
		Bucket: bucketName
	}
});
```
###  
To upload a file to Amazon S3 bucket, when stop recording

```javascript

 function dataURLtoFile(dataurl, filename) {
            var arr = dataurl.split(','), mime = arr[0].match(/:(.*?);/)[1],
                bstr = atob(arr[1]), n = bstr.length, u8arr = new Uint8Array(n);
            while(n--){
                u8arr[n] = bstr.charCodeAt(n);
            }
            return new File([u8arr], filename, {type:mime});
  }

 document.getElementById('btn-stop-recording').addEventListener("click", function(){
        this.disabled = true;

        recorder.stopRecording().then(function() {
            console.info('stopRecording success');

           var DataUrl = recorder.getDataURL();
           var random = Math.random( ); 
           DataUrl.then(function(result) {

                var url_file = dataURLtoFile(result, random +'.webm');

         var objKey = 'testing/' + url_file.name;
            var params = {
                Key: objKey,
                ContentType: url_file.type,
                Body: url_file,
                ACL: 'public-read'
            };

            var request = bucket.putObject(params);

            request.on('httpUploadProgress', function (progress) {
                percentage.innerHTML = parseInt((progress.loaded * 100) / progress.total)+'%'; 
                console.log("Uploaded :: " + parseInt((progress.loaded * 100) / progress.total)+'%');
               // console.log(progress.loaded + " of " + progress.total + " bytes");
            }).send(function(err, data){
                percentage.innerHTML = "File has been uploaded successfully.";
                listObjs();
            });


            cancelUpload.addEventListener('click', function() {
                    if(request.abort()){
                        percentage.innerHTML = "Uploading has been canceled.";
                    }
            });

    // here you can use the result of promiseB
});
           
            

            video.play();

            // Unmute video on preview
            video.muted = false;

            // Stop the device streaming
            recorder.stream.stop();

            // Enable record button again !
            document.getElementById('btn-start-recording').disabled = false;
        }).catch(function(error) {
            console.error('stopRecording failure', error);
        });
    }, false);

```
