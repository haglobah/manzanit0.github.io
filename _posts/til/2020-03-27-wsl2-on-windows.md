---
layout: post
title: Setting up WSL2 quick and easy on Windows 10
author: Javier Garcia
category: Windows
tags: Windows, WSL2
---

_Yet another note for myself, for the next time I have to set up WSL2 on a
Windows computer. I've taken the time to give it a bit of format so you may
follow the steps too._

## Install WSL

To install the original WSL, it's as easy as going to the *Windows Store* and
searching for "_wsl_". I personally installed the Ubuntu flavour. You will
probably need to enable the feature in Windows before installing. For that
simply run in PS:

```ps1
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

## Installing WSL 2

To access WSL2, as of March 2020, it's slightly tricker: you have to opt-in the
Insiders Program and get the most recent OS build, Windows 10 version 2004. To
do that, follow this [link][0].

Once you've registered in the Insider's program, you need to enable it in the
OS in order for Windows to download the latest updates. You can do that in
`Settings> Update & Security> Windows Insider Program`. Make sure you tick
those boxes, restart the system, and after that Windows will suggest
downloading the latest build under `Settings > Update & Security > Windows
Update`.

Next, to manually update the WSL 2 Linux kernel, download the following package
and run it - [link][1]

Lastly, once that's that, following the [official instruction][2], run:

```ps1
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Set the default version to 2

```ps1
wsl --set-default-version 2
```

And your Ubuntu WSL to version 2 too:

```ps1
wsl --set-version Ubuntu 2
```

To verify that all is in order, run: `wsl --list --verbose`

## The Extra Mile: Configuring IDEs

This is the easies step, assuming you've already set `Ubuntu` as your default
WSL with:

```
C:\Windows\System32\wslconfig /setdefault Ubuntu-18.04
```

For configuring any IDE to use WSL instead of CMD, point the terminal to the
path:

```
C:\Windows\System32\wsl.exe
```

In the case of Jetbrains products, you can find that under `Settings > Tools >
Terminal`.  This is the [original source][3].

[0]: https://insider.windows.com/
[1]: https://docs.microsoft.com/en-us/windows/wsl/wsl2-kernel
[2]: https://docs.microsoft.com/en-us/windows/wsl/wsl2-install
[3]:https://stackoverflow.com/questions/51912772/how-to-use-wsl-as-default-terminal-in-webstorm-or-any-other-jetbrains-products

