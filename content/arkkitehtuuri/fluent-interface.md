+++
date = "2016-08-22T03:08:34+03:00"
draft = false
title = "Ketjutettava rajapinta"

+++

Ohjelmoinnin puolella on olemassa kätevä konsepti nimeltä "Fluent interface". Paras suomennos tuolle lienee "ketjutettava rajapinta", joten käytän sitä.

Mikä tai millainen on ketjutettava rajapinta? Se on yksinkertaisesti rajapinta, joka mahdollistaa rajapintakutsujen ketjutuksen.

Otetaan esimerkkinä rajapinnasta, joka **ei** ole ketjutettava:

```php

// Rajapintaluokan 'Valot' kautta voi hallita talon valokatkaisijoita
class Valot {

  // Kukin toggle-metodi sytyttää valot jos ovat pois päältä,
  // ja sammuttaa valot jos ovat päällä.

  public function toggleVessa() {//...}
  public function toggleKeittio() {//...}
  public function toggleOlohuone() {//...}
  public function toggleMakuuhuone() {//...}
  public function toggleParveke() {//...}
}

```

```php

$valot = new valot();

// Kutsutaan metodeja
$valot->toggleKeittio();
$valot->toggleVessa();
$valot->toggleMakuuhuone();

```

Yllä meillä on tavallinen rajapinta. Kutsumme sitä metodi kerrallaan. Koska yksikään metodikutsu ei *palauta mitään palautusarvoa* - tai ainakaan emme mitään palautusarvoa ota vastaan - voimme olettaa, että kukin metodikutsu suorittaa jonkin ulkoisen muutoksen (engl. side effect). Jos metodikutsut eivät tuota ulkoisia muutoksia, koko rajapinnan käyttö on yksinkertaisesti turhaa.

> Mikäli metodikutsu ei palauta mitään eikä muokkaa yhdenkään ulkoisen tilamuuttujan arvoa, kyseessä on täysin tarpeeton metodikutsu. Sillä KAIKKI metodikutsut tehdään jomman kumman syyn takia; joko ne 1) *palauttavat jonkin arvon*, tai ne 2) *muokkaavat jotakin ulkoista tilamuuttujaa*. 
>
> Mitään kolmatta vaihtoehtoa ei ole olemassa, esimerkiksi tulosteen kirjoittaminen käyttäjän nähtäville on versio vaihtoehdosta #2 - siinä tietokoneen näyttöpäätteen tilamuuttujaa (= RAM-keskusmuistin sitä muistialuetta, johon kunkin pikselin tila on tallennettu) muokataan siten, että ihmiskäyttäjä näkee lukea jotain.

Nyt voimme muuntaa ylläolevan koodin *fluent interface*:ksi eli ketjutettavaksi rajapinnaksi hyvin yksinkertaisesti. 

```php

class Valot {

  public function toggleVessa() {
    //...
    // Palautusarvo on oleellista ketjutuksen kannalta. 
    // Palauttamalla kutsuttavan objektin voimme samantien kutsua sen
    // jotain metodia (vaikka tätä toggleVessa-metodia!) heti uudestaan.
    return $this;
  }
  public function toggleKeittio() {
    //...
    return $this;
  }
  public function toggleOlohuone() {
    //...
    return $this;
  }
  public function toggleMakuuhuone() {
    //...
    return $this;
  }
  public function toggleParveke() {
    //...
    return $this;
  }
}

```

```php


$valot = new Valot();

// Kutsutaan metodeja
$valot->toggleKeittio()->toggleVessa()->toggleMakuuhuone();

```

Yltä näemme mitä ketjutus tismalleen tarkoittaa; voimme kunkin metodikutsun palautusarvon *kierrättää* ja kutsua sen metodia. Ja koska **kunkin Valot-luokan metodikutsun palautusarvo on Valot-objekti itse**, ketjutus johtaa identtiseen lopputulemaan alkuperäisen esimerkin kanssa.

Itseasiassa seuraavat kolme koodipätkää johtavat kaikki identtiseen lopputulemaan:

```php 

// Tapa 1
$valot->toggleKeittio()->toggleParveke()->toggleKeittio();

// Tapa 2
$valot->toggleKeittio();
$valot->toggleParveke();
$valot->toggleKeittio();

// Tapa 3
// Tämä on huonoin tapa mitä tulee koodin selkeyteen, mutta toimii yhtäkaikki.
$valotKopio1 = $valot->toggleKeittio();
$valotKopio2 = $valotKopio1->toggleParveke();
$valotKopio3 = $valotKopio2->toggleKeittio();

```

Mitä etua ketjutus sitten tuo? Niin. Ei oikein mitään. Siinä säästää muutaman hassun merkin kun ei tarvitse toistaa '$valot'-sanaa uudestaan ja uudestaan. Ei kovin merkittävä hyöty. 

Jonkun mielestä ketjutus tekee koodista nätimpää tai helpommin luettavaa. Olen samaa mieltä, mutta kyseessä on ihan puhdas mielipidekysymys. 

> Huomio! Tässä ketjutettava rajapinta esiteltiin PHP-kielen kautta. Ketjutuksen konsepti ei ole sidonnainen PHP-kieleen, vaan pätee kutakuinkin kaikissa funktiokutsuja tukevissa ohjelmointikielissä.








