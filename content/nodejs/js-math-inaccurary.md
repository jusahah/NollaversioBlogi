+++
date = "2017-11-05T06:22:32+02:00"
draft = true
title = "Beware JS cumulating math inaccuracies"

+++

One of the fun things about programming is that math operations on floating point values are inherently inaccurate. This can be seen in Javascript:

```javascript

var b = 0.362 * 100;

console.log(b); // 36.199999999999996

```

Math operation above should produce 36.2, but instead it spews out something else. It is not a large inaccurary, but it is an inaccuracy nevertheless.

Most of the time those small inaccuracies do not cause any troubles; after all, Javascript is not meant to be used in high-precision scientific computing. Javascript is a scripting language for the Web.

However, as always, there is a big gotcha to watch out for: *cumulating inaccuracies during render loop*.

### Small inaccurary turns into a big one

Here is an example how things can quickly go haywire:

```javascript

var c = 0.362 * 100 - 35.2; // Should produce value 1

var frames = 60 * 60; // One minute at 60 FPS

while (frames--) {
  // 1 * 1 should be 1, thus c should never change!
  c = c*c;
}

// c should be 1, but...
console.log("Eventual c: " + c); // 0


```

In the code above we are running a simulated game loop. Every loop run simply multiplies *c* by itself. As this is supposed to be game loop, it spins approximately **60 times a second**.

What happens is that originally small and meaningless inaccuracy quickly *cumulates itself* into a devastating error. At the end of the loop, variable *c* contains value zero.

This is a type of bug that will certainly cause troubles within your program. First of all, it is pretty hard to find in testing because of its accumulating nature. 

Like most multithreading bugs, likelihood of the bug appearing increases with the duration of the program has been running. 

But again... does this bug actually occur in practice?

Yes. I had this bug happen in my Javascript game. I was using PaperJs library, and this bug periodically messed up scales of my PaperJS objects. Following pseudocode demonstrates:

```javascript

var paperObject = new paper.Circle(...);

// This gets called on every render frame.
function setScaleToObject(newScale) {
	paperObject.scaling = {
		x: newScale,
		y: newScale
    };
}

```

Setting scale-values right into paperJS object caused problems. Because, for example, if I expected *newScale* to be
1 but it instead was 0.999999999, PaperJs would store 0.999999999 to its internal data structures. And then somehow that value got repeatedly multiplied until suddenly object just disappeared from the screen. 

> Sudden disappearance is due the fact that the inaccuracy grows slowly at first, but eventually it reaches "critical mass" and starts to grow exponentially. 
>
>
>
> For example: **0.99999 ^ 2** is still pretty close to 0.99999, but **0.9 ^ 2** is clearly different (0.9 vs 0.81). 
>
>
>
> If you think about this in terms of pixels, **0.99999 ^ 2 multiplied by 1000 pixels** still rounds to 1000 pixels. But **0.9 ^ 2 multiplied by 1000 pixels** is only 810 pixels. A huge difference. 

The fix to avoid cumulating errors is to introduce auto-correction to the code and stop setting scale-value directly to paperJS object:

```javascript

var paperObject = new paper.Circle(...);

function scaleObject(newScale) {
	
	// We use objects current scale to auto-adjust our scale change.
	var currentScale = paperObject.getScaling().x;

	// We know currentScale and newScale; now we can calculate how much to scale
	// to achieve newScale given currentScale.

	var change = newScale / currentScale;

	paperObject.scale(change);
}

```

The code above is *auto-correcting*; meaning that if currentScale starts to drift away from epexted exact value (e.g. 0.99999 vs 1), our change calculation will take it into account. This saves the day.











