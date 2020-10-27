# `ffmpeg` and `imagemagick` cheatsheet
## `ffmpeg`
```sh
#/bin/sh
# get screen resolution
xdypinfo | grep dimensions | awk '{print $2;}' # 3840x1080`
####################################################################
#  get first frame of all the avi files in the directories recursively with same name and file type to jpg
## bash flags:
##      -maxdepth 1             # only at current directory
## ffmpeg flags: 
##      -n                      # skip process if output already exist
##      -vf                     # video frame, `-af` if it is audio frame
##      -select='eq(n\,1)'      # get 1st frame
##                              # if you want multiple frames: select='eq(n\,1)+eq(n\,200)+eq(n\,400)', 
##                              # `\,` is to escape `,`
##                              # `+` means OR, `*` means AND
##      -vframes number         # set the number of video frames to output
find . -type f -name "*.avi" -exec sh -c 'ffmpeg -n -i "$1" -vf "select=eq(n\,1)" -vframes 1 "${1%.avi}.jpg"' sh {} \;
####################################################################
#  record the whole screen, output output.mkv file
## bash flags:
##      xdpyinfo                # display information utility for X
## ffmepg flags:
##      -f fmt                  # force format
##      -s size                 # set frame size
ffmpeg -f x11grab -s $(xdpyinfo | grep dimensions | awk '{print $2;}') -i :0.0 out.mkv
####################################################################
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
## concatenate video
ffmpeg -i v1.mp4 -i v2.mp4 -filter_complex "[0:v] [1:v] concat=n=2:v=1 [v]" -map "[v]" out.mp4
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
```
---
## `imagemagick`
```sh
#/bin/sh
####################################################################
#  filename globbing
magick *.jpg images.gif
## imagemagick flags:
##      -resize 100x40          # resize to a dimension
##      -crop 40x30+10+10       # (width)x(height)+(x)+y
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

# make, merge a pdf
convert *jpg out.pdf
convert "*.{png,jpeg}" -quality 100 out.pdf
convert img{0..19}.jpg out.pdf

# add a colored border
convert -bordercolor red -border 5x5 flower.png flower-border.png
convert -mattecolor black -frame 5x5 beach.png beach-frame.png

# thumbnailing all the JPEGs in the current directory
for img in *.jpg
do
  convert -sample 25%x25% $img thumb-$img
done
####################################################################
# `montage` is to create a composite image by combining several separate images
## montage two images intoa single composite
montage -background `#336699` -geometry +4+4 rose.jpg red-ball.png montage.jpg
## add some decorations
## montage flags:
##       -label string           # assign a label to an image
##       -frame geometry	        # surround image with an ornamental border
##       -background color       # background color
montage -label %f -frame 5 -background '#336699' -geometry +4+4 rose.jpg red-ball.png frame.jpg
## append image side-by-side
montage [0-5].png -tile 5x1 -geometry +0+0 out.png
####################################################################
# `mogrify` is a more compact command for converting all files of one type
mogrify -format png *.bmp       # convert all images(bmp) to png
mogrify -format jpg -quality 85 *.png
# thumbnailing all the jpg in current directory
mogrify -sample 25%x25% *.jpg   
mogrify -format png -sample 25%x25% *.jpg
```