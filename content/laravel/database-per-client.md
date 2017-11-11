+++
date = "2017-11-11T07:47:42+02:00"
draft = false
title = "Tietokanta per asiakas"

+++

Tyypillinen pieni/keskisuuri Laravel-applikaatio rakentuu yhden tietokannan päälle. Tuo yksi tietokanta sisältää kaiken datan, jota Laravel-sovellus tallentaa/käyttää. 

Tyypillinen web-applikaatio kuitenkin tarjoaa käyttöoikeuden usealle erilliselle käyttäjälle/loppuasiakkaalle. Varsin yleinen tapaus vieläpä on, että kunkin loppuasiakkaan data elää täysin erillään muiden asiakkaiden datasta. Tällöin jokainen asiakas muodostaa oman universuminsa tietokannan sisälle; useimmiten tämä "privaatti maailma" rakennetaan käyttämällä avokätisesti *viiteavaimia* (foreign key). 

Näitä viiteavaimia sitten ripotellaan ympäri tietokannan rakennetta; lähes jokainen tietokantataulu sisältää sarakkeen, jossa viiteavain määrittelee kenen asiakkaan universumiin kyseinen tietue (rivi) kuuluu.

Toinen vaihtoehto on tehdä asiat konseptuaalisesti yksinkertaisemmin; **annetaan jokaiselle asiakkaalle oma tietokanta!** 

Tällöin viiteavaimia ei tarvita, sillä yksittäisessä tietokannassa on aina vain yhden asiakkaan data. 

> Tietokannan luominen jokaiselle asiakkaalle erikseen sisältää paljon hyviä puolia. Mutta kuten aina, trade-off on olemassa. Hyvä kokonaiskatsaus näihin kahteen eriävään strategiaan löytyy esim.: https://stackoverflow.com/questions/18910519/pros-cons-using-multiple-databases-vs-using-single-database

Tutkitaan seuraavaksi, miten Laravel-applikaatio voidaan rakentaa käyttämään *yhtä tietokantaa per asiakas*.

### Tietokanta === subdomain

Yksi erinomainen tapa mahdollistaa usean tietokannan käyttö järkevästi on kytkeä looginen yhtäläisyys *tietokannan* ja *alidomainin* välille.

Tämä tarkoittaa, että esimerkiksi domain *nokia.app.fi* valitsee käyttöönsä Nokia-tietokannan, ja *atria.app.fi* valitsee käyttöönsä Atria-tietokannan. Molemmat asiakkaat (Nokia ja Atria) jakavat yhteisen Laravel-applikaatiopalvelimen, ja mahdollisesti myös fyysisen tietokantapalvelimen, mutta Laravel valitsee *kunkin sisääntulevan palvelupyynnön yhteydessä* sopivan tietokannan dynaamisesti.

Koodirajapinnan tasolla tämä voisi näyttää kutakuinkin tältä:

```

// routes/api.php

Route::group(['domain' => '{company}.' . ENV('APP_DOMAIN')], function() {
	
	Route::get('/users', 'UserController@all');

}); 

```

Route-tiedostomme siis ottaa alidomainin sisään dynaamisena muuttujana. Tuota muuttujaa voidaan käyttää Controllerin puolella:

```
// Controller/UserController.php

class UserController extends Controller
{

    public function index(Request $request, $company) {

    	if ($company === 'nokia') {
    		// Käytä Nokian tietokantaa
    	} else if ($company === 'atria') {
    		// Käytä Atrian tietokantaa.
    	}

    	// Tässä kohtaa Eloquent on kytketty oikeaan tietokantaan.

    	return User::all();
    }


 }

 ```

 Ikävää ylläolevassa on tietenkin se, että meidän tarvitsee jokaikisessä Controllerissa tehdä tietokannan valinta. Helpompaa on siirtää tietokannan dynaaminen valinta middlewareen:

 ```

// Http/Kernel.php

class Kernel extends HttpKernel
{
	//... muita asetuksia...

    protected $middlewareGroups = [
        'api' => [
            \App\Http\Middleware\ValitseTietokanta::class, 
            'throttle:60,1',
            'bindings',
        ]
    ]; 

    // ... muita asetuksia...
}

```

```

// Middleware/ValitseTietokanta.php

class ValitseTietokanta
{

    public function handle($request, Closure $next)
    {
        $company = $request->route('company');

    	// Määritämme globaalin vakion, jota voidaan käyttää
    	// missä tahansa applikaatiokoodissa. Tällä tavoin
    	// mikä tahansa funktio saa tarvittaessa tietoonsa minkä
    	// asiakkaan kontekstissa se suoritetaan.
        if (!defined('COMPANY_SUBDOMAIN')) {
            define('COMPANY_SUBDOMAIN', $company);
        }

        // Ylikirjoita default-config.
        \Config::set('database.connections.mysql.database', 'appi_db_' . $company);
        // Ota uusi tietokantayhteys
        \DB::reconnect('mysql');

        return $next($request);
    }
}

```

Ylläoleva tekee tietokannan valinnan jokaiselle API-routelle. Se ei tee suuremmin virhetilanteiden hallintaa. On mahdollista, että tietokantaa ei ole olemassa. Tällöin myöskään alidomainia ei pitäisi olla olemassa, eli ympäröivän www-palvelimen tulisi estää sisääntuleva yhteys.

Ylläoleva tarvitsee vielä config-tiedostoon lisäyksen.

```
// config/database.php

return [

	// muita asetuksia

    'connections' => [


        'mysql' => [
            'driver' => 'mysql',
            'host' => env('DB_HOST', 'localhost'),
            'port' => env('DB_PORT', '3306'),
            // Tämä attribuutti korvataan middlewaressa.
            'database' => env('DB_DATABASE', 'appi_db_default'),
            'username' => env('DB_USERNAME', 'forge'),
            'password' => env('DB_PASSWORD', ''),
            'charset' => 'utf8',
            'collation' => 'utf8_unicode_ci',
            'prefix' => '',
            'strict' => true,
            'engine' => null,
        ],
];

```

Homma toimii siten, että middlewaressa ylikirjoitamme *database*-attribuutin mysql-configista. Ylikirjoituksen jälkeen kutsumme *DB::reconnect()*, joka lataa (muunnetun) configin uusiksi ja ottaa uuden tietokantayhteyden.

> Ylläoleva koodiesimerkki tekee ikävän oletuksen siitä, että kaikki asiakkaat käyttävät tietokannassa samaa salasanaa, tunnusta ja hostia. Tämä estää tietokannan siirtämisen ulkoiselle palvelimelle, esimerkiksi asiakasyrityksen omalle palvelimelle.
>
>
>
> Äärimmäinen dynaamisuus on saavutettavissa siten, että luomme erillisen taulun *"_asiakkaat"*, jonne tallennamme tiedot kunkin asiakkaan tietokannasta. Tämän jälkeen middlewaressa asetamme kaikki mysql-configin attribuutit asiakastietokannan asetusten mukaisiksi.
>
>
>
> Mutta minne luomme "_asiakkaat"-taulun? Nokian vai Atrian tietokantaan? Ei kumpaankaan. Loogisin paikka on erillinen *admin-tietokanta*, joka on rakenteeltaan erilainen kuin asiakkaiden tietokannat. Toinen vaihtoehto on käyttää .env-tiedostoa, ja tunkea kaikkien asiakkaiden tietokantatiedot sinne. Tärkeintä on, että asiakkaiden tietoja ei päästetä versiohallinnan piiriin, eli config/database -tiedostoon niitä EI saa laittaa.
