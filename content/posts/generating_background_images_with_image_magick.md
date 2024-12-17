---
title: 'Generating Background Images with ImageMagick'
date: 2024-12-17T15:39:44+03:00
tags: [ "subtle-patterns", "imagemagick", "background" ]
---

# Generating Background Images with ImageMagick

## Requirements

- [ImageMagick](https://imagemagick.org/)
- Image Pattern
  + You can draw it yourself or download one from a website like [Subtle Patterns](https://www.toptal.com/designers/subtlepatterns)

## Generating image

For this purpose imagemagick's tile option fits our needs. Use the following command:

```bash
magick -size {RESOLUTION} tile:{YOUR PATTERN} {OUTPUT FILE}
```

Resolution should be `width x height` (for example `1920x1080`) and pattern should be a valid image file. It is that simple.
