<!-- vscode-markdown-toc -->
* 1. [Quick Shell Cheatsheet](#QuickShellCheatsheet)
* 2. [`ffmpeg`](#ffmpeg)
	* 2.1. [video](#video)
	* 2.2. [audio](#audio)
	* 2.3. [video and audio](#videoandaudio)
	* 2.4. [AVFilter](#AVFilter)
		* 2.4.1. [Rules](#Rules)
		* 2.4.2. [Filter Syntax](#FilterSyntax)
		* 2.4.3. [`=parameters` format](#parametersformat)
		* 2.4.4. [filterchain syntax](#filterchainsyntax)
		* 2.4.5. [Examples](#Examples)
* 3. [`imagemagick`](#imagemagick)
	* 3.1. [`mogrify`](#mogrify)
	* 3.2. [`convert` and `magick`](#convertandmagick)
	* 3.3. [`montage`](#montage)
	* 3.4. [`identify`](#identify)
* 4. [`pdftk`](#pdftk)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

##  1. <a name='QuickShellCheatsheet'></a>Quick Shell Cheatsheet
```sh
#/bin/sh
# make sure your filename have spaces in replacement in two ways
-exec <command> '{}' \;
| xargs -I{}
####################################################################
# get screen resolution
xdypinfo | grep dimensions | awk '{print $2;}'  # 3840x1080
```

##  2. <a name='ffmpeg'></a>`ffmpeg`
###  2.1. <a name='video'></a>video
```sh
#  crop the video start point 03sec and last 24sec from that
## crop video to 1080p on the right side screen
## ffmpeg flags:
##      -ss timer_out           # set the staart time offset
##      -t duration             # record or transcode duration seconds of audio/video
##      -filter filter_graph    # set frame filtergraph
ffmpeg -ss 00:00:03.0 -i out.mkv -t 24 -filter:v "crop=1920:1080:1920:0" out.mp4
####################################################################
## 2x speed
ffmpeg -i in.mp4 -filter:v "setpts=0.5*PTS" out.mp4
## 0.5x speed
ffmpeg -i in.mp4 -filter:v "setpts=2.0*pts" out.mp4
## reverse video
ffmpeg -i in.mp4 -vf reverse rev.mp4
####################################################################
## scale image
ffmpeg -i in.png -vf scale=iw/2:-1 out.png # width/2
ffmpeg -i in.png -vf scale=-1:ih/2 out.png # height/2
####################################################################
## crop the video resolution (width:height:start_width:start_height)
ffmpeg -i in.mp4 -filter:v "crop=1920:1020:1920:30" out.mp4
####################################################################
## convert avi to mp4
ffmpeg -i in.avi -c:v libx264 out.mp4
## convert and compress video a lot
ffmpeg -i in.mp4 -c:v libx264 0.5M out.mp4
####################################################################
## generate gif from mp4
## the more compressed video is, gif will be larger
ffmpeg -i out1.mp4 -vf scale=1080:-1 out.gif
####################################################################
## concatenate video
ffmpeg -i v1.mp4 -i v2.mp4 -filter_complex "[0:v] [1:v] concat=n=2:v=1 [v]" -map "[v]" out.mp4
## merge two video clips into one, placing them next to each other
## ffmepg flags:
##      -filter_complex hstack  # place each video horizontally
##                              # streams must be of same pixel format and of same height
##                              # this filter is faster than using overlay
##                              # and pad filter to create same output
##      -hstack=inputs=3        # stack more than 2 videos
##      -filter_complex vstack  # place each video vertically
ffmpeg -i left.mp4 -i right.mp4 -filter_complex hstack output.mp4

ffmpeg -i activity.mp4 -i mobile.mp4 -filter_complex \
" nullsrc=size=2560x720 [base]; \
 [0:v] setpts=PTS-STARTPTS, scale=1280x720 [upperleft]; \
 [1:v] setpts=PTS-STARTPTS, scale=1280x720 [upperright]; \
 [base][upperleft] overlay=shortest=1 [tmp1]; \
 [tmp1][upperright] overlay=x=720" -c:v libx264 output.mp4
####################################################################
#  get first frame of all the avi files in the directories recursively with same name and file type to jpg
## bash flags:
##      -maxdepth 1             # only at current directory
## ffmpeg flags:
##      -n                      # skip process if output already exist
##      -vf                     # video frame, `-af` if it is audio frame
##      -select='eq(n\,1)'      # get 1st frame
##                              # if you want multiple frames: select='eq(n\,1)+eq(n\,200)+eq(n\,400)'
##                              # `\,` is to escape `,`
##                              # `+` means OR, `*` means AND
##      -vframes number         # set the number of video frames to output
find . -type f -name "*.avi" -exec sh -c 'ffmpeg -n -i "$1" -vf "select=eq(n\,1)" -vframes 1 "${1%.avi}.jpg"' sh {} \;
```
###  2.2. <a name='audio'></a>audio
```sh
## change speed of audio
## ffmpeg flags:
##      atempo                   # limit to [0.5,2.0]
##                               # series multiple atempo to have more than its limit
ffmpeg -i input.mkv -filter:a "atempo=2.0" -vn output.mkv             ## 2x speed of audio
ffmpeg -i input.mkv -filter:a "atempo=2.0,atempo=2.0" -vn output.mkv  ## 4x speed of audio

## prepone/postpone audio
## ffmpeg flags:
##      -filter_complex "adelay=3000|3000" # add 3000ms delay to both audio channels
ffmpeg -i ogg.ogg -i 1.avi -filter_complex "adelay=3000|3000" output.avi
```
###  2.3. <a name='videoandaudio'></a>video and audio
```sh

#  record the whole screen, output output.mkv file
## bash flags:
##      xdpyinfo                # display information utility for X
## ffmepg flags:
##      -f fmt                  # force format
##      -r rate                 # -r 30: 30fps
##      -s size                 # set frame size
##      -i :0.0                 # your current screen
##      -f alsa -i default      # record sound
##      -c:v libx264            # video codec
##      -c:a flac               # audio codec
ffmpeg -f x11grab -s $(xdpyinfo | grep dimensions | awk '{print $2;}') -i :0.0 -f alsa -i default -c:v libx264 -r 30 -c:a flac out.mkv
####################################################################
## ffmepg flags:
##      -map [v]                # [v] is a linklabel, a defined output in the filter
##                              # graph(the line above with filter_complex
##      [0:v:0]                 # video stream 0 from input 0
##      [1:v:0]                 # video stream 0 from input 1
##      [0:a:0]                 # audio stream 0 from input 0

## multiple filters, change video(x2) and audio(x2) speed
fmpeg -i input.mkv -filter_complex "[0:v]setpts=0.5PTS[v];[0:a]atempo=2.0[a]" \
 -map "[v]" -map "[a]" output.mkv

## concatenate two videos side-by-side, merge audio, make it stereo
## ffmepg flags:
##      -amerge                 # combine the audio from both inputs
##                              # into a single, multichannel audio stream
##      -ac 2                   # make it stereo
##                              # without this option the audio stream may
##                              # end up as 4 channels if both inputs are stereo
ffmpeg -i left.mp4 -i right.mp4 -filter_complex \
"[0:v][1:v]hstack=inputs=2[v]; \
 [0:a][1:a]amerge[a]" \
-map "[v]" -map "[a]" -ac 2 output.mp4

## put four videos in one 2x2 grid
ffmpeg -i in1.mp4 -i in2.mp4 -i in3.mp4 -i in4.mp4 -filter_complex \
"[0:v]pad=iw*2:ih*2[a];[a][1:v]overlay=w[b];[b][2:v]overlay=0:h[c];[c][3:v]overlay=w:h" out.mp4
####################################################################
## merge audio and video
ffmpeg -i a.wav -i a.avi out.avi
## remove video sound 
ffmpeg -i input.avi -vcodec copy -an output.avi
## remove video and  keep sound
ffmpeg -i input.mp4 -avodec copy -vn output.wav
```

###  2.4. <a name='AVFilter'></a>AVFilter
Similar to __DirectShow__ filter. A filter without input is a source filter, a filter without output is a sink filter.
####  2.4.1. <a name='Rules'></a>Rules
- separate each __Filter__ with `,`
- separate every filter's __attributes__ with `:`
- link attribute and filter with `=`
- form a __FilterChain__ with multiple __Filters__, seperate with `;`, e.g. `-vf`, `-af`, `-filter_complex`
- `-filter_complex` is used for multiple input sources
- `-vf`, `-af` is used for single input video, audio source

####  2.4.2. <a name='FilterSyntax'></a>Filter Syntax
```
[in_link_1]...[in_link_N]filter_name=parameters[out_link_1]...[out_link_2]
```
`[in_link_N]`,`[out_link_N]` used to label input and output, can use anyname and put bracket around it.
####  2.4.3. <a name='parametersformat'></a>`=parameters` format

```sh

## use `:` to separate one key-value pair
ffmpeg -i input -vf scale=w=iw/2:h=ih/2 output
## use `:` to separate values
ffmpeg -i input -vf scale=iw/2:ih/2 output
ffmpeg -i input -vf scale=iw/2:h=ih/2 output
```
####  2.4.4. <a name='filterchainsyntax'></a>`FilterChain` syntax
```
"filter1, filter2, ... filterN-1, filterN"
```
separate multiple filters with `;`
```sh
[in]split[main][tmp];[tmp]crop=iw:ih/2,vflip[flip];[main][flip]overlay=0:H/2[out]
```
__Explanation:__
- Input stream `[in]` is been split into two stream `[main]` and `[tmp]`
- For `[tmp]`, it changes to `[flip]` after two filters: crop and vflip
- Overlay `[flip]` and `[main]` onto `out`
- Every node is a __filter__, e.g. crop, vflip
- Word wrap with bracket is __FilterPad__, e.g. tmp. main, flip
```
# filtergraph
                [main]
in    --> split ---------------------> overlay --> out
            |                             ^
            |[tmp]                  [flip]|
            +-----> crop --> vflip -------+
```
####  2.4.5. <a name='Examples'></a>Examples
```sh
# add a watermark image onto in.flv at pixel (5,5)
ffmpeg -i in.flv -vf movie=annotate.jpg[wm];[in][wm]overlay=5:5[out] out.flv
# add a watermark input.png onto in.mkv
## if each input source only have one stream
ffmpeg -i in.mkv -i input.png -filter_complex 'overlay' out.mkv
## ffmepg flags:
##      -map                    # map is for label
ffmpeg -i in.mkv -i input.png -filter_complex 'overlay[out]' -map'[out]' out.mkv
## if each input source only have multiple streams
ffmpeg -i in.mkv -i input.png -filter_complex '[0:v][1:v]overlay[out]' -map'[out]' out.mkv
####################################################################
# mirror left half of an video, pad the frame
ffmpeg -i in.flv -vf crop=iw/2:ih:0:0,split[left][tmp];[tmp]hflip[right];[left]pad=iw*2[a];[a][right]overlay=w out.flv
####################################################################
# modify curve to apply filter
ffmpeg -i in.flv -vf curves=vintage out.flv
ffmpeg -i in.flv -vf curves=psfile='test.acv':green='0.45/0.53' out.flv
```
---
##  3. <a name='imagemagick'></a>`imagemagick`
###  3.1. <a name='mogrify'></a>`mogrify`
`mogrify` is a more compact command for converting all files of one type
```sh

mogrify -format png *.bmp       # convert all images(bmp) to png
mogrify -format jpg -quality 85 *.png
# thumbnailing all the jpg in current directory
mogrify -sample 25%x25% *.jpg
mogrify -format png -sample 25%x25% *.jpg
## make all png in current directory 2x larger and put under /newdir
mogrify -path newdir -resize 200% *.png
## add one watermark
mogrify -pointsize 16 -fill black -weight bolder -gravity southeast -annotate +5+5 "danyu.li" *.jpg
```
###  3.2. <a name='convertandmagick'></a>`convert` and `magick`
```sh
#  filename globbing
magick *.jpg images.gif
## imagemagick flags:
##      -resize 100x40          # resize to a dimension, (width)x(height)
##      -crop 40x30+10+10       # (width)x(height)+(start_hori)+start_ver
##      -flip                   # vertical
##      -flop                   # horizontal
##      -transpose              # flip vertical + rotate 90deg
##      -transverse             # flip horizontal + rotate 270deg
##      -trim                   # trim image edges
##      -rotate 90              # rotate 90deg
####################################################################
#  make all images monochrome
## in bash shell
for file in *.png
do
    convert $file -colorspace Gray -separate -average $file $file
done
## in fish shell
for file in *.jpg; convert -colorspace Gray $file $file;end;

# make image monochrome
convert -monochrome in.gif out.gif

# images -> pdf
# make, merge a pdf
convert *jpg out.pdf
convert "*.{png,jpeg}" -quality 100 out.pdf
convert img{0..19}.jpg out.pdf

# pdf -> image
## convert flags:
##    -density number         # specify output image DPI, on Mac, default is 72, not clear
##    -flatten                # when transfer from pdf to jpg, it can have black background
##                            # this flag choose white background, but won't have multiple images
convert -density 150 -flatten  'download.pdf[0]'  first_page.jpg
## convert pdf -> images
convert -density 150 -background white -alpha remove download.pdf download.jpg

# add a colored border
convert -bordercolor red -border 5x5 flower.png flower-border.png
convert -mattecolor black -frame 5x5 beach.png beach-frame.png
########### add watermark ###################################
## add one transparent watermark onto the image
## convert flags:
##    -draw "text x,y,string" # add text, can also draw circle etc
##    -fill color             # choose the color of the text
convert in.jpg -gravity southeast -fill 'rgba(221, 34, 17, 0.25)' \
-pointsize 36 -draw "text 5,5 'danyu.li'" out.jpg

## add multiple transparent watermark onto the image
## convert flags:
##    -size                   # canvas size
##    xc                      # X constant image, same as canvas
##                            # e.g. xc: color, none, transparent
##                            # to define the canvas color, default is white
##    -resize                 # shrink to how many percent of original image
##                            # when -pointsize <14, -draw rotate won't work
##    miff:-                  # miff: declare middle piped image format as MIFF
##                            # -: put before pipe meaning IM result as output
##                            #    put after pipe meaning get from output
##    -tile                   # tile spread around
##    -desolve                # transparent level
convert  -size 200x200  xc:none  -fill '#d90f02'  -pointsize 18  \
-gravity center  -draw 'rotate -45 text 0,0 "danyu.li"'  \
-resize 60%  miff:-  |  composite  -tile  -dissolve 25  -  in.jpg  out.jpg


# thumbnailing all the JPEGs in the current directory
for img in *.jpg
do
  convert -sample 25%x25% $img thumb-$img
done
# add a witdh=5, length=10, red border to input
convert -bordercolor red -border 5x10 input.png output.png

# put 4 images into a 2x2 grid
convert ^
  ( a.jpg b.jpg +append ) ^
  ( c.jpg d.jpg +append ) ^
  -append ^
  out.jpg
convert -background none                               \
     1.jpg -resize 1680x1050!                          \
  \( 2.jpg -resize 1920x1200! \) +append               \
  \( 3.jpg -resize 1920x1080! -gravity east \) -append \
  result.png
```
###  3.3. <a name='montage'></a>`montage`
`montage` is to create a composite image by combining several separate images
```sh
## montage two images into a single composite
montage -background `#336699` -geometry +4+4 rose.jpg red-ball.png montage.jpg
## add some decorations
## montage flags:
##       -label string           # assign a label to an image
##       -frame geometry	        # surround image with an ornamental border
##       -background color       # background color
montage -label %f -frame 5 -background '#336699' -geometry +4+4 rose.jpg red-ball.png frame.jpg
## append image side-by-side
montage [0-5].png -tile 5x1 -geometry +0+0 out.png
```
###  3.4. <a name='identify'></a>`identify`
`identify` can identify images' formats and characteristics
```sh
identify *
identify abc.jpg
identify -verbose abc.jpg
## gives
## abc.jpg JPEG 1952x3264 1952x3264+0+0 8-bit DirectClass 1.111MB 0.000u 0:00.000

identify -format "%f: %wx%h\n" *.jpg *.gif *.png
## gives
## 7T6Dj.jpg: 1920x1080
## a.jpg: 400x463
## b.jpg: 400x463
## back.jpg: 906x603
## background.jpg: 906x603

## recursive list dimensions of all images in a folder recursively
find . -name "*.jpg" -exec identify {} \;

## find all images with a certain pixel size
## '%w %h %i' gives the width, height, full pathname of the image
find . -iname "*.jpg" -type f -exec identify -format '%w %h %i' '{}' \; | awk '$1<300 || $2<300'
find . -iname "*.jpg" -type f | xargs -I{} identify -format '%w %h %i' {} | awk '$1<300 || $2<300'
find . -iname "*.png" -type f -exec identify -format '%i %wx%h\n' '{}' \; | grep '75x75'
```

### `composite`
### `display`
If you have a x server system, it can display and change images
### `animate`
If you have a x server system, it can display animate images
### `compare`
Compare different images mathmatically and change its property
##  4. <a name='pdftk'></a>`pdftk`
```sh
# get page 42, 43 from input.pdf into a new pdf
pdftk A=input.pdf cat A42 A43 output pages_42_43.pdf
pdftk original.pdf stamp link.pdf output final.pdf
```

```sh
convert -channel rgba -pointsize 100 -strokewidth 2 logo: ^( -background transparent -stroke #000b -fill #000a label:"mark" -blur 3x3 -stroke none  -fill #ffff label:"mark" -flatten ) -compose dissolve -define compose:args=50 -composite -quality 100 test.jpg
```