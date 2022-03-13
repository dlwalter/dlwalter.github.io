---
layout: post
title:  "The Mars Lander Problem"
date:   2022-03-12 17:24:33 -0800
categories: cpp
---
I was recently introduced to a clever coding problem called [The Mars Lander - Episode 2](https://www.codingame.com/training/medium/mars-lander-episode-2) and found it a fun application of some basic control theory.

![](/assets/img/mars-lander.png)

## The Problem

A simplified 2-D space craft is trying to land autonomously on Mars.  Controlling the angle and thrust of the aircraft, you need to write an algorithm that can land on level ground within certain parameters (speed, angle, etc) before running out of fuel.

You are presented with a game loop that spits out state information to stdin, and then expects a response to stdout for the next iteration.

## A Solution

Here's how I solved the problem.  The general flight control is 2 cascaded proportional controllers to control horizontal position and velocity, and then a single proportional controller to control vertical velocity.

If you want to try out my solution I've saved [a gist located here](https://gist.github.com/dlwalter/dc3a7119548f71f0a6133030bb3ae219)


### Finding the Landing Zone

The first challenge is to find the landing zone.  The 2-D terrain points are first sent to stdout before the game loop starts, so we can read in those x, y coordinates and then parse them.  The landing zone will be the flat part, where two successive coordinates have the same y value.  In addition to the coordinates we also want the "radius" of the landing zone - I know its 2-D so really its just half the width of the flat portion.

I used a `std::pair<double, double>` to store the (x, y) coordinates of the center of the landing zone.  Using typedef to keep the verbosity down.

{% highlight cpp %}
typedef std::pair<double, double> CoordXY;
{% endhighlight %}

I also define a `landing_setpoint_t` type so I can pass the landing center coordinate and radius to the controller.

{% highlight cpp %}
typedef struct /_landing_setpoint {
    CoordXY landing_coordinate;
    double landing_radius;
} landing_setpoint_t;
{% endhighlight %}

So now I am ready to read the surface coordinates from stdin.  I'll use a std::vector<CoordXY> - a vector of our CoordXY object.

```
std::vector<CoordXY> surface_coordinates_xy;
```

### Controlling Position


### Ensuring Feasible Control


 Here's the [replay of the solution in action](https://www.codingame.com/replay/613172447)


{% highlight cpp %}
void foo(int a);
foo(1);
// prints
{% endhighlight %}

[The Mars Lander Problem]: https://www.codingame.com/training/medium/mars-lander-episode-2
[My Solution]: https://gist.github.com/dlwalter/dc3a7119548f71f0a6133030bb3ae219
[My GitHub]: https://github.com/dlwalter

