---
layout: post
title:  "Making a GStreamer Plugin with OpenCV (Part 2)"
date:   2022-09-22 12:34:00 -0800
---
In the last post I covered the basic of getting a GStreamer pipeline sending test video through a basic plugin block, then streamed to the host computer over UDP.  The next step is to incorporate the OpenCV framework into our plugin and then convert the GStreamer video buffers into something OpenCV can ingest.

3. Add the OpenCV libraries to the plugin.  This involves changing the Gstreamer boilerplate `meson.build` to compile with `c++` instead of `c`.
4. In the plugin get the video frame out of the `GstBuffer` format and into `cv::Mat` so OpenCV can use it.  Then I'll have to extract the data from the `cv::Mat` and put it back into the `GstBuffer` so it can run through the remaining downstream GStreamer pipeline.

## Adding the OpenCV dependency

We'll first need to add the OpenCV dependencies into our `meson.build` file.

```
opencv_dep = dependency('opencv4', version : '>= 0.29.2',
  required : true)
```

Then we'll need to add that dependency to the MyFilter plugin

```
gstmyfilterplugin = library('gstmyfilter',
  gstmyfilter_sources,
  cpp_args: plugin_cpp_args,
  dependencies : [gst_dep, gstbase_dep, opencv_dep],
  install : true,
  install_dir : plugins_install_dir,
)
```

Now we should be able to add the OpenCV includes to the top of gstmyfilter.cpp and build the application

```
#include <opencv2/opencv.hpp>
#include "opencv2/core/types.hpp"
#include "opencv2/imgproc.hpp"
#include "opencv2/tracking.hpp"
```

## Converting the GstBuffer to cv::Mat

Great.  Next we need to take the GstBuffer coming in to `myfilter` and change it into something that OpenCV can ingest - a `cv::Mat`.  To keep things simple, let's assume the video color format is BGR - which plays really nice with OpenCV.  We'll want to detect the size of the buffer for referencing later.

* Add member variables to the filter `guint height` and `guint width`.  These get accessed with `filter->height` for example.
* in `gst_my_filter_sink_event` we'll detect a GST_CAPS event and extract the height and width of the buffer from the caps.
* in `gst_my_filter_chain` we'll extract the buffer and pop it into a cv::Mat
* finally we'll put the cv::Mat back into a GstBuffer so the rest of the GStreamer pipeline can run



{% highlight cpp %}
static gboolean
gst_my_filter_sink_event (GstPad * pad, GstObject * parent,
    GstEvent * event)
{
  GstMyFilter *filter;
  gboolean ret;

  filter = GST_MYFILTER (parent);

  GST_LOG_OBJECT (filter, "Received %s event: %" GST_PTR_FORMAT,
      GST_EVENT_TYPE_NAME (event), event);

  switch (GST_EVENT_TYPE (event)) {
    case GST_EVENT_CAPS:
    {
      GstCaps *caps;

      gst_event_parse_caps (event, &caps);
      /* do something with the caps */
      GstStructure *caps_struct = gst_caps_get_structure(caps, 0);
      if( !caps_struct) {
        g_warning("caps have NULL structure");
      }

      if( !gst_structure_get_int(caps_struct, "width", &filter->width) ||
          !gst_structure_get_int(caps_struct, "height", &filter->height)) {
        g_warning("caps have no HEIGHT, WIDTH");
      }

      g_info("[myfilter] caps: %s\n", gst_caps_to_string(caps));

      /* and forward */
      ret = gst_pad_event_default (pad, parent, event);
      break;
    }
    default:
      ret = gst_pad_event_default (pad, parent, event);
      break;
  }
  return ret;
}

{% endhighlight %}

With that in place when we start the pipeline we should get an GST_EVENT_CAPS that let's our filter identify the height/width of the buffer.

Here's my output when I run the application:

```
*[main][/workspace/myapp]$ ./build/main
** INFO: 16:35:19.940: building gst pipeline
** INFO: 16:35:19.997: Setting pipeline to PLAYING
** INFO: 16:35:19.997: Starting main loop
** INFO: 16:35:19.998: [myfilter] caps: video/x-raw, format=(string)BGR, width=(int)320, height=(int)240, framerate=(fraction)30/1, multiview-mode=(string)mono, pixel-aspect-ratio=(fraction)1/1, interlace-mode=(string)progressive
```

Having the height and width of the buffer will let us create a `cv::Mat` object we can use to store the buffer.  This work will happen in the `gst_my_filter_chain` function. I'm also going to add a check to make sure we have the height/width of the buffer and print a warning if we never received the GST_EVENT_CAPS.


{% highlight cpp %}

/* chain function
 * this function does the actual processing
 */
static GstFlowReturn
gst_my_filter_chain (GstPad * pad, GstObject * parent, GstBuffer * buf)
{
  GstPluginTemplate *filter;

  filter = GST_PLUGIN_TEMPLATE (parent);


  if(filter->height == 0 or filter->width == 0){
      g_warning("did not recieve caps height/width before first buffer");
    } else {

    GstMapInfo map;
      if(gst_buffer_map(buf, &map, GST_MAP_READ)) {

        cv::Mat cvmat(cv::Size(filter->width, filter->height), CV_8UC3, (char*)map.data, cv::Mat::AUTO_STEP);
      }
    }

  /* just push out the incoming buffer without touching it */
  return gst_pad_push (filter->srcpad, buf);

}

{% endhighlight %}

Note that the `return` at the end of the function is just sending out the original, unmodified buffer so the pipeline will work while we deal with the OpenCV part.

First thing I did was use `gst_buffer_map` to run a memory map operation on the buffer to get a `GstMapInfo` object.
From the `GstMapInfo` object we can access the underlying buffer data as `guint8*`

I'm creating a `cv::Mat` with the given size derived from the height/width. In the constructor I'm going to set the matrix a `CV_8UC3` which means 8-bit with 3 color channels.  We'll load in the BGR three channel color buffer into the constructor with the `(char)*map.data` using the `GstMapInfo` object we created.

Now we can do something with the `cv::Mat` object.  Let's try the following.

1. Convert to Grayscale
2. Run the video through a Gaussian Blur
3. Run Canny edge detection on the video.
4. Convert the image back into BGR

{% highlight cpp %}

  GstMapInfo map;
  if(gst_buffer_map(buf, &map, GST_MAP_READ)) {

      cv::Mat cvmat(cv::Size(filter->width, filter->height), CV_8UC3, (char*)map.data, cv::Mat::AUTO_STEP);
      cv::cvtColor(cvmat, cvmat, cv::COLOR_BGR2GRAY);
      cv::blur(cvmat, cvmat, cv::Size(3, 3));
      cv::Canny(cvmat, cvmat, 100, 200, 3, FALSE);
      cv::cvtColor(cvmat, cvmat, cv::COLOR_GRAY2BGR);
  }

{% endhighlight %}

Note I'm using the same `cv::Mat` object as the input and output image for these operations, so I'll lose the original video.

Finally as the last step, we'll need to put the `cv::Mat` data back into a GstBuffer object to send out the `src` pad of `myfilter`


{% highlight cpp %}
// convert back to GstBuffer
// get the number of bytes needed to capture buffer
gsize size_bytes = filter->height * filter->width * 3;

GstBuffer *buffer = gst_buffer_new_wrapped_full( (GstMemoryFlags)0,
   (gpointer)(cvmat.data), size_bytes, 0, size_bytes, NULL, NULL);

// apply timing info back to buffer
buffer->pts = buf->pts;
buffer->dts = buf->dts;
buffer->duration = buf->duration;
buffer->offset = buf->offset;

ret =  gst_pad_push (filter->srcpad, buffer);

{% endhighlight %}

If we rebuild and install (`ninja -C build install`) the plugin we should get the following video on our host machine over UDP.


![](/assets/mov/ball-udp-canny.mov)









