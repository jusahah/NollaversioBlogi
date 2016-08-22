+++
date = "2016-08-12T06:46:20+03:00"
draft = false
title = "Poikkeuksen väärinkäyttö?"

+++

Yksi suht usein tarvittava algoritmi on tietyn arvon etsiminen binaaripuusta. Etsinnän voi suorittaa esimerkiksi näin:

```javascript

function etsiArvoBinaaripuusta(puu, arvo) {
  var loytynyt = false;	

  function etsiAlipuu(juuri, arvo) {
    if (juuri.arvo === arvo) loytynyt = true;
    // Käy läpi vasemman ja oikeanpuoliset alipuut rekursiivisesti
    etsiAlipuu(juuri.vasenHaara, arvo);
    etsiAlipuu(juuri.oikeaHaara, arvo);
  }
  // Aloita rekursio kutsumalla alipuun etsintäfunktiota.
  etsiAlipuu(puu, arvo);

  return loytynyt;

}

// Kutsutaan etsintäfunktiota
var binaaripuu = /* rakenna puu, ei oleellista etsinnän kannalta */
var tulos = etsiArvoBinaaripuusta(binaaripuu, 'hauki');
console.log(tulos); // true tai false

```

Ylläoleva toimii. Mutta etsintä käy koko puun rekursiivisesti läpi *kaikissa tapauksissa*, ml. siinä erikoistapauksessa, että arvo löytyy heti koko puun juuresta.

Arvokas huomio funktion tehokkuuden kannalta onkin huomata, että heti kun arvo on löytynyt, ei jäljellä olevan puun läpikäyminen ole järkevää. Se on vain ajanhukkaa.

Asia on korvattavissa pitämällä huolen, että arvon löytyessä puuetsintää ei jatketa ko. oksan kohdalta alaspäin:

```javascript

function etsiArvoBinaaripuusta(puu, arvo) {
  var loytynyt = false;	

  function etsiAlipuu(juuri, arvo) {
    if (!juuri) return;
    if (juuri.arvo === arvo) loytynyt = true;
    // Käy läpi vasemman ja oikeanpuoliset alipuut rekursiivisesti,
    // mutta vain jos arvoa ei löytynyt!
    else {
      etsiAlipuu(juuri.vasenHaara, arvo);
      etsiAlipuu(juuri.oikeaHaara, arvo);   	
    }

  }
  // Aloita rekursio kutsumalla alipuun etsintäfunktiota.
  etsiAlipuu(puu, arvo);

  return loytynyt;

}

```

Ylläoleva on himpun verran parempi tapa hoitaa etsintä. Mutta edelleenkin etsintä jatkuu tarpeettoman kauan. Asian voi tarkistaa seuraavalla ajatuskokeella; puu jaetaan kahteen haaraan, vasen ja oikea. Kumpikin haara etsitään *erikseen*. **Jos arvo löytyy heti vasemman haaran alkupäästä, ainoastaan vasemman haaran etsintä stoppaa**. Oikean haaran etsintä joutuu yhä käymään läpi koko oikean puolen puun.

Tämä huomio johtaa meidät pieneen ongelmaan. Binaaripuulle on ominaista suorittaa etsintä binaarisesti - eli jakamalla jäljellä oleva puu aina kahteen osaan. Kumpikin osa saa oman "etsintäpartionsa". 

Mutta optimaalisinta olisi jos nuo kaksi etsintäpartiota voisivat kommunikoida keskenään. Näin ei kummassakaan ylläolevassa ratkaisussa ole. Kommunikaatio ei ole mahdollista - vasen partio ja oikea partio rämpivät täysin toisistaan erillään ja itsenäisesti.

Haluamme saavuttaa tilanteen, jossa **heti kun oikean puolen etsintäpartio löytää arvon, se viestittää tiedon vasemman puolen partiolle**.

Kuinka saavuttaa tälläinen kommunikaatio?

### Poikkeus apuun

Käytännössä kaikki yleisimmät ohjelmointikielet tarjoavat konseptin nimeltä *poikkeus* (engl. exception). Poikkeus on tarkoitettu ohjelman ajon aikana tapahtuvien virhetilanteiden hallintaan. Jos esimerkiksi yrität jakaa nollalla, ohjelma heittää poikkeuksen, joka kertoo että metsään mentiin.

Mikään laki ei estä käyttämästä poikkeuksia myös muihin tarkoituksiin kuin ns. aitojen virhetilanteiden käsittelyyn. 

Voimme luoda *keinotekoisen virhetilanteen*, joka heittää poikkeuksen. Tuollainen keinotekoinen "virhe" voi olla esimerkiksi halutun arvon löytyminen binaaripuusta:

```javascript

function etsiArvoBinaaripuusta(puu, arvo) {
  var loytynyt = false;	

  function etsiAlipuu(juuri, arvo) {
    if (!juuri) return;
    if (juuri.arvo === arvo) throw new Error("Löytyi!");
    // Käy läpi vasemman ja oikeanpuoliset alipuut rekursiivisesti,
    etsiAlipuu(juuri.vasenHaara, arvo);
    etsiAlipuu(juuri.oikeaHaara, arvo);   	
    
  }
  // Aloita rekursio kutsumalla alipuun etsintäfunktiota.
  try {
    etsiAlipuu(puu, arvo);
  } catch (e) {
    loytynyt = true;
  }

  return loytynyt;

}

```

Ylläoleva koodi toimii halutusti. Mutta mikä parasta, ylläolevassa koodissa *kaikki* etsintäpartiot heittävät hanskat tiskiin heti kun arvo on löytynyt. Miksi näin? Koska heittämällä poikkeuksen - heti kun arvo löytyy - koodinajo *rullaa* itsensä suoraan **catch-komentoon**.

Toisin sanoen, heti kun arvo löytyy, hyödynnämme Javascriptin sisäänrakennettua poikkeusten hallintaa ja luomme keinotekoisen virhetilanteen. Tuo virhetilanne *abortoi* kaiken käynnissä olevan etsinnän ja siirtää koodinajon catch-komennon riville:

```javascript
// Heti kun arvo on löytynyt, koodi pomppaa tänne
catch (e) {
  loytynyt = true;
}
```

Catch-komennon sisällä yksinkertaisesti merkkaamme arvon löydetyksi. Tämän jälkeen koodinajo jatkaa catch-komentoa seuraavalta riviltä.

Ylläolevaa koodia voi vielä hiukan parantaa. Ei ole mikään pakko heittää *geneeristä* poikkeusta, vaan luokaamme suosiolla sopivasti nimetty *spesiaalipoikkeus*:

```javascript

// Spesiaalipoikkeuksen määritys
// Huom! Spesiaalipoikkeuksen täytyy ekstentoida Error-objektia.
function ArvoLoytyi() {};
ArvoLoytyi.prototype = new Error();

function etsiArvoBinaaripuusta(puu, arvo) {
  var loytynyt = false;	

  function etsiAlipuu(juuri, arvo) {
    if (!juuri) return;
    if (juuri.arvo === arvo) throw new ArvoLoytyi("Löytyi!");
    // Käy läpi vasemman ja oikeanpuoliset alipuut rekursiivisesti,
    etsiAlipuu(juuri.vasenHaara, arvo);
    etsiAlipuu(juuri.oikeaHaara, arvo);   	
    
  }
  // Aloita rekursio kutsumalla alipuun etsintäfunktiota.
  try {
    etsiAlipuu(puu, arvo);
  } catch (err) {
    if (err instanceof ArvoLoytyi) {
    	// Arvo on löytynyt
    	loytynyt = true;
    } else {
    	// Jotain muuta meni pieleen, arvo ei löytynyt.
    	// Heitä poikkeus uudelleen, joku muu huolehtikoot...
    	throw err;
    }
    
  }

  return loytynyt;

}

```


> Loppukaneetti: monet tahot suhtautuvat *erittäin* epäilevästi poikkeusten väärinkäyttöön ylläolevan esimerkin tavoin. Epäilevässä suhtautumisessa on perusteensa - poikkeukset on luotu ohjelman ajon aikana tapahtuvien virheiden käsittelyyn, ja valtaosa ohjelmoijista lähtee tästä oletuksesta liikkeelle. Mikäli poikkeusta käyttää muuhun tarkoitukseen, on asia syytä selkeästi ilmaista lähdekoodin kommenteissa - tällä tavalla (ehkä, kenties) vältytään väärinkäsityksiltä.





