+++
date = "2016-08-29T13:39:19+03:00"
draft = false
title = "Lodash: toPairs + sortBy"

+++

Löysin kivan patternin tallentaa objektin attribuuttien keskinäinen järjestys osaksi objektia.

Sanotaan esimerkkinä, että meillä on *asukasluettelo*. Tuo luettelo on objekti, jossa *avaimena* toimii asukkaan nimi, ja *arvona* asukkaan iän kertova objekti:

```javascript

var asukasLuettelo = {
  'Matti' : {ika: 16},
  'Pekka' : {ika: 28},
  'Pirjo' : {ika: 35},
  'Lauri' : {ika: 21},
  // jne.	
}

```

Haluamme muuntaa asukasluettelon muotoon, jossa jokaisen iän yhteydeen on kirjattu *kuinka mones nousevassa ikäjärjestyksessä tuo asukas on*.

Eli haluamme lopputuloksen:

```javascript

var asukasLuettelo = {
  'Matti' : {ika: 16, jarj: 1},
  'Pekka' : {ika: 28, jarj: 3},
  'Pirjo' : {ika: 35, jarj: 4},
  'Lauri' : {ika: 21, jarj: 2},
  // jne.	
}

```

Kuinka tehdä tuo muutos helposti? Yksinkertainen pätkä ketjutettuja Lodash-funktiokutsuja riittää:

```javascript

_.chain(asukasLuettelo)
// Muunna objekti listaksi.
.toPairs()
// Lajittele asukkaat ikäjärjestykseen.
.sortBy(function(asukasL) { return asukasL[1].ika})
// Asukkaat nyt ikäjärjestyksessä.
// Talletetaan kunkin asukkaan kohdalle tieto hänen järj.numerostaan.
.each(function(asukasL, idx) { asukasL[1].jarj = idx+1})
// Pakotetaan Lodash evaluoimaan kutsuketju
.value()

```

Koska teemme muutoksen suoraan asukas-objektiin, meidän ei tarvitse tallentaa funktioketjun paluuarvoa mihinkään. 

Nyt jokaisen asukkaan yhteyteen on tallennettu hänen ikäjärjestysnumeronsa.

> Loppukaneetti: ylläolevan kutsuketjun lopussa kutsumme apufunktiota *value()*. Tämä kutsu on syytä suorittaa vaikka emme tarvitsekaan palautusarvoa mihinkään! Tämä siksi, että Lodash käyttää konseptia nimeltä *lazy evaluation* kun se kohtaa tuollaisen kutsuketjun. 
>
> Laiskana miehenä Lodash ei tee yhtään mitään ennenkuin se näkee value()-kutsun - tuon nähdessään se käy läpi koko kutsuketjun, ajaen tarpeelliset funktiot järjestyksessä loppuun saakka.


