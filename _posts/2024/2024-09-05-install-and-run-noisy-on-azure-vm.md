---
title: How to install and run noisy on an Azure VM
date: 2024-09-04 10:00
tags: [Azure, networking, hub-and-spoke, noisy, powershell, python, git, script, windows, linux, shell]
excerpt: "a PowerShell and shell script that facilitates the installation of Python and Git, subsequently download the repository of Noisy from GitHub and run it with the default configuration"

header:
  overlay_image: https://live.staticflickr.com/65535/52755090506_6cf0808a3c_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/52755090506)"
---

In this blog post, I show a a couple of scripts that are designed to facilitate the installation of Python and Git and subsequently download the repository of Noisy from GitHub, initiating it with the default configuration, on both Windows and Linux VMs.

> [Noisy](https://github.com/1tayH/noisy) is an elegant yet robust Python script that is programmed to generate arbitrary HTTP/DNS traffic.

I primarily utilize it on my 'hub-and-spoke playground' to generate simulated traffic. This facilitates the exploration and manipulation of Azure firewall configurations.

> the [hub-and-spoke playground](https://github.com/nicolgit/hub-and-spoke-playground) is instead a composite collection of BICEP/ARM templates. These templates are designed to be deployed on Azure, forming a hub and spoke network topology that is in alignment with the Microsoft Enterprise scale landing zone reference architecture. This setup serves as an experimental playground for testing and studying.

The following steps need to be executed from an administrative PowerShell terminal if the machine runs Windows:

```powershell
#
# STEP1 : install python
#
$version = "3.12.5"
$directory = "Python312"

$pythonSetupFile = "python-" + $version + "-amd64.exe"
$downloadUri = 'https://www.python.org/ftp/python/' + $version + '/' + $pythonSetupFile
Invoke-WebRequest -UseBasicParsing -Uri $downloadUri -OutFile $pythonSetupFile

$installPythonCommand = '.\' + $pythonSetupFile
Start-Process -wait $installPythonCommand -ArgumentList @('/quiet', 'InstallAllUsers=1', 'PrependPath=1', 'Include_test=0')

$env:PATH += "C:\Program Files\" + $directory + ";"

#
# STEP2: install git
#
$gitVersion = "2.46.0"
$gitSetupFile = "Git-" + $gitVersion +"-64-bit.exe"
$gitDownloadUri = "https://github.com/git-for-windows/git/releases/download/v" + $gitVersion + ".windows.1/" + $gitSetupFile
Invoke-WebRequest -UseBasicParsing -Uri $gitDownloadUri -OutFile $gitSetupFile

$installGitCommand = '.\' + $gitSetupFile
Start-Process -wait $installGitCommand -ArgumentList @('/SILENT')

$env:PATH += "C:\Program Files\Git\cmd;"

#
# STEP3: download noisy repo from github
#
python -m pip install requests
git clone https://github.com/1tayH/noisy.git
cd noisy

#
# STEP4: execute noisy 
#
python noisy.py --config config.json

```

If you run an Ubuntu machine, you already have `python` and `git` installed, so only steps 3 and 4 are required, as shown below:

```sh
sudo apt update
sudo apt upgrade

git clone https://github.com/1tayH/noisy.git
cd noisy

python3 noisy.py --config config.json

```

