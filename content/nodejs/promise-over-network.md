+++
date = "2017-11-30T07:30:17+02:00"
draft = false
title = "Promise yli netin"

+++

Promise on hieno keksintö. Se mahdollistaa asynkronoidun operaation odottamisen yli yksittäisen Javascript-tapahtumaloopin pyörähdyksen (tick), ja tekee mm. virhetilanteiden hallinnasta helppoa. 

Useimmissa tilanteissa Promise hoitaa kaiken koordinoinnin automaattisesti ohjelmoijan puolesta; ohjelmoijalle riittää kirjoittaa Promise-kutsu ja haluttu koodi, joka ajetaan Promisen täytyttyä.

```javascript

import Promise from 'bluebird'

Promise.resolve("Kutsuttava async-operaatio")
.then(function() {
	console.log("Ajettava koodi")	
})

```

Mutta jotta ylläoleva toimisi ja tarjoaisi helppokäyttöisen API:n ohjelmoijalle, täytyy pinnan alla tapahtua aika paljon. Promise-objektin täytyy sisällään koordinoita sille annettujen callback-funktioiden kutsumista. 

Entä jos Promisen suorittama asynkronoitu operaatio suoritetaan internet-yhteyden yli, siis osana operaatiota otetaan yhteys johonkin toiseen tietokoneeseen. Esimerkkinä seuraavan internet-moninpeli-applikaation koodinpätkä:

```javascript

// Applikaatio kuvaa kaksinpeliä, jossa pelaajat
// tekevät vuorotellen siirtoja.

// Pelin business-logiikka.	
var game = new Game();

var loopMoves = function(player1, player2) {
	var askForMove = function(player) {
		// Palauttaa Promisen, joka odottaa pelaajan tekevän siirron.
		return player.makeMove()
		.then(function(move) {
			// Tee siirto ja vahvista sen laillisuus
			var legal = game.applyMove(move);

			if (!legal || game.gameOver()) {
				throw new GameOver();
			}
		});
	}
	// Pelaajan 1 siirtovuoro
	return askForMove(player1)
	// Pelaajan 2 siirtovuoro
	.then(askForMove.bind(null, player2))
	// Jos peli ei päättynyt, looppaa takaisin
	// jotta pelaajat voivat tehdä uudet siirrot.
	.then(loopMoves.bind(null, player1, player2);
}

// Player1 ja player2 tulevat ulkoa.
loopMoves(player1, player2)
.catch(GameOver, function(gameOver) {
	console.log("Game over");
})

```

Ylläolevan kaltainen koodi tekee game-loopin kirjoittamisesta helppoa online-multiplayer-pelille. Kaiken ytimessä on kutsu *player.makeMove()*, joka palauttaa Promisen, joka puolestaan täyttyy pelaajan antamalla siirrolla.

Mutta miltä tuo *makeMove*-funktio näyttää? Ongelmana on, että makeMove-funktion tulee ottaa yhteys yli internetin siihen pelaajaan, jonka siirtovuoro on kyseessä. Tyypillisessä arkkitehtuurissa tuo yhteys on TCP-yhteyden välityksellä, web-applikaatioissa lähes poikkeuksetta WebSocket-protokollan avulla. 

WebSocketin käyttö osana siirtovuoro-Promisea vaatii jonkin verran koordinointia. Tarvitsemme tavan yhdistää pelaajalle lähetetty pyyntö ("tee siirto") myöhempään sisääntulevaan vastaukseen ("tässä siirtoni"). Ongelmana on, että pelaaja voi saada näiden kahden ajanhetken välillä useita eri viestejä palvelimelta, ja kaikki viestit välitetään samalla WebSocket-yhteydellä. 

Tästä syystä meidän täytyy jotenkin tallentaa palvelimen päässä tieto lähetetystä siirtovuoro-pyynnöstä, ja myöhemmin osata yhdistää sisääntullut vastaus aiempaan pyyntöön, jotta voimme täyttää siirtovuoro-Promisen (joka makeMove-metodista palautetaan):

```javascript

function Player(webSocket) {
	// Esim. socket.ion tuottama socket-objekti.
	this.webSocket = webSocket;

	this.init = function() {
		// Ohjaa socketista tulevat siirtoviestit omaan receive-metodiimme.
		this.webSocket.on('answerToMakeMove', this.receiveMoveFromClient.bind(this));
	}

	// Tämä objekti pitää kirjaa pelaajan suuntaan lähetetyistä pyynnöistä,
	// joihin pelaaja ei ole vielä antanut vastausta.	
	this.pendingMoveRequests = {};	

	this.makeMove = function() {
		var moveRequestId = generateUUID(); 

		return new Promise(function(resolve, reject) {
			// Talleta resolve-callback, jotta voimme myöhemmin
			// löytää sen ja palauttaa pelaajalta saadun vastauksen
			// alkuperäiselle kutsujalle.
			this.pendingMoveRequests[moveRequestId] = resolve;

			// Lähetä tieto pelaajalle 
			this.webSocket.emit('makeMove', {
				answerId: moveRequestId
			});
		}.bind(this))
	}

	this.receiveMoveFromClient = function(moveMsg) {
		var answerTo = moveMsg.moveRequestId;
		var move = moveMsg.move;

		// Etsi resolver hyödyntäen clientin mukana kuljettamaa moveRequestId-arvoa.
		if (this.pendingMoveRequests[answerTo]) {
			var resolver = this.pendingMoveRequests[answerTo];
			delete this.pendingMoveRequests[answerTo];

			// Tämä täyttää Promisen, joka aikaa sitten palautettiin makeMove-metodista.
			resolver(move);
		}
	}
}

```

Ylläoleva vaatii clientin puolella sen, että client käyttää saamaansa moveRequestId-tunnistetta antaessaan vastauksen takaisin palvelimen suuntaan. Jos client tämän muistaa tehdä, voimme palvelimen puolella helposti matchata lähetetyn siirtopyynnön ja sisääntulleen siirtovastauksen toisiinsa.

Itse ylimmällä tasolla voimme laittaa pelin käyntiin esim. seuraavasti:

```javascript

var p1;
var p2;
var game = new Game();

// Socket.io odottaa sisääntulevia yhteyksiä
socketio.on('connect', function(socket) {
	// Aseta disconnect-handler.
	socket.on('disconnect', function() {
		// Client on sulkenut yhteyden
		if (game.running()) {
			game.end();
		}
	});

	if (!p1) {
		// Ensimmäinen pelaaja
		p1 = new Player(socket);
		return;
	}

	// Toinen pelaaja
	p2 = new Player(socket);

	game.startGame();
  p1.init();
  p2.init();

	// Molemmat pelaajat paikalla, aloita siirtojen looppaus.
	loopMoves(p1, p2)
	.catch(GameOver, function() {
		// Peli päättynyt, disconnectoi pelaajat
		p1.webSocket.disconnect();
		p2.webSocket.disconnect();
	});
});


```


