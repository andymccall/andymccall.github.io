---
layout: post
title:  "Agon Light 2 Acade Controller Board"
date:   2024-07-09 20:11:47 +0100
categories: electronics retrocomputing agon basic
---
Over the last few weeks I've been playing with the Agon Light 2.  It's a great little 8bit retro computer that uses a eZ80 as it's CPU and an ESP32-PICO-D4 as a graphics and sound co-processor. [It's essentially everything that the Commander X16 was supposed to be](https://www.youtube.com/watch?v=mwnNEq3e6cw&t=7s), that's not to say I don't like the Commander X16, I've got one of those as well, it's just the project has taken a life of it's own and it's now something different to what it was originally supposed to be.

This week I built the EIGHTSBITWIDE Arcade board for the Agon so I can use game controllers with it. It's a neat little PCB that uses ten 10k resistors, two 9-pin angled D-Sub connectors and a 34-pin connector. Combined with the Agon Light 2, you basically turn your machine into the [Agon Console8](https://heber.co.uk/agon-console8/).

Building the card was really nice easy, which I was grateful for as I've not soldered for a couple of years. I had difficulty finding resistors that fitted, as the first two batches I ordered had legs were too thick, but after asking in the Discord channel for some advice on the exact parts I needed EIGHTBITSWIDE himself sent me some in the post along with a second PCB. The 34-pin connector is also difficult to get, but eBay came to the rescue and I've now got more connectors that I'll ever need.

![Agon Light 2 EIGHTBITSWIDE Arcade](/assets/images/20240709_eightbitswide_arcade.png)

The card itself is well layed out and getting access to solder each part is nice and easy. The silkscreen has the following 15 line test program in BASIC included on it:

{% highlight ruby %}
   10 UP=253 : DOWN=247 : LEFT=223
   20 RIGHT=127 : FIRE=215
   30 PRINTTAB(0,0)
   40 PRINT "158 = ";:PRINT GET(158)
   50 PRINT "162 = ";: PRINT GET(162)
   60 PRINT
   70 X=GET(158)
   80 IF (X AND UP) = X THEN PRINT "UP"
   90 IF (X AND DOWN)=X THEN PRINT "DOWN"
  100 IF (X AND LEFT)=X THEN PRINT "LEFT"
  110 IF (X AND RIGHT)=X THEN PRINT "RIGHT"
  120 X=GET(162)
  130 IF (X AND FIRE)=X THEN PRINT "FIRE"
  140 Y=Y+1 : IF Y=50 THEN CLS:Y=0
  150 GOTO 70
{% endhighlight %}

Including this on the silkscreen is a really good idea as armed with the card and the Agon Light 2 you've got enough to check everything is working. When developing games for the card EIGHTBITSWIDE recommends that you use software debounce as multiple signals can be send from a single button press, but I don't think I'll be doing that for a while as my assembly is still a long way from working with external devices.

Overall I really enjoyed the proejct. It's been years since I built something and having my electronic kit out has spurred me on to fix some of the broken retro computers I've got, which I'll post about another day. If you want to buy a one pre-assembled, you can get one direct from the boards [creator PCBWay site](https://www.pcbway.com/project/shareproject/Agon_ARCADE_Joystick_Interface_893b9d84.html).