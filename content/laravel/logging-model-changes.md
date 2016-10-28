+++
date = "2016-10-28T21:26:18+03:00"
draft = false
title = "Laravel: seuraa datan muutoksia"

+++

Tänään törmäsin mielenkiintoiseen kysymykseen Laravellin englanninkielisellä keskustelupalstalla Laracast.com:ssa.

Kysymys meni näin:

> I have a classic create() function to create elements, but changes I wish to save in a separate table, like history. There is table: element_changes and also model created named ElementChange, but in my ElementController, how can I tell to save it in a separate table?

Vapaasti suomennettuna siis:

> Minulla on tyypillinen luontifunktio, joka luo uusia malleja. Mutta haluaisin erilliseen tietokantatauluun kirjata ylös luontihistorian. Eli kun luon uuden objektin mallin pohjalta (tai *muutan* olemassaolevaa mallia), järjestelmä kirjaa lokitiedon asiasta erilliseen tauluun. Kuinka saavuttaa tämä?

Hyvä kysymys. Olen itse tarvinnut vastaavaa. 

Miksi tuollainen lokihistoria sisältäen muutokset on hyödyllinen? Selkeä käyttötarkoitus on järjestelmissä, joille vallitseva laki asettaa vaatimuksia. Yksi yleinen vaatimus on, että järjestelmän tulee pitää tarkkaa kirjaa *kaikista* järjestelmän sisällä tapahtuvista muutoksista.

Tälläinen kirjanpito on järkevä hoitaa lokihistorian avulla, jonne kirjaa lyhyen tiedoksiannon jokaisesta muutoksesta.

> Otetaan esimerkkinä ydinvoimalan hallintajärjestelmä. Siellä tuollainen muutos - jonka haluamme kirjata ylös - voisi olla reaktorin polttoainesauvan liikuttaminen. 
>
>Kun järjestelmän ylläpitäjä antaa järjestelmälle komennon siirtää polttoainesauvaa kolme senttiä ylöspäin, järjestelmän on syytä kirjata lokitieto asiasta.
>
>Sillä jos jotain menee pieleen, poliitikot haluavat tietää *tismalleen mitä ja miksi meni pieleen*! Lokihistoria auttaa.

### Toteutus

Jälleen kerran Laravel tekee lokihistorian pitämisen laittoman helpoksi. Käytännössä homma toimii näin; määrität kullekin *malliluokalle* muutaman metodin, joita Laravel-kehys kutsuu aina tietokantaa päivittäessään. Näiden metodien sisällä pusket lokitiedon lokihistoria-tauluun.

Otetaan hypoteettisena esimerkkinä tuo ydinvoimala. 

Meillä on malliluokka nimeltä "Polttoainesauva", joka on tämän näköinen:

```php

class Polttoainesauva extends Model {
	
  public function nostaYlos() {//...}
  public function laskeAlas() {//...}
}

```

Malliluokkamme on varsin yksinkertainen; sille on määritelty ohjelmoijan toimesta vain kaksi metodia. 

Ensimmäinen metodi nostaa sauvan ylös, toinen laskee sen takaisin alas. Metodien tarkemmat määritykset eivät ole oleellisia. 

Oletamme, että sauvojen asento/sijainti on kunakin hetkellä tallennettuna tietokantaan. Oikeassa maailmassa "tietokantana" toimisi ydinreaktori, mutta tämä on web-applikaatio, joka simuloi oikeaa maailmaa.

Jossain kohtaa applikaatiota meillä on seuraava koodinpätkä:

```php

$polttoainesauva->nostaYlos();

```

Ylläolevaa koodinpätkää voi ydinlaitoksen huoltoteknikko kutsua jonkinlaisen rajapinnan kautta. 

Ydinkysymys: **miten saamme järjestettyä siten, että polttoainesauvan nostosta jää yksiselitteinen lokitieto järjestelmän historiaan?**

Annoin vastauksen jo tämän kappaleen alkupuolella. Tutkitaan kuitenkin ensin pari huonoa tapaa hoitaa homma.

### Tapa 1

Yksi tapa on muokata ylläolevaa koodinkutsua seuraavanlaiseksi:

```php

$polttoainesauva->nostaYlos();
// Kirjaa lokiin
Loki::write('Polttoainesauva nostettu');

```

Ratkaisu on yleisellä tasolla huono, sillä entä jos useampi rajapintafunktio nostelee sauvaa? Tällöin lokikirjauksen tekeminen tulisi muistaa tehdä kaikkialle erikseen! 

Tämä on vaarallista ihan siksi, että ennemmin tai myöhemmin joku puolikätinen ohjelmoija pöllähtää paikalle ja muokkaa rajapintaa *unohtaen* lokikirjauksen lisäyksen! 

### Tapa 2

Huomattavasti parempi tapa on siirtää lokikirjaus suoraan Polttoainesauva-luokan metodien oheen:

```php

class Polttoainesauva extends Model {
	
  public function nostaYlos() {
    // Tee nosto
    Loki::write('Polttoainesauva nostettu');


  }

  public function laskeAlas() {
    // Tee lasku
    Loki::write('Polttoainesauva laskettu');  

  }
}

```

Nyt voimme olla varmoja, että sauvoja ei nosteta/lasketa ilman lokikirjausta. 

Vai voimmeko? Entä jos koodarimme menee typeryyspäissään kirjoittamaan uuden rajapintafunktion tyyliin:

```php

function vedenPintaKriittisenAlhaalla() {
  // Kiireellä sauva pois matalasta vedestä!
  // (Disclaimer: en tiedä lainkaan toimisiko tälläinen
  // varotoimenpide oikeassa elämässä...dont try at home!)
  $polttoainesauva->asento = 'ylös';
  $polttoainesauva->save();

  // Unohtuiko jotain...?
}

```

Kirjataanko tuossa mitään lokiin? Ei, sillä uusi noviisiohjelmoija meni muuttamaan sauvan asentoa *ohitse* meidän nostaYlos-metodimme. Siispä lokikirjausta ei tehty.

No, ydinvoimalat eivät palkkaisi diplomi-insinöörejä, joten ylläolevaa ei pääse tapahtumaan. Mutta on hyvä tiedostaa riskit.

Eikä siinä vielä kaikki. Tuossa lokikirjausten tekemisessä Polttoainesauva-luokkaan on toinenkin ongelma: entä jos meillä on *sadoittain* vastaavia malliluokkia ympäri applikaatiotamme?

Meidän tulisi *jokaikiseen* kirjata *jokaikisen* tietokantaa muokkaavan metodin kohdalle lokikirjaus! Helvetinmoinen urakka, muuten.

### Tapa 3

Paras keino on luottaa [Trait](http://php.net/manual/en/language.oop5.traits.php)-konseptin* voimaan.

Lisäämällä kirjaustoiminnot sisältävä Trait kunkin malliluokan oheen, meidän ei tarvitse huolehtia juuri mistään muusta! Laravel-kehys huolehtii siitä, että Traitin sisältämät *kuuntelijafunktiot* kutsutaan aina kun tietokantaa muokataan.

Huono puoli tässäkin on - meidän tulee edelleen muistaa sisällyttää tuon Trait jokaisen malliluokan oheen. Mutta ainakaan meidän ei tarvitse enää huolehtia yksittäisistä metodeista. Yksi lisäys per malliluokka riittää. 

Ja mikä parasta, **yksi ja sama Trait kelpaa kaikkiin malliluokkiin**.

Tämä viimeisin pointti on tärkeä; vaikka meillä olisi tuhat malliluokkaa, yksi Trait edelleen riittäisi.

Traitin avulla jokainen malliluokan metodi tulee automaattisesti "suojelluksi" - tarkoittaen, että **tietokannan muokkaus mistä ikinä metodista tulee kirjatuksi lokiin**.

Miltä tuo Trait näyttää? Tältä:


```php

trait Trackable {
  // Laravel kutsuu tätä metodia osana käynnistys-ajoaan.
  public static function bootTrackable() {

    static::creating(function ($model) {
      // Kirjataan tieto objektin luonnista
      Loki::write('Luonti: ' . get_class($model));
    });

    static::updating(function ($model) {
      // Kirjataan tieto objektin muokkauksesta!
      // HUOM! Emme tiedä millainen muokkaus on kyseessä, 
      // mutta objekti itse tietää!
      Loki::write('Muokkaus: ' . get_class($model) . $model->printData());
    });

    static::deleting(function ($model) {
      // Kirjataan tieto objektin kuolemasta!
      Loki::write('Kuolema: ' . get_class($model));
    });
  }
}

```

Ylläolevaa traittia voimme käyttää missä tahansa malliluokassa seuraavasti:

```php

class Polttoainesauva extends Model {
  use Trackable;
  // jne..
}

class Reaktori extends Model {
  use Trackable;
  // jne..
}

class Vesiallas extends Model {
  use Trackable;
  // jne..
}

class Lampomittari extends Model {
  use Trackable;
  // jne..
}

```

Muuta ei tarvita! Laravel-kehys hoitaa loput. Se pitää huolen, että aina kun tietokantaa muokataan jonkun em. malleista osalta, lokiin kirjataan tieto.

> Onko suojaus nyt täydellinen, täysin diplomi-insinööri-proof? Ei. Jos tietokantaa muokataan suoraan SQL-koodilla, lokikirjaus jää edelleen tekemättä. Mutta ainakin ohjelmoijilla on nyt vain yksi elinehto: **älä ohita Laravel-kehyksen omaa tietokanta-abstraktiota.**

*Perusidea on, että traitin sisältö copypastataan sellaisenaan siihen kohtaan koodipohjaa, jossa traitia käytetään (*use*).



