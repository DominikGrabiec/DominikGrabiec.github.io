---
layout: post
title: "Lessons Learnt: Know when to bend or break user requirements"
date: 2022-09-18 12:00:00 +0100
category: "Lessons Learnt"
excerpt_separator: <!--more-->
---

> I know that some of these lessons might be very simple or appear to be common sense to some readers here, but ever experienced software developer was once a junior like I was in this story. I also believe that there should be no shame in asking what would appear to be simple or stupid questions, or in sharing stories from long ago.

This lesson is all about learning to evaluate software requirements that you get from the customer and figuring how to bend or break them in order to efficiently apply them to the target hardware.

<!--more-->

I learnt this lesson at my first software development job at the start of my career, pretty much straight out of University. It was a small medical device startup and I joined as the first employee and therefore first software developer on the team. We were building a hand held medical device which was powered by an (integer only) ARM based [XScale processor](https://en.wikipedia.org/wiki/XScale#PXA27x), something like 32 megabytes of RAM, and an 18bpp 240x320 display. It was a pretty fun job for me as I got to design and write the software for the whole system and I managed to get something produced in the few months that I was working solo.

The main application being developed had to display a couple of graphs on the screen based on real time data, and the requirement that is relevant to this story is that it had to show something like 3 or 5 seconds worth. I'm actually not sure  of the exact number of seconds but it doesn't matter that much. Of course I took this very seriously and figured out that we'd need to process something like 533.33 samples per column of the graph in the default view. So I went about designing the architecture and building out the systems.

I ended up building a real time graph system which was able to handle this by keeping a running total of how many samples the graph was over/under while processing the data for each new column, and process more samples when needed to keep the sample count and real world time in sync. So it would end up processing 533, 533, 534 samples in sequence repeatedly. It ended up working pretty well and had the ability to zoom in and out on the graph while real time data was feeding in continuously. 

After a few months of solo development we actually got our second software engineer and first experienced person on the team. He was able to see the work I had done and while I did make some mistakes in the architecture which we needed to change, it wasn’t that bad overall. 

The relevant point to the story is that he showed me was that we didn't need to follow that particular requirement so strictly. So instead of processing 533.33 samples per column we could instead pick a nice round number like 512. Not only did this simplify the processing code, it also unlocked the ability for us to use optimised SIMD functions for big performance gains. It did cost us a reduction in the amount of time that we could display on the x-axis for the data, but it was only a small fraction of a second and as it turned out it wasn't really a hard requirement anyway.

At this time as a junior I was not experienced enough to know that the user requirements were nothing more than guidelines around which to build the software. Granted some user requirements are critical to the proper functioning of the software and must be adhered to, but other requirements are more flexible or aspirational in nature and therefore can be changed or negotiated to better suit the hardware or platform you're building for.

I hope you enjoyed this little tale of a lesson learnt, and if you have any comments or questions feel free to email or contact me on social media.
