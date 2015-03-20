## Hybrid Web Apps

By now, you have probably seen many different frameworks out in the wild that will allow you to build a mobile application using a combination of web technologies like HTML5, CSS3 and Javascript.

One of the more popular frameworks of late is the [Ionic Framework](http://ionicframework.com) which is based on [Angular JS](https://angularjs.org/) from Google, on top of [Cordova](http://cordova.apache.org/) for generating apps for both Apple's iOS and Google's Android platforms.  I have been using this framework since mid 2014, and written several small apps with even while it was still in Beta state, with excellent results.

It is truly one of the most flexible, and easiest to learn mobile development frameworks that I have used.  There are plenty of blog posts here and elsewhere detailing how to get started with Ionic, so I thought that I would focus on one key feature which I had to tackle recently, which is the fairly new Web Audio API.

## Making Sounds In A Web App

There are several plugins available for Cordova for generating sound within a hybrid web app.  From short once off sound effects, to playing a constant stream of music in the background, and indeed mixing the two together (for example, setting a spooky background theme for a game, as well as overlaying sounds of weapons or player voiceovers during the game).

The problem with these plugins though, is that they are usually initiated and called within the Javascript code.  This is not in itself a bad thing, however you have to bear in mind that Javascript is an interpreted language, not precompiled, so your instructions are parsed, interpreted and executed by the web browser on the device when your app is run.

This works fine for non critical audio, such as an ambient background track, but what if you wanted some precisely timed audio that absolutely HAD to play at a predetermined time interval?

### The case of the meandering metronome

I came across this issue recently when developing my [MusicKata](https://itunes.apple.com/us/app/musickata/id954752855?mt=8) app for musicians to help them with their practice routines.  Using Ionic, the building of the app went smoothly right up until I got to one particular feature - the Metronome.

I wanted a metronome within the app which the user could activate at any time while practicing a piece of music so that they could improve their rhythm and timing.  For that reason, a metronome has to be absolutely precise.  I mean absolutely.

If you set the metronome for 120 beats per minute, you HAD to get precisely 120 beats per minute.  Not one less, not one more.  Exactly 2 beats per second with NO variation in the gaps between the metronome 'ticks'.

I initially achieved the metronome sound via the Ionic `$timeout` function, which is really a wrapper for the Javascript `setInterval()` function, commonly used to call a Javascript procedure at, well, a set interval.

Here is an abbreviated example using `$timeout` to play a click at 90 beats per minute (bpm).

```javascript
angular.module('AudioTest', [])
.controller('MetronomeCtrl', function($scope, $timeout) {
    var mytimeout = null; // the current timeoutID
    var bpm = 90;
    
    // actual timer method
    $scope.metronomeClick = function() {
        // Play the metronome 'click' sound
        ...
        mytimeout = $timeout($scope.metronomeClick, 60000/bpm);
    };
    
    // starts the metronome
    $scope.startTimer = function() {
        mytimeout = $timeout($scope.metronomeClick, 60000/bpm);
    };
    
    // stops the metronome
    $scope.stopTimer = function() {
        $timeout.cancel(mytimeout);
    };
});
```

This worked up to a point.  80% of the time, the function to play the metronome `metronomeClick()` would fire at the correct timing interval.  However, because Javascript is interpreted, other things happening in the background on the device, such as the operating system doing some garbage collection or housekeeping, or the user activating a touch event on the device screen, or an email being processed in the background, would throw the interval time off, so you would get a pause or a skip in the metronome beats.

Totally **not** acceptable when you are trying to [practice 'Flight of the Bumbelbee' at 200 BPM](https://www.youtube.com/watch?v=zx00BCUqI7U).  Even a 5 millisecond delay was very noticeable, and would throw the musician off.

I tried using background web workers in Javascript etc., but to no avail.

But then, I came across the Web Audio API, a new proposal to add complex audio processing to the core browser itself.

## Building a Web Audio app

Enough preamble already.  Lets actually build an Ionic app that uses the [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) so that we can see it in action.

We will start with a blank Ionic project, and build it out from there.  In your terminal session, lets go to a folder on your hard drive and create a new project called **WebAudioTest**.

```
ionic start WebAudioTest blank
cd WebAudioTest
ionic platform add ios
ionic platform add android
```

The project will be prepared for both iOS and Android devices.

### The platform headache

The Web Audio API is available on all iOS devices which run iOS version 6 or better.  But please be aware that a lot of Android devices, even those running the latest version of Android, might not support Web Audio as yet, because they use an older web browser built in.

For this reason, we install the [Crosswalk](https://crosswalk-project.org/) browser on our Android project, as Crosswalk fully supports the Web Audio API.

To add the Crosswalk browser to our Android builds:

```
ionic add browser crosswalk
```

Note that it can take a while for the crosswalk browser to download and be integrated with your project.

### Looped or Triggered?

Now we are ready to build our project, we need to look at how we will generate the metronome clicks.  We have two choices really:

**Triggered** - This is where the clicks are triggered by the code, on a timer, much like the sample code I mentioned above.  However, a downside of this technique is, as I have already said, that the interval control is not 100%, and you can suffer some serious lag times if other asynchronous tasks hog the processor.

**Looped** - Web Audio, like other audio libraries, allow you to automatically **loop** an audio track.  This is handy for ambient background music etc. which is designed to play continuously and restart itself automatically when it gets to the end. This is essentially a 'set and forget' method, and the audio track itself seems to be handed off to a different threading model that is FAR more robust and accurate in terms of timeslice management than the Javascript engine.


### Loopy de loop

As you might have gathered, we will be using the **loop** method to generate our clicks in this exercise, given it's far better accuracy, and cleaner handling once it is started.

But this method means solving a few problems along the way, including:

#### A different kind of beep

Musicians are a fussy lot.  Guitarists and people playing in electric bands like to hear either a wooden drumstick 'click' or else a robotic 'beep'.  Piano players tend to like the lazy 'click' of the old swing arm metronomes.  Classical players like a more subtle 'blip' that does not overpower the smooth sounds of their instruments.

To this end, I wanted to give the end user the option to choose the type of intervallic 'click' that they wanted.  I easily found several sample sounds freely available on the internet, but here also came my first problem.

Each sound had a different decay.  Some were only 0.2 of a second long with no reverb or echo, but some were nearly 1 whole second long with a long delay tail.  This could cause problems if the interval was actually shorter than the sound sample.

#### The loop track length

The looping functionality in Web Audio is desgined to take a whole audio stream, start from the beginning, play to the end, and then go back to the beginning to start playing again.

The end point of the track is when the audio basically finished playing.  This means if your 'click' audio stream is 0.25 of a second long, Web Audio will play the track for 0.25 of a second, then restart playing it again immediately.  Result is you will always get 4 clicks per second.

A sample of a click audio on a timeline:

![Single Click](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/1-SingleClick.png)

The sample click on a repeat loop:

![Single Click Looped](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/2-MultiClick.png)

What if you only wanted 1 click per second?  Well then, you would have to take your click audio with a length of 0.25 of a second, then add 0.75 seconds of 'dead space' to pad out the audio track to 1.0 seconds.  Then repeat the padded stream.

Adding a click to a padded buffer:

![Padding The Click](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/3-AddPadding.png)

Playing a padded click on a loop:

![Padded Click Looped](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/4-PaddedClick.png)

## The Player

Lets first of all create a very plain and simple player screen in Ionic.  

![WebAudioTest Player](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/WebAudioTest-Screenshot.png)

Go to your `www/index.html` file and make the changes to the bottom section of code to add a start/stop toggle for the metronome, as well as a slider to set the bpm (beats per minute).

```markup,linenums=true
!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="initial-scale=1, maximum-scale=1, user-scalable=no, width=device-width">
    <title></title>

    <link href="lib/ionic/css/ionic.css" rel="stylesheet">
    <link href="css/style.css" rel="stylesheet">

    <!-- IF using Sass (run gulp sass first), then uncomment below and remove the CSS includes above
    <link href="css/ionic.app.css" rel="stylesheet">
    -->

    <!-- ionic/angularjs js -->
    <script src="lib/ionic/js/ionic.bundle.js"></script>

    <!-- cordova script (this will be a 404 during development) -->
    <script src="cordova.js"></script>

    <!-- your app's js -->
    <script src="js/app.js"></script>
    <script src="lib/AudioSampleLoader.js"></script>
  </head>
  <body ng-app="starter">

    <ion-pane>
      <ion-header-bar class="bar-stable">
        <h1 class="title">Web Audio Test</h1>
      </ion-header-bar>
      <ion-content>
        <div class="list list-inset">
          <ion-toggle toggle-class="toggle-energized" ng-model="metronome.metronomeOn" ng-checked="metronome.metronomeOn" ng-change="toggleMetronome(metronome.metronomeOn)" class="item item-icon-left">
            <i class="icon ion-speedometer energized"></i>
            Metronome &nbsp;&nbsp;<strong>{{ metronome.metronomeRate }} bpm</strong>
          </ion-toggle>
          <div class="item range range-energized center" ng-if="metronome.metronomeOn == true">
            <button class="button button-small button-outline button-energized" ng-click="changeTempo(-1)">-</button>
            <input type="range" name="metronome" min="40" max="200" value="{{ metronome.metronomeRate }}" ng-model="metronome.metronomeRate" ng-change="changeTempo(0)">
            <button class="button button-small button-outline button-energized" ng-click="changeTempo(1)">+</button>
          </div>
        </div>
      </ion-content>
    </ion-pane>
  </body>
</html>
```

As you can see, this is a really simple screen on the app.  When the Metronome toggle is OFF, the bpm slider is hidden from view, but when you toggle the Metronoma ON, then the slider appears and you can either move the slider to change the tempo, or else use the '+' and '-' buttons on the ends to make fine tuned adjustments.

Now lets move to the back end controller Javascript code that will do all the work.

#### Third Party Libraries

To make our job easier, we will be incorporating a library called [AudioSampleLoader](https://github.com/ScottMichaud/AudioSampleLoader) which was put together by Scott Michaud.  You can pull down the single Javascript file from his [Github](https://github.com/ScottMichaud/AudioSampleLoader) page.

Simply copy the `AudioSampleLoader.js` file into your `www/lib` folder and remember to add the line 

    <script src="lib/AudioSampleLoader.js"></script>

in your `index.html` (as above).  This library will do a lot of the heavy lifting for us, in terms of loading up the 'click' audio file into the Web Audio play buffer.

#### Audio files

I have also placed the 'click' audio files into the `www/audio` folder in my project.  If you want to download the files I am using, you can get them here:

* [Classic.mp3](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/Classic.mp3)
* [Klunk.wav](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/Klunk.wav)
* [Ping.wav](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/Ping.wav)
* [Robot.wav](https://s3.amazonaws.com/blazesharedfolder/blogs/airpair/Robot.wav)

Note that you can use `mp3` or `wav` files without issue in Web Audio.

Just replace the name of the file in line 22 of the Javascript below with a different file to see the difference.  If you wanted to get really tricky, you could add a drop down list to the view so that the user could choose the sound file at run time, but we won't get into that in our example.

## The Web Audio API Javascript

Normally, the following code would go into a Controller that was responsible for a particular view in Ionic, but because we are just using a simple blank Ionic app here, we can just add this code into the `www/js/app.js` file.

Note: If your code was in a controller, you would also use `$scope` in place of `$rootScope` as best practice.

```javascript,linenums=true
angular.module('starter', ['ionic'])

.run(function($ionicPlatform, $rootScope) {
  $ionicPlatform.ready(function() {
    // Hide the accessory bar by default (remove this to show the accessory bar above the keyboard
    // for form inputs)
    if(window.cordova && window.cordova.plugins.Keyboard) {
      cordova.plugins.Keyboard.hideKeyboardAccessoryBar(true);
    }
    if(window.StatusBar) {
      StatusBar.styleDefault();
    }

    // Metronome functions
    $rootScope.metronome = {};
    $rootScope.metronome.metronomeOn = false;
    $rootScope.metronome.metronomeRate = 90;

    var audioCtx = new window.webkitAudioContext; // (window.AudioContext || window.webkitAudioContext)();
    var audioLoader = new AudioSampleLoader();
    audioLoader.src = "audio/Classic.mp3";
    audioLoader.ctx = audioCtx;
    audioLoader.onload = function() {
       metronomeBuffer = audioLoader.response;
    };
    audioLoader.onerror = function() {
      console.log("Error loading Metronome Audio");
    };
    audioLoader.send();

    var metronomeSource = null;

    prepareMetronome = function(bpm) {
      var frameCount = audioCtx.sampleRate * (60/bpm);
      var numberOfChannels = metronomeBuffer.numberOfChannels;
      var paddingBuffer = audioCtx.createBuffer(1, frameCount, audioCtx.sampleRate);
      for (var i=0; i<numberOfChannels; i++) {
        var clickData = paddingBuffer.getChannelData(i);
        clickData.set(metronomeBuffer.getChannelData(i));
      };
      metronomeSource = audioCtx.createBufferSource();
      metronomeSource.connect(audioCtx.destination);
      metronomeSource.buffer = paddingBuffer;
      metronomeSource.loop = true;
    };

    $rootScope.toggleMetronome = function(metronomeOn) {
      if (metronomeOn == true) {
        prepareMetronome($rootScope.metronome.metronomeRate);
        metronomeSource.start(0);
      } else {
        metronomeSource.stop(0);
        $rootScope.metronome.metronomeOn = false;
      };
    };

    $rootScope.changeTempo = function(val) {
      $rootScope.metronome.metronomeRate = Number($rootScope.metronome.metronomeRate) + Number(val);
      metronomeSource.stop(2);
      prepareMetronome($rootScope.metronome.metronomeRate);
      metronomeSource.start(2);    
    };

  });
})
```

## Explanation of the Code

Ok, lets break down the Javascript above to see what we are doing.

The rough steps that we carry out are:
* Create and audio context
* Load the click audio and attach it to the audio context
* Create a buffer for the click audio

Then
* Create a padded buffer with silence which is exactly the length of the desired metronome click duration
* Merge the loaded click audio into the padded buffer
* Create an audio source pointer to the new merged buffer

Then
* Process start and stop play commands against the buffer source

Firstly, we are defininng an object under the `$rootScope` which will hold some critical information that we need - namely the current beats per minute (bpm) setting and a flag to denote whether the metronome is running or not.

    $rootScope.metronome = {};
    $rootScope.metronome.metronomeOn = false;
    $rootScope.metronome.metronomeRate = 90;

As mentioned earlier, if you were using these functions in a Controller, it is FAR better to define the objects under the controller `$scope` instead of the root scope.

The next section of code initialises the Web Audio objects that we will need

    var audioCtx = new window.webkitAudioContext;
    var audioLoader = new AudioSampleLoader();
    audioLoader.src = "audio/Classic.mp3";
    audioLoader.ctx = audioCtx;
    audioLoader.onload = function() {
       metronomeBuffer = audioLoader.response;
    };
    audioLoader.onerror = function() {
      console.log("Error loading Metronome Audio");
    };
    audioLoader.send();

    var metronomeSource = null;

Note: if you have problems with WebAudio when testing in your browser, you may need to change 

    var audioCtx = new window.webkitAudioContext;

to:

    var audioCtx = new window.AudioContext;

This line sets up `audioCtx` which is the Audio Context that we will be using to generate the audio sounds in the browser.

The next couple of lines uses the `AudioSampleLoader` library to load up the actual sound snippet file for our 'click' into `audioLoader` and attaches it to the audio context.

At this stage, we have set up the buffer `metronomeBuffer` which we store the 'click' audio for us, but we havent defined a buffer source, so `metronomeSource` is set to `nil` for now.

The `prepareMetronome` function is the interesting one.  This is where we actually create the audio buffer, and add the blank padding to the audio and create a buffer source so we can start playing.

    prepareMetronome = function(bpm) {
      var frameCount = audioCtx.sampleRate * (60/bpm);
      var numberOfChannels = metronomeBuffer.numberOfChannels;
      var paddingBuffer = audioCtx.createBuffer(1, frameCount, audioCtx.sampleRate);
      for (var i=0; i<numberOfChannels; i++) {
        var clickData = paddingBuffer.getChannelData(i);
        clickData.set(metronomeBuffer.getChannelData(i));
      };
      metronomeSource = audioCtx.createBufferSource();
      metronomeSource.connect(audioCtx.destination);
      metronomeSource.buffer = paddingBuffer;
      metronomeSource.loop = true;
    };

The function takes one parameter - **bpm**, which is the beats per minute that we want.

It then calculates the sampling rate of the current audio context to work out how many frames (`frameCount`) will be required to fill out each actual 'click' duraction.

Note also that Web Audio supports multiple channels for mono and stereo output, so we need to find out how many channels our preloaded source audio file has and store that in `numberOfChannels`.

Next, we create a *new* audio buffer `paddingBuffer` which is really an empty buffer (silence) with the duraction of `frameCount` - one whole metronome 'click'.

The following `for` loop just cycles through each channel on our source 'click' audio file and then just adds the 'click' audio data to the start of our `paddingBuffer`.

Essentially we have a padded audio stream with the exact length of each click, and we are just adding the custom click sound to the beginning of that padded stream.

The last few lines of the function simply attache the `metronomeSource` source identifier to the buffer so that we can actually play it.  At this point, we also tell Web Audio that we want to `loop` the audio track repeatedly.

Please note that `prepareMetronome` is called each time the **bpm** setting is changed by the user.  This is because we need to calculate the padding again for the beats per minute, and recreate the appropriate length padded buffer to copy the click audio into it.

The `toggleMetronome()` and `changeTempo()` functions are tied to the toggle and slider elements on the user interface.

    $rootScope.toggleMetronome = function(metronomeOn) {
      if (metronomeOn == true) {
        prepareMetronome($rootScope.metronome.metronomeRate);
        metronomeSource.start(0);
      } else {
        metronomeSource.stop(0);
        $rootScope.metronome.metronomeOn = false;
      };
    };

    $rootScope.changeTempo = function(val) {
      $rootScope.metronome.metronomeRate = Number($rootScope.metronome.metronomeRate) + Number(val);
      metronomeSource.stop(2);
      prepareMetronome($rootScope.metronome.metronomeRate);
      metronomeSource.start(2);    
    };

The toggle slider simply calls `toggleMetronome()` to set the `$rootScope.metronome.metronomeOn` flag to `true` or `false`.  This then simply calls the `start()` and `stop()` methods on the buffer source.  Simple.

The slider is linked to the `$rootScope.metronome.metronomeRate` object, and sets the **bpm** rate.  Every time the slider is moved, or the '+' or '-' buttons are tapped, the `changeTempo()` function is called to increment or decrement the **bpm** rate.

## Conclusion

That is really about it, when it comes to an introduction to the [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API).  Of course there is a great deal of depth to this rich API, and you can do things like perform convolution processing for reverb and delay effects etc, as well as modify the input audio signal to create special FX etc., but that is outside the scope of this article.

Hopefully this post will give you the confidence to get started with the Web Audio API and help it to become a standard part of all browsers in the future.




