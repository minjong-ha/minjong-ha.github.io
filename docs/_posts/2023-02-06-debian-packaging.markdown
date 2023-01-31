---
layout: posts
title:  "Debian Packaging""
author: Minjong Ha
published: false
date:   2023-02-06 19:39:00 +0900
---

In this posts, I will explain how I can package the source codes to deb.
It includes the way to use 'dpkg' command and a little tip for re-packaging existing packages.

## dpkg
<!-- Explain how I can use dpkg -->

## Package: deb

Debian package represents itself with 'deb': ${PACKAGE_NAME}.deb.
"deb" file has three components: debian-binary, control.tar.gz, and data.tar.gz

```shell
$ ar tv wget_1.12-2.1_i386.deb
rw-r--r-- 0/0 4 Sep 5 15:43 2010 debian-binary 
rw-r--r-- 0/0 2403 Sep 5 15:43 2010 control.tar.gz 
rw-r--r-- 0/0 751613 Sep 5 15:43 2010 data.tar.gz
```

'debian-binary' represents the version of deb file.
'control.tar.gz' has metadata about the package.
'data.tar.gz' has data files of the package.





## Re-Packaging
<!-- Explain repackaging with apt source -->


## Appendix
<!-- Appendix -->


## Reference
