# CastVideoLooping-receiver

This Google Cast receiver application shows how to loop video playback. 

## Dependencies
* A Google Cast Receiver Device (eg. Chromecast)
* The Google Cast Receiver SDK.

## Setup Instructions
* Check out the code from GitHub and host the index.html file on your web server.
* Create a custom receiver app ID in the Google Cast Developer console: https://cast.google.com/publish
* Enter the URL to the hosted index.html file for the custom receiver URL.
* Use the app ID in your sender app: https://developers.google.com/cast/docs/sender_apps

## Documentation
Google Cast receivers allows you to play video using the HTML5 media element. The video can be looped using the media element by specifying the “loop” attribute. However, auto looping introduces a few seconds delay between the end of the loop and the beginning of the next loop.

To make the transition seamless requires using the Media Source Extensions (MSE) which extends the media element to allow JavaScript to generate media streams for playback. MSE allows for segments of media to be handed directly to the HTML5 video tag’s buffer.

### Prepare the Video
Firstly you have to convert the video to the fragmented MP4 file format for streaming. A normal MP4 file consists of a header (moov box) and the media data (mdat box). The header is located at the end of the file. For streaming, the header has to be moved to the beginning of the file.

You will need these tools:
* FFmpeg: http://www.ffmpeg.org/download.html
* MP4Box: http://gpac.wp.mines-telecom.fr/downloads/gpac-nightly-builds/

Convert the file to have the correct codec using FFmpeg:
```
ffmpeg -i video.mp4 -an -codec:v libx264 -profile:v baseline -level 3 -b:v 2000k videocodec.mp4
```

Run the following command to put the header at the front of the file and to ensure that the fragments start with Random Access Points:
```
MP4Box -dash 1000 -rap -frag-rap videocodec.mp4
```
A new MP4 file is generated with a “_dashinit” postfix in the filename. Upload this file to your server.

### Play the Video
Now that the video file is in the correct format, MSE will be used to load and play the file with the HTML5 media element.

Firstly, define an HTML5 video element in your receiver web app:
```
<video id='vid' />
```
Get a reference to the video element in JavaScript:
```
window.video = document.getElementById('vid');
```
Create a MediaSource object and create a virtual URL using URL.createObjectURL with the MediaSource object as the source. Then assign the virtual URL to the media element’s “src” property:
```
window.mediaSource = new MediaSource();
window.video.src = window.URL.createObjectURL(window.mediaSource);
```
Wait for the MediaSource “sourceopen” event to tell you that the media source object is ready for a buffer to be added. Create a SourceBuffer using the MediaSource addSourceBuffer with the mime type of the video and then start downloading the file:
```
window.mediaSource.addEventListener('sourceopen', function(){	
	  window.sourceBuffer = window.mediaSource.addSourceBuffer(
	      'video/mp4; codecs="avc1.42c01e"');

	  fileDownload('videocodec_dashinit.mp4');              
    });
```
Use XMLHttpRequest to download the file as an ArrayBuffer:
```
function fileDownload(url) {
  var xhr = new XMLHttpRequest();
  xhr.open('GET', url, true);
  xhr.responseType = 'arraybuffer';
  xhr.send();
  xhr.onload = function(e) {
    if (xhr.status != 200) {
      onLoad();
      return;
    }
    onLoad(xhr.response);
  };
  xhr.onerror = function(e) {
    window.video.src = null;
  };
};
```
Append the file data to the SourceBuffer with appendBuffer:
```
function onLoad(arrayBuffer) {
  console.log("onLoad");
  if (!arrayBuffer) {
    window.video.src = null;
    return;
  }
  window.allSegments = new Uint8Array(arrayBuffer);
  window.sourceBuffer.appendBuffer(window.allSegments);
  processNextSegment();
}
```
Call the play method on the video element and append video segments to the source buffer when there is less than 10 seconds left in the playback pipeline:
```
function processNextSegment() {
  // Wait for the source buffer to be updated
  if (!window.sourceBuffer.updating &&   
       window.sourceBuffer.buffered.length > 0) {
    // Only push a new fragment if we are not updating and we have
    // less than 10 seconds in the pipeline
    if (window.sourceBuffer.buffered.end( 
          window.sourceBuffer.buffered.length - 1) -  
          window.video.currentTime < 10) {
      // Append the video segments and adjust the timestamp offset 
      // forward
	 window.sourceBuffer.timestampOffset =  
            window.sourceBuffer.buffered.end( 
             this.sourceBuffer.buffered.length - 1);
	        window.sourceBuffer.appendBuffer(window.allSegments);
    }
    // Start playing the video
    if (window.video.paused) {
      window.video.play();
    }
  }
  setTimeout(processNextSegment, 1000);
};
```
The video will keep looping forever and there won’t be any delays between each loop.

## References and How to report bugs
* Cast APIs: http://developers.google.com/cast/docs
* Design Checklist: http://developers.google.com/cast/docs/design_checklist
* If you find any issues, please open a bug here on GitHub

## How to make contributions?
Please read and follow the steps in the CONTRIBUTING.md

## License
See LICENSE

## Google+
Google Cast Developers Community on Google+ [http://goo.gl/TPLDxj](http://goo.gl/TPLDxj)
