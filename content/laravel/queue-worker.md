+++
date = "2016-09-27T09:44:12+03:00"
draft = false
title = "Laravel jonottaa puolestasi"

+++

Yksinkertaiset PHP-applikaatiot toimivat seuraavanlaisesti: 

1. Nettisurffaaja lähettää HTTP-pyynnön.
2. Palvelin ajaa PHP-koodin, joka käsittelee tuon pyynnön.
3. Koodinajon päätteeksi PHP-koodin luoma vastaus palautetaan surffaajalle.

Ylläoleva toimintamalli on ns. request-response -paradigman ytimessä. Yksi osapuoli tekee pyyntöjä (request), toinen osapuoli vastaan niihin pyyntöihin (response). 

Huomionarvoista on, että palvelin ei pysty tekemään pyyntöjä loppukäyttäjän suuntaan - se kun ei tiedä satunnaisen loppukäyttäjän IP-osoitetta. Satunnainen loppukäyttäjä sen sijaan tietää palvelimen IP-osoitteen.

> Loppukäyttäjän web-selain saa IP-osoitteen tietoonsa luonnollisesti domain-nimen kautta. Nettiselain huolehtii esim. "www.iltasanomat.fi"-osoitteen muuntamisesta IP-osoitteeksi. Ihmiskäyttäjän ei tarvitse asialla vaivata päätään.

Request-response -malli sopii erinomaisesti tyypilliseen tietokantapohjaiseen web-applikaatioon. 

Yksi PHP:lle ominainen ongelma kuitenkin nostaa päätään request-response -mallin yhteydessä. Koska vastaus käyttäjälle palautetaan vasta kun PHP-koodi on ajanut itsensä läpi, pitkäkestoinen koodinajo tarkoittaa pitkää odotusaikaa loppukäyttäjän päässä. 

Eli jos koodi suorittaa raskaan operaation, joka kestää viisi sekuntia, ei loppukäyttäjä saa vastausta takaisin kuin aikaisintaan viiden sekunnin kuluttua.

> Ylläoleva on hienoinen yksinkertaistus. Teknisesti on mahdollista kikkailla *flush()*-tyylisillä PHP-funktioilla, mutta tuollainen kikkailu on turhan sotkuista ja tuppaa aiheuttamaan ylläpidollisia ongelmia koodipohjalle pitkällä aikavälillä.

### Jonotus pelastaan päivän

Onneksi apunamme on Laravel-kehyksen erinomainen **Queue**-toiminnallisuus. Käytännössä jonotustoiminnon avulla voimme saavuttaa seuraavanlaisen tavan käsitellä sisääntuleva pyyntö.

1. Palvelupyyntö loppukäyttäjältä tulee sisään.
2. PHP-koodi puskee *työvaiheen* jonoon.
3. Palvelupyynnön vastaus palautetaan loppukäyttäjälle.
4. PHP-koodi aloittaa *työvaiheen* erillisessä prosessissa.
5. ...(aikaa kuluu, työvaihe on hidas suorittaa)
6. Työvaihe valmis.

Ylläoleva mahdollistaa juurikin *raskaiden ja hitaiden* työvaiheiden siirtämisen erillisen käyttöjärjestelmän prosessin suoritettavaksi. Tällä tavoin työvaiheen suoritus ei hidasta vastauksen palauttamista loppukäyttäjälle.

> Periaate on sama kuin loistohotellien concierge-palvelussa. Hotelliasiakas voi antaa conciergen hoidettavaksi vaikkapa varauksen suorittamisen illan teatteriesitykseen.
>
> Tässä tapauksessa asiakas tekee *requestin* concierge-palvelijan suuntaan. Palvelija ottaa pyynnön vastaan ja palauttaa *responsen* välittömästi asiakkaalle. Itse pyynnön toteutuksen - tässä tapauksessa lippujen hankkimisen teatteriin - palvelija hoitaa myöhempänä ajankohtana.
>
> Tärkeintä asiakaspalvelun laadun kannalta on se, että hotelliasiakkaan ei tarvitse toljottaa tyhjän panttina odottamassa että concierge saa teatteriliput ostettua. Sen sijaan hotelliasiakas voi vaikka käydä olusella teatterilippuja odotellessaan.

Vertaa ylläolevaa viiden kohdan listaa vanhaan malliin, jossa jonotusta ei käytetty:

Vanha malli:

1. Palvelupyyntö loppukäyttäjältä tulee sisään.
2. PHP-koodi aloittaa *työvaiheen* samassa prosessissa.
3. ...(aikaa kuluu, työvaihe on hidas suorittaa)
4. Työvaihe valmis.
5. Palvelupyynnön vastaus palautetaan loppukäyttäjälle.

### Käytännön toteutus

Laravel tekee kaikesta liian helppoa. Myös jonottamisesta. Mistä tahansa koodin osasta voimme yksinkertaisesti kutsua globaalia apufunktiota *dispatch*, joka siirtää halutun työvaiheen jonoon:

```php

// Controllers/TilausController.php

public function vastaanotaTilaus(Tilaus $tilaus) {
  
  Log::log("Tilaus vastaanotettu järjestelmään: " . $tilaus->id);
  // Pusketaan uusi työvaihe jonoon.
  dispatch(new IlmoitaTavaranToimittajille($tilaus));

  // Palautetaan vastaus loppukäyttäjälle välittömästi.
  return "Tilaus vastaanotettu - käsittelemme sen piakkoin.";

}

```

Tarvitsemme luonnollisesti *IlmoitaTavaranToimittajille*-luokan. Tämän luokan luoma objekti on lopulta se, joka *erillisessä prosessissa* ajetaan sitten joskus myöhemmin.

```php

// Jobs/IlmoitaTavaranToimittajille.php

class IlmoitaTavaranToimittajille implemets ShouldQueue {

  // Lisätoiminnallisuuksia jotka vaaditaan jonotusta varten.
  // Näistä ei koodarin tarvitse suuremmin välittää, kehys hoitaa.
  use InteractsWithQueue, Queueable, SerializesModels;	

  protected $tilaus;

  public function __construct(Tilaus $tilaus) {
    $this->tilaus = $tilaus;
  }
  // Handle-metodi kutsutaan kehyksen toimesta kun suoritus alkaa!
  public function handle() {
    $tilaus->tavarat->each(function($tavara) {
      $toimittaja = Tavaratoimittaja::haeToimittaja($tavara);
      try {
        $toimittaja->varaaYksiKappale($tavara);
      } catch (EiVarastossa $e) {
      	// Tilausta ei voida täyttää. Tee jotain.
      }
    });

    $tilaus->tavaratVahvistettu();
  }

}

```

Kaiken tämän lisäksi tarvitaan vielä käyttöjärjestelmän prosessi huolehtimaan jonon pyörittämisestä. Jonon käynnistys onnistuu suoraan komentoriviltä:

```
php artisan queue:work

```

Ja siinäpä se onkin. Jonoprosessi automaattisesti monitoroi jonoa, suorittaen sinne lisätyt työvaiheet sopivana ajanhetkenä. 




