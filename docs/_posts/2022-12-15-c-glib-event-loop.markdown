---
layout: posts
title:  "Glib"
author: Minjong Ha
published: false
date:   2022-12-15 12:00:00 +0900
---

GLib is a utility library for C that helping developers can implement the source codes more efficiently.
In this post, I will explain about the basic GLib features and focus on the event loop that providing asyncio.
<!--- Lets study about g_event_loop() --->

## GLib
<!--- What is g-lib? --->
<!--- Example - virtio-win-tools --->

GLib or glibc (GNU C Library), presents the core libraries for GNU and GNU/Linux systems. 
Since GTK provides glibc for Windows and macOS, it is transportable.
Although the C is not for the objective programming, glibc provides GObjects for OOP and developers can implement codes efficiently.
It also provides basic data structures.

However, in this post, I will focus on the event loop.
For better understanding about asyncio, you can reference [here](https://blahblahblah).

Usually, GLib / GTK application works on the main event loop that running on the main thread.
Main event loop handles keyboard and mouse I/O, displaying, and etc.

## Appendix
- [Repository for C glib with event loop study](https://github.com/minjong-ha/c-glib-study)
