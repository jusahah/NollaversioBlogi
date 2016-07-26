+++
date = "2016-07-25T17:28:05+03:00"
draft = false
title = "Filteröi epäonnistujat pois (reflect + filter)"

+++

Lupauskirjasto *Bluebirdin* yksi vähemmän tunnetuista apumetodeista on **reflect()**. Ainakin allekirjoittaneelle tuo apufunktio pysyi tuntemattomana hyvää matkaa toista vuotta - ei vain tullut pakottavaa tarvetta, ja ongelmat sai aina ratkottua muutenkin.

Näin jälkikäteen ajateltuna nuo "muut" ratkaisut olivat aika hirveitä sekasotkuja, jotka toimivat jos jaksoivat. 

Sittemmin otin reflectin käyttöön.

### Minkä ongelman Promise.reflect() ratkoo?

Varsin usein omissa applikaatioissani on toiminnallisuuksia, joiden onnistunut läpivienti *ei ole kriittistä*. Jotkut toiminnot ovat luonteeltaan sellaisia, että ei niin väliä mikäli toiminto epäonnistuu nolosti. **Tärkeintä on, että yhden vähäpätöisen toiminnon epäonnistuminen ei vedä mukanaan koko applikaatiota vessan pöntöstä alas.**

Puhtaan perinteisessä synkronoidussa koodissa on luonnollista, että epäonnistunut toiminto napataan kiinni *try-catch* -siepparilla. Esim.

```javascript

var _       = require('lodash');
var Promise = require('bluebird');

// SyncVarkaus.js

function kopioiBlogi(urls) {

	// Käydään yksitellen pihistämässä blogien HTML-sisältö.
	var blogiSisallot = _.map(urls, function(blogiURL) {
		var sisalto; // Täytetään sisällöllä
		try {
			sisalto = syncRequest(blogiURL);
		} catch (e) {
			// No, pöllintä ei onnistunut. Eipä hätiä.
			// Rikolliset aikeemme kohdistuvat seuraavaan uhriin.
			console.warn('Pölliminen epäonnistui - seuraava uhri sisään.');
		}	

		return sisalto;	

	})
	
	// blogiSisallot sisältää pöllityt sisällöt niistä blogeista,
	// joiden kähvellys EI epäonnistunut nolosti. Luuserit roikkuvat
	// vielä mukana undefined-arvoina. Mutta eivät pitkään.

	// Compact() suodattaa töpeksijät roskakoriin.
	return _.compact(blogiSisallot);
}

// Rikollisen uramme alkupiste.
var tulokset = kopioiBlogi([
	'http://www.pollitasta.fi',
	'http://www.tuplaamo.fi',
	'http://www.nollaversio.fi/public/blog'
]);

// Bestseller tiedossa.
koostaKirja(tulokset);

```

Ylläoleva on mukavaa perinteistä sync-koodia, jossa jokainen toiminto suoritetaan peräkanaa yksitellen. Ja try-catch-sieppari toimii kuin unelma.

Mutta kun Javascriptin (ja maalaisjärjen) luonteeseen kuuluu, että jokaista varkautta ei tarvitse tehdä perätysten. Niitä kun voi tehdä myös *samanaikaisesti*.

Siirrytään ihanaan lupausten maailmaan, ja suoritetaan rikossarjamme asynkronoidusti.

```javascript

// AsyncVarkaus.js

function kopioiBlogi(urls) {

	return Promise.resolve(urls)
	.map(function(blogiURL) {
		// Jokainen request lähtee liikkeelle yht'aikaa.
		return asyncRequest(blogiURL);
	})
	.catch(function(err) {
		// Jotain meni nolosti pieleen.
	})

}

kopioiBlogi([
	'http://www.pollitasta.fi',
	'http://www.tuplaamo.fi',
	'http://www.nollaversio.fi/public/blog'
]).then(function(tulokset) {
	return koostaKirja(tulokset);
}).then(function(kirja) {
	// Valitaan sopivan korruptoitunut kustantaja.
	talentumMedia.julkaise(kirja);
})

```

Kaikki näyttää pintapuolin hyvältä. Käyttämällä *Promise.map*-metodia ammumme kaikki pöllimisyritykset käyntiin *samanaikaisesti*. Kukin kähvellys joko onnistuu tai epäonnistuu. Epäonnistuminen tippuu kivasti *.catch()*-sieppariin, joka sitten tekee jotain.

Mutta asiassa on ongelma. **Jos yksikin yritys epäonnistuu, kaikki epäonnistuvat.**

Tämä on *.map*-metodin ominaisuus - map-lupaus julistaa itsensä onnistujaksi **vain** jos jokainen sen alaisuudessa hyörivistä lupauksista onnistuu.

Jos yksikin alainen töpeksii, map-lupaus vetää pultit ja rikkoo kaiken. Siis estää ketään muutakaan onnistumasta. 

Kerrataan vielä tämä tärkeä pointti uusiksi - **map-lupaus epäonnistuu jos yhdenkin blogin pölliminen epäonnistuu!**. Ja jos yksikin rosvous menee päin honkia, kaikki onnistuneet pöllimiset päätyvät jäteastiaan. Yksi kaikkien, ja kaikki yhden puolesta. Tämä ei tietenkään ole mitä haluamme.

Me haluamme, että jos yksi ryöstö menee reisille, muut ryöstöt voivat yhä onnistua. 

Sata kultakelloa on parempi kuin 99, mutta 99 kultakelloa on parempi kuin pyöreä nolla.

Joten miten korjata asia?

### Reflect()

Ratkaisu on sopivaan paikkaan sijoitettu reflect()-apumetodi. 

Miksi reflect() toimii? Koska reflect() nappaa kiinni sekä *onnistumiset* että *epäonnistumiset*, ja välittää tiedon eteenpäin ns. neutraalissa muodossa. 

Ikäänkuin luuseri ja maailmanmestari kävelisivät tasa-arvoisina rinta rinnan. Reflect() on koodimaailman emakko - kaikki kelpaa ruuaksi. Ja toisesta päästä tuleva tavara on aina vakioitua.

Katsotaanpa:

```javascript

// AsyncVarkaus.js

function kopioiBlogi(urls) {

	return Promise.resolve(urls)
	.map(function(blogiURL) {
		// Yksittäinen varkaus - kokeillaan onnistuuko?
		return asyncRequest(blogiURL).reflect();
	})
	.filter(function(varkausLupaus) {
		// Suodatetaan(!) pois epäonnistuneet ryöstöt!
		return varkausLupaus.isFulfilled();
	})
	.map(function(onnistunutVarkaus) {
		// Vain onnistuneet rosvoukset jäljellä.
		// Joudumme kutsumaan teknisen apufunktion joka
		// hakee lopullisen tuloksen lupauksen syövereistä.
		return onnistunutVarkaus.value();
	})

	// Huomion arvoista, että emme tarvitse -catch-siepparia lainkaan!
	// Miksi? Koska kaikki luuserit on jo siivilöity ylempänä.
	// Yksikään päivänpilaaja ei elä tänne saakka.

}

kopioiBlogi([
	'http://www.pollitasta.fi',
	'http://www.tuplaamo.fi',
	'http://www.nollaversio.fi/public/blog'
]).tap(function(tulokset) {

	// Tulokset sisältää listan niistä blogisisällöistä, 
	// joiden ryöstö meni nappiin. 

	// Kokoa sisällöistä ikioma kirja.
	var valmisKirja = _.chain(tulokset)
	.reduce(function(kirja, pollittyBlogi) {
		kirja.lisaaUusiLuku(pollittyBlogi);
	}, new Kirja('Pölli Tästä Reloaded'))
	.value();

	talentumMedia.julkaise(valmisKirja);
})

```

Ylläolevassa koodissa huomattavaa ovat seuraavat rivit:

```.filter(function(varkausLupaus) {...}```

Apumetodi *.filter* (nimensä mukaisesti) suodattaa luuserit pois.

```return asyncRequest(blogiURL).reflect();```

Teemme requestin, ja kutsumme heti apumetodia *reflect()*. Kutsumalla reflectiä saamme luotua
(ja palautetta ympäröivästä funktiosta ulos) uudentyyppisen lupauksen, joka *itse osaa napata
omat virheensä kiinni.* 

Jännittävää. Olemme ikäänkuin koulineet parannetun Pokemonin, joka on sisäsiisti.

> Loppukaneetti: kun tietty operaatio on *valinnainen*, toisin sanoen sen onnistuminen ei ole kriittisen tärkeää, on syytä muistaa *reflect() + filter()* -kikka.

Ps. Huomasitko muuten, että käytimme yllä metodia **tap()** perinteisen **then**-metodin sijaan:

```.tap(function(tulokset) {...}```

Tämä siksi, että julkaisun suorittava anonyymi funktiomme on päätepiste. Se ei palauta mitään takaisin liukuhihnalle. [Lisää tap vs. then eroista aiemmasta blogikirjoituksessani.](http://nollaversio.fi/blog/public/nodejs/promise-tap/)





