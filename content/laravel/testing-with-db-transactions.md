+++
date = "2016-08-16T06:37:20+03:00"
draft = false
title = "Yksikkötestaus ja tietokanta-transaktio"

+++

Yksikkötestaus (engl. Unit Testing) on tehty Laravellissa helpoksi. Ei muuta kuin määrittää testiluokan, ja pinnan alla testiajuri hoitaa loput.

Tähän tyyliin:


```php

class LentokoneTesti extends TestCase {
	
  public function lentokoneella_on_kaksi_siipea() {
    // Oletetaan, että meillä on Lentokone-malli olemassa.
    $lentokone = new LentoKone()

    // Varmistetaan, että siipien lkm on kaksi.
    $this->assertEquals($lentokone->siivet->count(), 2);

  }
}

```

Kaikki hyvin yllä. Luomme Eloquent-mallin pohjalta objektin nimeltä *lentokone*, ja tarkistamme, että tuolla lentsikalla on kaksi kpl siipiä. 

Huomionarvoista on, että ylläolevassa testissä emme käytä tietokantaa lainkaan. Tämä on ihanteellista. Mutta joissain testeissä on kovin vaikea välttää tietokannan käyttöä:


```php

class LentokenttaTesti extends TestCase {
	
public function kentta_evaa_laskeutumisluvan_jos_kiitoradat_taynna() {

  $helsinkiVantaa = Lentokentta::create(['kiitoradat' => [
    'itäinen', 'läntinen', 'pohjoinen'
  ]]);

  // Luodaan neljä kappaletta lentokoneita
  // Laravellin factory-apumetodi auttaa.
  factory(Lentokone::class, 4)->create();

  // Lentokoneet ja lentokenttä on lisätty tietokantaan! 
  // Toisin sanoen, meidän on käytettävä tietokantaa suorittaaksemme testin loppuosan.

  // Varmistetaan, että lentokoneet tosiaan ovat tietokannassa.
  $koneet = Lentokone::all();

  // Lentokoneita tulisi siis olla neljä kpl
  $this->assertEquals($koneet->count(), 4);

  // Assignoidaan kullekin koneelle yksi kiitorata laskeutumiseen.
  $helsinkiVantaa->assignoiKiitorata($koneet[0]);
  $helsinkiVantaa->assignoiKiitorata($koneet[1]);
  $helsinkiVantaa->assignoiKiitorata($koneet[2]);

  // Nyt Helsinki-Vantaan kaikki kolme kiitorataa ovat käytössä, joten
  // viimeinen kone EI voi saada omaa kiitorataansa.

  // Varmistetaan, että lentokenttä ei sisällä vapaita kiitoratoja.
  $this->assertEquals($helsinkiVantaa->vapaatKiitoradat()->count(), 0);

  // Varmistetaan, että yritys assignoida olematon kiitorata johtaa virhetilanteeseen!
  // (En ole itsekään ihan varma miten tämä toteutetaan, mutta jotenkin seuraavasti...)
  $this->expectException(EiVapaitaKiitoratoja::class);

  $helsinkiVantaa->assignoiKiitorata($koneet[3]);

  // Nyt äskettäin asetetun exception handlerin tulisi olla lauennut.

  }
}

```

Ylläoleva testi käyttää tietokantaa. Ensin se luo tietokantaan yhden lentokentän ja neljä lentokonetta. Sen jälkeen testi suorittaa testilogiikan tietokantaan turvautuen.

Ylläolevan ongelma on, että kun testi on valmis, testin aikana luodut objektit jäävät lojumaan tietokantaan. Tämä on epämieluisa tilanne. Parhaimmillaan se on pelkkä suorituskykyongelma, pahimmillaan se johtaa tilanteisiin, joissa testi menee pieleen koska tietokanta sisältää ennalta-arvaamatonta roskaa.

### Use DatabaseTransactions

Tietokannan resetointi testin jälkeen on helppoa. Suorastaan laittoman helppoa. Lisätään vain yksi rivi koodia:

```php

class LentokenttaTesti extends TestCase {

  // Uusi rivi
  use DatabaseTransactions;
	
  public function kentta_evaa_laskeutumisluvan_jos_kiitoradat_taynna() {

    // Kuten aiemmin

  }
}

```

Lisäämällä rivin *use DatabaseTransactions* Laravel-kehys huolehtii omatoimisesti tietokannan putsaamisesta testin päätteeksi. 

DatabaseTransactions on siis *Trait*, joka käytännössä copypastaa *LentokenttaTesti*-luokkaan sopivat putsaustoiminnot. Testi suorituu nyt näin:

```php

class LentokenttaTesti extends TestCase {

  use DatabaseTransactions;
	
  public function kentta_evaa_laskeutumisluvan_jos_kiitoradat_taynna() {

    // Puhdas tietokanta

    // Kuten aiemmin, luodaan objekteja tietokantaan.
    // Sitten testataan, testataan niin pirusti.

    // Tyhjennä tietokanta

  }
}

```

Varsin kätevää.

> Tietokannan resetointi alkuperäiseen tilaan noudattaa nk. "same world"-periaatetta. Periaate tarkoittaa, että tietty testi ajetaan aina vakioidussa ympäristössä. Tässä tapauksessa tuo vakioympäristö on tyhjä tietokanta.




