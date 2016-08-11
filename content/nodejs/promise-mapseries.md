+++
date = "2016-08-10T03:08:44+03:00"
draft = false
title = "Rekursiivinen lupausketju ajurina? (osa 2)"

+++

(Tämä on jatkoa postaukselle [Rekursiivinen lupausketju ajurina? (osa 1)](http://nollaversio.fi/blog/public/nodejs/promise-workflow-manager))

Rakennan parhaillaan ohjelmakehystä (työ)nimeltään *PromiseMonopoly*. Tuon kehyksen tarkoitus on valmistuessaan mahdollistaa keskuspalvelimen kautta toimivien vuoropohjaisten pelien helpompi toteuttaminen.

Kehys abstraktoi vastuulleen yhteyksien hallinnan ja ns. game-loopin pyörittämisen. Jälkimmäinen vastuualue on keskeinen osa mitä tahansa vuoropohjaista peliä. Ajatellaan vaikka Monopolia; meillä on viisi pelaajaa, jotka kukin tekevät siirtonsa vuorollaan. Siirtovuoro kiertää ympyrää kullakin hetkellä pelissä mukana olevien pelaajien kesken kunnes lopulta jäljellä on vain yksi pelaaja. Tämä viimeinen mohikaani on pelin voittaja.

Vastaava ympyrää kiertävä siirtovuorojärjestys on ominainen käytännössä kaikille vuoropohjaisille peleille. Ainoa mikä vaihtelee on pelaajien määrä. 

Esimerkiksi shakissa siirtovuoro hyppii kahden pelaajan välillä. Shakkipeli päättyy heti kun toinen pelaajista ei enää kykene tekemään siirtoa (eli laudalla on matti tai patti).

Rakennusvaiheessa oleva ohjelmistokehykseni abstraktoi siirtovuorojen hallinnan seuraavalla tavalla:

```javascript

var peliTila = new Peli();

var SIIRTO_MAX_AIKA = 5000; // Siirtoaika max. viisi sekuntia.

function aloitaPeli(pelaajat) {
    // Pyydä kutakin pelaajaa yksitellen tekemään siirtonsa  
	return siirtoKierros(pelaajat)
	// Pelaajat, jotka eivät jatka seuraavalle siirtokierrokselle saivat 
	// palautusarvonaan "null", joten heidät voi filteröidä pois.
	.then(_.compact)
	.then(function(mukanaOlevatPelaajat) {

	  if (mukanaOlevaPelaajat.length <= 1) {
		// Vain yksi tai nolla pelaajaa enää mukana, lopeta peli.
		throw new LopetaPeli();
	  }

	  // Peli jatkuu, aloita uusi siirtoKierros 
	  // Vain yhä mukana olevat pelaajat pääsevät mukaan
	  // uudelle siirtokierrokselle.
	  return siirtoKierros(mukanaOlevatPelaajat);
	})
	.catch(LopetaPeli, function() {
		// Peli on päättynyt
		// Älä rekursoi
		console.log("Peli päättynyt");
	})
}

function siirtoKierros(pelaajat) {
  return Promise.mapSeries(pelaajat, function(pelaaja) {
    if (pelaaja.hasDisconnected()) return null;
    return __pyydaSiirtoa(pelaaja);
  });	
}

function __pyydaSiirtoa(pelaaja) {
  // pelaaja.teeSiirto() lähettää pelaajalle pyynnön tehdä siirto.
  // .timeout() määrittää maksimiajan jonka puitteissa tuo siirto on tehtävä.
  return pelaaja.teeSiirto().timeout(SIIRTO_MAX_AIKA)
  .tap(function(siirto) {
    // Throws "Laitonsiirto"-Error jos kyseessä laiton siirto.
    return tarkistaSiirronLaillisuus(siirto);
  })
  .then(function(siirto) {
    // Jos pääsemme tänne, siirto on ollut laillinen
    // Muokkaamme pelin tämän hetkistä tilaa siirron pohjalta.
    // Pelitila yksinkertaisesti tarkoittaa pelin tämän hetkistä pelitilannetta, esim.
    // shakkipelissä pelitila tarkoittaa laudalla olevaa asemaa.
    var uusiPelitila = toteutaSiirto(siirto);
    // Ilmoitamme uuden tilapäivityksen kaikille pelin osanottajille.
    // (jotta he pysyvät kärryillä pelin etenemisestä).

    viestiPelaajille(pelaajat, {
      aihe: 'uusi_siirto_tehty',
      siirto: siirto,
      pelitila: uusiPelitila
    });
    // Palautamme pelaajan sillä hän jatkaa mukana pelissä.
    return pelaaja;

    // 
  })
  .catch(Laitonsiirto, function() {
    // Pelaaja yritti tehdä laittoman siirron.
    // Palauta vuoro pelaajalle ja pyydä tekemään laillinen siirto.
    // Kutsumme rekursiivisesti tätä funktiota uudestaan.
    return this.__pyydaSiirtoa(pelaaja);

  })
  .catch(TimeoutError, function() {
    // .timeout(aika) metodimme heitti virheen, eli
    // pelaaja ei ehtinyt tekemään siirtoaan ajoissa.

    // Vakioasetuksena pelaaja häviää pelin, eli palautamme arvon null.
    return null;
  })

	
}

function toteutaSiirto(siirto) {
  // Muokkaa pelitilaa siirron pohjalta jotenkin ja palauta muokattu pelitila.
  // Muokkaaminen on pelikohtaista ja kehyksen käyttäjä määrittää muokkausfunktion.

  return peliTila;
}

function viestiPelaajille(pelaajat, viesti) {
  // Kutsu kunkin pelaajan "lahetaViesti"-metodia, joka
  // hoitaa kommunikoinnin pelaajan suuntaan.
  _.map(pelaajat, function(pelaaja) {
    pelaaja.lahetaViesti(viesti);
  });
}

```

Yllä on yksinkertaistettu versio asynkronoidusta game-loopista, joka pyytää vuorotellen pelaajia tekemään siirtojaan kunnes lopulta vain yksi pelaaja on jäljellä.

Koko loopin keskiössä on **Promise.mapSeries**, joka yksi kerrallaan kutsuu *__pyydaSiirtoa*-funktiota kullekin pelaajalle. **Promise.mapSeries**-kutsun palautusarvo sisältää listan pelaajista, jotka jatkavat peliä seuraavalle kierrokselle. 

Tämä lista rakentuu pelaaja pelaajalta sen mukaan, mitä *__pyydaSiirtoa*-funktio palauttaa. Jos *__pyydaSiirtoa* palauttaa *null*, pelaaja ei jatka seuraavalle kierrokselle (= hän on hävinnyt pelin). Jos *__pyydaSiirtoa* palauttaa *Pelaajan*, pelaajan jatkaa seuraavalle kierrokselle.

Ylimmällä tasolla funktio **aloitaPeli** laittaa pyörät pyörimään. Se kutsuu rekursiivisesti aina uutta siirtokierrosta pelattavaksi. Kunkin siirtokierroksen päätteeksi se tarkistaa onko peli päättynyt (= vähemmän kuin kaksi pelaajaa jäljellä). Jos ei ole, se aloittaa uuden siirtokierroksen.

Asynkronoidun game-loopin perusominaisuus on, että kaikki funktiot palauttavat *lupauksen*. Tämä lupaus voidaan sitten ketjuttaa osaksi suurempaa lupausketjua. Poikkeuksena on funktio kuten **viestiPelaajille**, jonka oletetaan suorittavan tehtävänsä välittömästi (viestien lähettäminen kullekin pelaajalle yksinkertaisesti oletetaan onnistuvaksi, myöhemmässä versiossa oletuksesta luovutaan ja käytetään erillistä "disconnect"-handleria reagoimaan yhteysvirheisiin pelaajan ja palvelimen välillä).

Ylläolevasta koodista puuttuu vielä *tärkein* ohjelmistokehykselle ominainen aspekti; mahdollisuus kutsua kehyksen käyttäjän määrittämiä lisäfunktioita. Koska esimerkiksi shakkipelissä tehtävän siirron laillisuuden tarkistaminen on varsin erilainen prosessi kuin pokeripelissä tehtävän siirron laillisuuden tarkistaminen, on kehyksen käyttäjän pystyttävä *pluggaamaan sisään* haluamansa tarkistusfunktio.

Toisin sanoen, tämä kohta koodia:

```javascript

.tap(function(siirto) {
   // Throws "Laitonsiirto"-Error jos kyseessä laiton siirto.
   // Kutsumme staattisesti valittua tarkistusfunktiota.
   return tarkistaSiirronLaillisuus(siirto);
})

```  

menee kutakuinkin muotoon

```javascript

.tap(function(siirto) {
   // Throws "Laitonsiirto"-Error jos kyseessä laiton siirto.
   // Kehyksen käyttäjä tarjoaa meille tarkistusfunktion osana
   // "laajennukset"-objektia, jonka hän on määrittänyt.
   return laajennukset['tarkistaSiirronLaillisuus'](siirto);
})

``` 

Antamalla käyttäjälle vapauden valita *laajennukset*-objektin funktioiden toteutukset, kehyksen käyttäjä kykenee toteuttamaan haluamansa pelilogiikan kehyksen pohjalle. Esimerkiksi timeout-virhetilanteen hallinta:

Vanha muoto:

```javascript

.catch(TimeoutError, function() {
	// .timeout(aika) metodimme heitti virheen, eli
	// pelaaja ei ehtinyt tekemään siirtoaan ajoissa.

	// Vakioasetuksena pelaaja häviää pelin, eli palautamme arvon null.
	return null;
})

```

ja uusi muoto:

```javascript

.catch(TimeoutError, function() {
	// .timeout(aika) metodimme heitti virheen, eli
	// pelaaja ei ehtinyt tekemään siirtoaan ajoissa.

	// Annamme kehyksen käyttäjän tarjoaman funktion 
	// päättää miten reagoidaan
	return laajennukset['aikaKuluiUmpeen']();
})

```

Huomioitavaa on, että kehyksen käyttäjän tarjoama kutsufunktio voi sisältään myös heittää virhetilanteita, jotka sitten kehyksen lupausketju nappaa kiinni. Tällä tavoin kutsufunktio voi esimerkiksi päättää pelin ennenaikaisesti (= ennen kuin vain yksi tai nolla pelaajaa on jäljellä).

```javascript

laajennukset['aikaKuluiUmpeen'] = function() {
	throw new LopetaPeli();
}

```

Jatketaan tästä ensi kerralla.









