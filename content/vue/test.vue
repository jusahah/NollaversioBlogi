<template>
	<div>
		<!-- Render delete buttons for game objects above canvas -->
		<button 
			v-for="player in players" 
			v-on:click="deletePlayer(player.id)"
			:key="player.id"
		>Delete {{player.id}}</button>
		<!-- Canvas paper.js uses to draw game stuff -->
		<canvas id="forpaper"></canvas>
	</div>
</template>

<script>

import _ from 'lodash';
import paper from 'paper';

function Player(paper) {
	
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
			players: []
		}
	},

	mounted: function() {

		// Init Paper to our canvas (not implemented here)

		// Create 4 players to start with
		_.times(4, this.createPlayer.bind(this));
	},

	methods: {
		createPlayer: function() {
			var player = new Player(paper);

			// This update will cause button to be inserted to DOM for the player.
			this.players.push(player);
		},
		deletePlayer: function(id) {

			// First we remove our wrapping object, which causes button to disappear.
			var removedPlayer = _.remove(players, function(p) { return p.id === id})[0];
			// ...then actual Paper.js object.
			removedPlayer.paperObject.remove();

		}
	}

}

</script>