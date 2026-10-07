---
title: "Enabling Intel N100 Hardware Transcoding for Plex and Jellyfin"
date: 2026-10-07T14:39:08+01:00
draft: false
description: "How I passed the Intel GPU into both containers, enabled Quick Sync, proved a real Jellyfin transcode and checked Plex's hardware path."
tags: [plex, jellyfin, intel-quick-sync, docker, homelab]
---

A film that plays perfectly on a television can still fail in a web browser. That was the symptom which sent me back into my Plex and Jellyfin setup.

The media files were healthy and direct play worked on compatible clients. The problem appeared when a client needed the server to convert the video. My small Intel N100 server has an integrated GPU designed for exactly this job, but having the hardware in the machine does not mean a container can use it.

I found two different states:

- Jellyfin could see the Intel render device, but its transcoding settings needed tightening and a real playback test.
- Plex had no GPU device inside its container at all, so hardware acceleration could never work regardless of the checkbox in its web interface.

The fix was not to buy a larger server. It was to connect each layer properly and verify the result from the application all the way down to the GPU.

## Why direct play hid the problem

A media server normally chooses one of three paths:

- **Direct play:** send the original video and audio untouched.
- **Direct stream:** repackage compatible streams into another container without re-encoding them.
- **Transcode:** decode and encode one or more streams into something the client accepts.

Direct play is the cheapest option, but it only proves that the client understands the file. It says nothing about the transcoder.

Browsers are useful test clients because their codec and container support is narrower than many television apps. In this case, deliberately lowering the browser's playback quality forced a transcode and exposed the missing hardware path.

## The complete path has to work

For Intel Quick Sync to work in a container, all of these need to be true:

1. The host exposes the Intel Direct Rendering Infrastructure devices, normally under `/dev/dri`.
2. The container receives those devices.
3. The account running the media server can open the render device.
4. The application is configured to use the correct acceleration method.
5. The codec conversion being requested is supported by the hardware and application.

A green container status only proves that its main process is running. It does not prove any of the above.

On the host I first checked for the devices:

```bash
ls -l /dev/dri
```

The important entries were the card device and `renderD128`. I then compared that with the view from each container:

```bash
docker exec plex ls -l /dev/dri
docker exec jellyfin ls -l /dev/dri
```

Jellyfin had the devices. Plex did not. That made the first Plex failure unambiguous.

## Passing the Intel GPU into Docker

My Compose definition now exposes the DRI directory to each media server:

```yaml
services:
  plex:
    devices:
      - /dev/dri:/dev/dri

  jellyfin:
    devices:
      - /dev/dri:/dev/dri
```

Depending on the image and host, the container account may also need membership of the host's render or video group. Group IDs are host-specific, so copying someone else's number is a poor shortcut. Check the ownership of the actual device and the identity inside the container instead:

```bash
getent group render
getent group video
docker exec plex id
docker exec jellyfin id
```

After changing a Compose definition, I recreated only the affected container rather than restarting unrelated services.

## Jellyfin: Quick Sync and tone mapping

In Jellyfin's playback settings I selected **Intel Quick Sync (QSV)** and the Intel render device. I enabled the hardware decoders appropriate for the library, hardware encoding, and VPP tone mapping for HDR-to-SDR conversion.

Tone mapping matters when an HDR source has to be displayed on an SDR client. It is also one of the fastest ways to turn a comfortable transcode into heavy CPU work if it falls back to software.

I also enabled two practical controls:

- throttling, so Jellyfin does not transcode the entire film as fast as possible when the client is consuming it slowly;
- deletion of old transcode segments, so temporary files do not accumulate indefinitely.

Those settings improve behaviour, but they are not proof of acceleration.

## Plex: the missing device was the real issue

Plex requires an active Plex Pass for hardware-accelerated streaming. With that available, the transcoder settings can enable:

- hardware acceleration when available;
- hardware-accelerated video encoding;
- HDR tone mapping where needed.

In my case, changing those settings alone would have achieved nothing because `/dev/dri` was absent from the container. Once the device mapping was added and the Plex container recreated, the hardware path became available.

That distinction is worth remembering: an application's hardware-acceleration switch expresses intent. The container still needs permission and a device to carry it out.

## The test that mattered

I used a normal film from my own library and opened it in a browser. I then selected a lower playback quality to force conversion instead of direct play.

Jellyfin completed a real browser transcode and reported hardware processing. I also observed GPU activity on the host while playback was running.

For Plex, I verified `/dev/dri` access and completed a synthetic VAAPI H.264 encode with its bundled transcoder. I did not independently complete the final Plex playback check. That still needs a forced conversion whose active session is labelled `Transcode (hw)` in Plex's dashboard, not merely `Transcode`.

A useful host-side observation tool is `intel_gpu_top` from the Intel GPU tools package:

```bash
sudo intel_gpu_top
```

Run it while a forced transcode is active. Video engine activity is stronger evidence than a successful page load or a low CPU percentage on its own.

## A compact verification checklist

This is the order I would use next time:

1. Confirm `/dev/dri/renderD128` exists on the host.
2. Confirm the same device exists inside the media-server container.
3. Check the container process identity and render-device permissions.
4. Select QSV or hardware acceleration in the application.
5. Start with one ordinary SDR H.264 transcode.
6. Confirm the application reports hardware decoding or encoding.
7. Observe Intel GPU activity on the host.
8. Then test HEVC, subtitles and HDR tone mapping separately.

Testing one complication at a time matters. A subtitle burn-in, unsupported audio format or tone-mapping stage can change the pipeline even when basic hardware transcoding is working.

## What this fixed—and what it did not

The Intel N100 is now doing the work it was bought to do for Jellyfin, which completed a real hardware-accelerated browser transcode. Plex's container gained the previously missing `/dev/dri` access and its bundled transcoder completed a synthetic VAAPI H.264 encode. A real Plex playback session still needs to show `Transcode (hw)` before I can call its end-to-end path verified.

This does not guarantee that every file will direct play or that every possible conversion is supported. Client capability, subtitle format, audio conversion, HDR metadata and codec profiles still affect the decision.

The lesson was simpler: **verify the whole path**. Check the host, container, permissions, application settings and a real playback session. Hardware acceleration is not working because a box is ticked. It is working when the requested stream uses the GPU and the client receives playable video.

## References

- [Jellyfin hardware acceleration documentation](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)
- [Jellyfin Intel GPU documentation](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/)
- [Plex: Using hardware-accelerated streaming](https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/)
