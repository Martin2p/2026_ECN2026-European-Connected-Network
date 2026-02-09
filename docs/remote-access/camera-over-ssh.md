# Remote Camera Access over SSH (MJPEG)

## Overview

This document describes a minimal and secure method to access a **USB camera attached to a remote Linux system** using **SSH as transport**, without running persistent services or exposing additional ports.

The setup was implemented as part of **ECN2026** to demonstrate secure, on-demand remote device access.

---

## Design

- Camera access via **Video4Linux2 (v4l2)**
- Capture using **FFmpeg**
- **MJPEG passthrough** (no re-encoding)
- Video streamed over **SSH STDOUT**
- Rendering on the client via **ffplay**

No web interfaces, no daemons, no open ports.

---

## Architecture
USB Camera → /dev/videoX → ffmpeg → SSH → ffplay

## Prerequisites

**Camera host (Laptop)**
- Linux (tested on Linux Mint)
- `ffmpeg`, `v4l2-utils`
- User in `video` group

**Remote client**
- `ffplay`
- SSH access (optionally over WireGuard)

---

## Device Detection

```bash
v4l2-ctl --list-devices
ffplay /dev/video2
```
## MJPEG Stream over SSH

Executed on the remote client:
```bash
ssh -T user@laptop-ip \
  "ffmpeg -nostdin -f v4l2 -input_format mjpeg \
   -framerate 20 -video_size 640x480 \
   -i /dev/video2 -c:v copy -f mjpeg -" \
| ffplay -fflags nobuffer -flags low_delay \
  -analyzeduration 0 -probesize 32 -f mjpeg -i -
```

## Notes

- MJPEG provides low latency and high image quality
- Bandwidth-heavy; reduce resolution or FPS for WAN links
- Camera is active only while the command runs
- Stream integrity requires silent, non-interactive SSH shells

## ECN2026 Relevance

Demonstrates:

- Secure on-demand device access
- SSH as a controlled data transport
- Conscious latency vs. bandwidth trade-offs
- End-to-end system integration (hardware → network → application)
