---
layout: post
title:  "Making a GStreamer Plugin with OpenCV (Part 1)"
date:   2022-09-18 12:34:00 -0800
---
I decided to dust off the GStreamer toolbox and try doing something fancier than just passing buffers using OpenCV.  To change it up, I also wanted to incorporate the code as a GStreamer plugin - that way it could be brought in to any pipeline with the gst factory method `gst_element_factory_make("myfilter", "myfilter0")`.  I usually lean on the `appsink`/`appsrc` method and pass the buffers between the two blocks, but figured this would be a good excercise and would scale better.

The first thing I did was setup a docker image with the GStreamer and OpenCV development frameworks installed.  I'll make a post on how I built out the Dockerfiles another time.

So the sequence looked something like this:
1. Setup a basic GStreamer pipeline using a GLib mainloop to generate a test video of a ball bouncing around and send it over UDP to my host computer.
2. Setup a GStreamer plugin called `myfilter` and add it to my pipeline in `main.cpp`.  To start it wouldn't do anything, just pass the buffers from the sink pad to the src pad.
3. Add the OpenCV libraries to the plugin.  This involves changing the Gstreamer boilerplate `meson.build` to compile with `c++` instead of `c`.
4. In the plugin get the video frame out of the `GstBuffer` format and into `cv::Mat` so OpenCV can use it.  Then I'll have to extract the data from the `cv::Mat` and put it back into the `GstBuffer` so it can run through the remaining downstream GStreamer pipeline.


## A Basic UDP Pipeline.

First thing is to get some code up to run a basic pipeline.

```
videotestsrc -> capsfilter -> myfilter -> videoconvert -> capsfilter -> x264enc -> rtph264pay -> udpsink
```

On my host OS I'll have run pipeline to display the video.

```
udpsrc -> rtpjitterbuffer -> rtph264depay -> decodebin -> videoconvert -> glimagesink
```

Some notes:
- I threw in the capsfilters on either side of my plugin to enforce the video format.   There's probably a better way to do this in the plugin itself, but this was my first run at it.
- To make it easier on myself, I'll set the caps on the first capsfilter to `BGR` since this plays really nice with open CV.  The resolution is 320x240, so the `cv::Mat` size will be 320x240x3 to capture the three color channels for the video frames.
- The videoconvert is necessary because the x264 encoder won't work with `BGR` and needs something like `I420`. I'll use the second capsfilter to enforce this and make sure the videoconvert block changes to the appropriate caps.
- Originally the UDP video was really jittery and I found the rtpjitterbuffer really smoothed out the quality of the ball jumping around.

Let's start with the `meson.build` file to compile a little `main.cpp` test harness.

Here's the directory structure:

```
myapp
 |- meson.build
 |- src
     |- main.cpp
```

{% highlight yaml %}
project('main', 'c', 'cpp',
  version : '0.0.1',
  meson_version : '>= 0.54',
  default_options : ['cpp_std=c++17', 'warning_level=1', 'buildtype=debugoptimized'])

gst_version = '1.20.3'

glib_dep = dependency('glib-2.0', version : '>= 2.56.0',
  fallback: ['glib', 'libglib_dep'])

gst_dep = dependency('gstreamer-1.0', version : '>= 1.20',
  required : true, fallback: ['gstreamer', 'gst_dep'])

gstbase_dep = dependency('gstreamer-base-1.0', version : '>= 1.20',
  fallback: ['gstreamer', 'gst_base_dep'])

plugins_install_dir = join_paths(get_option('libdir'), 'gstreamer-1.0')
plugin_cpp_args = ['-DHAVE_CONFIG_H']
api_version = '1.0'

warning_flags = [
  '-Wmissing-declarations',
  '-Wmissing-prototypes',
  '-Wredundant-decls',
  '-Wundef',
  '-Wwrite-strings',
  '-Wformat',
  '-Wformat-nonliteral',
  '-Wformat-security',
  '-Wold-style-definition',
  '-Waggregate-return',
  '-Winit-self',
  '-Wmissing-include-dirs',
  '-Waddress',
  '-Wno-multichar',
  '-Wdeclaration-after-statement',
  '-Wvla',
  '-Wpointer-arith',
]

cc = meson.get_compiler('cpp')

executable('main', 'src/main.cpp', dependencies : [glib_dep, gst_dep])
{% endhighlight %}

That should be enough to make a little executable to build a test harness using `src/main.cpp`

```
#include "gst/gstelementfactory.h"
#include <glib.h>
#include <iostream>
#include <gst/gst.h>

int
main(int argc, char *argv[]) {

    g_info("building gst pipeline");

    static GMainLoop *gloop;
    gloop = g_main_loop_new(NULL, FALSE);

    gst_init(&argc, &argv);

    gloop = g_main_loop_new(NULL, FALSE);

    /* videotestsrc pattern=ball \
    * ! x264enc bitrate=4000 \
    * ! rtph264pay config-interval=1 \
    * ! udpsink host=$1 port=$2
    */

    // setup the GStreamer elements
    GstElement* pipeline;
    GstElement* src;
    GstElement* capsfilter0;
    GstElement* myfilter;
    GstElement* videoconvert;
    GstElement* capsfilter1;
    GstElement* encoder;
    GstElement* rtph;
    GstElement* sink;

    // run the factory make method
    pipeline = gst_pipeline_new("pipeline0");
    src = gst_element_factory_make("videotestsrc", "src0");
    capsfilter0 = gst_element_factory_make("capsfilter", "capfilter0");
    myfilter = gst_element_factory_make("identity", "myfilter0");
    videoconvert = gst_element_factory_make("videoconvert", "videoconvert0");
    capsfilter1 = gst_element_factory_make("capsfilter", "capfilter1");
    encoder = gst_element_factory_make("x264enc", "encoder0");
    rtph = gst_element_factory_make("rtph264pay", "rtph0");
    sink = gst_element_factory_make("udpsink", "sink0");

    // make sure the elements got created
    if (!src) {
        g_warning("error creating videotestsrc");
    }
    if (!capsfilter0) {
        g_warning("error creating capsfilter");
    }
    if (!app) {
        g_warning("error creating myfilter");
    }
    if (!videoconvert) {
        g_warning("error creating videoconvert");
    }
    if (!capsfilter1) {
        g_warning("error creating capsfilter1");
    }
    if (!encoder) {
        g_warning("error creating x264enc");
    }
    if (!rtph) {
        g_warning("error creating rtph264pay");
    }
    if (!sink) {
        g_warning("error creating sink");
    }

    // set the caps for myfilter
    g_object_set(G_OBJECT(capsfilter0), "caps",
            gst_caps_new_simple("video/x-raw",
                "format", G_TYPE_STRING, "BGR",
                "width", G_TYPE_INT, 320,
                "height", G_TYPE_INT, 240,
                //"framerate", GST_TYPE_FRACTION, 15, 1,
                NULL), NULL);

    // set the caps for the x264enc for the videoconvert to convert to
    g_object_set(G_OBJECT(capsfilter1), "caps",
            gst_caps_new_simple("video/x-raw",
                "format", G_TYPE_STRING, "I420",
                "width", G_TYPE_INT, 320,
                "height", G_TYPE_INT, 240,
                //"framerate", GST_TYPE_FRACTION, 15, 1,
                NULL), NULL);

    // we'll add some basic settings for myfilter
    //g_object_set(G_OBJECT(myfilter), "silent", TRUE, "enabled", TRUE, NULL);

    g_object_set(G_OBJECT(src), "pattern", 18, NULL);

    g_object_set(G_OBJECT(rtph), "config-interval", 1, NULL);

    g_object_set(G_OBJECT(sink), "host", "192.168.65.2", "port", 14500, NULL);

    // add all the elements to the pipeline and link them
    gst_bin_add_many(GST_BIN(pipeline),
            src,
            capsfilter0,
            app,
            videoconvert,
            capsfilter1,
            encoder,
            rtph,
            sink,
            NULL);

    gst_element_link_many(
            src,
            capsfilter0,
            app,
            videoconvert,
            capsfilter1,
            encoder,
            rtph,
            sink,
            NULL);

    g_info("Setting pipeline to PLAYING");
    gst_element_set_state(pipeline, GST_STATE_PLAYING);

    g_info("Starting main loop");
    g_main_loop_run(gloop);

    gst_element_set_state(pipeline, GST_STATE_NULL);
    gst_object_unref(pipeline);
    g_main_loop_unref(gloop);

    return 0;
}
```

Since we haven't build the plugin `myfilter` yet, note that we are using `"identity"` in the `gst_element_factory_make` call. Later on, we'll replace this with `myfilter` once we've built and installed the plugin.

Setup the build directory and compile the code with

```
meson setup build
ninja -C build
```

run the application with:
```
GST_DEBUG=2 G_MESSAGES_DEBUG=all ./build/main
```

To recieve the video, you need to run the following GStreamer pipeline on the host:

```
gst-launch-1.0 -e \
    udpsrc port=$1 \
    caps="application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" \
    ! rtpjitterbuffer \
    ! rtph264depay \
    ! decodebin \
    ! videoconvert \
    ! glimagesink
```

You should get a window with a ball bouncing around like so:

![](/assets/mov/ball-udp-recieve.mov)

## Adding a Custom Plugin

The next step is we'll want a custom GStreamer Plugin.  We'll use this plugin to run any kind of OpenCV code on the video buffers, but for now let's just get a basic passthrough into the pipeline.  We'll also need to modify the `meson.build` file.

Grab the boilerplate code from `https://gitlab.freedesktop.org/gstreamer/gst-template`

Follow the instructions at `https://gstreamer.freedesktop.org/documentation/plugin-development/basics/boiler.html?gi-language=c`

You'll run the command `../tools/make_element myfilter` and it will generate the `gstmyfilter.cpp` and `gstmyfilter.h` in the gst-template repo directory.

I'll modify my folder structure a little with the following:

```
myapp
 |- meson.build
 |- src
     |- main.cpp
     plugin
       |-gstmyfilter.cpp
       |-gstmyfilter.hpp
```

I coped the gst-plugin.c and gst-plugin.h into the `src/plugin` folder and renamed them to `cpp/hpp`

Then add the following to the `meson.build` file:
{% highlight yaml %}

plugins_install_dir = join_paths(get_option('libdir'), 'gstreamer-1.0')
plugin_cpp_args = ['-DHAVE_CONFIG_H']
api_version = '1.0'

cdata = configuration_data()
cdata.set_quoted('PACKAGE_VERSION', gst_version)
cdata.set_quoted('PACKAGE', 'gst-template-plugin')
cdata.set_quoted('GST_LICENSE', 'LGPL')
cdata.set_quoted('GST_API_VERSION', api_version)
cdata.set_quoted('GST_PACKAGE_NAME', 'GStreamer template Plug-ins')
cdata.set_quoted('GST_PACKAGE_ORIGIN', 'https://gstreamer.freedesktop.org')
configure_file(output : 'config.h', configuration : cdata)

cc = meson.get_compiler('c')

# The myfilter Plugin
 gstmyfilter_sources = [
  'src/plugin/gstmyfilter.cpp',
  ]

gstmyfilterplugin = library('gstmyfilter',
  gstmyfilter_sources,
  cpp_args: plugin_cpp_args,
  dependencies : [gst_dep, gstbase_dep, opencv_dep],
  install : true,
  install_dir : plugins_install_dir,
)

{% endhighlight %}

### Build and Install the plugin.

If use the above `main.cpp` with the `myfilter` plugin you can build everything and install your myfilter plugin to the 
GST_PLUGIN_PATH directory.

You'll need to change the `identify` block to our `myfilter` 

```
    myfilter = gst_element_factory_make("identity", "myfilter0");
    myfilter = gst_element_factory_make("myfilter", "myfilter0");
```
You may want to set that in your `.zshrc` or `.bashrc`

export GST_PLUGIN_PATH=/usr/local/lib/aarch64-linux-gnu/gstreamer-1.0

```
ninja -C build install
```

You can test if you plugin build by running

```
gst-inspect-1.0 myfilter
```

You should get the following output:

```
Factory Details:
  Rank                     none (0)
  Long-name                MyFilter
  Klass                    FIXME:Generic
  Description              FIXME:Generic Template Element
  Author                   root <<user@hostname.org>>

Plugin Details:
  Name                     myfilter
  Description              myfilter
  Filename                 /usr/local/lib/aarch64-linux-gnu/gstreamer-1.0/libgstmyfilter.so
  Version                  1.20.3
  License                  LGPL
  Source module            gst-template-plugin
  Binary package           GStreamer template Plug-ins
  Origin URL               https://gstreamer.freedesktop.org

GObject
 +----GInitiallyUnowned
       +----GstObject
             +----GstElement
                   +----GstMyFilter

Pad Templates:
  SINK template: 'sink'
    Availability: Always
    Capabilities:
      ANY

  SRC template: 'src'
    Availability: Always
    Capabilities:
      any

```

Running the application now should result in the same ball video you saw before.


