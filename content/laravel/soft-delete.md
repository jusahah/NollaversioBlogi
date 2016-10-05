+++
date = "2016-10-05T18:05:16+03:00"
draft = false
title = "Laravel ja pehmeä tuho"

+++

Laravel tarjoaa ohjelmoijan käyttöön konseptin nimeltä "soft delete". Suomennan tuo "pehmeäksi tuhoksi", koska termi on niin hauska.

Pehmeä tuho tarkoittaa seuraavaa: kun tietue poistetaan tietokannasta, sitä *ei oikeasti poistetakaan*, se vain merkitään näkymättömäksi.

Vastakohtana on tietenkin "kova tuho" - eli siis tuikitavallinen poisto-operaatio, jossa tietue ihan aidosti poistetaan tietokannasta.

### Miksi pehmeä tuho?

Herää kysymys, että mitä järkeä koko pehmeän tuhon konseptissa on? Poistamme tietueen, mutta emme poistakaan sitä. Häh? Miksi halusimme alunperinkään poistaa, jos emme sitten halunneetkaan.

Kaiken ytimessä on ajatus siitä, että *applikaation* tasolla tietue on saavuttamattomissa. Applikaatio siis elää käsityksessä, että tietue on ihan oikeasti tuhottu. Samaan aikaan kuitenkin yrityksen muut komponentit - esim. Business Intelligence - haluaa, että tietue on visusti tallessa. 

Tämä eri komponenttien erilainen tarve tietueen olemassaololle johtuu komponenttien eriävistä vaatimuksista:

> Applikaation ydinkoodille on ensisijaisen tärkeää, että poistetut tietueet ovat poissa. Eli että ne eivät väärään aikaan yhtäkkiä hyppää silmille.

> Business Intelligence väelle taas on tärkeää, että jos jokin tietue on *kerran asuttanut applikaation tietokantaa*, on tuosta tietueesta *ikuinen jälki jossain*. Tällä tavoin mitään informaatiota ei huku bittiavaruuteen; jokainen tietue on ikuisesti tallessa.

Oleellista on, että yksi ja sama tietokanta voi pehmeää tuhoa hyväksikäyttäen tarjota soveltuvat toiminnallisuudet sekä *applikaatiokoodille* että *Business Intelligence väelle*!

Tämä onnistuu yksinkertaisesti siten, että kaikki applikaatiokoodin haut tietokantaan ajetaan yhdessä kontekstissa, ja kaikki Business Intelligencen haut ajetaan toisessa kontekstissa.

Yksinkertaisemmin: **applikaatiokoodin haut *jättävät huomioimatta pehmeästi tuhotut tietueet*, kun taas Business Intelligence *sisällyttää kaikki tietueet*.**

### Toteutus Laravellissa

Laravellin puolella pehmeän tuhon käyttö on helppoa. Käytännössä käyttöönotossa on vain kaksi vaihetta:

1. Käytettävään tietokantatauluun lisätään "deleted_at"-sarake.
2. Käytettävä malli ottaa käyttöön *SoftDeletes*-toiminnallisuuden.
3. Käytettävän mallin tulee sisältää *dates*-attribuutin.

Kas, näin:

```php

// App\Models\Pankkitili.php

class Pankkitili extends Model
{
  // Vaatimus 2.
  use SoftDeletes;
  // Vaatimus 3.
  protected $dates = ['deleted_at'];

  // jne. muut mallin normimetodit
}

```

Vaatimuksen nro 1 täyttämiseksi meidän tulee luoda taulu, jossa on deleted_at-sarake. Esimerkiksi:

```

// Taulu: pankkitilit


id | tilinumero | omistaja    | created_at | deleted_at
-- | ---------- | ----------- | ---------- | ----------
1  | FI23932118 | 070278-262M | 2016-10-01 | 2016-10-03
2  | FI88001921 | 261188-771S | 2015-02-27 | NULL

```

Ylläolevassa taulussa sarake *deleted_at* kertoo milloin tietue on "tuhottu", eli siis pehmeästi tuhottu. 

Jos sarakkeen arvo on NULL, tietue on vielä olemassa. Tällöin siis sekä Business Intelligence että applikaatiokoodi näkevät tietueen.

Applikaatiokoodin puolella Laravel huolehtii siitä, että pehmeästi tuhotut mallit eivät tule mukaan hakutuloksiin.

```php
// Koska vain malli ID #2 on applikaation näkökulmasta olemassa,
// seuraava haku palauttaa lukumääräksi 1.
Pankkitili::all()->count(); // 1

``` 

> Soft Delete-toiminnallisuus mahdollistaa helposti *append-only*-tyylisen tiedonhallintaratkaisun luomisen. *Append-only*-ratkaisussa mitään tietoa ei koskaan poisteta; vanhentunut tieto yksinkertaisesti merkitään jollain ruksilla (*deleted_at*), joka kertoo, että tietoa ei pidä sisällytettävän applikaation tietokantahakuihin.








