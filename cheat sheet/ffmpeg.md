---
title: cheat sheet/ffmpeg
---

- cut, re-encode

  ```sh
  # -ss after -i + re-encode: cut lands on the exact timestamp instead of the nearest keyframe
  ffmpeg -i in.webm -ss 00:09:27 -to 00:09:46 -c:v libsvtav1 -crf 24 -c:a copy out.mp4
  ```
