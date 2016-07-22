+++
date = "2016-07-11T06:35:00+03:00"
draft = false
title = "Nodejs - riippuvuuksien injektointi"

+++

Riippuvuuksien injektointi (engl. dependency injection) on varsin vahva tapa varmistaa modulaarinen koodipohja. Kun tietyn komponentin jokainen alikomponentti otetaan vastaan "ulkoa annettuna", on komponenttia mahdollista muokata
rakentamalla se eri palikoista.

Alla esimerkki komponentista, joka hallitsee itse riippuvuuksiaan (alikomponenttejaan):

```javascript

// Termostaatti.js

module.exports = Termostaatti;

function Termostaatti() {
	this.lampomittari = new Lampomittari();
	this.tuuletus = new Tuuletusjarjestelma();

	...

}

function Lampomittari() {...}
function Tuuletusjarjestelma() {...}


```

Ylläoleva Termostaatti-komponentti paitsi itse päättää omat alikomponenttinsa, myös sisältää
alikomponenttien koodin sisuksissaan. Kyseessä on äärimmilleen viety tapa "paketoida" komponentti
loogiseksi kokonaisuudeksi, ikäänkuin mustaksi laatikoksi.

Termostaatin loppukäyttäjän kannalta ratkaisu on peräti toimiva, olettaen, että loppukäyttäjä vain
haluaa termostaatin käyttöönsä annetussa muodossa.

Ongelmana kuitenkin on, että esimerkiksi lämpömittarin koodipohjalla olisi ehkä käyttöä muuallakin, esimerkiksi komponenttia **Leivinuuni** rakennettaessa. Jos lämpömittarin koodi elää **Termostaatin** sisuksissa, se on käytännössä vangittuna ikuiseen tyrmään.

Täten helppo tapa parantaa koodia on refaktoroida **Termostaatti** muotoon, jossa lämpömittari elää omassa kooditiedostossaan, täten helposti siirrettävissä muihin tarkoituksiin.

```javascript

// Termostaatti.js

var Lampomittari = require('Lampomittari');

module.exports = Termostaatti;

function Termostaatti() {
	this.lampomittari = new Lampomittari();
	this.tuuletus = new Tuuletusjarjestelma();
	...
}

function Tuuletusjarjestelma() {...}

```
```javascript


// Lampomittari.js

module.exports = Lampomittari;

function Lampomittari() {...}


```

**Lampomittari-alikomponentti** otetaan ylläolevassa esimerkissä erikseen käyttöön osaksi **Termostaatti-komponenttia**. Lämpömittari ei siis enää elä **Termostaatin** sisällä. Selkeä parannus aiempaan siinä mielessä, että eri komponenttien koodipohjat ovat entistä paremmin jaoteltuina omiin tiedostoihinsa.

Varsinainen otsikon ongelma ei silti ratkennut - Termostaatti itse hallitsee alikomponentin ottamisen käyttöön.

Seuraava parannus on siirtää päätäntävalta pois Termostaatin ulottuvilta. Termostaatin vastuulla ei pidä olla lämpömittarin valinta. Termostaatin vastuulla on huolehtia lämpömittarin mitta-asteikon lukemisesta. Oleellista on, että termostaatti saa käyttöönsä luettavissa olevan lämpömittarin. 

Oletetaan esimerkin nimissä, että meillä on kaksi eri tyyppistä lämpömittaria; digitaalinen mittari ja elohopeamittari.

Termostaattia ei kiinnosta kumpi mittari on sen käytettävissä KUNHAN VAIN molemmat mittarit ovat luettavissa ongelmitta.

Mutta meitä huoneiston omistajina asia saattaa kiinnostaa. Emme halua elohopeamittaria, sillä elohopea on ympäristömyrkky. Olemme viherhihhuleita, ja suosimme digitaalista mittaria (jonka toiminta ei perustu elohopean lämpölaajenemiseen).

Käytännössä meillä on kaksi tapaa toteuttaa koodipohja siten, että termostaatti ei ole edes tietoinen millaisen mittarin se saa käyttöönsä.

### Tapa 1 ("tiedosto-interface")

Helpoin tapa ratkoa ongelma on hoksata, että Lampomittari.js -tiedoston määrittämä komponentti otetaan käyttöön Termostaatti.js-tiedostossa nimellä "Lampomittari". 

Toisin sanoen, mitä ikinä Lampomittari.js-tiedosto määrittääkään, termostaatti näkee sen nimellä "Lampomittari". Kyseessä on puhdas interface, joka pätee tiedostojärjestelmän tasolla. Niin kauan kuin Lampomittari.js-tiedoston nimi ei muutu, voimme *kontrolloida termostaatin sisäistä toimintaa ilman että meidän tarvitsee koskea lainkaan Termostaatti.js-tiedostoon.*

Eli:

```javascript

// Termostaatti.js

var Lampomittari = require('Lampomittari');

module.exports = Termostaatti;

function Termostaatti() {
	this.lampomittari = new Lampomittari();
	this.tuuletus = new Tuuletusjarjestelma();
	...
}

function Tuuletusjarjestelma() {...}

```
```javascript


// Lampomittari.js

module.exports = DigitaalinenLampomittari;

function DigitaalinenLampomittari() {...}

function ElohopeaLampomittari() {...}


```

Ylläoleva koodipohja antaa Lampomittari.js-tiedostolle tilaisuuden kontrolloida termostaatin sisäistä toimintaa. Jos haluamme vaihtaa termostaatin lämmonmittauksen vanhan koulukunnan menetelmiin, riittää yksi muutos:

```javascript
// Muutos Lampomittari.js koodiin
module.exports = ElohopeaLampomittari;
```

Vielä parempaa - voimme käyttää koko Lampomittari.js-tiedostoa yhtenä suurena "dispatchina". Tällöin kaikki eri tyyppiset mittarit elävät omissa tiedostoissaan, ja Lampomittari.js-tiedoston tehtäväksi jää valita niistä yksi ja tarjota sitä ulkopuolisille "Lampomittari"-interfacen nimissä.

```javascript
// Lampomittari.js
var ElohopeaMittari = require('ElohopeaLampomittari');
var DigitalMittari  = require('DigitaalinenLampomittari');
var SaunaMittari    = require('SaunaLampomittari');

module.exports = SaunaMittari;

```

Mutta asiassa on ongelma. Entä jos huoneistoon halutaan *useampi* termostaatti? Entä jos eri termostaatit eivät halua käyttää samaa lämmönmittaustapaa? 

Niin kauan kuin Lampomittari.js-tiedosto toimii interfacena, se pystyy tarjoamaan vain yhden tavan mitata lämpötila. Lisäksi tuo tapa on kirjoitettu suoraan lähdekoodiin. Tarkoittaen, että ohjelman ajon aikana tuo valittu tapa on vakio - sitä ei pysty muuttamaan.

Yksi suht typerä tapa ratkaista ongelma on luoda erillinen Termostaatti-tiedosto jokaista erilaista termostaattia varten:

```javascript
// ElohopeaTermostaatti.js

var Lampomittari = require('ElohopeaLampomittari');

module.exports = Termostaatti;

function Termostaatti() {
	this.lampomittari = new Lampomittari();
	this.tuuletus = new Tuuletusjarjestelma();
	...
}

```
```javascript
// DigitaalinenTermostaatti.js

var Lampomittari = require('DigitaalinenLampomittari');

module.exports = Termostaatti;

function Termostaatti() {
	this.lampomittari = new Lampomittari();
	this.tuuletus = new Tuuletusjarjestelma();
	...
}

```

Ylläolevassa ei ole mitään järkeä. Huomattavaa on, että eri tiedostojen välillä vain yksi koodirivi muuttuu - valitun lämpömittarin nimi.

Parempikin tapa on.

### Tapa 2 ("riippuvuuksien injektointi moduuliin")

Kaiken päämääränä on se, ettei meidän tarvitse koskea Termostaatti.js-tiedostoon silloin, kun haluamme vaihtaa termostaatin lämmönmittaustapaa. Yllä saavutimme tavoitteen require-komennon kautta; otimme käyttöön require-toiminnolla Lampomittari.fi -komponentin - joka ei itse asiassa ollut komponentti lainkaan, vaan ainoastaan *esitti* komponenttia. Oikea komponentti oli piilossa Lampomittari.js-tiedoston selän takana.

Vaihtoehtoinen tapa toteuttaa tavoitteemme on yksinkertaisesti syöttää tarvittavat alikomponentit sisään samalla kun luomme termostaattia.

Huomioitavaa on, että syötämme alikomponentit sisään *ohjelman ajon aikana*. Toisin sanoen, valinta käytetyistä alikomponenteista on tiedossa vasta ohjelman ajon aikana. 

Tämä on fundamentaaline ero aiempiin ratkaisuyrityksiimme. Aiemmissa ratkaisuissa valinta oli aina **kirjattu suoraan lähdekoodiin**. 

```javascript

// Termostaatti.js

module.exports = function(lampomittari, tuuletusjarjestelma) {
	// Onko lampomittari digitaalinen vai elohopea? 
	// Emme tiedä. Emme välitä.
	return new Termostaatti(lampomittari, tuuletusjarjestelma);
}

function Termostaatti(lampomittari, tuuletusjarjestelma) {
	
	...
}


```

Tämä toimintamalli eroaa aiemmista siten, että Termostaatti ottaa vastaan alikomponentit täysin ulkoa annettuina. Termostaatti.js-tiedoston tehtäväksi jää *rakentaa termostaatti kytkemällä ulkoatulevat komponentit osaksi kokonaisuutta*. Tästä ajattelumallista käytetään nimitystä "Factory" eli tehdas. Termostaatin käyttäjä voi vapaasti syöttää haluamansa lämmönmittausmenetelmän sisään termostaattia projektiin lisätessään.

```javascript

// Asuinhuoneisto.js

var termostaattitehdas = require('Termostaatti');

var saunanTermostaatti = termostaattitehdas(new SaunaMittari(), new Tuuletus());
var eteisenTermostaatti = termostaattitehdas(new ElohopeaMittari(), new Tuuletus());

...

```

Luonnollisesti tapojen #1 ja #2 välillä trade-off. Tapa 1 mahdollistaa loppukäyttäjän olevan auvoisen tietämätön mistään termostaatin sisäisistä aspekteista. Loppukäyttäjä vain ottaa termostaatin käyttöönsä, luottaen sen toimintaan. Tapa 2 antaa loppukäyttäjälle mahdollisuuden *määritellä kustomoituja* termostaatteja. Loppukäyttäjä voi itse rakentaa haluamansa termostaatin ikäänkuin LEGO-palikoita kokoamalla. Jokaisen palikan hän voi valita itse.

Ero on vastaava kuin Applen läppärin ja itsekootun pöytätietokoneen välillä. Applen läppäri on käytännössä yksi iso musta laatikko, ja sen sisäisten alikomponenttien muuttaminen vaatii Apple-sertifioidun ammattilaisen apua. 

Itsekoottu pöytäkone taas... riittää, että ruuvaa sivukannen auki, vetää muistikamman irti, asettaa uuden muistikamman tilalle. Noin kahden minuutin juttu.

