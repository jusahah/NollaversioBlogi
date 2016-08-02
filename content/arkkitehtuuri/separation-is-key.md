+++
date = "2016-08-01T13:50:30+03:00"
draft = false
title = "Neljä lähestymistapaa toiminnallisuuksien abstraktointiin"

+++

Aloitetaan heti esimerkillä. Tehtävämme on luoda pieni skripti, joka käy hakemassa dataa (listan numeroita) palvelimelta, käsittelee tuon datan ja näyttää käsittelyn tulokset ihmiskäyttäjälle.

Verrataan neljää eri ratkaisua, jotka kaikki toteuttavat em. vaatimuksen, mutta käyttävät erilaisia lähestymistapoja mitä tulee koodin strukturointiin. 

*(Käytän kielenä Javascriptiä, mutta artikkelin aihe ei ole Javascriptiin sidottu)*

Ratkaisu #1:

```javascript

$.ajax({
  url: 'serveri.php'
}).done(function(data) {
  // Käsittele data
  var summa = _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);

  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
});

```

Ratkaisu #2:

```javascript

var vastaanotaData = function(data) {
  // Käsittele data
  var summa = _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);

  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(vastaanotaData);

```

Ratkaisu #3:

```javascript

var kasitteleData = function(data) {
  // Käsittele data
  return _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);
}

var naytaSumma = function(summa) {
  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(function(data) {
  var summa = kasitteleData(data);
  naytaSumma(summa);
});

```

Ratkaisu #4:

```javascript

var vastaanotaData = function(data) {
  var summa = kasitteleData(data);
  naytaSumma(summa);
}

var kasitteleData = function(data) {
  // Käsittele data
  return _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);
}

var naytaSumma = function(summa) {
  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(vastaanotaData);

```

#### Neljä eri ratkaisua samaan ongelmaan - mikä on paras?

Lähdetään analysoimaan eri ratkaisuja. 

### Ratkaisu #1

```javascript

$.ajax({
  url: 'serveri.php'
}).done(function(data) {
  // Käsittele data
  var summa = _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);

  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
});

```

Ensimmäisessä ratkaisussa kaikki koodi elää kivasti sisäkkäin *ajax*-kutsun sisällä. Datan vastaanotto, käsittely ja näyttäminen käyttäjälle ovat eroteltuna **riveittäin**, eivät **funktioittain**. Ratkaisu #1 edustaa alhaisinta abstraktion tasoa - eri toiminnallisuudet on kytketty suoraan toistensa perään ilman mahdollisuutta erotella niitä toisistaan.

Mutta miksi kukaan haluaisi erotella niitä toisistaan? Tämä kysymys on erittäin keskeisessä roolissa kaikessa ohjelmoinnissa. Yleensä *eri loogiset toiminnot* halutaan erotella toisistaan siksi, että yksittäisiä toimintoja voi uudelleenkäyttää muualla. Esimerkiksi datan käsittely on hyödyllinen konsepti ihan itsessään - se on siis hyödyllinen ilman, että käsiteltävä data tulee palvelimelta ja että käsitelty data näytetään käyttäjälle! 

Tällä tavoin on loogista, että datan käsittelyä **ei ole** liitetty betonivalulla yhteen niiden toimintojen kanssa, jotka vastaavat vuoropuhelusta palvelimen ja näyttöruudun kanssa.

Ykkösratkaisuissa tämä yhteenliittymä on juurikin betoniin valettu. Eri toimintoja sisältävät koodirivit seuraavat toisiaan kiltisti peräkanaa.

Ratkaisun hyvä puoli on vähäinen koodimäärä. Merkittävin huono puoli on, että *datan käsittely* ja *datan näyttäminen käyttäjälle* on sidottuna teräslangalla yhteen - et voi käsitellä dataa ilman, että myös näyttäisit sen käyttäjälle. 

### Ratkaisu #2

```javascript

var vastaanotaData = function(data) {
  // Käsittele data
  var summa = _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);

  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(vastaanotaData);

```

Kakkosratkaisussa koodi selkenee hiukan. Erottelemme datan vastaanoton ja käsittelyn + näytön erilleen. Ajax-kutsulle tarjotaan palanpainikkeeksi *vastaanotaData*-niminen funktio, joka sisältää "käsittelykoodin" ja "näyttökoodin".

Ratkaisu on hyvä alku - olemme saaneet alkuperäisestä pyhästä kolminaisuudesta (vastaanotto, käsittely, näyttö) yhden osan lohkaistua irralleen. Siirrytään eteenpäin.

### Ratkaisu #3

```javascript

var kasitteleData = function(data) {
  // Käsittele data
  return _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);
}

var naytaSumma = function(summa) {
	// Näytetään summa käyttäjälle
	alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(function(data) {
	var summa = kasitteleData(data);
	naytaSumma(summa);
});

```

Kolmosratkaisussa kaksi jäljellejäänyttä yhteenliitettyä toimenpidettä (käsittely ja näyttö) erotetaan omiksi funktioikseen. Tällä tavoin kaikki kolme toisiinsa kahlittua toimintoa on saatu eroteltua erilleen.

Tämä erilleen erottelu on ohjelmoinnin keskiössä oleva konsepti. Kun erottelu tehdään funktioita käyttäen, sitä kutsutaan nimellä *funktionaalinen abstraktio*. Käytännössä se vain tarkoittaa, että tietty pala koodia *irrotetaan erilleen* ja paketoidaan pakettiin nimeltä 'funktio'. Tuota funktiota voi käyttää eri puolilta applikaatiota uudestaan ja uudestaan.

Funktionaalinen abstraktio toimii siis näin:

```javascript
// Ei abstraktoitu

var a = 2;
// Tehdään potenssiin korotus
var potenssiluku = 3;
var potenssilaskunTulos = 1;
for (var i = 0, i < 3; i++) {
  potenssilaskunTulos = potenssilaskunTulos * a;
}
// Potenssiin korotus valmis
console.log(potenssilaskunTulos); // 8

```

```javascript
// Funktionaalinen abstraktio suoritettu!

// Luodaan funktio
function nostaPotenssiin(luku, potenssi) {
  var potenssilaskunTulos = 1;
  for (var i = 0, i < potenssi; i++) {
    potenssilaskunTulos = potenssilaskunTulos * luku;
  }
  return potenssilaskunTulos;
}

// Tehdään potenssiin korotus
var tulos = nostaPotenssiin(2, 3);
// Potenssiin korotus valmis
console.log(tulos); // 8

```

Ei sen kummempaa. Siirrytään seuraavaan ratkaisuun.


### Ratkaisu #4

```javascript

var vastaanotaData = function(data) {
  var summa = kasitteleData(data);
  naytaSumma(summa);
}

var kasitteleData = function(data) {
  // Käsittele data
  return _.reduce(data, function(sum, datanum) {
    return sum + datanum;
  }, 0);
}

var naytaSumma = function(summa) {
  // Näytetään summa käyttäjälle
  alert("Summa on " + summa);
}

$.ajax({
  url: 'serveri.php'
}).done(vastaanotaData);

```

Ratkaisussa numero 4 viemme funktionaalisen abstraktion vielä yhden askeleen pidemmälle. Saimme jo kolmosratkaisussa eroteltua erilleen alkuperäiset kolme toimintoa - vastaanoton, käsittelyn, ja näytön. Nelosratkaisun ero kolmoseen verrattuna on, että ajax-kutsun kyytipojaksi annettu done-metodin callback on erikseen nimetty ja määritelty funktio. Sen nimi on *vastaanotaData*.

Vastaavan funktion käyttä toi lisäarvoa ratkaisussa #2, joten takuulla se auttaa myös nyt, vai mitä?

Ei välttämättä.

Tässä kohtaa on hyvä huomata, että ratkaisussa #3 ei ollut funktiota nimeltä *vastaanotaData*. Se, että sen nimistä funktiota ei ole, ei tarkoita, etteikö toimintoa olisi olemassa. Toiminto oli olemassa jo ratkaisussa #3 - se vain sattui olemaan anonyyminä funktiona. Verrataan:

```javascript

// Ratkaisu 3 - vastaanotto anonyyminä funktiona

.done(function(data) {
  var summa = kasitteleData(data);
  naytaSumma(summa);
});

// Ratkaisu 4 - vastaanotto erikseen nimettynä funktiona

.done(vastaanotaData);

```

Nyt kysymys kuuluukin, onko ratkaisu #3 **automaattisesti** huonompi kuin ratkaisu #4?  

Juuri aiemmin mainitsin, että funktionaalinen abstraktio on ohjelmoinnin keskiössä, ja erittäin tärkeä työkalu. Voiko tästä johtaa, että pienikin lisäpotku funktionaalista abstraktiota poikkeuksetta parantaa koodin laatua?

Ei. 

Saimme jo ratkaisussa #3 abstraktoitua kaiken sen mitä halusimmekin - eli *vastaanoton*, *käsittelyn* ja *lopputuloksen näyttämisen ihmiskäyttäjälle*. 

Ylimääräinen abstraktio tuon kolmosratkaisun päälle ei enää paranna koodia - se saattaa jopa huonontaa sitä. Tässä tapauksessa muutos on lähinnä neutraali.

Huomioitavaa on, että tarjoamme Ajax-kutsun done-metodille kyytipojaksi funktion, joka *vastaanottaa* palvelimelta tuodun datan. Tuo funktio siis vastaanottaa datan riippumatta sen nimestä tai siitä onko sillä nimeä lainkaan.

Toinen huomiotava asia on, että tuo vastaanotaData-funktio *ei tee mitään varsinaista työtä*. Se ainoastaan koordinoi kahta kutsua muihin funktioihin. Nuo muut funktiot tekevät ns. oikeaa työtä. Koska vastaanotaData-funktio on ikäänkuin *esimies*, ei sen abstraktoinnista saa yhtä suurta hyötyä kuin *työmiehen* toiminnan abstraktoinnista. 

> Funktionaalinen abstraktio - eli "koodirivien paketointi funktion sisälle" - on äärimmäisen tärkeä työkalu ohjelmoijan arsenaalissa.
>
> Mutta abstraktionkin voi viedä liian pitkälle. Abstraktio toimii vain siihen pisteeseen saakka, jossa viimeinenkin looginen toiminto on eroteltuna omaksi paketikseen. Tämän jälkeen abstraktion lisääminen tuppaa vain sotkemaan koodia. 

