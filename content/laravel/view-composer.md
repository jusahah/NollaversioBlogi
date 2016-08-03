+++
date = "2016-08-03T03:49:47+03:00"
draft = false
title = "Datan lähetys näkymään"

+++

Laravellin kaltaisten täysiveristen ohjelmistokehysten yksi hienoimmista ominaisuuksista on *datan käsittelyn* ja *datan näytön* erottaminen. Laravellissa konseptit erotetaan toisistaan tiedostotasolla; datan käsittely - ns. bisneslogiikka - elää yhdessä tiedostossa, ja näyttölogiikka elää toisessa tiedostossa.

Bisneslogiikasta vastaava tiedosto olkoot kutsumanimeltään *model*, näyttölogiikasta vastaavaa tiedostoa kutsukaamme nimellä *view*.

Yleensä näiden kahden välissä istuu vielä kolmas konseptuaalinen palikka - tiedosto kutsumanimeltään *controller*. Laravel-kehys perustaakin vahvasti toimintansa nk. MVC-periaatteeseen.

MVC:ssä osa-alueet **model**, **view** ja **controller** erotetaan toisistaan. Erottelun ansiosta applikaation koodipohja on selkeämmin luettavissa ja muokattavissa.

Me yksinkertaistamme kolmijakoa hiukan ja muunnamme sen *kaksijaoksi*; datan käsittely sisältää MC-kirjaimet, ja datan näyttö sisältää V-kirjaimen.

Tutkitaan kahta eri tapaa siirtää vastikään käsiteltyä dataa näkymän käytettäväksi:

### Tapa #1:

Tapa 1 - siirretään data suoraan view-tiedoston käyttöön.

```php

// Controller.php

public function index() {
  // Hankitaan dataa jotenkin
  $dataset = getDatasetSomehow();
  // Siirretään data eksplisiittisesti view-tiedoston käyttöön.
  return view('View')->with('dataset', $dataset);
}

```

```php

// View.php

// Renderöidään data ihmiskäyttäjäm nähtäville.
<?php echo $dataset; ?>

```

Tapa 1 on suositelluin tapa. Kutakin näkymätiedostoa (view) otettaessa käyttöön määritämme samalla datan, jonka avulla näkymätiedosto renderöidään lopulliseksi HTML-koodiksi. Voimme vapaasti määrittää mitä dataa siirrämme näkymän käyttöön. Voimme myös käyttää samaa tiedostoa usean eri datapaketin renderöimiseen:

```php

// Controller1.php

public function index() {.
  return view('View')->with('dataset', 1);
}

```

```php

// Controller2.php

public function index() {.
  return view('View')->with('dataset', 2);
}

```

```php

// Controller3.php

public function index() {.
  return view('View')->with('dataset', 3);
}

```

Eri datapaketilla renderöidyt näkymät tuottavat eri HTML-koodit.


### Tapa 2 - määritetään *globaalisti* näkymälle tarjottava data

Tapa nro. 1 on yleisin keino siirtää dataa näkymän käyttöön/näytettäväksi. 

Mutta entä jos meillä on seuraavanlainen tilanne... tietty näkymätiedosto renderöidään aina tietyn *vakiodatan* turvin. Tämän vakiodatan lisäksi tuo näkymä ottaa vastaan myös *muuttuvaa dataa*. Esimerkki tälläisestä on web-portaali; vakiodata on esimerkiksi paikalliset säätiedot, jotka ovat kaikille käyttäjille samat ja aina nähtävillä (esim. yläreunan widgetin kautta). 

Eli kaikki käyttäjät näkevät saman säätilatiedotteen portaalin yläreunassa. Nämä säätiedot ovat näkyvillä kaikilla portaalin alasivuilla.

Muuttuva data on puolestaan kullekin käyttäjälle yksilöllinen, esim. kunkin käyttäjän viimeisin sisäänkirjautuminen. Lisäksi kullakin alasivulla on oma muuttuva datansa.

Voisimme vallan mainiosti määrittää molemmat datapaketit aina näkymätiedostoa renderöidessämme:

```php

// KirjautumisTiedotController.php

public function index() {
  $saa = $this->getSaaTiedot();
  $viimeisinKirjautuminen = $this->getLoginInfo();
  return view('kirjautumistiedot')
    ->with('saatiedot', $saa)
    ->with('viimeisinKirjautuminen', $viimeisinKirjautuminen);
}

```

```php

// VastaanotetutViestitController.php

public function index() {
  $saa = $this->getSaaTiedot();
  $viestit = $this->getMessagesReceived();
  return view('viestit_vastaanotettu')
    ->with('saatiedot', $saa)
    ->with('viestit', $viestit);
}

```

```php

// LahetetytViestitController.php

public function index() {
  $saa = $this->getSaaTiedot();
  $viestit = $this->getMessagesSent();
  return view('viestit_lahetetty')
    ->with('saatiedot', $saa)
    ->with('viestit', $viestit);
}

```

Ylläolevassa esimerkissä meillä on kolme eri näkymätiedostoa - *kirjautumistiedot*, *viestit_vastaanotettu* ja *viestit_lahetetty*. 

Kukin niistä hoitaa oman leiviskänsä applikaation käyttöliittymän renderöimisestä ihmiskäyttäjän silmille sopivaksi. 

Mutta kullakin näkymällä on *ydintehtävän* lisäksi myös oheistehtävä, joka on säätietojen renderöinti. Säätiedot ovat näkyvillä kaikilla applikaation alasivuilla, eli kaikki applikaation näkymät joutuvat ne renderöimään. Huomaamme tämän ylläolevissa kolmessa näkymäesimerkissä - kukin näkymä sisältää rivin:

```->with('saatiedot', $saa)```

On myös parempi tapa. Koska säätiedot on sisällytettynä jokaiseen applikaation alasivuun, voimme abstraktoida säätietojen hakeminen ns. **näkymälaatijaan** (engl. view composer).

Näkymälaatija on oma komponenttinsa, joka huolehtii tietyn datapaketin viemisestä eri näkymien saataville *automatisoidusti ja keskitetysti*. Toisin sanoen, datapakettia ei tarvitse enää määritellä näkymän saataville Controller-tiedoston sisällä, vaan tuo määrittely voidaan tehdä globaalisti näkymälaatijan sisällä.

Näkymälaatijasta tarkempi kuvaus (englanniksi): [https://laravel.com/docs/5.1/views#view-composers](https://laravel.com/docs/5.1/views#view-composers).

Säätietojen hakeminen ja antaminen applikaation *kaikkien* näkymien saataville onnistuu näin:

```php

// SaatiedotLaatijaProvider.php

class SaatiedotLaatijaProvider
{
  public function boot()
  {
    view()->composer('*', function($view) {
      $saatiedot = $this->getSaaTiedot();
      $view->with('saatiedot', $saatiedot);
    });
  }

  protected function getSaaTiedot() {
    /* Hanki säätiedot jotenkin */
  }
}


```

Määriteltyämme ylläolevan *näkymälaatijan* - tässä tapauksessa anonyymi funktio - huolehtimaan säätiedoista, voimme poistaa säätietojen hallinnasta vastaavan koodin kustakin Controllerista.

```php

// KirjautumisTiedotController.php

public function index() {
  $viimeisinKirjautuminen = $this->getLoginInfo();
  return view('kirjautumistiedot')
    ->with('viimeisinKirjautuminen', $viimeisinKirjautuminen);
}

```

```php

// VastaanotetutViestitController.php

public function index() {
  $viestit = $this->getMessagesReceived();
  return view('viestit_vastaanotettu')
    ->with('viestit', $viestit);
}

```

```php

// LahetetytViestitController.php

public function index() {
  $viestit = $this->getMessagesSent();
  return view('viestit_lahetetty')
    ->with('viestit', $viestit);
}

```

Applikaation koodipohja selventyi huomattavasti - enää ei sama säätietojen hakeminen + tarjoaminen näkymän käyttöön elä *kolmessa* eri sijainnissa, vaan koko säätietoja hallinnoiva koodi elää yhdessä paikassa (laatijatiedoston sisällä).

> Laravellin *näkymälaatija* on tehokas konsepti poistamaan tarpeetonta duplikaatiota. Mikäli tietty datapaketti on renderöitävä usealle applikaation alasivulle, on syytä harkita näkymälaatijan käyttöä.

> Kuten aina, kyseessä on tradeoff. Näkymälaatijalla saadaan vähennettyä koodin duplikaatiota. Mutta huonona puolena on, että tietyn näkymätiedoston käytettävissä oleva data ei enää ole nähtävillä yhdessä koodilokaatiossa. 

> Ratkaisussa #1 kaikki näkymälle tarjottava data tuli Controller-tiedostosta. Ratkaisussa #2 osa datasta tulee Controllerista, toinen osa näkymälaatijalta.







