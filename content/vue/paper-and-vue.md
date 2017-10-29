+++
date = "2017-10-21T12:01:31+03:00"
draft = false
title = "Vue.js reactivity gotcha"

+++

![PaperJs object violated by Vue](/blog/public/img/paper-vue.png)

Vue is great framework. However, one must be careful when using it in apps requiring usage of animation loop (requestAnimationFrame).

Lately I've been using Vue with Paper.js. There is some great synergy between these two when building games or game-like javascript apps. Vue specializes in handling typical UI interactions, while Paper.js takes care of high-speed rendering and animations to canvas. 

In application I am building, Paper.js takes care of running the game (and game loop) and Vue provides HTML elements used to control gameplay. This works well, but there is a big gotcha.

### Beware Vue's reactivity octopus

Lets say we want to build a very simple HTML canvas based game. It is a game where some monster sprites (or whatever) move on the canvas. And then there are HTML buttons above canvas; one button for each monster. Clicking the button deletes the monster on the canvas. Each monster has its own button.

Creating buttons from dynamically changing arrays is something Vue is very good at, so we naturally use *v-for* directive to keep monsters and buttons in sync.

Now, one could build it like this:

```html

<template>
	<div>
		<!-- Render delete buttons for game objects above canvas -->
		<button 
			v-for="monster in monsters" 
			v-on:click="deleteMonster(monster.id)"
			:key="monster.id"
		>Delete {{monster.id}}</button>
		<!-- Canvas paper.js uses to draw game stuff -->
		<canvas id="forpaper"></canvas>
	</div>
</template>

<script>

import _ from 'lodash';
import paper from 'paper';

function Monster(paper) {
	
	this.id = /* generate random id*/

	this.paperObject = new paper.Circle(/*settings*/);

	this.moveTo = function(x, y) {
		// Delegate to Paper object which will takes care
		// of updating and drawing to the screen.
		this.paperObject.position = {x: x, y: y};
	} 

	//... etc
}

export default {

	data: function() {
		return {
			monsters: []
		}
	},

	mounted: function() {

		// Init Paper to our canvas (not implemented here)

		// Create 4 monsters to start with
		_.times(4, this.createMonster.bind(this));
	},

	methods: {
		createMonster: function() {
			var monster = new Monster(paper);
			// This push will cause button to be inserted to DOM 
			// for the monster.
			this.monsters.push(monster);
		},
		deleteMonster: function(id) {

			// First we remove our wrapping object, which causes 
			// corresponding button to disappear.
			var removedPlayers = _.remove(monsters, function(p) { 
				return p.id === id
			});
			var removedPlayer = removedPlayers[0];
			// ...then actual Paper.js object.
			removedPlayer.paperObject.remove();

		}
	}

}

</script>

```

Vue component above looks nice. All of the monster-related Paper.js stuff is nicely encapsulated inside Monster. We can freely design any API we want for Monster object, and Monster then internally calls Paper.js methods.

There is deep performance issue, however. 

First of all, notice that we are pushing Monster objects to monsters-array that is used to render HTML buttons. This monsters-array is component's data member, giving us all the reactivity magic Vue is so good at. But at what price? 

Consider what happens when we call *createMonster* method:

1. We call new Monster().
2. Monster's constructor builds up PaperJs object and saves it locally to a *property*.
3. Newly-created Monster is pushed to an array.
4. Vue notices this and *binds* get/set listeners to our Monster object's properties.
5. Virtual Dom is recreated and real DOM updated (new button shown on the screen).

Fourth step is problematic, because Vue binds reactivity listeners *recursively*. That is, it traverses Monster object's all normal properties and plunges right in if one of them happens to be Object or Array.

And Monster.paperObject is an Object.

**Thus what happens it that Vue ends up binding ALL the internal properties of Paper.js Circle object!** 

This means that every time *any* internal property of our Circle object changes, Vue's reactivity listener gets called. That might not sound that terrible, but consider this; our *Circle* represents a particular graphical object on the screen which is (by default) updated **60 times a second** via browser's own animation loop.

If the circle is constantly being animated (which it probably is... we are after all building a game), we end up calling Vue's reactivity listener 60 times per second. 

And that is for *one object*, and for its *one property* that is mutated somewhere deep down in the heart of PaperJS code.

Now imagine we have 100 Monster objects. That would cause 6000 totally unnecessary calls per second per property.

That is still vast underestimate. Most likely one update call to a Circle will mutate many of its properties. Position, rotation, size,... etc.

You can see this quickly gets out of hand. A massive slow-down ensues.

>This is something I experienced first-hand. Simply pushing one object *that had internal Paper.js linkage somewhere deep down* to an array Vue controls caused massive performance drop. This was  hard to notice at first, because I was developing with PC happily running FPS 60. That is, each frame still got processed in under 17 ms so there was no visual feedback.
>
>
>
>When I started using the app on mobile device, performance issues became apparent. Doing even the most elementary PaperJS stuff (like drawing a simple rectange over and over again) caused FPS to drop around 30-40.

I call this gotcha **Vue the Kraken**, because its feels like Vue deliberately tries to hunt down my Paper.js object with its long slimy tentacles. No matter how deep you hide your linkage to Paper, Vue will find it and fuck up everything.

> Of course, Vue is just doing its job to make the reactivity system work as expected. There is no way Vue could know which data-bound objects to walk through and which not. But still. Kraken Vue.

So beware. Keep Vue and PaperJs separate. They are still great match for building HTML5 games with nice UIs, but you must introduce some impenetrable layer between them.



