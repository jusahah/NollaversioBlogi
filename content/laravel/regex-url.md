+++
date = "2016-10-25T17:01:58+03:00"
draft = false
title = "Regex ja URL"

+++

Tarvitsin tänään Laravel-projektia koodatessani toiminnallisuutta, joka tsekkaa onko annettu tekstijono validi www-osoite. 

Laravel itsessään tarjoaa tälläisen tsekkauksen, mutta ikäväksekseni Laravel on varsin tiukkapipoinen: se ei hyväksy osoitetta *www.nokia.fi*, sillä osoitteen alusta puuttuu "http://"-alkuliite. Omassa projektissani en halua kiusata käyttäjiä mokoman http-alkuosan kirjoituspakolla, joten jouduin hylkäämään Laravellin tsekkarin.

Netistä löytyi varsin kiva regex (regular expression) hoitamaan URL:n tarkistus:

```
/^(https?:\/\/)?([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$/;

```

Niin että mitäs tuo sotku tarkoittaakaan? Itselläni ei ole juuri mitään hajua. Tai ei ollut ennen tätä päivää. Olen aina suosiolla ulkoistanut Regex-lauseiden muodostamisen Stack Overflown kaltaisille nettipalveluille. 

Nyt kuitenkin selvitin asiaa, vaikka vain tätä blogipostausta varten. Ja toisaalta onhan se hyvä osata jotain.

### Miten tuo tekstihirviö tarkistaa URL-osoitteen?

Ylläoleva regex tosiaan varmistaa, että sille annettu tekstijono on toimiva www-osoite eli URL. Miten ihmeessä? Tarkastellaan tekstimonsteria pala kerrallaan.

Koko monsteri oli siis:

```
/^(https?:\/\/)?([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$/

```

**Ensimmäinen kenoviiva**

```
/
```

Ensimmäinen kenoviiva avaa regex-ekspressionin.

**Http-alkuliitteen tarkistus**

```
^(https?:\/\/)?

```

Tässä päästään itse asiaa. Tämä osuus tarkistaa, että URL-osoittessa joko on *http://*-alkuliite, *https://*-alkuliite, tai ei alkuliitettä ollenkaan. Jokin noista kolmesta vaihtoehdosta tulee olla voimassa; muussa tapauksessa kyseessä ei ole URL ja regex loppuu siihen.

Hiukka merkistöstä ylläolevan regex-palasen suhteen.

1. Sulut ympäröivät tarkistettavan konseptin.
2. Kysymysmerkki merkitsee, että sitä edeltävä konsepti esiintyy joko *kerran* tai *ei lainkaan*. Esimerkiksi *s?* tarkoittaa, että osuuden *http* jälkeen tulee kirjain *s* joko kerran tai ei kertaakaan.

Mennään eteenpäin.

**Host-nimen tarkistus**

Seuraava palanen tarkistaa, että domainin host-osuus sisältää laillisia merkkejä. Host-osuus on domainissa se nimi, joka tulee ennen maatunnusta. Esimerkiksi domainissa *www.nokia.fi*, host-nimi on *nokia*.

```
([\da-z\.-]+)

```

Ylläoleva siis tarkistaa, että host-nimi sisältää numeroita (*\\d*) ja/tai laillisia kirjaimia (*a-z*). Ääkkösiä ei saa sisältää, sillä *a-z* sisältää vain englannin kielen kirjaimet. 

Lisäksi *a-z* tarkoittaa, että vain pieniä kirjaimia saa olla mukana. Isot kirjaimet eivät käy.

Tämä jälkeen tulee kohta  '*\\.-*', joka tarkoittaa, että host-nimi saa sisältää myös pisteitä ja väliviivoja. Muut merkit eivät ole sallittuja.

Mitä nuo hakasulut tekevät tuossa? En tiedä. Jotain kapturoinnista internet-haun mukaan, mutta en täysin ymmärtänyt mitä kapturoinnilla (siis "kiinniotolla" suomeksi) tarkoitetaan tässä kontekstissa.

Tärkeä sen sijaan on plus-merkki juuri ennen viimeistä sulkua. Se tarkoittaa, että koko aiempi litanja voi laillisesti toistua yhden tai useamman kerran. Ei siis nolla kertaa - vähintään yksi kerta tarvitaan.

Tämä tarkoittaa, että seuraavat host-nimet ovat laillisia:

1. *'nokia'*
2. *'nokia-puhelin007'*
3. *'nokia.puhelin007.ollila'*

Ylläolevat noudattavat sääntöjämme. Sen sijaan seuraavat host-nimet ovat laittomia:

1. *'Nokia'* (iso kirjain on laiton!)
2. *'huhtamäki'* (ääkkönen on laiton!)
3. ' ' (tyhjä merkkijono on laiton!)

Mennään eteenpäin.

**Pakollinen piste**

```
\.

```

Tämä on hyvin yksinkertainen palanen; vaadimme, että host-nimen jälkeen tulee yksi piste. Tämä piste vastaa pistettä host-nimen ja maatunnuksen välissä, esim. "nokia.fi".

**Maatunnus min. 2 merkkiä, max. 6 merkkiä**

Seuraavana tulee maatunnus, eli siis se *com*, *fi*, *org* tjms. 

```
([a-z\.]{2,6})

```

Ylläoleva vaatimus määrittää, että maatunnus voi sisältää vain *a-z* -kirjaimet. Se siis EI voi sisältää numeroita. Ja sitten tulee mielenkiintoinen: *{2,6}* tarkoittaa, että maatunnuksen pituus voi olla 2-6 merkkiä.

Eli *fi* menee alarajalta nipin napin läpi, se kun on kaksi merkkiä. Maatunnus *finland* ei menisi läpi, koska se on 7 merkkiä pitkä.

**Loppuosuus eli mahdolliset URI-päätteet**

Loppuosuus on aika sotku. 

```
([\/\w \.-]*)*\/?$/

```

Ylläoleva on tarkoitettu varmistamaan, ettei URL-osoitteen hakemistopolku sisällä laittomuuksia. Hakemistopolku on siis se loppuosuus, joka määrittää tarkan resurssin, joka haetaan.

Esimerkiksi URL-tekstijonossa *www.nokia.fi/mobiili/ollila.jpg*, tuo hakemistopolun osuus on */mobiili/ollila.jpg*.

Ylläoleva regex aluksi varmistaa, että loppuosuus alkaa kenoviivalla. 

Sen jälkeen tulee merkki *\\w*, joka on mielenkiintoinen. Tuo tarkoittaa, että mikä tahansa alfanumeerinen merkki kelpaa. Eli siis pienet kirjaimet, isot kirjaimet ja numerot, ja vielä erikoismerkki *_* (alaviiva).

Sitten tulee merkki *\**. Se tarkoittaa, että koko aiempi litanja - joka on hakasulkujen sisällä - toistuu joko nolla kertaa, yhden kerran tai useammin. Eli siis kuinka monesti tahansa - kaikki käy.

Loppuosuus *\*\/?$/* merkkaa yksinkertaisesti, että syöte päättyy. Dollarimerkki käskyttää regex-moottoria ymmärtämään, että tekstijonon tulisi olla loppu tässä kohtaa.

Aika monimutkaista. 










