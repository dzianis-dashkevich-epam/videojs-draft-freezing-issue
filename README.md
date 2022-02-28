## Description
Hi!

I am working with STB and we are using Video.Js for HLS live playback. I noticed random 1-5sec freezes during playback session which impact QoE.

## Additional Information
Please include any additional information necessary here. Including the following:

I collected debug logs for ~6 hours of playback and noticed a lot of `vhs-gap-skip`, `vhs-unknown-waiting`, `vhs-live-resync`, `waiting` events.

But moreover, I noticed a lot of unnecessary segment appends after playlist switching because of bandwidth re-calculation.

Those are unnecessary because they do not change buffer - the same media sequence segments are already in the buffer from previous renditions (here is visualization):
<img width="1189" alt="freeze-snapshot" src="https://user-images.githubusercontent.com/94862693/155458563-e26b7de6-374b-4e2c-9047-0d17fcf72c14.png">


To narrow the investigation scope and to prove my theory, that the problem is because of those unnecessary appends after playlist switching - I decided to test the same stream, but with only one rendition allowed for the same playback time.

Result: 0 `vhs-gap-skip`, `vhs-unknown-waiting`, `vhs-live-resync`, `waiting` events were observed. The playback was flawless.

So, I dived into source code and noticed that `segmentLoader` calls `resetLoader` during playlist switch for live. It means that `currentTime` will be used instead of `bufferedEnd` for next segment calculation in the next `Playlist.getMediaInfoForTime` which is called by `chooseNextRequest_` during `fillBuffer_`.

In most cases `currentTime` is several segments before `bufferedEnd` and new `syncPoint` is calculated based on the new playlist. That is why `Playlist.getMediaInfoForTime` returns first `mediaSequance` available from the new playlist.
This can lead to unexpected behavior including freezing on low-performant devices.

I changed `resetLoader` to `resyncLoader` and it significantly improves QoE.
But it seems like it was an intentional change in the scope of [low low-latency HLS integration](https://github.com/videojs/http-streaming/pull/1201)

So, probably some new approach should be introduced eg: track media Sequences in the buffer and prevent unnecessary appends.

I suppose this issue is related: https://github.com/videojs/http-streaming/issues/1224
PS: It seems like `useNetworkInformationApi` and `experimentalBufferBasedABR` improve stability significantly, but still it fixes consequences - not a root cause.


## Steps to reproduce
I suppose, the easiest way is to throttle Network up and down during playback via chrome devtools.

## Results
Random 1-5sec freezes during playback session which impact QoE.

### Expected
Stream plays fine without freezing.


### videojs-http-streaming version
what version of videojs-http-streaming does this occur with?

I was testing videojs-http-streaming 2.9.2 and 2.13.1

### videojs version
what version of videojs does this occur with?

I was testing with videojs v7.14.3 (VHS v2.9.2) and v 7.18.0 (VHS v2.13.1)

### Browsers
what browsers are affected? please include browser and version for each

I was testing on Chromium 72.0.3626.121(Linux) and 98.0.4758.109(MacOS)

### Platforms
what platforms are affected? please include operating system and version or device and version for each

I was testing on MacOS and Linux

### Other Plugins
are any other videojs plugins being used on the page? If so, please list them with version below.
* No other plugins.

### Other JavaScript
are you using any other javascript libraries or frameworks on the page? if so please list them below.
* Issue occurs even without any other js libraries
