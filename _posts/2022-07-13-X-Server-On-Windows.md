---
title: Using the Linux Kernel on Windows to display graphic apps
published: true
---

First an XServer needs to be installed for Windows. Fortuantely there are a lot of options to choose from like [http://www.straightrunning.com/XmingNotes/](XMing) or [https://sourceforge.net/projects/vcxsrv/files/latest/download](VcXsrv)

After installing your XServer you also need to
- Add the following to your `~/.bashrc`
```
export DISPLAY=0.0.0.0:0
export LIBGL_ALWAYS_INDIRECT=1
```

    `LIBGL_ALWAYS_INDIRECT` tells OpenGL to never communicate with hardware directly when rendering and instead contact X.Org instead. 

    `DISPLAY` is the hostname of the system where the XServer has a physically connected monitor.

- Enable inbound rule for TCP port `6000` on Windows. 

You can see how I'm installing IDA Pro on my Linux WSL machine with a Windows XServer for debugging Windows apps. 

![IDA installation from WSL](/assets/idawsl.png)