+++
date = '2025-09-19T21:00:00-07:00'
title = 'Torrenting Videos on a Chromebook'
+++

## Step #0: Disclaimer

Before you dive in, a quick but important heads-up: while the technology of torrenting is perfectly legal, downloading and distributing copyrighted content without the proper rights is not. This guide is here to explain how the technology works, but it's up to you to use it responsibly and within the bounds of the law. Please be aware of the laws in your area and torrent with care.

## Step #1: Set Up Linux

Follow [this guide](https://support.google.com/chromebook/answer/9145439?hl=en) to set up Linux on your Chromebook. Once it's set up, a terminal should open automatically.

## Step #2: Install QBittorrent

[QBittorrent](https://www.qbittorrent.org/) is my preferred torrent client to install on a Chromebook for three reasons:

1) It has support for RSS feeds.
2) It supports running a script when a torrent finishes.
3) It has built-in search functionality.

To install it, run the following command in your Linux terminal:

```bash
sudo apt-get install --yes qbittorrent
```

Once that finishes, you should be able to open the QBittorrent app via the launcher.

## Step #3: Install Search Plugins

Follow [this guide](https://github.com/qbittorrent/search-plugins/wiki/Install-search-plugins) for installing search plugins. You can find a comprehensive list in [the "Plugins for Public Sites" section](https://github.com/qbittorrent/search-plugins/wiki/Unofficial-search-plugins#plugins-for-public-sites) of the QBittorrent wiki.

**Note** that the guide is a little outdated; you don't need to download the plugins anymore, you can just copy and paste the URL.

## Step #4: Share Download Folder

Open the "Files" app and right-click on the "Downloads" folder. Click the option that says "Share with Linux".

## Step #5: Configure Folders in QBittorrent

Go back to QBittorrent, and click the "Preferences" button (it's got a gear above it).

1) Make sure you are in the “Downloads” section of the “Preferences”.
2) Change the "Default Save Path" to `/mnt/chromeos/MyFiles/Downloads`.
3) Go to the "Advanced" section of the “Preferences”.
4) Find the "Disc I/O Type" option in the list.
5) Change the value to "POSIX-compliant".
6) Click "Okay" to save your changes.
7) Restart QBittorrent so that the changes take effect.

## Step #6: Install FFmpeg

A problem with Chrome OS is its very limited set of audio codecs, with no way to install new ones without jailbreaking.

I have tried different third-party video players like VLC, both in Linux or as an Android app. Either way I've found that the video playback is a bit choppy.

My preferred solution is to transcode the audio in any downloaded video files using [FFmpeg](https://ffmpeg.org/) so Chrome OS can play them.

Run the following command in your Linux terminal:

```bash
sudo apt-get install --yes ffmpeg
```

## Step #7: Create Transcode Script

Open the "Text" app and copy the following code:

```bash
#!/bin/bash

# Get the file or directory path from qBittorrent
INPUT="$1"

if [ -f "$INPUT" ]; then
  OUTPUT="${INPUT%.*}-transcode.${INPUT##*.}"
  if [[ "$INPUT" =~ \.avi$ ]]; then
    ffmpeg -i "$INPUT" -c:v copy -c:a pcm_s16le -ac 2 "$OUTPUT"
  elif [[ "$INPUT" =~ \.mkv$ ]]; then
    ffmpeg -i "$INPUT" -c:v copy -c:a pcm_s16le -ac 2 "$OUTPUT"
  elif [[ "$INPUT" =~ \.mp4$ ]]; then
    ffmpeg -i "$INPUT" -c:v copy -c:a libvorbis -ac 2 "$OUTPUT"
  fi
elif [ -d "$INPUT" ]; then
  for FILE in "$INPUT"/*.{avi,mkv,mp4}; do
    if [ -f "$FILE" ]; then
      "$0" "$FILE"
    fi
  done
fi
```

Save the file as "transcode-audio" **with no file extension** in your "Downloads" folder.

## Step #8: Move Transcode Script

Back in your Linux terminal, run the following command:

```bash
chmod +x /mnt/chromeos/MyFiles/Downloads/transcode-audio
sudo mv /mnt/chromeos/MyFiles/Downloads/transcode-audio /usr/bin/
sudo chown root:root /usr/bin/transcode-audio
```

This will make the script runnable and move it to the correct directory, so QBittorrent can find it.

## Step #9: Configure QBittorrent to Transcode

Go back to QBittorrent, and open the "Preferences" again.

1) Make sure you are in the "Downloads" section of the "Preferences".
2) Scroll right to the bottom and enable "Run on torrent finished".
3) Enter `transcode-audio %F` into the box.
4) Click "Okay" to save your changes.

## Step #10: Profit.

That's it, folks!

Note that once your download finishes, it will spend up to ten minutes in the "Moving" state while it is being transcoded. When it's done, you will find two copies of the video file in your "Downloads" folder. You should be able to double-click and play the one that ends in "-transcode".