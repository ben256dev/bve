---
title: Bens Video Editor
---

# Ben's Video Editor

## Learning ffmpeg

Lets decode the video of a clip into pngs and the audio into wav.

```bash
mkdir -p png

# first the video
ffmpeg -ss 00:00:00 -to 00:00:01 -i https://ben256.com/b/9494ce0c5cd4099458f7dfa4d6e205d1492dd00e08b05d00812ab3c650f1bb8a/reddit_video_017.mp4 \
    -vf fps=60 -start_number 0 png/clipA_%06d.png

# next the audio
ffmpeg -ss 00:00:00 -to 00:00:01 -i https://ben256.com/b/9494ce0c5cd4099458f7dfa4d6e205d1492dd00e08b05d00812ab3c650f1bb8a/reddit_video_017.mp4 \
    -vn -acodec pcm_s16le clipA.wav
```

Omitting either the ``-ss`` or ``-to`` flag will default to the respective start and end of the source clip.

Lets decode another clip so we can combine them.

```bash
ffmpeg -ss 00:00:00 -to 00:00:01 -i https://ben256.com/b/8fa9a67b0e08318e749a74378300a630f8dfb4a716e382d04ba6e39554c5c079/reddit_video_031.mp4 \
    -vf fps=60 -start_number 0 png/clipB_%06d.png

ffmpeg -ss 00:00:00 -to 00:00:01 -i https://ben256.com/b/8fa9a67b0e08318e749a74378300a630f8dfb4a716e382d04ba6e39554c5c079/reddit_video_031.mp4 \
    -vn -acodec pcm_s16le clipB.wav
```

Finally, this one-liner will combine the two sets of pngs and the two wav files.

```bash
ffmpeg \
    -framerate 60 -start_number 0 -i png/clipA_%06d.png \
    -framerate 60 -start_number 0 -i png/clipB_%06d.png \
    -i clipA.wav -i clipB.wav \
    -filter_complex "
        [0:v][1:v]concat=n=2:v=1:a=0[v];
        [2:a][3:a]concat=n=2:v=0:a=1[a]
    " \
    -map "[v]" -map "[a]" -c:v libx264 -c:a aac -pix_fmt yuv420p output.mp4
```

However, when we try to combine the two urls above, we get this error:

```bash
[Parsed_concat_0 @ 0x5e68f9b22b00] Input link in0:v0 parameters (size 1274x720, SAR 1912:1911) do not match the corresponding output link in0:v0 parameters (1280x666, SAR 3713:3716)
```

ffmpeg isn't going to automatically rescale videos with different aspect ratios, we have to specify one manually. Lets specify ``1080x1920`` as our resolution. We can force it by decoding the pngs to be in that resolution in the first place:

```bash
# in our decoding png commands from earlier, -vf fps=60 becomes...
...
-vf "fps=60,scale=1920:1080:force_original_aspect_ration=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1" \
# rest of the command remains the same
```

- ``scale=1920:1080:force_original_aspect_ratio=decrease`` scales the input to fit our desired resolution *without* stretching.
- ``pad=1920:1080:(ow-iw)/2:(oh-ih)/2`` adds black bars if necessary. The image should be centered no matter what.
- ``setsar=1`` is important because it ensures that our pixels are always "square" i.e. their height and width are equal.

We can also make the clips stretch to fit:

```bash
-vf "fps=60,scale=1920:1080,setsar=1"
```

In the case of the clips we are using, stretching is more optimal, but it will depend upon the clips.

Lets experiment with rendering portions of our frames. Instead of the ``-ss`` and ``-to`` flags we can decode the whole clips and only render portions. To do this we need the ``trim`` filter for video and ``atrim`` for audio. We also shouldn't forget to re-demux our audio into .wav to get the full audio for each clip.

```bash
ffmpeg \
    -framerate 60 -start_number 0 -i png/clipA_%06d.png \
    -framerate 60 -start_number 0 -i png/clipB_%06d.png \
    -i clipA.wav -i clipB.wav \
    -filter_complex "
        [0:v][1:v]concat=n=2:v=1:a=0[v];
        [2:a][3:a]concat=n=2:v=0:a=1[a];
        [v]trim=start=10:end=12,setpts=PTS-STARTPTS[vout];
        [a]atrim=start=10:end=12,asetpts=PTS-STARTPTS[aout];
    " \
    -map "[vout]" -map "[aout]" -c:v libx264 -c:a aac -pix_fmt yuv420p output.mp4
# above we are using "vout" and "aout" instead of original "v" and "a"
```

Rendering is far faster in this case, so it is preferred in many cases. Although, it can be a bit clunky.

The above rendering command *works* but its not ideal. There is a simpler way to trim without post-processing.

```bash
ffmpeg \
    -framerate 60 -start_number 0 -i png/clipA_%06d.png \
    -framerate 60 -start_number 0 -i png/clipB_%06d.png \
    -i clipA.wav -i clipB.wav \
    -filter_complex "
        [0:v][1:v]concat=n=2:v=1:a=0[v];
        [2:a][3:a]concat=n=2:v=0:a=1[a]
    " \
    -map "[v]" -map "[a]"  -ss 10 -to 12  -c:v libx264 -c:a aac -pix_fmt yuv420p output.mp4
#                         ╰───── * ─────╯
```

Note the ``*`` section that was added. If we want to use frames instead of seconds we can use fractions also. Just divide the frame by the fps.

ffmpeg -i https://ben256.com/b/9494ce0c5cd4099458f7dfa4d6e205d1492dd00e08b05d00812ab3c650f1bb8a/reddit_video_017.mp4 \
    -vf fps=60 -start_number 0 png/clipA_%06d.png
ffmpeg -i https://ben256.com/b/9494ce0c5cd4099458f7dfa4d6e205d1492dd00e08b05d00812ab3c650f1bb8a/reddit_video_017.mp4 \
    -vn -acodec pcm_s16le clipA.wav
ffmpeg -i https://ben256.com/b/8fa9a67b0e08318e749a74378300a630f8dfb4a716e382d04ba6e39554c5c079/reddit_video_031.mp4 \
    -vf fps=60 -start_number 0 png/clipB_%06d.png
ffmpeg -i https://ben256.com/b/8fa9a67b0e08318e749a74378300a630f8dfb4a716e382d04ba6e39554c5c079/reddit_video_031.mp4 \
    -vn -acodec pcm_s16le clipB.wav

### Cross fades

Here we process the audio and video seperately because it is more reliable. We will recombine them later.

```bash
ffmpeg -framerate 60 -ss 3 -i png/clipA_%06d.png -framerate 60 -i png/clipB_%06d.png \
-filter_complex "
    [0:v]format=yuv420p,setsar=1[v0];
    [1:v]format=yuv420p,setsar=1[v1];
    [v0][v1]xfade=transition=fade:duration=1:offset=6[v]
" -map "[v]"  out_video.mp4
```

In the ``xfade`` parameters we set ``duration`` to 1 and the ``offset`` to 6. For the inputs, we start he first clip 3 seconds in with the ``-ss`` flag. Changing where the clip starts won't have an effect on the duration the clip is on screen so long as the clip is long enough, but setting the duration of the fade to 1 and the offset to 6 means that the clip will be on screen for a total of 7 seconds. This is important to bear in mind for fading the audio.

```bash
ffmpeg -ss 3 -t 7 -i clipA.wav -i clipB.wav \
    -filter_complex "[0:a][1:a]acrossfade=d=1:c1=tri:c2=tri[a]" \
    -map "[a]" out_audio.wav
```

The audio works the same except for ``acrossfade`` we can't specify an offset. Therefore, we should specify the duration of the first audio file with ``-t`` as the 7 seconds we calculated earlier. This time we set ``d`` for duration.

Finally we can combine the video and audio:

```bash
ffmpeg -i out_video.mp4 -i out_audio.wav -c:v copy -c:a aac final.mp4
```
