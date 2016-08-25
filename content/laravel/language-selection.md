+++
date = "2016-08-25T06:22:35+03:00"
draft = false
title = "Monikielisyys Laravel-kehyksen turvin"

+++


Tyypillinen web-applikaatio tarjoaa käyttäjilleen HTML-sivuista koostuvan käyttöliittymän. Tuo käyttöliittymä sisältää luonnollisesti tekstiä. Monet pienemmät ohjelmistot sisältävät tekstin ainoastaan ensisijaisen käyttäjäryhmän äidinkielellä, mutta suuremmat ohjelmistot ovat lähes poikkeuksetta *monikielisiä*.

Monikielisyys toteutetaan loppukäyttäjän kannalta usein niin, että käyttöliittymän yläpalkissa (tai vastaavassa) on valikko, josta kielivalinnan voi määrittää. 

Laravel tekee kielivalintojen käytöstä helppoa. Monikielisyys pohjaa kahteen toimenpiteeseen:

1. Määritä kullekin kielivalinnalle oma *kielihakemisto*, joka sisältää käännökset (joko yhdessä tai useammassa tiedostossa) kaikkiin käyttöliittymässä esiintyviin tekstipätkiin. Oleellista on, että kunkin kielihakemiston sisäinen tiedostorakenne on samankaltainen muiden kielihakemistojen kanssa. 

2. Koodaa käyttöliittymä siten, että kaikkialla viitataan tiettyyn käännöstiedoston nimeen. Ei siis tiettyyn käännöstiedostoon (eli *tiedoston täydelliseen tiedostopolkuun*), vaan ainoastaan tiedostonimeen. Missään **ei** aktiivisesti viitata tiettyyn kielihakemistoon. Kielihakemiston valinnan hoitaa Laravel pinnan alla.

Käytännössä siis kullekin kielelle luodaan ensin oma hakemisto. Tuonne hakemistoon luodaan kielitiedostot.

```php

// resources/lang/en/tervehdykset.php

return [
  'tervetuloviesti' => 'Hi and Welcome!'
];

```

```php

// resources/lang/fi/tervehdykset.php

return [
  'tervetuloviesti' => 'Tervetuloa!'
];


```

Nyt kaikkialla applikaation käyttöliittymän koodipohjassa viittamme tuohon lista-indeksiin *tervetuloviesti*. Pinnan alla Laravel osaa tällä tavoin hakea oikean tekstin riippuen siitä, mikä kieli on kulloinkin valittuna.

```php

// resources/views/etusivu.blade.php

<h1>{{trans('tervehdykset.tervetuloviesti')}}</h1>

```

Ylläoleva tuottaa loppukäyttäjän näkyville joko h1-tagilla ympäröidyn tekstin *Hi and Welcome!* (mikäli englanti on valittuna), tai *Tervetuloa!* (mikäli suomi valittuna).

Huomaa funktiokutsu *trans()*, joka suorittaa käännöksen.

### Miten Laravel päättää mikä kieli on kulloinkin valittuna?

Yllä oletimme, että Laravel on valinnut tietyn kielen käyttöönsä, ja sen valinnan perusteella käy hakemassa *oikeasta hakemistosta* tarvittavan käännöstekstin. 

Vakiokielivalinnan voi kertoa Laravellille helposti suoraan config-tiedostossa:

```php

// config/app.php

// Muut asetukset...

// Applikaation vakiokieli
'fallback_locale' => 'en'

```

Asettamalla config-tiedoston vakiokieleksi englannin (en), Laravel osaa käyttää englannin käännöksiä *ellei sitä toisin ohjeisteta*.

Nyt sitten kysymys kuuluukin, kuinka ohjeistaa Laravellia toisin? Entä jos haluamme suomen käännökset käyttöön?

Kätevin tapa lienee koodata tieto käyttäjän kielitoiveesta suoraan osaksi URL-osoitetta:

```php

// routes.php

Route::group([
  'prefix' => 'app/{kielivalinta}', 
  'middleware' => 'asetaKieli'], 
  function() {
    Route::get('front', function() {/* ... */});
    // jne.
  }
)

```

```php

// app/Http/Middleware/Asetakieli.php

// Tämä middleware pitää muistaa rekisteröidä Laravellin käyttöön.

namespace App\Http\Middleware;

use Closure;

class AsetaKieli {

  public function handle($request, Closure $next) {
    // Aseta kielivalinta
    \App::setLocale($request->route('kielivalinta'));
    return next($request);
  }

}

```

Ylläolevan ansiosta voimme helposti määrittää haluamamme kielen osana URL-osoitetta:

Suomi:

```
http://www.testiohjelma.fi/app/fi/front

```

Englanti:

```
http://www.testiohjelma.fi/app/en/front

```

> Loppukaneetti: koodissa viittasimme muuttujaan/lista-indeksiin nimeltä "tervetuloviesti". Tämä muuttujan nimi on siis koodissa suomeksi. Pitäisikö myös tälle olla käännös? Ei, sillä loppukäyttäjä ei koskaan näe koodin sisällä käytettäviä muuttujien nimiä. 
>
> Nyrkkisääntönä on, että koodin muuttujien nimet määritetään englanniksi, koska valtaosa ohjelmoijista käyttää englantia työkielenään. Mikään pakko näin ei ole toimia tietenkään.







