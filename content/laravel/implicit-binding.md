+++
date = "2016-08-24T06:36:45+03:00"
draft = false
title = "Laravel injektoi mallin puolestasi"

+++

Laravel 5.2 -kehys toi mukanaan uuden ominaisuuden nimeltä *implicit model binding*. Suomennos on vaikea; "automaattinen mallin injektointi" kuvaa mielestäni parhaiten tuota konseptia. 

Sillä konseptin avulla voi pistää Laravellin tekemän raskas työ ja etsimään sopiva malliluokka, luomaan sen pohjalta uusi objekti, ja tarjoamaan objekti ohjelmoijan käyttöön.

Ero implisiittisen mallin injektoinnin ja ns. tavanomaisen koodiratkaisun välillä on seuraava:

**Vanha tapa (ei injektointia)**

```php

//routes.php

Route::get('sanomalehdet/{id}', function($id) {
  // Luodaan *eksplisiittisesti* Sanomalehti-objekti käyttäen id-parametriä.
  $lehti = Sanomalehti::findOrFail($id);

  return $lehti->sarjakuvat();

});

```

**Uusi tapa (injektointi käytössä)**

```php

//routes.php

Route::get('sanomalehdet/{id}', function(Sanomalehti $lehti) {
  // Meillä on käytössämme Sanomalehti-luokasta luotu $lehti-objekti.
  // $lehti luotiin automaattisesti id-parametrin perusteella.	

  return $lehti->sarjakuvat();

});

```

Huomaamme eron: vanhassa ratkaisussa *erikseen* haemme objektin tietokannasta ```Sanomalehti::find($id)```-kutsulla. Uudessa ratkaisussa Laravel-kehys hakee objektin tietokannasta meidän puolestamme.

Kumpikin ratkaisu toimii loppukäyttäjälle samalla tavalla - kutsumme URL-endpointia tyyliin:

```php

http://www.lehtiapp.fi/sanomalehdet/1

```

Objekti siis molemmissa tapauksissa haetaan tietokannasta - ero on vain siinä kuka hakee.

Tarkastalleen vaihe vaiheelta mitä oikeasti pinnan alla tapahtuu tuon URL-endpointin kutsun aikana:

1. Käyttäjä kirjoittaa URL:n osoiterivilleen.
2. Kutsu saapuu Laravel-applikaation HTTP-rajapintaan.
3. Laravel ohjaa kutsun määrittämäämme *Route::get('sanomalehdet/{id}')* callbackiin.
4. Pinnan alla Laravel tutkii tuon callbackin parametrilistan ja havaitsee tutkinnan seurauksena, että callback ottaa parametrikseen $lehti-objektin luokkatyyppiä Sanomalehti.
5. Laravel laskee yksi yhteen ja hoksaa, että URL:n sisällä tullut id-parametri on sama kuin callbackin parametriksi tulevan Sanomalehti-objektin id-attribuutti.
6. Laravel käy hakemassa sopivan Sanomalehti-objektin tietokannasta edelliseen päättelyyn pohjaten. Laravel siis tekee haun tyyliin: 

```php

Sanomalehti::where('id', $id)->first()

```

Kaiken tuon Laravel päättelee sen muutaman millisekunnin aikana, joka HTTP-kutsun vastaanottoon kuluu. Laravellilla on aika nopsat hoksottimet.

### ID-parametrin korvaaminen toisella attribuutilla

Entä jos haluamme, että voimme osoiteriville kirjoittaa seuraavanlaisen URL-lausekkeen:

```php

http://www.lehtiapp.fi/sanomalehdet/ristiinalainen

```

Koska termi 'ristiinalainen' ei ole id-attribuutti, Laravel-kehys ei löydä oikeaa lehteä sen avulla. Ellemme sitten *kerro Laravellille*, että haluamme lehden nimen (esim. 'ristiinalainen') toimivan hakuattribuuttina. 

Tämä on mahdollista määrittämällä uusi metodi Sanomalehti-malliin:

```php

// Sanomalehti.php

class Sanomalehti extends Eloquent {
	
	// Määritetään injektointiattribuutti, jota Laravel käyttää 
	// etsiäkseen oikean objektin tietokannasta.
	public function getRouteKeyName() {
		return 'nimi';
	}

}

```

```php

//routes.php

Route::get('sanomalehdet/{nimi}', function(Sanomalehti $lehti) {
  // Meillä on käytössämme Sanomalehti-luokasta luotu $lehti-objekti.
  // $lehti luotiin automaattisesti nimi-parametrin perusteella.	

  return $lehti->sarjakuvat();

});

```

Ylläoleva mahdollistaa meidän kutsuvan HTTP-endpointia tyyliin:

```php

http://www.lehtiapp.fi/sanomalehdet/ristiinalainen

```

Tämä on selkeä loppukäyttäjää helpottava parannus verrattuna aiempaan kutsuumme, jossa tietty lehti eriteltiin id-attribuutin avulla. Nyt lehdet eritellään niiden nimen avulla. Loppukäyttäjä ei osaa yhdistää id-numeroa tiettyyn lehteen. Lehden nimi taas heti kertoo mistä lehdestä on kyse.



