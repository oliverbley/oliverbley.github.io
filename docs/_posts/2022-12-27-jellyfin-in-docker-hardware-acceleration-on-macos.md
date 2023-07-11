---
layout: page
title:  "Jellyfin-in-docker hardware acceleration for transcoding on macOS"
date:   2022-12-27 22:34:44 +0000
categories: jellyfin
---
# Jellyfin-in-docker hardware acceleration for transcoding on macOS
With [Jellyfin] on macOS, to get hardware acceleration for transcoding videos
working, it has to be installed as an native app. Whereas when Jellyfin is
hosted inside a Docker container, then [jellyfin-ffmpeg], that is used for
transcoding videos, has no access to the videotoolbox framework, that macOS uses
for GPU-acceleration. Also on macOS it is not possible to share the hosts GPU
with Docker containers like it can be done on Linux. Thus within a Docker
container on macOS ffmpeg can only utilize the (slow) software encoders for
transcoding.

The approach described here circumvents the macOS restrictions by establishing
an SSH tunnel from the Jellyfin container into the host and directly running the
ffmpeg part natively on the macOS host side. Using this method provided me with
an approximately 3x speedup when transcoding VP9 to H264 videos on a MacBook Pro
15" (2019) with built-in Intel UHD 630 Graphics.

This solution is not secure if hosted on public servers, because it allows
unrestriced access to the host machine from within the Jellyfin container.

## Setup
This guide leans heavily on the [rffmpeg setup instructions]. There is a custom
`Dockerfile` and `docker-compose.yaml` hosted at [jellyfin-docker-macos] that is
prepared for this use case.

### Install ffmpeg/ffprobe on macOS
There is no custom [jellyfin-ffmpeg build] available for macOS, instead the
[official ffmpeg static build for macOS] can be used, that also supports
videotoolbox. The install location for both ffmpeg and ffprobe should be in
`/usr/local/bin`.

### Enable SSH remote login on macOS
Enabling SSH can be done via macOS System Preferences (`Sharing > Remote
Login`), although there is a bug on Catalina, where [ this workaround ](
https://apple.stackexchange.com/a/442380 ) could help.

To allow passwordless login via SSH key pair, edit `/private/etc/ssh/sshd_config`
and change the following values:
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

### Prepare SSH keys for Jellyfin container on macOS
If there is no SSH key pair, it has to be generated first, e.g. with `ssh-keygen`.
The public part of the key has to be added to `authorized_keys`:
```zsh
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

Also the macOS target host should be added to `known_hosts`:
```zsh
ssh-keyscan $(hostname) >> ~/.ssh/known_hosts
```

### Setup common accessible shared folders
The media and transcodes folders have to be accessible at the same location on the
Docker container side like on macOS. The transcodes will be generated on a `/tmp`
subdirectory, whereas the media folder has to be located at `/var/folders/media`.
A soft link can be used to link to the actual location of the media folder on
macOS like that:
``` zsh
ln -s *your_media_folder* /private/var/folders/media
```

Check the Docker Dashboard, that `/var/folders` is an entry in `Preferences >
File Sharing`.

### Start Jellyfin Docker container
When using the provided `docker-compose.yaml`, Jellyfin can now be started with:
``` zsh
docker compose up jellyfin 
```

The logs should now contain a entry like this:
```
[INFO] [1] MediaBrowser.MediaEncoding.Encoder.MediaEncoder: Available hwaccel types: ["videotoolbox"]
```

### Adjust Jellyfin Playback settings
To get the transcoding working, there have to be some additional settings made
manually in the Jellyfin web dashboard (`Dashboard > Playback`):
* Hardware acceleration: `Apple VideoToolBox`
* Enable hardware encoding: `True`
* Allow encoding in HEVC format: `False` (HEVC did not utilize GPU in my setup)
* FFmpeg path: `/opt/ffmpeg`
* Transcodes path: `/tmp/jellyfin/transcodes`

The GPU utilization under macOS can be monitored via the Activity Monitor.

[Jellyfin]: https://jellyfin.org
[jellyfin-ffmpeg]: https://github.com/jellyfin/jellyfin-ffmpeg
[jellyfin-ffmpeg build]: https://github.com/jellyfin/jellyfin-ffmpeg/releases
[rffmpeg setup instructions]: https://github.com/joshuaboniface/rffmpeg/blob/6458bc85b758b696073adb52af57cfa27eac0621/SETUP.md
[jellyfin-docker-macos]: https://github.com/ovbly/jellyfin-docker-macos
[official ffmpeg static build]: https://evermeet.cx/ffmpeg/
