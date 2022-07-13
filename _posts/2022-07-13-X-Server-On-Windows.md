---
title: Using the Linux Kernel on Windows to display graphic apps
published: true
---

First an XServer needs to be installed for Windows. Fortuantely there are a lot of options to choose from like [http://www.straightrunning.com/XmingNotes/](XMing).

After installing your XServer you also need to
- Add the following to your `~/.bashrc`
```
export DISPLAY=0.0.0.0:0
export LIBGL_ALWAYS_INDIRECT=1
```
- Enable inbound rule for TCP port `6000` on Windows. 


