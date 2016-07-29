+++
date = "2016-07-29T21:33:50+03:00"
draft = false
title = "Rekursiivinen lupausketju ajurina? (osa 1)"

+++

Olen epäilemättä varsin ihastunut lupauksiin ([Promise](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise)). Tässä blogissa on blogin ensimmäisen kuukauden aikana julkaistu neljä kirjoitusta, joiden keskiössä toimii lupausten käyttö. Ja tässä on viides.

Tänään mieltäni askarrutti seuraava lupausten hyödyntämiseen liittyvä ajatus:

**Entä jos rakentaisi lupausten varaan yleismaailmallisen "task-runnerin", johon kytkeä varsinaiset ominaisuudet *service provider*-tyyliin.**

>Service Provider on itselleni Laravellin puolelta tutuksi tullut termi. Se tarkoittaa ohjelmakomponenttia, joka ohjelman suorituksen alkuvaiheessa *lisää* jonkin palvelun osaksi (ohjelma)kokonaisuutta. 

>Jos itse ohjelmisto on F1-auto, Service Provider on varikkomekanikko, joka ruuvaa kiinni sivupeilit (= lisäominaisuus) osaksi auton runkoa. 

Pakko myöntää, etten itsekään ole täysin kärryillä mitä ajan tällä konseptilla takaa. Mutta jotain sen suuntaista, että haluaisin rakentaa lupausten varaan uuden ohjelmistokehyksen. Tuo kehys olisi suunnattu hyvin spesifiin käyttötarkoitukseen; vuoropohjaisten moninpelien ohjelmointiin.

### Voiko rekursiivinen lupausketju toimia ajurina?

Kaikkein yleisimmässä muodossaan lupausketju toimii siten, että ketjun osanen suoritetaan *heti* kun edellisen osanen on saanut oman työnsä päätökseen. Ketju etenee siis yksi osasuoritus kerrallaan järjestyksessä. 

Myös kaikki vuoropohjaiset pelit etenevät järjestyksessä; ensin on pelaajan #1 vuoro, sitten pelaajan #2, sitten pelaajan #3, jne. Kun kierros käyty läpi, vuoro siirtyy takaisin pelaajalle #1.

> Esimerkkejä vuoropohjaisten pelien siirtojärjestyksestä:
>
> Monopoli (3 pelaajaa): p1->p2->p3->p1->p2->p3->p1...
>
> Shakki (kaksinpeli): p1->p2->p1->p2->p1...

Monopoli, pokeri, shakki, snooker, curling, laivanupotus... vuoropohjaisia pelejä on paljon ja todella monenlaisia. Katsotaan esimerkillinen lupauksiin perustuva ajuri, joka suorittaa yhden vuorokierroksen (= kaikki pelaajat tekevät yhden siirron). Käytetään esimerkkinä kolmen pelaajan Monopoli-peliä:

```javascript

var Promise = require('bluebird');

// Vuorosimulaattori.js

// PeliLoppui-exception
function PeliLoppui() {};
PeliLoppui.prototype = new Error;

// Pelin update-metodi, jolla peliä viedään eteenpäin
function toteutaSiirto(pelaajaNimi, siirto) {
	// Tee siirto esim. shakkilaudalla.
}

// Apufunktio nopan heittämiseen, arpoo kaksi lukua 1-6.
function heitaNoppaa() {
	// [nopan silmäluku, toisen nopan silmäluku]
	return [Math.ceil(Math.random()*6), Math.ceil(Math.random()*6)];
}


// Pelaajan #1 siirtovuoro
function p1Siirto() {
	return new Promise(function(resolve, reject) {
		// Heitä noppaa
		var nopat = heitaNoppaa();
		// Tee siirto
		toteutaSiirto('p1', nopat);
		// Päätä vuoro täyttämällä lupaus.
		resolve();
	});
}

// Pelaajan #2 siirtovuoro
function p2Siirto() {
	// Vastaava kuin p1, mutta annetaan vuoro kakkospelaajalle.
}

// Pelaajan #3 siirtovuoro
function p3Siirto() {
	// Vastaava kuin p2, mutta annetaan vuoro kolmospelaajalle.
}

function aloitaVuorokierros(pelaajat) {
	// Kunkin pelaajan siirtofunktio on elementtinä *pelaajat*-listassa.
	// Kutakin funktiota kutsutaan järjestyksessä vuorotellen.
	
	// Promise.each-metodi käy pelaajat yksi kerrallaan läpi, antaen
	// siirtovuoron kullekin pelaajalle kertaalleen.

	Promise.each(pelaajat, function(annaVuoroPelaajalle) {
		// Muuttuja *annaVuoroPelaajalle* on funktio.
		// Se on joko *p1Siirto*, *p2Siirto* tai *p3Siirto*!
		return annaVuoroPelaajalle();
	})
	.then(function() {
		// Siirry seuraavalle kierrokselle!
		// HUOM! Ikuinen rekursio.
		// Ilman virhettä peli ei lopu koskaan.
		aloitaVuorokierros(pelaajat);
	})
	.catch(function() {
		// Pelissä tapahtui virhe, lopeta peli.
		// Peli lopetetaan heittämällä 'PeliLoppui',
		// joka napataan kiinni ylempänä call stäkissä.
		throw new PeliLoppui();
	})

}
// Luo kolme pelaajaa
var pelaajat = [p1Siirto, p2Siirto, p3Siirto];
// Aloita peli, johon nuo kolme pelaajaa osallistuvat.
aloitaVuorokierros(pelaajat)
.catch(PeliLoppui, function() {
	// Tässä on hyvä paikka kerätä roskat yms.
	// Tai esim. tallettaa pelin lopputulokset tietokantaan!
	console.log("Peli on loppunut, kiitos pelaajille.")
})

```

Ylläoleva koodi pyörii ikuista looppia *aloitaVuorokierros*-funktion ympärillä. Tällä tavoin se pystyy simuloimaan esimerkiksi Monopoli-peliä, joka ei pääty koskaan. Huomattavaa on, että koska tuo luuppi pyörii *asynkronoidusti*, on *p1Siirto*-funktion sisällä mahdollista **kysyä ihmispelaajalta hänen siirtoaan.**

Eli ihmispelaajalle voidaan *p1Siirto*-funktion sisältä käsin avata vaikka popup-ikkuna selaimessa, ja tuo popup-ikkuna tarjoaa ihmispelaajalle mahdollisuuden päättää siirrostaan. Kun pelaaja klikkaa popup-ikkunasta haluamaansa siirtoa, tieto välittyy palvelimelle, ja pelaajan siirtovuoro päättyy.

Tässä nopea naivi toteutus edellämainitusta:

```javascript

// Pelaajan #1 TCP-socket tjms. viestintäväylä
// Se miten tämä socket on luotu on tekninen sivuseikka,
// jonka vastuu jätettäköön *socket.io*:n kaltaiselle kirjastolle.
var p1socket = /* luo socket jotenkin */

function p1Siirto() {
	return new Promise(function(resolve, reject) {
		// Lähetä ihmispelaajalle tieto siitä, että
		// nyt on hänen siirtovuoronsa.
		p1socket.send('Sinun siirtovuorosi - tee siirto.');

		// Tärkeää!
		// Jää kuuntelemaan ihmispelaajan vastausta!
		// Ohjaa saatu vastaus suoraan lupauksen täyttävään
		// resolve-funktioon!
		p1socket.on('siirto', resolve);

	}

}

// p2Siirto ja p3Siirto vastaavanlaiset...

```

Erittäin kaunista. Kunkin pelaajan siirtofunktio vie tiedon ihmispelaajalle, ja jää odottamaan ihmispelaajan vastausta. Kun vastaus saapuu, aiemmin luotu lupaus täytetään ja vuorokierros pyörähtää yhden pykälän eteenpäin.

Ylläoleva algoritmi on toki naurettavan naivi siinä mielessä, että se ei ota juuri mitään erikoistilanteita tai sivuehtoja huomioon. Esimerkiksi siirtovuorolla ei ole aikarajaa - eli pillastunut pelaaja voi kieltäytyä tekemästä siirtoa lainkaan ja tällä tavoin koko peli jää jumiin.

Palataan aikarajaan ja muihin ongelmiin seuraavassa postauksessa. Samalla pääsemme näkemään josko *Promise.race*-metodista olisi johonkin...











