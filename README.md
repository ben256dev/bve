---
title: Benjamin's Video Editor
---

# Benjamin's Video Editor

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

### Overlays

Overlays are necessary for any kind of layering of video or images.

```bash
ffmpeg \
    -framerate 60 -i png/clipA_%06d.png \
    -i png/trollface.png \
    -filter_complex "[0:v][1:v]overlay=x=main_w/2-overlay_w/2:y=main_h/2-overlay_h/2[out]" \
    -map "[out]" -c:v libx264 -pix_fmt yuv420p output.mp4
```

- ``overlay=x=main_w/2-overlay_w/2:y=main_h/2-overlay_h/2`` Centers the image on the screen
- ``main_w`` and ``main_h`` are the **width** and **height** of the background
- ``overlay_w`` and ``overlay_h`` are the **width** and **height** of the foreground image

I this gets the point across, but I say we implement a *real* transition.

### Combining Effects

Lets make the trollface appear by zooming in while simultaneously fading in. You'll get the idea once you render the result.

```bash
ffmpeg \
    -framerate 60 -i png/clipA_%06d.png \
    -loop 1 -i png/trollface.png \
    -filter_complex "
        [1:v]format=rgba,
            scale='iw*min(1, 0.2+0.8*t/0.75)':'ih*min(1, 0.2+0.8*t/0.75)':eval=frame,
            fade=t=in:st=0:d=0.75:alpha=1,
            setpts=PTS-STARTPTS[ovr];

        [0:v][ovr]overlay=x='(main_w-overlay_w)/2':y='(main_h-overlay_h)/2':shortest=1
    " \
    -c:v libx264 -pix_fmt yuv420p output.mp4
```

- ``t`` is a global time variable.
- ``min(1, 0.2+0.8*t/0.75)`` is the calculation that does a linear crash-in effect for the scale.
- ``loop 1`` is necessary for our image input because unlike previously if we don't specify that our image loops, then only one frame will be drawn. To prevent issues arising from looping we also use ``setpts=PTS-STARTPTS[ovr];``

There are some really big issues right now. It would be more consistent if we used an equation to set the alpha similar to what we did with ``scale``. This isn't possible unfortunately. We also can't really use custom easing functions as a result of the limitations of these ffmpeg transitions. That's where ``ffmpeg-gl-transition`` comes in.

## xfade-easing Repository

I had a lot of pain trying to build [this library](https://github.com/scriptituk/xfade-easing?tab=readme-ov-file). Essentially it is a header library plus a patch and requires you to build ffmpeg yourself with a few of the changes and the new header file. It requires different flags from the regular ffmpeg compilation process. A good start is getting all the dependencies installed from the [ffmpeg compilation guide](https://trac.ffmpeg.org/wiki/CompilationGuide).

The dependency ``SVT-AV1`` required specific build steps for a compatible version:

```bash
cd ~/ffmpeg_sources
git clone https://gitlab.com/AOMediaCodec/SVT-AV1.git
cd SVT-AV1
git checkout v1.4.1
rm -rf build
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
make install
```

I wrote a giant script during my troubleshooting process to replicate my steps easier. Here it is:

```bash
#!/bin/bash

mkdir -p ~/ffmpeg_sources ~/bin && \
cd ~/ffmpeg_sources && \
wget -O xfade-easing.tar.gz https://github.com/scriptituk/xfade-easing/archive/refs/tags/v3.4.1.tar.gz && \
mkdir -p xfade-easing && \
tar -xzf xfade-easing.tar.gz -C xfade-easing --strip-components=1 && \
wget -O ffmpeg-snapshot.tar.xz https://ffmpeg.org/releases/ffmpeg-7.1.1.tar.xz && \
mkdir -p ffmpeg && \
tar -xf ffmpeg-snapshot.tar.xz -C ffmpeg --strip-components=1 && \
cd ffmpeg && \
rm libavfilter/vf_xfade.c && \
cp ../xfade-easing/src/vf_xfade.c libavfilter && \
cp ../xfade-easing/src/xfade-easing.h libavfilter && \
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --extra-libs="-lpthread -lm" \
  --ld="g++" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-gnutls \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libfreetype \
  --enable-libmp3lame \
  --enable-libopus \
  --enable-libsvtav1 \
  --enable-libdav1d \
  --enable-libvorbis \
  --enable-libvpx \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nonfree && \
PATH="$HOME/bin:$PATH" make ECFLAGS=-Wno-declaration-after-statement && \
make ECFLAGS=-Wno-declaration-after-statement install && \
hash -r
```

To verify a successful install, we run ``ffmpeg -hide_banner --help filter=xfade | grep easing``:

<pre><font color="#55FF55"><b>user@shell</b></font>:<font color="#5555FF"><b>~</b></font>$ <code class="nohighlight">ffmpeg -hide_banner --help filter=xfade | grep easing</code>
   <font color="#FF5555"><b>easing</b></font>            &lt;string&gt;     ..FV....... set cross fade <font color="#FF5555"><b>easing</b></font>
   reverse           &lt;int&gt;        ..FV....... reverse <font color="#FF5555"><b>easing</b></font>/transition (from 0 to 3) (default 0)</pre>

![wipedown demo](https://raw.githubusercontent.com/scriptituk/xfade-easing/e9d5b1c26cbe8f48116e40bd283964fa38e80cf8/assets/wipedown-cubic.gif)

The above is an example from the ``xfade-easing`` github.

```bash
ffmpeg -i first.mp4 -i second.mp4 -filter_complex "
    xfade=duration=3:offset=1:easing=cubic-in-out:transition=wipedown
    " output.mp4
```

Lets try our own version using our frame sequence.

```bash
ffmpeg \
  -framerate 60 -i png/clipA_%06d.png \
  -framerate 60 -i png/clipB_%06d.png \
  -filter_complex "xfade=transition=wipedown:duration=3:offset=1:easing=cubic-in-out" \
  -pix_fmt yuv420p output.mp4
```

![xfade-easing cupic wipe test](https://ben256.com/b/fb5e7082e1afc8e0278b6ca532c81411c18e1e7533e90cc3c3210476b6b278bd/wipetest.gif)

It works!

<!-- FOOTER:
-->
