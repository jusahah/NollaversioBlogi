+++
date = "2016-08-23T06:53:16+03:00"
draft = false
title = "Bluebird: Catch + Translate"

+++

Lupausketjuihin perustuvissa arkkitehtuureissa virhetilanteiden hallinta on helppoa. Useimmiten riittää, että asettaa sopivaan kohtaan lupausketjua *catch*-handlerin. Tuo handleri nappaa kiinni ketjun aiempien suoritusvaiheiden tuottamat virheet.

Bluebird tekee catch-handlerin käytöstä vieläkin kätevämpää tarjoamalla ikäänkuin automaattisen *virheiden ohjauksen* juuri oikeaan handleriin. Esim. seuraavasti:

```javascript

var jaateloKioski = /* luo */
var asiakas = /* luo */;

Promise.try(function() {
  return asiakas.valitseMaku();
})
.then(function(maku) {
  // Saattaa heittää virheen 'JaateloMakuLoppunut'
  return jaateloKioski.rakennaAnnos(maku)
})
.tap(function() {
  // Pyydä maksu
  // Saattaa heittää virheen 'EiRahaa'
  jaateloKioski.pyydaMaksu(asiakas);
})
.then(function(annos) {
	return asiakas.vastaanotaJaatelo(annos);
})
// Käsitellään virheet, kukin virhe yksitellen.
.catch(JaateloMakuLoppunut, function() {/* ...*/})
.catch(EiRahaa, function() {/* ...*/})

```

Ylläolevassa koodissa on mahdollista syntyä kaksi eri virhetyyppiä. Joko jäätelömaku on kiskalta toistaiseksi loppunut, tai asiakas havaitsee yllättäen, että hän on persaukinen.

Nämä kaksi eri virhettä käsitellään erikseen omissaan catch-handlereissa.

Mutta aina tilanne ei ole yhtä valoisa. Joskus tulee vastaan skenaario, jossa *kaksi eri loogista virhetyyppiä käyttävät saman tyypin virheobjektia.*

Esimerkki:

```javascript

// Yksittäisen siirron maksimiaika
var maksimiSiirtoaika = 2000; // 2 sek
// Koko peliin (=pelaajaan kaikkiin siirtoihin) varattu maksimiaika
var peliAikaaJaljella = 180000; // 3 min

Promise.try(function() {
  return pelaaja.teeSiirtosi()
  .timeout(maksimiSiirtoaika)
  .timeout(peliAikaaJaljella)
})
.then(/* käsittele siirto ja vähennä peliaikaa */)
.catch(Promise.TimeoutError, function() {
  // Pelaaja ei tehnyt siirtoaan ajoissa.
  // Mutta kumpi timeout laukesi?
	
})

```

Ylläoleva esimerkki on melko suoraan koodistani. Osana peliserveriäni lupausketjun tulee tietää onko pelaaja ylittänyt *siirtokohtaisen aikansa* vai *pelikohtaisen aikansa*.

Ongelmana on, että molemmat ylityksen heittävät identtisen virheobjektin. Itse asiassa Bluebird-kirjasto tekee tuon heiton, joten sitä ei ole helppo edes kontrolloida.


### Ratkaisu: Muunna geneerinen virhetyyppi domain-spesifiksi virhetyypiksi

Mutta voimme aina napata toisen heiton ja muuntaa (**translate**) sen toiseksi virhetyypiksi. Riittää, että asetamme ylimääräisen catch-handlerin sopivaan kohtaan.

```javascript

// Yksittäisen siirron maksimiaika
var maksimiSiirtoaika = 2000; // 2 sek
// Koko peliin (=pelaajaan kaikkiin siirtoihin) varattu maksimiaika
var peliAikaaJaljella = 180000; // 3 min

Promise.try(function() {
  return pelaaja.teeSiirtosi()
  .timeout(maksimiSiirtoaika)
  .catch(Promise.TimeoutError, function() {
    throw new MaksimiSiirtoAikaYlitetty();
  })
  .timeout(peliAikaaJaljella)
})
.then(/* käsittele siirto ja vähennä peliaikaa */)
.catch(Promise.TimeoutError, function() {
  // Pelaajan kokonaispeliaika umpeutui!	
})
.catch(MaksimiSiirtoAikaYlitetty, function() {
  // Pelaajan siirtokohtainen aika umpeutui!	
})

```

Yltä huomaamme, että nappaamme ensimmäisen mahdollisen TimeoutErrorin kiinni *juuri sopivasti* ennen toista kutsua, joka tuottaa myös TimeoutErrorin. Nappaamalla ensimmäisen virheen kiinni ja muuntamalla sen toiseen muotoon - eli toiseen virhetyyppiin - meidän ei tarvitse myöhemmin vaivata päätämme sen suhteen, mistä virhe lähti alunperin liikkeelle!

Tämä on siis **catch + translate** -patterni. Virhe napataan ja muunnetaan eri muotoon, ja muunnoksen jälkeen palautetaan takaisin "putkeen".

Bluebird tarjoaa peräti juuri tätä catch+translate -tarkoitusta varten erillisen apumetodin: **catchThrow()**. Ylläoleva koodi menee muotoon:

```javascript

// Yksittäisen siirron maksimiaika
var maksimiSiirtoaika = 2000; // 2 sek
// Koko peliin (=pelaajaan kaikkiin siirtoihin) varattu maksimiaika
var peliAikaaJaljella = 180000; // 3 min

Promise.try(function() {
  return pelaaja.teeSiirtosi()
  .timeout(maksimiSiirtoaika)
  .catchThrow(Promise.TimeoutError, new MaksimiSiirtoAikaYlitetty())
  .timeout(peliAikaaJaljella)
})
.then(/* käsittele siirto ja vähennä peliaikaa */)
.catch(Promise.TimeoutError, function() {
  // Pelaajan kokonaispeliaika umpeutui!	
})
.catch(MaksimiSiirtoAikaYlitetty, function() {
  // Pelaajan siirtokohtainen aika umpeutui!	
})

```

> Loppukaneetti: Ihannearkkitehtuuri myös siirtokohtaisen ajan ylitys muunnettaisiin domain-spesifiin virhetyyppiin. Tällöin emme lupausketjun lopussa nappaisi kiinni geneeristä TimeoutErroria lainkaan, vaan esim. KokonaisPeliAikaYlitetty-virheen.