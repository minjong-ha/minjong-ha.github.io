---
layout: posts
title:  "Debian Packaging""
author: Minjong Ha
published: false
date:   2023-02-06 19:39:00 +0900
---

In this posts, I will explain how I can package the source codes to deb.
It includes the way to use 'dpkg' command and a little tip for re-packaging existing packages.

## Package: deb

Debian package represents itself with 'deb': ${PACKAGE\_NAME}.deb.
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


## dpkg
<!-- Explain how I can use dpkg -->

Creating deb package does not always require source code.
However, almost every debian packages opens their sources for the community.
For better understanding, I create my test project written in C.
Some programming languages such as python can be much easier to build package since they does not require compile.

```c
#include <stdio.h>

int main() {
    printf("Hello World!\n");

    return 0;
}
```

Above codes represent my simple application that just print some text on the screen.


Before you run the following command, change your directory name of source code to format '${PACKAGEi\_NAME}-${VERSION}'.
For example, 'test-package-0.0.1'.
Beware you cannot use '\_'.

```shell
dh-make --createorig
```
'dh-make' create a directory for debian pacakaging: 'debian'.
Under the 'debian/', there are files to build deb package.
Files having '.ex' suffix, are examples and not essentials.
Essential files to build deb are 'changelog', 'control', and 'copyright'

It is also possible that building multiple deb packages in a single package.
In this case, you should distinguish each maintainer scripts like: "package1.postinst", "package2.postinst".


```shell
dpkg-buildpackage -us -uc -b
```
'-us' allows to build unsigned source.
'-uc' allows to build unsigned change.
-'b' allows to build all.



## Re-Packaging
<!-- Explain repackaging with apt source -->

In case of re-packaging exist application, you can use following way.

First, you should download the components of source package: \*.dsc, \*.org.tar.gz, \*.debian.tar.xz.
Collect them in the same directory, and run:
```shell
gbp import-dsc *.dsc
```

It will create a local repo with git that not having remote origin.

Since the repository follows 3.0(quild) debian/source/format, modifying the codes in upstream should be proceed as patch.
```shell
gbp pq import
```

Above command will create a new branch.
If your branch name is 'master', it would be 'patch-queue/master'.

Now you create another branch that based on 'patch-queue/master'; such as 'patch-queue/master/change\_some\_code'.
Change the code you want to, and commit.
Then create MR to 'patch-queue/master': 'patch-queue/master/change\_some\_code' -> 'patch-queue/master'.

If the MR is completed, run following command in 'patch-queue/master'
```shell
gbp export
```

Now you can see the patch files in 'patch-queue/master'.
Push 'patch-queue/master' to remote repository.


## Appendix
<!-- Appendix -->


## Reference
