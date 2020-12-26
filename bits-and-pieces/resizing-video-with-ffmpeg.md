---
description: How to make a large video smaller
---

# Resizing video with ffmpeg

## Intro

I've recorded a video on a MacBook Pro 2019, 16'' in QuickTime player. It's about 100 Mb.

I wanted to convert it to something smaller without losing much quality.

## TLDR

Use this:

```text
ffmpeg -i input.mov -vcodec libx264 -crf 28 -f mov -acodec copy output.mov
```

## Details

First I've looked at this answer:

[https://unix.stackexchange.com/questions/28803/how-can-i-reduce-a-videos-size-with-ffmpeg](https://unix.stackexchange.com/questions/28803/how-can-i-reduce-a-videos-size-with-ffmpeg)

Which gives some command line using `ffmpeg`.

I've installed ffmpeg \(`brew install ffmpeg`\) and became interested in what codecs/containers it supports and what all this stuff generally means.

Also, by following advice from the article above I could convert my file easily from 100 Mb to 9 Mb using this command line:

```text
ffmpeg -i input.mov -vcodec libx265 -crf 28 output.mp4
```

However, when I tried to send it via WhatsApp, it was sent fine \(although I need to send it as a document, it's not recoginised when I try to send it as audio/video, i.e. the file is just not visible, so I can't select it as a video\), but when I tried to open it on iPhone SE, it didn't work.

So ok, I googled a little bit: iPhones should be opening .mp4 containers without any issues, so I was not sure why it didn't work.

Eventually it turned out that the problem was with H.265 codec \(which is more efficient, but less supported\), in fact for better support one should use more popular H.264 codec which is supported well on iPhones \(and presumably other devices\).

If I encode my files like this:

```text
ffmpeg -i input.mov -vcodec libx264 -crf 28 -f mov output.mov
```

It works fine, on macOS I can immediately see preview of the video, I can send it as a video in WhatsApp and it's played normally on iPhone.

Also, if you want to keep the sound as is, there is `-c:a copy` option \(which means "codec:audio = copy":

```text
ffmpeg -i input.mov -vcodec libx264 -crf 28 -f mov -acodec copy output.mov
```

## Useful command for inspecting files information

```text
ffprobe -show_streams
```

## Links

[https://ffmpeg.org/ffmpeg.html](https://ffmpeg.org/ffmpeg.html) -- ffmpeg doc

 [https://slhck.info/video/2017/02/24/crf-guide.html](https://slhck.info/video/2017/02/24/crf-guide.html) -- a nice description of CRF parameter with plots

 [https://en.wikipedia.org/wiki/Comparison\_of\_video\_container\_formats](https://en.wikipedia.org/wiki/Comparison_of_video_container_formats) -- containers and codecs big table

