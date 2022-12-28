# video-encoding-benchmark

This repo is a demonstration of a pure re-encoding of an mp4 video file with the web APIs.

There are four steps to re-encode a video:
* Extracting samples from the mp4 file format (demuxing), done with [mp4box.js](https://github.com/gpac/mp4box.js/).
* Decoding the samples into pixels, done with [WebCodec window.VideoDecoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder).
* Encoding pixels into samples, done with [WebCodec window.VideoEncoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder)
* Putting samples into an mp4 file (muxing)
  * Done with [mp4box.js](https://github.com/gpac/mp4box.js/) in mp4box.html
  * Done with [mp4wasm](https://github.com/mattdesl/mp4-wasm) in mp4wasm.html

The code is very dense so I'm going to split it up and explain it here.

## Demuxing

We start with a simple `<input type="file" />` in order to select a video file from the file system.

```html
<p>
  Select a video to re-encode:
  <input type="file" onchange="reencode(event.target.files[0])"></input>
  <div id="progress"></div>
</p>
```

We use the FileReader API to conver the file into an ArrayBuffer.

```javascript
var reader = new FileReader();
reader.onload = function() {
  // this.result is the ArrayBuffer
};
reader.readAsArrayBuffer(file);
```

mp4box's API is designed to work without loading the entire video file at once in memory since they can be very big. They designed the API with an appendBuffer method and then flush to process it.

```javascript
const mp4boxInputFile = MP4Box.createFile();
// ...
reader.onload = function() {
  this.result.fileStart = 0; // where in the file the buffer actually is
  mp4boxInputFile.appendBuffer(this.result);
  mp4boxInputFile.flush();
};
```

Once the first buffer is flushed, mp4box parses the headers and calls onReady with the parsed information. We can set it up to extract the first video track and start the extraction process.

```javascript
mp4boxInputFile.onReady = async (info) => {
  const track = info.videoTracks[0];
  mp4boxInputFile.setExtractionOptions(track.id);
  mp4boxInputFile.start();
};
```

which will give us all the samples from the buffers loaded at once.

```javascript
mp4boxInputFile.onSamples = async (track_id, ref, samples) => {
  for (const sample of samples) {
    // Do something with each sample
  }
};
```

## Decoding

The VideoDecoder uses a similar API as mp4box but this time it's not because it doesn't load the whole file into memory. The way video compression works is that some frames are full encoded images and some frames are just a delta of another frame, as images in video frequently are small variations of the nearby ones. I say nearby because there's something called "B-Frames" (bi-directional frames) that are deltas of both previous and next frames.

So the way the API is setup is that you send it multiple frames to decode at once and it's going to call the output function for all the frames it has enough information to decode all at once. In the end it's going to decode the same amount of frames, in the same order, but the order is not 1 frame in, 1 frame out.

We first start by creating a decoder object with its output callback. We can use createImageBitmap to get all the pixels of the frame. We need to call frame.close() to help with garbage collection since the images are very heavy and the browser vendors don't want to rely on JavaScript engines default garbage collection.

```javascript
decoder = new VideoDecoder({
  async output(frame) {
    const bitmap = await createImageBitmap(frame);
    frame.close();
  },
  error(error) {
    console.log(error);
  }
});
```

We feed the decoder all the samples wrapped in an EncodedVideoChunk and some options that need to be massaged from the mp4 file format to the generic one the browser wants.

```javascript
for (const sample of samples) {
  decoder.decode(new EncodedVideoChunk({
    type: sample.is_sync ? "key" : "delta",
    timestamp: 1e6 * sample.cts / sample.timescale,
    duration: 1e6 * sample.duration / sample.timescale,
    data: sample.data
  }));
}
```

So far, the APIs were a bit convoluted but it was fairly generic. Now we're going to need to do some H264-specific shenanigans. In order to decode a video frame, h264 has a bunch of configuration options called PPS (Picture Parameter Set) & SPS (Sequence Parameter Set). Their content isn't super interesting, you can [read on it here](https://www.cardinalpeak.com/blog/the-h-264-sequence-parameter-set). We need to find them and give them to the decoder.

The inside of mp4 files is structured as many nested boxes that contain JSON-like values (all encoded differently using a binary format). The box `trak.mdia.minf.stbl.stsd.avcC` contains the PPS and SPS configuraiton we are looking after. So we use the following piece of code to extract it out and pass it to the encoder.

```javascript
let description;
const trak = mp4boxInputFile.getTrackById(track.id);
for (const entry of trak.mdia.minf.stbl.stsd.entries) {
  if (entry.avcC || entry.hvcC) {
    const stream = new DataStream(undefined, 0, DataStream.BIG_ENDIAN);
    if (entry.avcC) {
      entry.avcC.write(stream);
    } else {
      entry.hvcC.write(stream);
    }
    description = new Uint8Array(stream.buffer, 8); // Remove the box header.
    break;
  }
}

decoder.configure({
  codec: track.codec,
  codedHeight: track.video.height,
  codedWidth: track.video.width,
  description,
});
```
