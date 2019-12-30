---
title: "Splitting media with FFmpeg"
date: 2019-12-30T11:47:10Z
draft: false
---

The below command[^1] tells ffmpeg to split into several chunks by i-frames. This means that each chunk can be played individually.

```
ffmpeg -i test.mp4 -f segment -segment_time chunk_length -reset_timestamps 1 -c copy test%02d.mp4
```

To explain the flags better, the command is annotated with additional newlines

```
ffmpeg -i test.mp4 \
  -f segment \
  -segment_time <chunk_length> \
  -reset_timestamps 1 \
  -c copy \
  test%02d.mp4
```

The input is test.mp4 specified by

```
-i test.mp4
```

The format is specified by `-f`. The segment format[^2] tells FFmpeg to output several segments.

```
-f segment
```

Segment time denotes approximately how long each of the segments are. The reason this is an approximation is because
FFmpeg will split the input at the i-frames (aka key frames). If an i-frame is not found at a particular timestamp,FFmpeg will continue seeking to the next i-frame. You can however force i-frame insertion with another option (-force_key_frames[^3]) so that each segment is the exactly the same length.

```
-segment_time <chunk_length>
```

Reset the timestamps of each segment to 0, this is meant to make playback easier.

```
-reset_timestamps 1
```

The `-c` flag is used to specify encoder or decoder. Remember that flags have different meaning when specified before
inputs or outputs. The `-c` flag is used to specify encoder when placed before the outputs, and decoder when specified before inputs.

The `-c copy` is a special codec value which means that the stream is simply copied without any re-encoding.

```
-c copy
```

The output files are zero padded and numbered.

```
test%02d.mp4
```


[^1]: https://stackoverflow.com/questions/33595056/dividing-processing-and-merging-files-with-ffmpeg
[^2]: https://ffmpeg.org/ffmpeg-formats.html#segment_002c-stream_005fsegment_002c-ssegment
[^3]: http://paulherron.com/blog/forcing_keyframes_with_ffmpeg