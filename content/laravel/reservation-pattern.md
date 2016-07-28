+++
date = "2016-07-26T11:32:03+03:00"
draft = false
title = "Varausten hallinta tietokannan tasolla"

+++

Otetaan esimerkki seuraavankaltaisesta applikaatiosta. Applikaatio mahdollistaa uhanalaisten sarvikuonojen ostamisen lemmikeiksi. Afrikan salametsästäjät (tai tässä tapauksessa 'salakidnappaajat') tuovat järjestelmään uusia sarvikuonoja, joita eurooppalaiset intoilijat ostavat.

Ostoprosessi ei kuitenkaan ole yksinkertainen. Kukin sarvikuono varataan ostoprosessin ajaksi - mikäli ostoprosessi menee onnistuneesti läpi, sarvikuonopolo rahdataan Eurooppaan uudelle isännälleen. Mikäli ostoprosessi ei mene lävitse, sarvikuono vapautuu takaisin markkinapaikalle.

Tietokanta voisi olla esim. tämän kaltainen:

```
// Sarvikuonot-taulu

| id |   ostaja  | hinta | tullut_myyntiin |
| -- | --------- | ----- | --------------- |
| 1  |   NULL    |  25   |   1.6.2016      |
| 2  |   NULL    |  32   |   3.6.2016      |
| 3  |   2       |  26   |   4.6.2016      |

```

```
// Ostajat-taulu

| id |   nimi    |  maa  |   email         |
| -- | --------- | ----- | --------------- |
| 1  |   Pekka   |  FI   |   pekka@24.fi   |
| 2  |   Mikko   |  FI   |   m85@gmail.com |

```

Järjestelmä toimii ylläolevia tauluja hyödyntäen. Ostaja kirjautuu sarvikuonojen markkinapaikalle - se miten tuo kirjautuminen tapahtuu ei ole tässä esimerkissä oleellista. Sen jälkeen hän selaa ostettavissa olevia sarvikuonoja. *Sarvikuonot*-taulusta saadaan helposti haettu vapaana (ostomielessä) olevat kuonokkaat - vapaalla sarvikuonolla *ostaja*-sarake on tyhjä (NULL).

``` SELECT * from Sarvikuonot WHERE ostaja=NULL ```

Mutta kuten alussa mainitsimme, haluamme myös sallia sarvikuonon *varauksen* itse ostoprosessin ajaksi. 

Miksi tämä on tärkeää? **Siksi, että muuten saattaisi hyvinkin käydä niin, että useampi henkilö yrittäisi samanaikaisesti ostaa samaa kuonokasta.**

Ongelman ydin on siinä, että sarvikuonon *ostaja*-sarake päivitetään vasta aivan ostoprosessin lopussa. Tämä on loogista siinä mielessä, että ostos ei ole vahvistettu kuin vasta prosessin lopussa. 

Mutta järjestelmän muiden asiakkaiden tämä ei ole optimaalista. On varsin ikävää jos joku heistä aloittaa oman ostoprosessinsa sarvikuonosta, jota sinä olet parhaillaan maksamassa Nordean nettipankissa. Kun maksusi menee läpi, tuo toinen asiakas on umpikujassa. 

Hänen kannaltaan on varsin ikävää, mikäli ostos epäonnistuu aivan kalkkiviivoilla. Vähemmästäkin ihminen repii juurikasvunsa.

Ongelman ydin siis on, että ostoprosessilla on **alku** ja **loppu**. Mikäli ostoprosessi olisi pistemäinen tapahtuma, mitään ongelmaa ei olisi. Varaus ja osto tapahtuisivat tismalleen samalla ajan hetkellä, joten tarve varauksen olemassaololle poistuisi.

Esimerkkinä tälläisestä pistetapahtumasta on ruokakaupassa käynti. Sanotaan, että maitohyllyllä on tasan yksi maitopurkki. Kauppaan saapuu kaksi perhekuntaa maito-ostoksille. 

Kumpi poppoo tuon maitopurtilon saa mukaansa? Kumpi ensimmäisenä sen hyllyltä nappaa. Voittaja vie maidon. Seuraava käsi hapuilee pelkkää tyhjää ilmaa. Tyhjyyttä kohti kurotteleva kyllä varsin nopeasti hoksaa, että maito meni jo, joten hänen ei tarvitse jatkaa ostoprosessiaan eteenpäin. Ainoastaan voittaja kävelee kohti kassapistettä.

Ratkaiskaamme sarvikuonojen *varaus vs. osto* -ongelma lisäämällä erillinen varaus-sarake tietokantatauluun.

### Varaus ja osto eriteltynä tietokannassa

Uusi Sarvikuonot-taulu näyttää tältä:

```
// Sarvikuonot-taulu

| id |   ostaja  | varaaja | hinta | tullut_myyntiin |
| -- | --------- | ------- | ----- | --------------- |
| 1  |   NULL    |  NULL   |   25  |   1.6.2016      |
| 2  |   NULL    |  1      |   32  |   3.6.2016      |
| 3  |   2       |  2      |   26  |   4.6.2016      |

```

Järjestelmä toimii seuraavanlaisesti: heti kun potentiaalinen kuonoaddikti aloittaa ostoprosessin, hänen ostajanumeronsa lisätään sarvikuonon *varaaja*-kenttään.

Tällä tavoin taulu sisältää tiedon siitä, että kyseistä sarvikuonoa *ollaan parhaillaan ostamassa*. Muille asiakkaille tuota sarvikasta ei tarvitse näyttää listauksissa - heidän kannaltaan sarvikuono on jo myyty. Täten ostettavissa olevat sarvikuonot haetaan tietokantakomennolla:

``` SELECT * from Sarvikuonot WHERE varaaja=NULL ```

Kun varausta tekevä taho sitten lopulta *vahvistaa* kuonokkaan oston, tieto vahvistuksesta päivitetään tauluun *ostaja*-sarakkeeseen. 

```
// Sarvikuonot-taulu
// Sarvikuonon #2 osto vahvistettu ostajalle #1.

| id |   ostaja  | varaaja | hinta | tullut_myyntiin |
| -- | --------- | ------- | ----- | --------------- |
| 1  |   NULL    |  NULL   |   25  |   1.6.2016      |
| 2  |   1       |  1      |   32  |   3.6.2016      |
| 3  |   2       |  2      |   26  |   4.6.2016      |

```
Kaksi eri asiakasta eivät voi enää aloittaa ostoprosessia samasta sarvikuonosta samanaikaisesti. Erinomaista. Onko järjestelmämme nyt täydellinen? 

Ei todellakaan. 

Entä jos ostoprosessi ei menekään läpi? Koska *varaaja*-kenttä on jo täytetty, sarvikuono on muiden asiakkaiden näkökulmasta ostettu. Mutta jos ostoprosessi menee pieleen (ehkä ostaja tulee katumapäälle kesken maksamisen), tuo sarvikuono on ikuisesti jumissa limbossa.

Tarvitsemme siis mekanismin, joka jollain tavoin *vapauttaa* limboon joutuneet sarvikuonot. Mekanismiksi on kaksi hyvää vaihtoehtoa.

### Varausten vapautus - aktiivinen vs. passiivinen

Kerrataan, tietokantataulumme näyttää tältä:

```
// Sarvikuonot-taulu

| id |   ostaja  | varaaja | hinta | tullut_myyntiin |
| -- | --------- | ------- | ----- | --------------- |
| 1  |   NULL    |  NULL   |   25  |   1.6.2016      |
| 2  |   NULL    |  1      |   32  |   3.6.2016      |
| 3  |   2       |  2      |   26  |   4.6.2016      |

```

Ja potentiaalinen ostajamme #1 yllättäen saa aivoinfraktin ja poistuu linjoilta. Hän ei tule enää koskaan ostamaan edes tikkukaramellia saati savannin hyökkäysvaunua. Joten tehtävämme on jollain tavoin *poistaa* varaus sarvikuonolta #2.

#### Aktiivinen poisto

Yksi tapa hoitaa poistot on pitää yllä erillistä *poisto-ohjelmaa*, joka tasaisin väliajoin käy etsimässä + poistamassa *erääntyneitä* varauksia.

Vastaavan kaltainen systeemi on käytössä hotelleissa - jos et ole viimeistään klo 18 vastaanottamassa huoneesi avainta, varauksesi poistetaan asiakaspalvelijan toimesta.

Meidän sarvikuonomarkkinapaikkamme kohdalla loogisinta on kirjata ylös ajankohta, jolloin ostoprosessi alkoi. Vaadimme ostajilta, että heidän tulee suorittaa ostoprosessinsa läpi yhden tunnin aikana. Jos ostoprosessi on epäonnistunut (tai yhä kesken!) tuon yhden tunnin rajapyykin umpeuduttua, varaus poistetaan. 

Muokataan tauluamme, jotta saamme kirjattua ylös varauksen tekoajankohdan:

```
// Sarvikuonot-taulu

| id |   ostaja  | varaaja | hinta | tullut_myyntiin |   varaus_tehty    |
| -- | --------- | ------- | ----- | --------------- | ----------------- |
| 1  |   NULL    |  NULL   |   25  |   1.6.2016      |       NULL        |
| 2  |   NULL    |  1      |   32  |   3.6.2016      | 26.7.16 klo 12.15 |
| 3  |   2       |  2      |   26  |   4.6.2016      | 21.7.16 klo 17.33 |

```

Ohessa pieni skripti, joka pyörii ikäänkuin *taustapalveluna*, käyden 
tasaisin väliajoin poistamassa erääntyneet varaukset:

```php

// Tätä skriptiä kutsutaan esim. käyttöjärjestelmän cron-tabin toimesta.
// Esimerkiksi aina 1 minuutin välein.

// Aloita tietokanta-transaktio
DB::transaction(function() {
	// Rajapyykkinä toimii ajankohta yksi tunti sitten.
	$aikaRajapyykki = Carbon::now()->subHour();

	// Sarvikuonot, jotka ovat erääntyneet, 
	// mutta ei ostettu ('ostaja' on NULL),
	// tyhjennetään varaustiedot
	Sarvikuono
		::where('varaus_tehty', <, $aikaRajapyykki)
		->where('ostaja', NULL)
		->update([
			'varaaja' => NULL,
			'varaus_tehty' => NULL
		]);
});

```

Kun ylläoleva skripti on käynyt poistamassa varauksen tiedot, on sarvikuono #2 jälleen muiden
markkinapaikan kävijöiden nähtävissä.

On myös toinen keino, ns. passiivinen poisto.

#### Passiivinen

Aktiivisessa poistossa meillä on erillinen, itse itseään kontrolloiva/ajastava prosessi (=*käyttöjärjestelmän prosessi*), joka käy tasaisin väliajoin tekemässä poistot. Tuo prosessi elää omaa elämäänsä irrallaan siitä prosessista, joka pyörittää markkinapaikkaamme.

Toinen vaihtoehto on jättää varauksen tiedot maatumaan tietokantaan, ja suorittaa erääntyneiden varausten käsittely suoraan applikaatiomme ydinkoodin puolella.

Tämä on läpeensä sysimusta idea, mutta esimerkin omaisesti esittelen myös sen.

Sarvikuonot-taulu ei muutu mihinkään. Se on edelleen tälläinen: 

```
// Sarvikuonot-taulu

| id |   ostaja  | varaaja | hinta | tullut_myyntiin |   varaus_tehty    |
| -- | --------- | ------- | ----- | --------------- | ----------------- |
| 1  |   NULL    |  NULL   |   25  |   1.6.2016      |       NULL        |
| 2  |   NULL    |  1      |   32  |   3.6.2016      | 26.7.16 klo 12.15 |
| 3  |   2       |  2      |   26  |   4.6.2016      | 21.7.16 klo 17.33 |

```

Erillisen skriptin sijasta meillä on *suoraan applikaatiomme sisuksiin koodattu sopivat reagoinnit* erääntyneisiin varauksiin.

Esimerkiksi ostettavissa olevien kuonojen listaus näyttää nyt tältä:

```php
// SarvikuonoController.php

public function vapaatKuonot() {

	// Vapaat kuonot ovat niitä, joilla pätee joko:
	// 1) 'varaaja' on tyhjä (NULL)
	// 2) 'varaus_tehty' ajankohta yli 1 tunti sitten

	$aikaRajapyykki = Carbon::now()->subHour();
	$vapaat = Sarvikuonot
		::where('varaus_tehty', <, $aikaRajapyykki)
		->where('ostaja', NULL)
		->get();

	return View::make('listaus', compact('vapaat'));	

	
}

```

Muut toiminnot joutuvat nyt turvautumaan vastaavaan logiikkaan. Esimerkiksi ostoprosessin lopussa on vielä kerran varmistettava, että varaus on yhä voimassa. Homma toimii, joten kuten.

Passiivisessa lähestymistavassa on puolensakin. Ylimääräinen prosessin (aktiivinen) olemassaolo lisää järjestelmän kuormitusta ja luo uudenlaisen bugityypin - jos erillinen poistoprosessi kaatuu, varaukset eivät enää eräänny lainkaan. Passiivisessa mallissa tätä riskiä ei ole, sillä "erääntyminen" on koodattu suoraan osaksi ydinalgoritmia.

> Loppukaneetti: passiivisen ja aktiivisen lähestymistavan ero on pohjimmiltaan filosofinen, ja siitä löytyy oikean elämän esimerkkejä kosolti. Otetaan esimerkiksi käteisen rahan käyttö. 

> Yksi nostaa joka kuukausi 100 euroa käteistä, ja sujauttaa setelit lompakkoonsa. Jos hän kuukauden aikana tarvitsee käteistä, hän voi luottaa siihen, että sitä lompakosta löytyy. Hänen ei tarvitse jokaisen kirppariostoksen kohdalla erikseen miettiä asiaa.

> Toinen ei nosta käteistä rahaa, vaan kantaa mukanaan yksinomaan muovirahaa. Hänen ei tarvitse huolehtia kuukausittaisesta Otto-automaatilla vierailusta. Mutta jos hän joskus sattuu tarvitsemaan käteistä, hänellä ei sitä ole. Toisin sanoen, jokaista ostosta tehdessään hänen täytyy erikseen varmistaa, että muoviraha käy.  

> Kyseessä on klassinen tradeoff. 









