+++
date = "2016-07-13T06:48:47+03:00"
draft = false
title = "Lupausten mahti - pätkä koodiani"

+++

Lupaukset (engl. Promise) ovat varsin mahtavia. Siinä missä muuten asynkronoidun funktiokutsun matkapojaksi joutuisi lähettämään callback-funktion, lupaus mahdollistaa koodaustyylin, jossa callback liikkuu *varjoissa* - siis pinnan alla. Lupaus on käytännössä pieni wrapperi, kuin lahjapaketti, joka kaitsee isällisellä otteella callbackia.

Ehkä suurin hyöty lupauksen käytöstä on kuitenkin se, että virhetilanteet tulevat asianmukaisesti hoidetuksi. Lisäksi ne virheet tulevat hoidetuksi oikeassa paikassa - *lupausketjun lopussa*. Harva asia on hirveämpää kuin joutua kirjoittamaan *varsinaista bisnes-koodia* ja *virhetilanteisiin reagoivaa hätäapukoodia* sikin sokin. Lupausten avulla virhekoodi voi elää visuaalisesti kaukana varsinaisesta koodista. Tämä helpottaa koodinlukua.

Otan esimerkin *lupausketjusta*, jossa virheisiin reagoiva koodi on upotettu pahnanpohjimmaiseksi. Esimerkki on suoraan applikaatiostani, joka analysoi PGN-shakkipelitiedoston ja raportoi käyttäjälle takaisin pelaajien tekemät huonot siirrot. Huono siirto tarkoittaa siirtoa, jonka seurauksena vastustajan voittomahdollisuudet paranivat merkittävästi.

```javascript

// Lupausketju

// Lupausketju koostuu kuudesta osavaiheesta, jotka suoritetaan järjestyksessä peräkkäin!
// Näiden jälkeen on virhetilanteet käsittelevä ekstravaihe.

// #1 PGN-datan separointi eli pelien erottelu toisistaan
// #2 Jokaisen pelin (map-apufunktio!) muuntaminen kasaksi peliasemia (FEN-muoto)
// #3 Asemien filteröinti niin, että vain tiettyjen siirtojen asemat ovat mukana
// #4 Asemien analysointi valittua shakkimoottoria käyttäen.
// #5 Analysointitulosten paketointi pelikohtaisesti
// #6 Asemien filteröinti, jätetään vain asemat joissa pelaaja tunaroi
// #7 Virhetilanteiden hallinta

function processPGN(pgnText) {
		return Promise.resolve(pgnText)
		// #1
		.then(function(pgnText) {
			return separateGames(pgnText);
		})
		// #2
		.then(function(separatedGames) {
			return _.map(separatedGames, function(game) {
				var gameID = uuid.v4(); 
				// Every position is associated with game id
				// so we can later know from which game each 
				// position came from (position.fromgame)
				return separateIndividualPositions(game, gameID);
			});
		})
		// #3
		.then(function(allPositions) {
			// Filter out those not in movenumber range
			return _.filter(allPositions, function(position) {
				return position.movenum >= 10 && position.movenum <= 30;
			});
		})
		// #4
		.map(analyzePosition, {concurrency: 4} /*Num of parallel engine instances to use?*/)
		// #5
		.then(function(results) {
			// Pack analyzed positions back into games
			var groupedIntoGames = _.groupBy(results, function(result) { 
				return result.fromgame;
			});
			// Sort positions by movenumber
			return _.mapValues(groupedIntoGames, function(positions) {
				return _.sortBy(positions, function(p) { return p.movenum})
			});
		})
		// #6
		.then(function(groupedIntoGames) {
			return _.mapValues(groupedIntoGames, function(positions) {
				if (!positions || positions.length < 2) return [];
				var currPosition = positions[0];

				return _.compact(_.map(_.tail(positions), function(position) {
					var thisEval = parseFloat(position.evaluation);
					var evalDiff = Math.abs(thisEval - parseFloat(currPosition.evaluation));

					var oldFen = currPosition.fen;
					var oldBest = currPosition.bestmove;
					var oldEval = currPosition.evaluation;

					// Replace old with the current for next loop run
					currPosition = position;
					// Evaluation changed > 75 centipawns -> bad move
					if (evalDiff > 75) {
						// Mistake found
						return {
							fenBefore: oldFen,
							fenAfter: position.fen,
							evalBefore: oldEval,
							evalAfter: position.evaluation,
							movenum: position.movenum,
							evalDiff: evalDiff,
							playedMove: position.move,
							bestMove: oldBest
						};
					}

					return null; // Nulls are filtered out later

					
				}));

				


			});
		})
		// #7
		.catch(function(err) {
			// Something went wrong, lets bail.
			console.log("PGN analysis went wrong");
			Log::error(err);
		})	


}


```	

Ylläoleva koodi voitaisiin helposti vielä muuttaa *todella* helppolukuiseen muotoon
abstraktoimalla varsinainen bisneskoodi.

```javascript

function processPGN(pgnText) {

		return Promise.resolve(pgnText)
		.then(separateGames)
		.then(separatePositionsForEachGame)
		.then(filterOutPositionsNotInMoveRange)
		.map(analyzePosition, {concurrency: 4})
		.then(packResultsIntoGames)
		.then(filterOutPositionsWhereNoMistakeWasMade)
		.catch(handleErrors)	
}

function separateGames(...) {...}
function separatePositionsForEachGame(...) {...}
// jne.


```	

Aika kaunista.