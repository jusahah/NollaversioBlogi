+++
date = "2016-10-04T17:40:46+03:00"
draft = false
title = "Yksi taulu, useampi objekti (part 2)"

+++

*(jatkoa edelliselle postauksella)*

Eli kysymys siis on: **Miten yhdistämme usean eri luokan yhteen tauluun ja miksi haluamme niin tehdä?**

Yksi tapa vetää mutkat suoriksi on tehdä yksinkertainen taulu, joka sisältää objekti-ID:n ja sitten tekstimuodossa valinnaisen datan, joka kuvaa objektia:

```

id | data
-- | ----------------------------------
1  | {name: 'jaakko', osoite: '...'}
2  | {pankki: 'OP', puhelin: '...'}
3  | {yhteys: 'SQLDriver', args: '...'}

```

Tällä tavalla on helppo saada eri tyyppiset objektit menemään samaan tauluun. Riittää, että objektin sisältö kyetään mahduttamaan data-kenttään, ja avot. 

Mutta hetkinen, jotain puuttuu. Miten erotamme eri tyyppiset objektit toisistaan? Tarvitsemme uuden sarakkeen:

```

id | tyyppi  | data
-- | ------- | ----------------------------------
1  | Henkilo | {name: 'jaakko', osoite: '...'}
2  | Pankki  | {pankki: 'OP', puhelin: '...'}
3  | Ajuri   | {yhteys: 'SQLDriver', args: '...'}

```

Nyt meidän *tyyppi*-sarake kertoo millainen objekti kyseiselle riville on tallennettu. Teoriassa tuon objektin tyypin olisi voinut tallentaa osaksi data-attribuuttia, mutta parempi näin. Sillä nyt pystymme tekemään hakuja *tyyppi*-attribuuttia hyödyntäen.

Muokataan ylläolevaa meidän kommunikaatioesimerkkiä varten:

```

// Taulu: 'kommunikaatiot'

id | tyyppi     | data
-- | ---------- | --------------------------
1  | Savumerkki | {savunvari: 'harmaa', ...}
2  | Valomerkki | {aallonpituus: '30', ...}
3  | Puhelin    | {numero: 0409351405, ...}

```

Jokainen rivi sisältää tiedon siitä millainen **konkreettinen** kommunikaatiotapa on kyseessä, ja tarvittavan lisäinfon tuon tavan käyttämiseksi applikaatiokoodissa.

Miten sitten applikaatiokoodi tietää luoda oikeanlaisen objektin kunkin rivin pohjalta?

Muistutetaan mieleen, että tämä oli koko "yhden taulun periytyvuuden"-lähtökohta; kyky luoda *eri* objekteja *saman* taulun tietueista. Olemme kivasti onnistuneet koodaamaan tietuetyypin osaksi riviä ("tyyppi"-sarake!), mutta kuinka luoda objekti tuon sarakkeen avulla?

Laravellissa homma onnistuu laittoman helposti; *voimme nätisti korvata vakioluontimetodin omalla metodillamme, joka tarkastaa tyyppi-sarakkeen ja valitsee oikean objektiluokan sarakkeen arvon perusteella!*

Kas, näin:

```php

use App\Models\Puhelin;
use App\Models\Valomerkki;
use App\Models\Savumerkki;

class Kommunikaatio extends Model {
	
  public function newFromBuilder($attributes = array(), $connection = null) {

    $m;

    $tyyppi = $attributes->tyyppi;

    // Voisimme myös instantoida suoraan "tyyppi"-attribuuttia käyttäen:
    // $m = new $tyyppi($attributes->data);
    // Tällöin emme tarvitsisi if-lausekkeita lainkaan!

    if ($tyyppi === 'Puhelin') {
      $m = new Puhelin($attributes->data);
    } 
    else if ($tyyppi === 'Savumerkki') {
      $m = new Savumerkki($attributes->data);
    } 
    else if ($tyyppi === 'Valomerkki') {
      $m = new Valomerkki($attributes->data);
    }     	
    else {
      throw new \Exception('Missing type: ' . $tyyppi);
    }

    return $m;
  }
}

```

Nyt meidän abstrakti konseptimallimme *Kommunikaatio* - joka on suoraan kytketty *kommunikaatiot* tietokantatauluun - tekee päätöksen lopullisesta *konkreettisesta* objektiluokasta, jonka perusteella objekti luodaan!

Tämän päätöksen Kommunikaatio tekee tarkastelemalla *tyyppi*-attribuuttia, ja valitsemalla sopivan mallin. Tuon sopivan mallin pohjalta luotu uusi objekti sitten palautetaan ulos metodista.

Kaiken hienous on siinä, että metodia kutsutaan Laravellin itsensä toimesta. Eli kun applikaatiokoodini hakee tietyn kokoelman *kommunikaatioita* tietokannasta, kukin kommunikaatio rakennetaan ylläolevan *newFromBuilder*-metodin kautta!

```php

// Esimerkki
Kommunikaatio::all(); // [Puhelin, Valomerkki, Valomerkki, Puhelin, ...]

```

Toisin sanoen pystyn yhdellä ylätason kutsulla *Kommunikaatio::all()* luomaan kokoelman, joka sisältää eri objekteja. Tämä on aika hienoa. Koska nyt voin käsitellä noita eri objekteja miten haluan. Niin kauan kuin ne kaikki noudattavat jotain kommunikaatiokanaville yhteistä käyttöliittymää, ei ongelmia synny.

```php

// Esimerkki
$kommunikaatiot = Kommunikaatio::all();

$kommunikaatiot->each(function($komm) {
  // Tässä on hienous! Voimme polymorfisesti kutsua
  // tiettyä metodia tietämättä lainkaan mikä konkreettinen
  // objekti "$komm" itse asiassa on!

  // Puhelin, Valomerkki, Savumerkki kaikki tarjoavat "send"-metodin.
  $komm->send('Haloo!');
});

```

> Single-table inheritance - yhden taulun periytyvyys - antaa mahdollisuuden tallentaa yhteen ja samaan tauluun eri tyyppisiä objekteja. Mikä parasta, Laravellin avulla voimme luoda kokoelmia, jotka sisältävät noita eri tyyppisiä objekteja. Kaiken huippuna voimme käsitellä kokoelmia ilman, että tiedämme mitä tyyppiä kukin objekti on. Riittää, että kukin objekti tarjoaa tietyn yhteisen käyttöliittymän (interface).

