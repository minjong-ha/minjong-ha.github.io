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

### Event loop in GLib

```c
#include <glib.h>

static GThread *my_thread;
static GMainLoop *my_loop;

static void
add_idle_to_my_thread (GSourceFunc    func,
                       gpointer       data)
{
  GSource *src;

  src = g_idle_source_new ();
  g_source_set_callback (src, func, data, NULL);
  g_source_attach (src,
                   g_main_loop_get_context (my_loop));
  g_source_unref (src);
}

static void
add_timeout_to_my_thread (guint          interval,
                          GSourceFunc    func,
                          gpointer       data)
{
  GSource *src;

  src = g_timeout_source_new (interval);
  g_source_set_callback (src, func, data, NULL);
  g_source_attach (src,
                   g_main_loop_get_context (my_loop));
  g_source_unref (src);
}

static gpointer
loop_func (gpointer data)
{
  GMainLoop *loop = data;

  g_main_loop_run (loop);

  return NULL;
}

static void
start_my_thread (void)
{
  GMainContext *context;

  context = g_main_context_new ();
  my_loop = g_main_loop_new (context, FALSE);
  g_main_context_unref (context);

  my_thread = g_thread_create (loop_func, my_loop, TRUE, NULL);
}

static void
stop_my_thread (void)
{
  g_main_loop_quit (my_loop);
  g_thread_join (my_thread);
  g_main_loop_unref (my_loop);
}
```

## Appendix

- [Repository for C glib with event loop study](https://github.com/minjong-ha/c-glib-study)
