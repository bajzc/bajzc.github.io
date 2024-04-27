---
category: |
  Review
date: "2023-08-12 16:42:50 UTC+07:00"
slug: |
  vlc-how-to-generate-m3u-playlists
tags:   ["VLC"]
title: "VLC: how to generate .m3u playlists"
---

![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e6/VLC_Icon.svg/1920px-VLC_Icon.svg.png){.align-center
width="100px"}

# VLC: how to generate .m3u playlists {#vlc-how-to-generate-.m3u-playlists-1}

::: contents
:::

:::: note
::: title
Note
:::

The new C language version can be found here: (30.12.2023 update)

It\'s 6x faster than the bash script (1:52/11:08)

<script src="https://gist.github.com/bajzc/9f3d06154fee7c1f527654672b59a5cf.js"></script>
::::

VLC is a free and open-source media player, and it is PORTABLE and
CROSS-PLATFORM.

In order to have a better user experience, without paying a penny, it is
the best choice.

> \"M3U (MP3 URL or Moving Picture Experts Group Audio Layer 3 Uniform
> Resource Locator in full) is a computer file format for a multimedia
> playlist. One common use of the M3U file format is creating a
> single-entry playlist file pointing to a stream on the Internet. The
> created file provides easy access to that stream and is often used in
> downloads from a website, for emailing, and for listening to Internet
> radio.\"
>
> \-\-- Wikipedia

As .m3u is one of the playlist format that VLC supports, and its format
can be easily followed by a shell script, I chose it to make a list of
all my Internet media files.

## 1 Syntax of .m3u

An .m3u file is a plain text file that specifies the locations of one or
more media files. Each entry carries one specification, which can be:

1.  an absolute local path:`C:\Music\I_love_this_one.mp3`
2.  a relative path: `This_one_as_well.mp4`
3.  an URL: `www.music.com/123.mkv`

The extended .m3u syntax we will use in the script: 1. #EXTM3U: file
header, should be the first line 2. #EXTINF: track information:
`#EXTINF:123,Title`

All the syntax for .m3u can be checked on
[Wikipedia](https://en.wikipedia.org/wiki/M3U).

## 2 My file system

`tree Music -L 1`

    Music/
    ├── Dr._Dre
    │   ├── 02 - Deep Cover.flac
    │   ├── ...
    ├── Imagine_Dragons
    ├── LOL_BGM
    ├── OneRepublic
    ├── The_Fat_Rat
    └── The_Music_of_Grand_Theft_Auto_V

And all those files are published on the Internet, so they can be
accessed by **http://ipv4.bajzc.com:81/Music/\...** (use port 81 as the
NSP blocked the default port: 80)

## 3 Script

:::: note
::: title
Note
:::

This script is only fit for my needs, you can modify it by following the
explanation for each function.
::::

``` bash
#! /bin/bash
set -e
export ListPath=/var/www/html/PlayLists
Backup(){
  cd $ListPath
  for name in $(ls -c)
  do
    mv $name ./.$name.bak
  done
}

ListMovie(){
cd $ListPath/../
echo "#EXTM3U" >> $ListPath/ALL.m3u
find Movie/ -mindepth 1 -maxdepth 2 | while read line
do
    if [ -d "$line" ]; then
    echo -n "#EXTGRP " >> $ListPath/ALL.m3u
    echo $(echo "$line" | sed -e "s/\([[:print:]]*\/\)*//g") >> $ListPath/ALL.m3u
  else
    duration=$(ffmpeg -i "$line" 2>&1 | grep "Duration"| cut -d ' ' -f 4 | sed s/,// | sed 's@\..*@@g' | awk '{ split($1, A, ":"); split(A[3], B, "."); print 3600*A[1] + 60*A[2] + B[1] }')
    if [ "$duration" == "0" ] || [ "$duration" == "" ]; then
      printf "$line is not a media file, duration: $duration\n"
      continue
    fi
        echo -n "#EXTINF:$duration," >> $ListPath/ALL.m3u
    echo $(echo "$line" | sed -e "s/\([[:print:]]*\/\)*//g") >> $ListPath/ALL.m3u
    echo $(echo "$line" | sed -e 's/^/http:\/\/ipv4.bajzc.com:81\//') >> $ListPath/ALL.m3u
    echo >> $ListPath/ALL.m3u
    fi
done
}

ListAlbum(){
  cd /var/www/html/
  find Music/ -maxdepth 1 -mindepth 1 -type d | while read line
  do
    Album=$(echo "$line" | sed -e "s/\([[:print:]]*\/\)*//g")
    echo $Album
    echo "#EXTM3U" > $ListPath/$Album.m3u
    echo "#EXTGRP" >> $ListPath/$Album.m3u
    echo "#EXTGRP" >> $ListPath/ALL.m3u
    find $line -type f | while read path
    do
      duration=$(ffmpeg -i "$path" 2>&1 | grep "Duration"| cut -d ' ' -f 4 | sed s/,// | sed 's@\..*@@g' | awk '{ split($1, A, ":"); split(A[3], B, "."); print 3600*A[1] + 60*A[2] + B[1] }')
      if [ "$duration" == "0" ] || [ "$duration" == "" ]; then
        printf "$path is not a media file, duration: $duration\n"
        continue
      fi
    header="#EXTINF:$duration,$(echo "$path" | sed -e "s/\([[:print:]]*\/\)*//g")"
    url="$(echo "$path" | sed -e 's/^/http:\/\/ipv4.bajzc.com:81\//')"
    printf "$header\n$url\n\n" >> $ListPath/$Album.m3u
    printf "$header\n$url\n\n" >> $ListPath/ALL.m3u
    done
  done
}

compress(){
  cd $ListPath
  tar -czvf PlayList.tar.gz *.m3u
  zip -qr PlayList.zip *.m3u

}
Backup
ListMovie
ListAlbum
compress
unset ListPath
```

:::: note
::: title
Note
:::

You may need to install `ffmpeg`, `zip` on your host system.
::::

Backup()

> line 6-9: backup all files in `$ListPath`

ListMovie()

> line 13: change the directory to where the `Movie` is.
>
> line 14: print .m3u header to the output file `$ListPath/ALL.m3u`
>
> line 15: list all the files and directories in `Movie/` and assign the
> path to `line`
>
> line 17: whether `line` is a path to a directory
>
> line 19: use regular expression to get the name of directory
>
> line 21: use ffmpeg to get the duration of a media file in seconds
>
> line 28: use regex again to convert the relative path into URL

ListAlbum()

> line 36: only search for directories in one depth
>
> line 38: use regex to get the name of album

compress()

> line 60: compress into .zip
>
> line 61: compress into .tar.gz

The output can be seen [here](http://ipv4.bajzc.com:81/PlayLists/)

## 4 Summary

This script still has some limitations, such as it cannot deal with the
`SPACE` in directories\' name. Because `find` won\'t return the path in
C-style (but `ls` can do it). For example:
`The Music of Grand Theft Auto V [FLAC]`, should be
`The\ Music\ of\ Grand\ Theft\ Auto\ V\ \[FLAC\]`. Otherwise `Bash`
cannot locate it.

However, if the file name contains those characters will be fine. Thanks
to ffmpeg, path can be passed as a string. (Line 21)

After all, open a m3u file in VLC:

![image](/images/VLC-Playlist.png){.align-center}

:::: note
::: title
Note
:::

Copyright (C) 2023 Jason Li

License:

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your
option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
Public License for more details.

You should have received a copy of the GNU General Public License along
with this program. If not, see \<<https://www.gnu.org/licenses/>\>.
::::
