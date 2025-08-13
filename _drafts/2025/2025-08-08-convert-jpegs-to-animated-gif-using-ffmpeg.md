---
title: "Converting JPEG Images to Animated GIF using PowerShell and FFmpeg"
date: 2025-08-08 10:00
tags: [powershell, ffmpeg, gif, automation, image-processing]
excerpt: "Learn how to create animated GIFs from a sequence of JPEG images using a PowerShell script and FFmpeg. Perfect for creating smooth animations from image sequences."

header:
  overlay_image: https://live.staticflickr.com/65535/54686926861_527ec886b1_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/54686926861)"
---
Creating animated GIFs from a sequence of images is a common task in digital content creation, whether you're building tutorials, showcasing animations, or creating engaging social media content. In this post, I'll walk you through a PowerShell script that automates the conversion of JPEG images to animated GIFs using FFmpeg.

## Prerequisites

You'll need FFmpeg installed on your system. 

```
winget install --id=Gyan.FFmpeg
```

## The Complete Script

Here's the PowerShell script that handles the entire conversion process:

```powershell

function Convert-JpegsToGif {
    param(
        [Parameter(Mandatory = $true)]
        [string]$SourceDir,
        [Parameter(Mandatory = $true)]
        [string]$OutputDir,
        [Parameter(Mandatory = $true)]
        [string]$OutputFileName,
        [Parameter(Mandatory = $true)]
        [float]$FramesPerSecond
    )
    
    $inputPattern = Join-Path $SourceDir "%02d.jpeg"
    $outputMp4 = Join-Path $OutputDir "$OutputFileName.mp4"
    $outputGif = Join-Path $OutputDir "$OutputFileName.gif"

    ffmpeg -framerate $FramesPerSecond -i $inputPattern -c:v libvpx-vp9 -lossless 1 -vf "scale=1024:768" -y $outputMp4
    $palette = "palette.png"
    $filters = "fps=12,scale=1072:-1:flags=lanczos"
    ffmpeg -v warning -i $outputMp4 -vf "$filters,palettegen" -y $palette
    ffmpeg -v warning -i $outputMp4 -i $palette -lavfi "$filters [x]; [x][1:v] paletteuse" -y $outputGif
    Remove-Item $palette
    Remove-Item $outputMp4
}


```

## Usage Example

To use this script, ensure your JPEG files are numbered sequentially (01.jpeg, 02.jpeg, etc.) in the source directory, then call:

```powershell
Convert-JpegsToGif -SourceDir "welcome" -OutputDir "../video" -OutputFileName "welcome" -FramesPerSecond 0.5
```

This creates a slow-motion GIF (0.5 fps) from images in the "welcome" directory.

## Conclusion
Whether you're creating technical documentation, social media content, or presentation materials, this script can save you significant time while ensuring consistent, high-quality results.

> **Pro tip**: For best results, ensure your source images have consistent dimensions and naming conventions before running the script!
```
