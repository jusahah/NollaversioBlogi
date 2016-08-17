+++
date = "2016-08-17T03:40:31+03:00"
draft = false
title = "Require vs Include"

+++

PHP:ssa on mahdollisuus *sisällyttää* yhden tiedoston koodipätkä toisen tiedoston sisälle skriptiä ajettaessa. Tämä sisällytys onnistuu joko *require* tai *include* komennoilla:

```php

// Auto.php

require 'Ratti.php';

class Auto {
  public function __construct(Ratti $ratti, $autoMerkki) {...}
  //...
}

$volvo = new Auto(new Ratti, 'Volvo');

```
```php

// Ratti.php

class Ratti {
	//...
}

```

tai 

```php

// Auto.php

include 'Ratti.php';

class Auto {
  public function __construct(Ratti $ratti, $autoMerkki) {...}
  //...
}

$volvo = new Auto(new Ratti, 'Volvo');

```
```php

// Ratti.php

class Ratti {
	//...
}

```

Yllä koodiesimerkit eroavat toisistaan vain yhden rivin suhteen; ensimmäinen esimerkki turvautuu PHP:n komentosanaan *require*, jälkimmäinen esimerkki käyttää termiä *include*. Mitä eroa näillä kahdella on?

### Require vs. include

On ensin syytä ymmärtää näiden kahden termin yhtäläisyys; molemmat tuovat ulkoisen tiedoston sisältämän koodin osaksi sitä tiedostoa, jossa termi sijaitsee.

Ne siis käytännössä *copypastaavat* palan koodia tismalleen siihen kohtaan, jossa require/include-termiä käytetään.

Kahden termin välinen ero on yksikertainen. 

**Require vaatii, että copypastattava tiedosto on olemassa.**

**Include EI vaadi copypastattavan tiedoston olemassaoloa.**

Require on siis hiukka tiukkapipoisempi versio include-käskystä. Mutta mitä tarkoittaa "vaatia tiedoston olemassaolo"?

Se tarkoittaa yksinkertaisesti sitä, että jos *require* yrittää sisällyttää olemattoman tiedoston, PHP-skripti räjähtää käsiin. Teknisesti tarkempi termi tälle posahtamiselle on keskeyttää skriptin suoritus virhekoodilla "Fatal error". Yhtäkaikki, asiat menevät päin honkia.

Jos puolestaan *include* yrittää sisällyttää olemattoman tiedoston, PHP-skripti ei räjähdä käsiin, vaan jatkaa suoritustaan kuin mitään ei olisi tapahtunut. 

Tästä kaikesta herää kysymys; jos haluamme sisällyttää yhden kooditiedoston sisältämän koodin osaksi toista tiedostoa, kaipa me vaadimme tuon tiedoston olemassaolon?

Asia ei aina välttämättä ole näin. Esimerkkinä tilanne, jossa meillä on tietyt vakioasetukset PHP-skriptillemme. Nuo vakioasetukset määritetään koko applikaation elinkaaren ensihetkillä.

Vakioasetukset voidaan kuitenkin ylikirjoittaa erillisen *asetustiedoston* avulla. Jos asetustiedosto on olemassa, sen sisältämä koodi *korvaa* vakioasetukset omilla asetuksillaan. 

Jos asetustiedostoa ei ole olemassa, vakioasetukset jäävät voimaan.

Ylläolevan esimerkin mukaisen rakenteen voi toteuttaa *include*-käskyllä näin:

```php

// applikaatio.php

// Vakioasetukset
$tcpPortti = "8080";
$tcpTimeout = 5000;

// Tuodaan sisään korvaavat asetukset sisältävä tiedosto
// HUOM! Jos tiedosto ei ole olemassa, mitään ei tapahdu
// ja vakioasetukset jäävät voimaan!
include "kayttajan_asetukset.php";

// ... rakenna applikaatio yms. käyttäen yllämääriteltyjä asetuksia

```

```php

// kayttajan_asetukset.php

// Käyttäjän erilliset, korvaavat asetukset
$tcpPortti = "3000";
$tcpTimeout = 12000;

```

Jos *kayttajan_asetukset.php*-tiedostoa ei ole olemassa, vakioasetukset jäävät voimaan. Jos tuo tiedosto on olemassa, käyttäjän omat asetukset korvaavat (muuttujat alustetaan uusiin arvoihin!) vakioasetukset.

> Include-käsky on toimiva tapauksissa, joissa sisällytettävä koodi *tuo valinnaisia lisäominaisuuksia* ympäröivän koodin käyttöön.
>
> Require-käsky on asianmukainen tapauksissa, joissa sisällytettävä koodi on elintärkeä applikaation toiminnan kannalta, ja tiedoston puuttuminen on syytä nähdä virhetilanteena.