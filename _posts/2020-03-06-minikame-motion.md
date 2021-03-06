---
layout: default
title: miniKame Motion
---

<script src="/js/stimulus.js"></script>
<style>
  fl-br { display: inline-block; line-height: 1em; height: 1em; }
  fr-bl { display: inline-block; line-height: 1em; height: 1em; }
</style>
<script type="module">
  window.customElements.define('fl-br', class extends HTMLElement {
    constructor() {
      super();
    }
    connectedCallback() {
      this.innerHTML = '<svg xmlns="http//www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" width="1em" height="1em"><path d="M5 3a2 2 0 00-2 2v2a2 2 0 002 2h2a2 2 0 002-2V5a2 2 0 00-2-2H5z M11 13a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z" /></svg>';
    }
  });
  window.customElements.define('fr-bl', class extends HTMLElement {
    constructor() {
      super();
    }
    connectedCallback() {
      this.innerHTML = '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" width="1em" height="1em"><path d="M5 11a2 2 0 00-2 2v2a2 2 0 002 2h2a2 2 0 002-2v-2a2 2 0 00-2-2H5z M11 5a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V5z" /></svg>';
    }
  });
  window.changeInput = (selector, newValue) => {
    const el = document.querySelector(selector);
    el.value = newValue;
    el.dispatchEvent(new Event('input'));
  };
  window.checkCheckbox = (selector) => {
    const el = document.querySelector(selector);
    el.checked = true;
    el.dispatchEvent(new Event('input'));
  };
</script>

# Minikame Motion

A few months ago, I built my own version of [miniKame][1], a 3D printed quadruped
robot powered by an ESP8266. I built it mostly as per the instructions in the
GitHub wiki, but redesigned all the parts in Fusion 360 based off of the original
FreeCAD files as an exercise to gain experience using Fusion 360. I also didn't
reuse the project's code as-is, but instead implemented a very small UDP listener
for the ESP8266 that simply reads values for the eight servos from an UDP packet.
I implemented the actual control software in Python on my computer, which gave
me the ability to experiment more quickly, without having to wait for the ESP8266
to finish flashing after every change. Sending servo values via UDP turns out to
be fast enough on my WiFi network, with only occasional lags and hickups.

To get my miniKame walking, I ported some of the original C code to Python, which
worked well after some fiddling. I got the robot moving.

<video controls>
  <source src="/images/minikame.mp4" type="video/mp4" />
  Your browser does not support the video tag.
</video>

Other stuff came up and I set the project aside for a while, but a recent
conversation with a friend prompted me to look at the code again. I found that
I didn't understand how the code got the robot to move as smoothly as it does
anymore. I probably never truly did, I honestly mostly just ported the C code and
fiddled around with the values. But I definitely didn't understand what was going
on anymore looking at it after a couple of months had passed. So I went to work
reverse engineering the original C code and what I'd ported to Python a few
months back. These are my results.

## How miniKame is built

miniKame is a quadruped, which means that it has four legs and four feet that it
can use to move around. Each leg is powered by two servos, one controlling the
XY-direction (I call that one the "brace"), and one controlling the Z-direction
(I call that one the "foot")[^2].

<img src="/images/minikame-labeled.jpg" alt="miniKame with labels" />

Each servo has a limited useful range of motion due to the way miniKame is
constructed -- at some point the XY-brace simply cannot move further because it's
up against miniKame's body. So we'll have to limit the range of motion in
software.

## What makes miniKame move

In the original miniKame code and in my Python port, each servo is controlled
by a sine wave oscillator. That means that for each servo, there's one sine
wave that controls its position at a certain point in time. All oscillators are
on the same "clock" -- in my Python code, I simply use the milliseconds elapsed
since the start of the program. I update the servos once every millisecond in
my Python code.

```
while True:
  time.sleep(1.0 / 1000.0)
  t = datetime.now().timestamp() * 1000.0
  # leg_fl = leg front left, leg_br = leg back right
  leg_fl.xy_servo.set_angle(leg_fl_xy_osc.value(t))
  leg_br.xy_servo.set_angle(leg_br_xy_osc.value(t))
  # Other servos are similar
```

But *how* do these eight sine waves (two for each leg) result in a smooth walking
motion that actually moves the robot forward? Just looking at the thing moving,
then looking at the code, then looking at the moving robot again didn't make
things click for me. I could not figure out how the sine waves were connected to
the actual motion. So I built some simulations.

## Properties of a sine wave

First, some basics about sine wave. Most of this will probably be familiar from
high school. We will discuss amplitude, period, phase shift, and offset of a
sine wave[^3]. Feel free to skip this part if you're comfortable with those
terms.

We can get a basic sine wave using the formula y(t) = sin(t).

<div data-controller="sine"
     data-sine-display-ticks-value="false"
     data-sine-x-labels-value="0 π 2π 3π 4π 5π 6π 7π 8π 9π 10π"
     data-sine-period-value="628"
     data-sine-ticks-value="3141"
     class="py-6"
  >
  <canvas class="w-full h-36" id="plain" data-sine-target="canvas"></canvas>
</div>

This plot shows the value of sin(t) from 0 to 10 * π. An unmodified sine wave
will range from -1 to 1, starting at zero and reaching zero again for every
multiple of π.

We can introduce a few parameters to modify the sine wave. First, we'll talk
about the *period*. The period is the distance on the x axis between two peaks
on the y axis. In the example above that'd be 2π: the first peak occurs at
1.5π, the second peak at 3.5π. You can play around with the period in the
following example. I've changed the values a bit so that we're no longer dealing
in multiples of π, but in nice round numbers instead.

y(t) = sin(period * t)

<div data-controller="sine"
     data-sine-display-ticks-value="false"
     data-sine-x-labels-value="0 500 1000 1500 2000"
     data-sine-period-value="1000"
     data-sine-ticks-value="2000"
     class="flex flex-col space-y-4 py-6"
  >
  <canvas class="w-full h-36" data-sine-target="canvas"></canvas>
  <div class="flex flex-col space-y-4 md:flex-row md:space-x-6 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="period">Period</label>
      <input
        data-sine-target="period"
        data-action="sine#changePeriod"
        type="range"
        min="100"
        max="1000"
        step="1"
        name="period"
      />
      <input type="text" data-sine-target="periodDisplay" readonly="true" />
    </div>
  </div>
</div>

Next up: amplitude. The amplitude of a sine wave determines the height of its
peaks and the depth of its valleys. The default amplitude of an unmodified sine
wave is 1, which is why the plots above show the sine wave ranging from -1 to
1.

y(t) = amplitude * sin(period * t)

<div data-controller="sine"
     data-sine-display-ticks-value="false"
     data-sine-x-labels-value="0 500 1000 1500 2000"
     data-sine-period-value="1000"
     data-sine-ticks-value="2000"
     class="flex flex-col space-y-4 py-6"
  >
  <canvas class="w-full h-36" data-sine-target="canvas"></canvas>
  <div class="flex flex-col space-y-4 md:flex-row md:space-x-6 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="period">Period</label>
      <input
        data-sine-target="period"
        data-action="sine#changePeriod"
        type="range"
        min="100"
        max="1000"
        step="1"
        name="period"
      />
      <input type="text" data-sine-target="periodDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="amplitude">Amplitude</label>
      <input
        data-sine-target="amplitude"
        data-action="sine#changeAmplitude"
        type="range"
        min="0"
        max="2"
        step="0.1"
        name="amplitude"
      />
      <input type="text" data-sine-target="amplitudeDisplay" readonly="true" />
    </div>
  </div>
</div>

Third, we can shift the sine wave around on the x axis. This is called a
*phase shift*, or simply *phase*. The phase is specified in radians. A phase
shift of 1/2π means that at x = 0 the sine wave will be at its maximum (as
determined by the amplitude), instead of at zero.

y(t) = amplitude * sin(period * (t + phase))

<div data-controller="sine"
     data-sine-display-ticks-value="false"
     data-sine-x-labels-value="0 500 1000 1500 2000"
     data-sine-period-value="1000"
     data-sine-ticks-value="2000"
     class="flex flex-col space-y-4 py-6"
  >
  <canvas class="w-full h-36" data-sine-target="canvas"></canvas>
  <div class="flex flex-col space-y-4 md:flex-row md:space-x-6 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="period">Period</label>
      <input
        data-sine-target="period"
        data-action="sine#changePeriod"
        type="range"
        min="100"
        max="1000"
        step="1"
        name="period"
      />
      <input type="text" data-sine-target="periodDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="amplitude">Amplitude</label>
      <input
        data-sine-target="amplitude"
        data-action="sine#changeAmplitude"
        type="range"
        min="0"
        max="2"
        step="0.1"
        name="amplitude"
      />
      <input type="text" data-sine-target="amplitudeDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="phase">Phase</label>
      <input
        data-sine-target="phase"
        data-action="sine#changePhase"
        type="range"
        min="0"
        max="7"
        step="0.01"
        name="phase"
      />
      <input type="text" data-sine-target="phaseDisplay" readonly="true" />
    </div>
  </div>
</div>

Finally, we can shift the whole wave vertically. Right now it's always centered
around zero on the y axis, which we can change using an *offset*.

y(t) = amplitude * sin(period * (t + phase)) + offset

<div data-controller="sine"
     data-sine-display-ticks-value="false"
     data-sine-x-labels-value="0 500 1000 1500 2000"
     data-sine-period-value="1000"
     data-sine-ticks-value="2000"
     class="flex flex-col space-y-4 py-6"
  >
  <canvas class="w-full h-36" data-sine-target="canvas"></canvas>
  <div class="flex flex-col space-y-4 md:flex-row md:space-x-6 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="period">Period</label>
      <input
        data-sine-target="period"
        data-action="sine#changePeriod"
        type="range"
        min="100"
        max="1000"
        step="1"
        name="period"
      />
      <input type="text" data-sine-target="periodDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="amplitude">Amplitude</label>
      <input
        data-sine-target="amplitude"
        data-action="sine#changeAmplitude"
        type="range"
        min="0"
        max="2"
        step="0.1"
        name="amplitude"
      />
      <input type="text" data-sine-target="amplitudeDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="phase">Phase</label>
      <input
        data-sine-target="phase"
        data-action="sine#changePhase"
        type="range"
        min="0"
        max="7"
        step="0.01"
        name="phase"
      />
      <input type="text" data-sine-target="phaseDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="offset">Offset</label>
      <input
        data-sine-target="offset"
        data-action="sine#changeOffset"
        type="range"
        min="-1"
        max="1"
        step="0.01"
        name="offset"
      />
      <input type="text" data-sine-target="offsetDisplay" readonly="true" />
    </div>
  </div>
</div>

## How sine waves are used to move miniKame's braces

After this refresher on sine waves: How do we actually *use* sine waves to move
the robot? Let's start with the braces, the parts that move a leg in the
X-Y-plane.

Using a sine wave to move these is actually pretty straightforward: We tweak
the amplitude until we get the desired range of motion out of the servo, using
the offset to adjust for its initial zero position (which can always vary a bit
because of different mounting positions). Then we tweak the period until the
speed of motion is just right.

<div data-controller="single-brace" class="flex flex-col space-y-4 py-6">
  <div class="flex flex-row">
    <canvas class="w-1/2 h-36" data-single-brace-target="braceCanvas"></canvas>
    <canvas class="w-1/2 h-36" data-single-brace-target="waveCanvas"></canvas>
  </div>
  <div class="flex flex-col md:flex-row md:space-x-6 space-y-4 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="period">Period</label>
      <input
        data-single-brace-target="period"
        data-action="single-brace#changePeriod"
        type="range"
        min="100"
        max="2000"
        step="1"
        name="period"
      />
      <input type="text" data-single-brace-target="periodDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="amplitude">Amplitude</label>
      <input
        data-single-brace-target="amplitude"
        data-action="single-brace#changeAmplitude"
        type="range"
        min="0"
        max="2"
        step="0.1"
        name="amplitude"
      />
      <input type="text" data-single-brace-target="amplitudeDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="phase">Phase</label>
      <input
        data-single-brace-target="phase"
        data-action="single-brace#changePhase"
        type="range"
        min="0"
        max="7"
        step="0.01"
        name="phase"
      />
      <input type="text" data-single-brace-target="phaseDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="offset">Offset</label>
      <input
        data-single-brace-target="offset"
        data-action="single-brace#changeOffset"
        type="range"
        min="-1"
        max="1"
        step="0.01"
        name="offset"
      />
      <input type="text" data-single-brace-target="offsetDisplay" readonly="true" />
    </div>
  </div>
</div>

When looking at a video of miniKame moving, you'll notice that not all braces
move in unison. The front left and back right braces <fl-br></fl-br> are always
moving in the opposite direction of the front right and back left <fr-bl></fr-bl>
braces. To achieve this, we use a phase shift of 1.5π, or 270 degrees, on the
front right and back left <fr-bl></fr-bl> brace sine waves and a phase shift of
0.5π, or 90 degrees, on the front left and back right <fl-br></fl-br> braces.

<div data-controller="four-braces" class="flex flex-col space-y-4 py-6">
  <div class="flex flex-row">
    <canvas class="h-36 w-1/2" data-four-braces-target="braceCanvas"></canvas>
    <div class="flex flex-col w-1/2">
      <canvas class="h-18 w-full" data-four-braces-target="waveBRFLCanvas"></canvas>
      <canvas class="h-18 w-full" data-four-braces-target="waveBLFRCanvas"></canvas>
    </div>
  </div>
  <div class="flex flex-col space-y-4 md:flex-row md:space-x-6 md:space-y-0">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="phaseBRFL">Phase FL/BR <fl-br></fl-br></label>
      <input
        data-four-braces-target="phaseBRFL"
        data-action="four-braces#changePhaseBRFL"
        type="range"
        min="0"
        max="7"
        step="0.01"
        name="phaseBRFL"
      />
      <input type="text" data-four-braces-target="phaseBRFLDisplay" readonly="true" />
    </div>
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="phaseBLFR">Phase FR/BL <fr-bl></fr-bl></label>
      <input
        data-four-braces-target="phaseBLFR"
        data-action="four-braces#changePhaseBLFR"
        type="range"
        min="0"
        max="7"
        step="0.01"
        name="phaseBLFR"
      />
      <input type="text" data-four-braces-target="phaseBLFRDisplay" readonly="true" />
    </div>
  </div>
</div>

## How sine waves are used to move miniKame's feet

Now we can move our legs in the X-Y-plane by rotating the braces. To actually
move the robot forward, we also have to move its feet. What we want is to have
the feet touch the ground when the braces are in their frontmost position and
stay down while they move backwards. Then we'll lift the feet up again as the
braces reach their back positions. As long as the ground isn't too slippery, this
will move the robot forward.

The motion itself is simple. Because of the way the feet are mounted, all we have
to do is move the servo for each foot back and forth to move the foot up and down
from the ground. A simple sine wave, not unlike the ones that we've used above to
move the brace. Amplitude and offset will probably have to be different to achieve
the correct range of motion, but all in all still a simple sine wave. But how do
we achieve the specific timing we need: move the foot down when the brace is in
front, keep it down while the brace moves to the back, and then move it up again?

The way we've set up the oscillators for the brace servos, the sine wave reaches
its peak position when the brace is in its front position and is at its lowest
point when the brace is in its back position. What we want is for the leg
oscillator to move through one complete cycle -- from peak to valley and back to
peak, from its up position to its down position and back up -- while the brace
oscillator moves through half a cycle -- from front to back.

We can achieve that by setting the period of the feet oscillators to half that
of the brace oscillators and using a phase of 0.5π / 90 degrees, the same as
the front left / back right <fl-br></fl-br> brace oscillators. Both sets of
oscillators start in their peak position, but since the period of the feet
oscillators is half that of the brace oscillators, the feet oscillators will
complete a full cycle (up, down, up) in the time it takes the brace oscillators
to complete a half cycle (front, back). Try it in the simulation below. You can
use the slider to move through time.

<div data-controller="foot-and-brace" class="flex flex-col space-y-4 py-6">
  <div class="grid gap-2 grid-rows-2 grid-cols-2">
    <canvas class="col-start-1 row-start-1 h-40 w-full" data-foot-and-brace-target="braceCanvas"></canvas>
    <canvas class="col-start-1 row-start-2 h-40 w-full" data-foot-and-brace-target="footCanvas"></canvas>
    <canvas class="col-start-2 row-start-1 h-40 w-full" data-foot-and-brace-target="braceWaveCanvas"></canvas>
    <canvas class="col-start-2 row-start-2 h-40 w-full" data-foot-and-brace-target="footWaveCanvas"></canvas>
  </div>
  <div class="flex flex-row space-x-6">
    <div class="flex flex-col flex-shrink min-w-0 space-y-1">
      <label class="font-semibold" for="time">Time</label>
      <input
        data-foot-and-brace-target="time"
        data-action="foot-and-brace#changeTime"
        type="range"
        min="0"
        max="3000"
        step="10"
        name="time"
        id="foot-and-brace-time"
      />
      <input type="text" data-foot-and-brace-target="timeDisplay" readonly="true" />
    </div>
    <div class="flex-shrink min-w-0 space-y-1">
      <input
        data-foot-and-brace-target="activeSide"
        data-action="foot-and-brace#toggleActiveSide"
        type="checkbox"
        name="active-side"
        id="foot-and-brace-filter"
      />
      <label class="font-semibold" for="active-side">Period Filter</label>
    </div>
  </div>
</div>

This seems like it works at first, but there's a problem. In the first cycle,
everything works fine. At <a href="javascript:void(0)" onClick="changeInput('#foot-and-brace-time', 0);">t = 0</a>,
the brace is in the front position and the foot in the up position. When the
brace is halfway through its motion to the back position, at
<a href="javascript:void(0)" onClick="changeInput('#foot-and-brace-time', 250);">t = 250</a>,
the foot is in the down position, touching the ground. When the brace is in the
back position, at <a href="javascript:void(0)" onClick="changeInput('#foot-and-brace-time', 500);">t = 500</a>,
the foot is back up again. Our robot has moved forward. But if we keep increasing
the time, we see that the brace naturally starts to move back to its front
position. But the foot keeps moving too! In fact, at <a href="javascript:void(0)" onClick="changeInput('#foot-and-brace-time', 750);">t = 750</a>,
it's back on the ground. That will undo the forward motion we just made. We
want the foot to move *only* when the brace is moving from front to back, not
when it's moving from back to front.

To solve this, the miniKame code uses a little trick. It disables the foot
oscillator for half of the period.

```
while True:
  time.sleep(1.0 / 1000.0)
  t = datetime.now().timestamp() * 1000.0
  # leg_fl = leg front left, leg_br = leg back right
  leg_fl.xy_servo.set_angle(leg_fl_xy_osc.value(t))
  if int(t / 500.0) % 2 == 0:
    leg_fl.z_servo.set_angle(leg_fl_z_osc.value(t))
  # Other servos are similar
```

When the result of `t / 500.0` is even, we're in the first half of the 1000 tick
long period of the brace, which means that we're in the first half of its motion,
i.e. the front-to-back motion. If it's odd, then we're in the second half, the
back-to-front motion, so we simply don't pass the oscillator values to the servo.
To see this in action,
<a href="javascript:void(0)" onClick="checkCheckbox('#foot-and-brace-filter')">
activate the period filter checkbox</a> above and move the time slider. You'll
see that the foot stays up when the brace moves from back to front.

We can use the same trick to make the up-down motion work for both the front left/
back right <fl-br></fl-br> legs and the front right/back left <fr-bl></fr-bl> legs.
Remember that we used a phase shift to make both pairs move in opposite directions
from one another. We need the front right/back left <fr-bl></fr-bl> feet to move
up and down when their corresponding braces are going through the
front-to-back motion. Because of the phase shift, that's precisely the time when
the front left/back right <fl-br></fl-br> braces are going through their
back-to-front motion, which is the time slot that we "blocked" through the
period filter above. If we reverse the period filter for the front right/back
left <fr-bl></fr-bl> feet oscillators, their movement will be synced up with the
front right/back left <fr-bl></fr-bl> brace front-to-back motion.

## The complete motion

Put all of the above together, and you get a miniKame robot that can move
forward. To show how it all works together, I built a small simulation using
[Three.js][2][^1]. You can see the individual sine waves below the simulation
and inside the simulation itself.
Click and drag to rotate the camera. Hold shift and drag to move. Scroll to zoom.
If you move the camera a bit, it's pretty easy to see how it all plays together.

<div id="sides" class="flex flex-col space-y-4">
  <canvas id="scene" class="w-full"></canvas>
  <div class="grid grid-cols-2 grid-rows-2 gap-2">
    <div class="col-start-1 row-start-1">
      <div><strong>Front Left</strong></div>
      <div class="flex">
        <canvas class="h-full w-1/2" id="graph-fl-brace"></canvas>
        <canvas class="h-full w-1/2" id="graph-fl-leg"></canvas>
      </div>
    </div>
    <div class="col-start-1 row-start-2">
      <div><strong>Front Right</strong></div>
      <div class="flex">
        <canvas class="h-full w-1/2" id="graph-fr-brace"></canvas>
        <canvas class="h-full w-1/2" id="graph-fr-leg"></canvas>
      </div>
    </div>
    <div class="col-start-2 row-start-1">
      <div><strong>Back Left</strong></div>
      <div class="flex">
        <canvas class="h-full w-1/2" id="graph-bl-brace"></canvas>
        <canvas class="h-full w-1/2" id="graph-bl-leg"></canvas>
      </div>
    </div>
    <div class="col-start-2 row-start-2">
      <div><strong>Back Right</strong></div>
      <div class="flex">
        <canvas class="h-full w-1/2" id="graph-br-brace"></canvas>
        <canvas class="h-full w-1/2" id="graph-br-leg"></canvas>
      </div>
    </div>
  </div>
</div>

<script type="module">
import { setup3DView, animate3DView, setupStimulusControllers } from '/js/minikame.js';

const scene = setup3DView();

requestAnimationFrame(t => animate3DView(t, scene));

setupStimulusControllers();
</script>

[1]: https://github.com/JavierIH/miniKame
[2]: https://threejs.org

[^1]: If you want to learn Three.js, check out https://threejsfundamentals.org. I had absolutely zero prior knowledge of Three.js and the last time I did any graphics was years ago. Using their guide, I was able to build this simple simulation in a couple of hours.
[^2]: Z-direction is not strictly true since it also moves in the XY-direction, but I *think* that's irrelevant for understanding how the basic motion works.
[^3]: I think I got the definitions right. Not a mathematician, though.