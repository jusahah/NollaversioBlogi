+++
date = "2016-09-07T18:03:38+03:00"
draft = false
title = "CQRS ja Laravel"

+++

CQRS (Command Query Responsibility Separation) on vahva keino selkeyttää vastuunjakoa ohjelma-arkkitehtuurissa. 

Sen perusidea on *datan haun* ja *datan muokkauksen* erottaminen toisistaan. Tämä tarkoittaa pohjimmiltaan sitä, että tietty operaatio joko hakee dataa tai muokkaa dataa, mutta **ei koskaan molempia yhtaikaa.**

Kun operaatio on joko *hakubisneksessä* tai *muokkausbisneksessä*, mutta ei ikinä molemmissa, voi operaatio optimoida itsensä valitun "bisneksen" mukaan. Esimerkiksi hakuoperaatio voidaan optimoida käyttämään datalähdettä, jossa data on valmiiksi käsitelty helposti haettavaan muotoon. Muokkausoperaatio puolestaan voi käyttää datalähdettä, jossa data on käsitelty helposti muokattavaan muotoon.

Useimmiten ylläoleva tarkoittaa, että datasta on kaksi kopiota; yksi hakua varten, toinen muokkausta varten. Kopiot pidetään ajan tasalla toisiinsa nähden esimerkiksi rakentamalla hakukopio puhtaalta pöydältä aina kun muokkauskopioon tulee päivitys (=dataa muokataan).

CQRS ei itsessään vaadi datakopioiden olemassaoloa. Haku- ja muokkausoperaatioiden erottelu voidaan suorittaa siten, että molemmat operaatiot käyttävät samaa datalähdettä, mutta vaatimukset esim. virhetilanteiden käsittelylle ovat erilaiset.

### Hakuoperaatio (Query)

Hakuoperaation luonteeseen kuuluu, että haku ei voi mennä *kriittisellä* tavalla pieleen. Kriittisellä tarkoitan tässä, että jos operaatio epäonnistuu, datalähde ei ole moksiskaan. Operaation epäonnistuminen rajoittuu operaatioon itseensä; ympäröivä järjestelmä ei kärsi vaurioita.

Miksi näin? Luonnollisesti ihan siksi, että hakuoperaatio - nimensä mukaisesti - *hakee* tietoa. Tuo haku joko onnistuu tai epäonnistuu. Riippumatta operaation lopputulemasta, datalähde pysyy intaktina. 

### Muokkausoperaatio (Command)

Muokkausoperaation luonteeseen taas kuuluu, että operaatio muokkaa datalähdettä. Esimerkiksi puhelinnumeron muokkaus Facebookin profiilissa on selkeä muokkausoperaatio; uusi puhelinnumero tulee tallentaa jonnekin. Uuden datan tallennus (tai vanhan muokkaus) on operaatio, joka *ei* jätä datalähdettä intaktiin tilaan. 

### Mitä haku vs. muokkaus tarkoittaa koodin tasolla?

Koska hakuoperaatio ei voi edes teoriassa sotkea datalähdettä, tuo operaatio voidaan suorittaa varsin "vapaamielisesti". Toisin sanoen vailla huolen häivää.

Itse tuppaan suorittamaan hakuoperaatiot suoraan Controllerista käsin. Controller on siis perinteisessä web-MVC-arkkitehtuurissa se osanen, joka vastaa sisääntulevan palvelupyynnön käsittelystä ja vastauksen (response) muodostamisesta. 

Ihannearkkitehtuurissa Controller ei ole se paikka, josta tehdään tietokantakutsuja, mutta mikään laki ei estä tietokantakutsuja suorittamasta. Ja koska hakuoperaation kohdalla vaatimukset tietokantakutsuille ovat niin löyhät, voi tuollaisia kutsuja suorittaa huoletta.

```php

// Controller/LainausController.php

public class LainausController {
	
  public function list() {

    // Tietokantakutsu käyttäen Kirja-mallia.
    $kirjat = Kirja::all();


    return view('kirjat.lista', compact('kirjat'));
  }
}

```

Muokkausoperaation kohdalla en lähtökohtaisesti tee tietokantakutsuja Controllerista käsin. Miksi? Koska muokkausoperaation epäonnistuminen voi pahimmillaan tuhota koko tietokannan eheyden. Siksi on tärkeää, että muokkausoperaatio suoritetaan johdonmukaisesti ja turvatoimenpiteet huomioiden.

Turvatoimenpiteellä tarkoitan lähinnä sitä, että moniosainen muokkaus tehdään *tietokantatransaktion* sisällä.

Koska tietty muokkausoperaatio on varsin mahdollista suorittaa useammasta eri Controllerista käsin, on syytä abstraktoida muokkausoperaatio erilliseen apuluokkaan:

```php

// Usecases/Lainaakirja.php

public class LainaaKirja {
	
  public function suorita(User $user, $koodi) {
    // Kirjan lainaus muokkaa sekä kirjan tietoja että lainaajan tietoja.
    // Muokkaukset on syytä tehdä transaktion sisällä jotta ne molemmat
    // joko onnistuvat tai epäonnistuvat. 

    // Missään tapauksessa ei saa käydä niin, että käyttäjä rekisteröi 
    // lainauksen, mutta kirja ei rekisteröi lainaajaa.

    $kirja = Kirja::findOrFail($koodi);
    // Onko kirja saatavilla?
    if ($kirja->parhaillaanLainassa()) {
      throw new KirjaJoLainassa($koodi);
    }

    // Aloitetaan transaktio.
    // Huomattavaa on, että joku toinen saattaa 
    // juuri tässä kohtaa lainata kirjan. Jos näin käy,
    // transaktio epäonnistuu rivillä '$kirja->rekisteroiLainaaja($user)'
    
    DB::transaction(function () use ($user, $kirja) {
      // Jos jompi kumpi epäonnistuu, molemmat epäonnistuvat.
      $user->rekisteroiLainaus($kirja);
      $kirja->rekisteroiLainaaja($user);
    });
  }
}

```

```php

// Controller/LainausController.php

public class LainausController {
	
  public function lainaaKirja($kirjaKoodi) {
    $user = Auth::user();
    (new LainaaKirja)->suorita($user, $kirjaKoodi);

  }
}

```

Ylläolevassa koodissa Controllerin tehtäväksi jää kutsua apuluokkaa, joka suorittaa varsinaisen muokkausoperaation. Tuo apuluokka yksinkertaisesti enkapsuloi sisäänsä tarvittavan logiikan, jonka avulla lainaus suoritetaan.

Ero hakuoperaation ja muokkausoperaation välillä on selkeä: **hakuoperaatio suoritetaan suoraan Controllerista käsin, muokkausoperaatio delegoidaan apuluokalle, joka huolehtii tarvittavista lisätoimenpiteistä (kuten transaktion luonti).** 

> Loppukaneetti: Controllerista käsin tietokantakutsujen tekeminen on useimpien mielestä kyseenalaista. Höpsis. Jos tietokantakutsu on turvallinen ja yksinkertainen, ei ole mitään syytä lähteä abstraktoimaan sitä sen enempää. Kunhan vain kutsut tietokantaa ja sillä sipuli.

> Muokkausoperaation kohdalla tilanne on toinen. Vaativissa applikaatioissa muokkausoperaatiot voivat olla erittäin monimutkaisia ja sisältää monta askelta. Tällöin on tärkeää, että mahdolliset virhetilanteet käsitellään asianmukaisesti. Muokkausoperaation voi suorittaa Controllerista käsin, mutta applikaation rakenteen kannalta on selkeämpää, että elintärkeä ja mutkikas muokkaus eristetään omaksi apuluokakseen. Tämä eristys myös mahdollistaa, että useampi eri Controller voi uudelleenkäyttää tuota muokkauslogiikkaa mikäli tarve niin vaatii.





