+++
date = "2016-09-19T09:43:34+03:00"
draft = false
title = "Fasaadin feikkaus"

+++

Laravel hyödyntää runsaasti konseptia / design patternia nimeltä "Facade". Kehys tarjoaa kehittäjän käyttöön tarttumapinnan moniin aputoiminnallisuuksiin juurikin fasaadien kautta, esim. applikaation oman välimuistin käsittely käy helposti *Cache*-fasaadin avulla:

```php

// Cache-fasaadi tarjoaa meille globaalin tarttumapinnan 
// Laravellin omaan välimuistiin.
$nimi = Cache::get('pelaajan_nimi');

```

Fasaadin käytössä on myös heikkoutensa. Pääasiallinen heikkous on, että fasaadin kutsuminen on *staattinen kutsu*; toisin sanoen, kutsuttava luokka on määritelty suoraan koodiin. 

Toinen vaihtoehtohan on *olla määrittämättä* kutsuttavaa luokkaa suoraan koodiin. Miten ihmeessä se on mahdollista? Käyttämällä konseptia nimeltä *dependency injection*, eli riippuvuuksien injektointi.

Vertaa näitä kahta tapaa:

**Fasaadin käyttö**

```php

public function tallennaNimi() {
  // Cache-fasaadi tarjoaa meille globaalin tarttumapinnan välimuistiin.
  Cache::set('pelaajan_nimi', Auth::user()->name);	
}

```

**Riippuuvuuden injektointi**

```php

public function tallennaNimi(ICache $valimuisti) {
  // ICache-rajapintaa noudattava objektin ei tarvitse olla Cache-luokasta,
  // vaan se voi olla *mikä tahansa* objekti joka implementoi ICachen.
  $valimuisti->set('pelaajan_nimi', Auth::user()->name);	
}

```

Kahden ylläolevan esimerkin välinen ero on juurikin siinä, että **ensimmäisessä versiossa kutsumme staattisesti Cache-luokan metodia.** Jälkimmäisessä puolestaan **kutsumme dynaamisesti sisäänsaadun objektin metodia.**

Jälkimmäistä kutsua kutsumme nimeltä *polymorphinen* kutsu. Tämä tarkoittaa, että koodia *kirjoitettaessa* meillä ei ole varmaa tietoa siitä, mikä pätkä koodia lopulta tulee ajetuksi kun metodikutsu *$valimuisti->set()* suoritetaan.

Mitä hyötyä tuollaisesta polymorphisesta kutsusta on? Se, että voimme ulkoakäsin määritellä millainen ICache-rajapintaa noudattava objekti halutaan käyttöön.

```php

public function tallennaNimi(ICache $valimuisti) {
  // ICache-rajapintaa noudattava objektin ei tarvitse olla Cache-luokasta,
  // vaan se voi olla *mikä tahansa* objekti joka vain implementoi ICachen.
  $valimuisti->set('pelaajan_nimi', Auth::user()->name);	
}

// Vaihtoehto #1, Laravellin default-välimuisti
tallennaNimi(new Cache());
// Vaihtoehto #2, käytetään lokaalia tekstitiedostoa
tallennaNimi(new Loki('pelaajat.txt'));
// Vaihtoehto #3, käytetään Googlen nettilevyä
tallennaNimi(new HTTPCache('http://www.docs.google.com/jrk5u5emsdmk'));

```

Riippuvuuden injektointi on siis joustavampi kuin fasaadin käyttö. 

### Fasaadin feikkaus

Mutta.

Laravel 5.3 kehyksessä fasaadia käyttävän kutsun voi myös muuttaa polymorphiseksi. Muutos vain täytyy tehdä koko applikaatiolle kerrallaan. 

Tärkeä huomio: *yksittäistä fasaadikutsua ei voi muuttaa polymorphiseksi, mutta koko fasaadin voi.*

Tämä tarkoittaa, että kun defaulttina **Cache**-fasaadi johtaa Laravellin omaan välimuistiin, on mahdollista asettaa **Cache**-fasaadi johtamaan johonkin muuhun luokkaan. Muutos koskee koko applikaatiota.

Laravel 5.3 tarjoaa sisäänrakennetun korvausmekanismin. Kullekin fasaadille on määritelty *fake*-metodi, joka mahdollistaa korvata fasaadiin kytketty vakioluokka jollain muulla luokalla.

Otetaan esimerkkinä tuo Cache-fasaadi. Haluamme että Cache-fasaadi tallentaa välimuistitiedot Dropboxiin:

```php

class Cache extends Facade {

  public function fake() {
    // Korvaamme vakiotoiminnot tarjoavat luokan jollain toisella luokalla.
    // Tässä siis kytketään fasaadi siten, että missä ikinä
    // käytämmekään *Cache*-fasaadia, se vie meidät 
    // *NettiLevyValimuisti*-luokan metodeihin.
    static::swap(new NettiLevyValimuisti('dropbox.com/j53jySD'));
  }
	

}

```

Ylläoleva ei vielä ihan riitä. Meidän täytyy jotenkin ilmaista Laravel-kehykselle, että haluamme tuon swappauksen tehdä, eli haluamme ottaa nettilevyn käyttöön. Ilmoitus tehdään yksinkertaisesti:

```php

// Swapataan.
Cache::fake();

```

Tästä eteenpäin voimme *Cache*-fasaadin kautta tallettaa tietoja suoraan Dropboxiin.

```php

// Swapataan.
Cache::fake();

// Swappaus suoritettu.
// Pinnan alla HTTP-kutsu lähtee matkaan kohti Dropboxin palvelinta.
Cache::set('pelaajan_nimi', 'Jussi'); 

```

### Milloin fasaadin korvaus, milloin injektointi?

Yllä näimme kaksi tapaa järjestää rajapintakutsu. Ensimmäinen tapa turvaa fasaadin käyttöön. Toinen tapa turvaa sopivan objektin injektointiin ja sen objektin metodikutsuun.

On tärkeä huomata, että vaikka fasaadin "vakio-ohjaus" voidaan pinnan alla korvata kustomoidulla ohjauksella, on injektointi edelleenkin joustavampi tapa. Tämä johtuu siitä, että fasaadin tapauksessa korvaus on aina **globaali**. Tietty fasaadi johtaa aina tiettyyn implementaatioon. 

Injektointi taas mahdollistaa **lokaalin** korvauksen. Injektoinnin avulla kukin injektoidun objekti voi johtaa eri toiminnallisuuksiin:

```php

public function tallennaNimi(ICache $valimuisti) {
  $valimuisti->set('pelaajan_nimi', Auth::user()->name);	
}

// Eri toiminnallisuuksia voi olla rajaton määrä...
tallennaNimi(new Cache());
tallennaNimi(new Loki('pelaajat.txt'));
tallennaNimi(new HTTPCache('http://www.docs.google.com/jrk5u5emsdmk'));
tallennaNimi(new CDLevy());
tallennaNimi(new SaviTaulu());

// jne jne...

```
Fasaadia käytettäessä korvaus voidaan tehdä vain kerran.

```php

public function tallennaNimi() {
  Cache->set('pelaajan_nimi', Auth::user()->name);	
}

// Vakiotoiminnallisuuden voi korvata vain kerran.

tallennaNimi(); // Tallentaa vakio-välimuistiin.
tallennaNimi(); // Tallentaa vakio-välimuistiin.
tallennaNimi(); // Tallentaa vakio-välimuistiin.
Cache::fake(); // Suoritetaan korvaus
tallennaNimi(); // Tallentaa nettilevylle;
tallennaNimi(); // Tallentaa nettilevylle;
tallennaNimi(); // Tallentaa nettilevylle;
// jne jne...

```

> Injektointi on suositeltava tapa silloin kun on syytä dynaamisesti kesken business-koodin kyetä muuttamaan metodikutsun määränpäätä. Fasaadien käyttö on täysin ok jos tälläistä kykyä ei tarvitse. Testauksen kannalta molemmat ovat ok - testejä ajettaessa riittää, että esimerkiksi välimuisti korvataan feikkivälimuistilla globaalisti.








