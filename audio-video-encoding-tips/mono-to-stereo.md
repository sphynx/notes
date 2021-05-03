---
description: >-
  How to change a video file, so that you can hear the audio in both channels,
  even when it was recorded with mono equipment
---

# Mono to stereo

## Introduction

Normally when you record an audio or a video with an external microphone -- it just records one channel. For example this happens to me if I record a video using QuickTime player and use my external microphone plugged into an audio interface. Then, as a result, when you listen to that video with your headphones, you can just hear the sound in one ear: either left or right. This is of course, not what you want, you want to hear the same channel in both left and right ears.

I've tried to figure out how to do it using different tools: iMovie, DaVinci Resolve and command line `ffmpeg` tool. 

iMovie can't do it right now \(as per version 10.2.1\), in DaVinci Resolve you can do it, by changing the clip attributes \(setting both channels to the same input\), which is explained here: [https://www.youtube.com/watch?v=Iws-fPaMLfM](https://www.youtube.com/watch?v=Iws-fPaMLfM)

But I must admit that's it's rather an overkill: to install several gegabytes of a serious video editing tool DaVince Resolve to just do a simple thing like this. So I'd prefer to quickly do it with a couple of `ffmpeg` commands. However, as usual, `ffmpeg` options quite difficult to get right if you are not a regular user of that tool. So I had to spent some time on that, but at the end I think I've found a simple sequence of commands which I can understand and that do the job.

## ffmpeg way

We start with `i.mov` file which was recorded by QuickTime player \(File -&gt; New Movie Recording\) using an external microphone.

* First we need to extract the audio portion of that file:

```text
ffmpeg -i i.mov -vn -acodec copy orig.aac
```

This takes `i.mov` files, disables the video \(`-vn`\) and copies the audio \(`-acodec copy`\) and produces a new audio file `orig.aac` I am doing all of this on a Mac, so my videos are recorded using typical formats and codecs used by Apple.

* Now, we can look at this file and see the streams and their properties:

```text
> ffprobe -show_streams orig.aac
...
codec_name=aac
channels=2
channel_layout=stereo
...
```

You can see that interestingly enough, this is actualy a "stereo" file, i.e. it has two channels. Probably the second channel is just silence. This was actually causing the problems with other methods from StackOverflow or "Manipulating audio channels" [page](https://trac.ffmpeg.org/wiki/AudioChannelManipulation). I.e. I expected it to be a real mono, but in fact it was stereo with an empty channel.

* Now let's convert it to a real mono, since that's what it is. I do this using advice from "Manipulating audio channels" page above. I combine both channels into a single mono stream.

```text
ffmpeg -i orig.aac -ac 1 mono.aac
```

Let's look again at the result:

```text
> ffprobe -show_streams mono.aac
...
channels=1
channel_layout=mono
...
```

Now it's mono, and when players will play this, they will know that they need to play the same channel in both ears of the headphones. You can quickly try playing with, say, `ffplay mono.aac`

* Finally, we need to replace our original audio stream in our starting video file with this updated stream. We don't need to remix anything, since we haven't actually changed much, so we just copy video stream from the original file `i.mov` and copy audio stream from the `mono.aac` file to get the final result.

```text
ffmpeg -i i.mov -i mono.aac -c copy -map 0:v:0 -map 1:a:0 new.mov
```

This involves doing some mapping. The numeration there starts from zero, somewhat confusingly, but it just maps the video of the first stream \(`i.mov`\) to the first video stream of the output \(`-map 0:v:0`\), and the audio of the second stream to the first audio stream of the resulting file \(`-map 1:a:0`\).

That's it, you can just combine those commands into a handy shell script and use it to quickly fix your files.





