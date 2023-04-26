---
layout: post
title:  "Making a GStreamer Plugin with OpenCV (Part 3)"
date:   2023-04-25 12:34:00 -0800
---
In my earlier posts (Part 1 and Part 2) we setup a basic GStreamer Pipeline with a custom plugin and then use OpenCV to manipulate the income buffers before sending it down the rest of our pipeline.  Now we'll try something a little more advanced and run an OpenCV tracking algorithm on the image buffer to track the circle.

This won't be the best implementation - probably a bit hamfisted.  We'll use the OpenCV `tracking.hpp` library and I'll be using the CSRT Tracker - The Channel and Spatial Reliability Tracker.  You can read more about it here: [Disciminative Correlation Filter with Channel and Spatial Reliability - Alan Lukezˇicˇ et al.](https://openaccess.thecvf.com/content_cvpr_2017/papers/Lukezic_Discriminative_Correlation_Filter_CVPR_2017_paper.pdf)

Since we already have the OpenCV libraries included in our `meson.build` and we've already add the `opencv2/tracking.hpp` library to `mygstplugin.hpp`, all that's left is the implementation.

First we'll need to add an instance of the tracker to `gstmyfilter.hpp`, as well as a boolean to indicate the tracker is initialized and a green rectangle to add around the tracked object.  We'll use OpenCV to add the green rectangle.

```
struct _GstMyFilter
{
  GstElement element;

  GstPad *sinkpad, *srcpad;
  gboolean silent;
  gboolean enabled;
  gint width;
  gint height;
  gchar* format;

+++  gboolean tracker_init;
+++  cv::Ptr<cv::Tracker> tracker;
+++  cv::Rect_<int> bbox;

}
```

All the magic will again be happening in the `gst_my_filter_chain` method.  We'll need to do the following:
1. Check if the tracker is initialized, and if it isn't then initialize it.
2. Call the update method for the tracker.
3. Add a green rectangle where the tracker is indicating the pixels are.

Here are some details of how to do each:


1. Initialize the Tracker.  This is a little hacky in that we know the test video spawns the ball in the middle of the image.  If you wanted to be fancy you'd want to detect a coordinate on the image to start tracking.
This is where we setup the tracker as the CSRT method.

```
if(!filter->tracker_init) {
  // box coordinates are 0,0 -> topleft corner of buffer
  guint boxwidth= 50;
  guint topleft_x = filter->width/2.0 - boxwidth/2.0;
  guint topleft_y = filter->height/2.0 - boxwidth/2.0;
  g_info("initializing tracking box at (x,y)=%u, %u", topleft_x, topleft_y);

  //initialize bounding box
  filter->bbox = cv::Rect_<int>(topleft_x, topleft_y, boxwidth, boxwidth);
  filter->tracker = cv::TrackerCSRT::create();
  filter->tracker->init(cvmat, filter->bbox);

  // set the tracker as initialized
  filter->tracker_init = TRUE;
}
```

2. Call the update method of the tracker.

```
gboolean tracking_ok = filter->tracker->update(cvmat, filter->bbox);
```

3. Add a green rectangle to show the tracking.  This is after we convert to color back to BGR.
```
cv::rectangle(cvmat, filter->bbox, cv::Scalar(0, 255, 0), 2, cv::LINE_8);
```

Here is the entire `gst_my_filter_chain` method:

```
/* chain function
 * this function does the actual processing
 */
static GstFlowReturn
gst_my_filter_chain (GstPad * pad, GstObject * parent, GstBuffer * buf)
{

  GstMyFilter *filter;

  GstFlowReturn ret = GST_FLOW_ERROR;

  filter = GST_MYFILTER (parent);

  if(filter->height == 0 or filter->width == 0){
    g_warning("did not recieve caps height/width before first buffer");
  } else {

    if (filter-> enabled) {
      // pack GstBuffer into OpenCV cv::Mat
      GstMapInfo map;
      if(gst_buffer_map(buf, &map, GST_MAP_READ)) {

        cv::Mat cvmat(cv::Size(filter->width, filter->height), CV_8UC3, (char*)map.data, cv::Mat::AUTO_STEP);
        // do something with cv::Mat filter->cvmat
        cv::cvtColor(cvmat, cvmat, cv::COLOR_BGR2GRAY);
        cv::blur(cvmat, cvmat, cv::Size(3, 3));
        cv::Canny(cvmat, cvmat, 100, 200, 3, FALSE);
        //object tracking
+++        if(!filter->tracker_init) {
+++          // box coordinates are 0,0 -> topleft corner of buffer
+++         guint boxwidth = 50;
+++         guint topleft_x = filter->width/2.0 - boxwidth/2.0;
+++          guint topleft_y = filter->height/2.0 - boxwidth/2.0;
+++          g_info("initializing tracking box at (x,y)= %u, %u", topleft_x, topleft_y);

+++          //initial bounding box
+++          filter->bbox = cv::Rect_<int>(topleft_x, topleft_y, boxwidth, boxwidth);
+++          filter->tracker = cv::TrackerCSRT::create();
+++          filter->tracker->init(cvmat, filter->bbox);
+++          filter->tracker_init = TRUE;
+++        }

+++        gboolean tracking_ok = filter->tracker->update(cvmat, filter->bbox);
+++        cv::cvtColor(cvmat, cvmat, cv::COLOR_GRAY2BGR);

+++        cv::rectangle(cvmat, filter->bbox, cv::Scalar(0, 255, 0), 2, cv::LINE_8);

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

        //cleanup opencv
        cvmat.release();

      } else {
        g_warning("could not buffer map");
        ret = GST_FLOW_ERROR;
      }

    } else {
      g_debug("filter not enabled, passthrough buffer");
      ret =  gst_pad_push (filter->srcpad, buf);
    }
  }

  /* just push out the incoming buffer without touching it */
  return ret;
}
```

Let's build and run this.  Remember we'll need to set the plugin path if we haven't included it in our .zshrc.

```
ninja -C build install
GST_PLUGIN_PATH=/usr/local/lib/aarch64-linux-gnu/gstreamer-1.0 ./build/main
```

![](/assets/mov/ball-udp-tracking.mov)

Check out the code on github at https://github.com/dlwalter/gst-opencv-tracking

