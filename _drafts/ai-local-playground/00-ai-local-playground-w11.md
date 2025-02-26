---
title: Setup a local ai playground on your W11 machine
date: 2025-01-01 10:00
tags: [Azure OpenAI ]
excerpt: ""

header:
  overlay_image: https://live.staticflickr.com/65535/52755090506_6cf0808a3c_h.jpg
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/52755090506)"
---

The AI local playground: in this blog post I show how to configure your W11 machine
 to start experimenting with the most common artificial intelligence APIs
using Visual Studio Code, Python, and Jupyter Notebooks.

Starting from a clean Windows 11, the components to install and configure are the following:

* Windows Subsystem for Linux: It is a compatibility layer for running Linux binary executables natively on Windows 11. WSL allows developers to use a Linux environment, including most command-line tools, utilities, and applications, directly on Windows, without the overhead of a traditional virtual machine or dual-boot setup.
* Conda Python virtual environment: An environment created using the Conda package manager, which can manage dependencies and packages for Python and other languages.  
  > A Python environment is a context in which Python code is executed. It includes the Python interpreter and a set of installed packages and dependencies.
* Visual Studio Code: Visual Studio Code (VS Code) is a free, open-source code editor developed by Microsoft. It is available for Windows, macOS, and Linux. 

once these components are installed, I will show a pattern to use for experimenting with the following APIs:

* Azure OpenAI service (chat, completition)


