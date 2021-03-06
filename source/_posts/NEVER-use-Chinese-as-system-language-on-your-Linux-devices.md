---
title: NEVER use Chinese as system language on your Linux devices
date: 2017-09-10 15:32:46
category: essay
tags:
  - Linux
  - shell
---

During my attempt to execute a shell script provided by the manufacturer, I got a couple o' annoying error outputs(shown below).

![](p1.png)

Going through the script several times, it seemed that there were nothing wrong with it. But it suddenly dawned on me that I was using Chinese as system language for my CentOS 7.

Then I found this. It's obvious that `grep Disk` may not work as expected.

```bash
SIZE=`fdisk -l $DRIVE | grep Disk | awk '{print $5}'`
CYLINDERS=`echo $SIZE/255/63/512 | bc`
```

Gotcha..

![](p2.png)

The script worked after I changed system language to English. Perhaps we should **never** use Chinese(as well as other languages beside English) as system language. :)

P.S. In fact, you can do something like `export LC_ALL="en_US.UTF-8"` to switch locale insead of language, if you really wanna use other languages.
