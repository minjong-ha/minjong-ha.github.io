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
// test.c
#include <stdio.h>

int main() {
    printf("Hello World!\n");

    return 0;
}
```

```shell
# Makefile
all: test.c
    gcc -o test test.c
clean: 
    rm test
```

Above codes represent my simple application that just print some text on the screen.


Before you run the following command, change your directory name of source code to format '${PACKAGEi\_NAME}-${VERSION}'.
For example, 'test-package-0.0.1'.
Beware you cannot use '\_'.

```shell
dh_make --createorig
```
'dh\_make' create a directory for debian pacakaging: 'debian'.
Under the 'debian/', there are files to build deb package.
Files having '.ex' suffix, are examples and not essentials.
Essential files to build deb are 'changelog', 'control', 'rules', and 'copyright'


```shell
dpkg-buildpackage -us -uc -b
```
'-us' allows to build unsigned source.
'-uc' allows to build unsigned change.
-'b' allows to build all.

'dpkg-buildpackage' calls make with 'debian/rules' as the Makefile.
As a result, we can configurate 'debian/rules' and automate the tasks for packaging.
Followings are result of run 'dpkg-buildpackage -us -uc -b'.
```shell
$ dpkg-buildpackage -us -uc -b
pkg-buildpackage: info: source package test-package
dpkg-buildpackage: info: source version 0.0.1-1
dpkg-buildpackage: info: source distribution unstable
dpkg-buildpackage: info: source changed by minjong_ha <minjong_ha@unknown>
dpkg-buildpackage: info: host architecture amd64
 dpkg-source --before-build .
 debian/rules clean
dh clean
   dh_auto_clean
   dh_clean
 debian/rules binary
dh binary
   dh_update_autotools_config
   dh_autoreconf
   dh_auto_configure
   dh_auto_build
	make -j6 "INSTALL=install --strip-program=true"
make[1]: Enter '/home/minjong_ha/workspace/tmax/teamwork/test-package-0.0.1' directory
gcc -o test src/test.c
make[1]: Leave '/home/minjong_ha/workspace/tmax/teamwork/test-package-0.0.1' directory
   dh_auto_test
   create-stamp debian/debhelper-build-stamp
   dh_prep
   dh_auto_install
   dh_installdocs
   dh_installchangelogs
   dh_perl
   dh_link
   dh_strip_nondeterminism
   dh_compress
   dh_fixperms
   dh_missing
   dh_dwz -a
   dh_strip -a
   dh_makeshlibs -a
   dh_shlibdeps -a
   dh_installdeb
   dh_gencontrol
dpkg-gencontrol: warning: Depends field of package test-package: substitution variable ${shlibs:Depends} used, but is not defined
   dh_md5sums
   dh_builddeb
dpkg-deb: building package 'test-package' in '../test-package_0.0.1-1_amd64.deb'.
 dpkg-genbuildinfo --build=binary
 dpkg-genchanges --build=binary >../test-package_0.0.1-1_amd64.changes
dpkg-genchanges: info: binary-only upload (no source code included)
 dpkg-source --after-build .
dpkg-buildpackage: info: binary-only upload (no source included)
```

Since I create 'Makefile' that compiling source codes, dpkg-buildpackage automatically compiles it with 'dh\_auto\_build'.

It is also possible that building multiple deb packages in a single package.
In this case, you should distinguish each maintainer scripts like: "package1.postinst", "package2.postinst".


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


### Example

In 'Example' section, I will share my experience that repackaging 'libopenal-data', which is included in 'openal-soft'.

First, you should download the three package components: dsc, org.tar.ga, and debian.tar.xz.
Following is a github link for ('openal-soft')[https://github.com/kcat/openal-soft].
You can download each components using official site, but I personally prefer to use 'apt source'.

```shell
$ apt source libopenal-data

$ ls
openal-soft-1.19.1  openal-soft_1.19.1-2.debian.tar.xz  openal-soft_1.19.1-2.dsc  openal-soft_1.19.1.orig.tar.gz
```

'apt source' download three deb components with sources in 'openal-soft-1.19.1'.
However, 'openal-soft-1.19.1' does not include any git files.
To acquire the git included sources, you should run 'gpb import-dsc'.

```shell
$ rm -rf openal-soft-1.19.1/
$ gbp import-dsc openal-soft_1.19.1-2.dsc
$ cd openal-soft/
```

Now you have openal-soft source codes with git.
Since it does not remote origin, you cat allocate it to your personam repackaging git repository.

Rest of the tasks is same as I wrote in previous section.
You can update the source codes or configuration and run 'dpkg-buildpackage -us -uc -b' to create repackaged deb.
Just don't forget to change debian/changelog and debian/control.

## Appendix
<!-- Appendix -->


## Reference
