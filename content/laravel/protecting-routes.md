+++
date = "2016-07-20T20:42:23+03:00"
draft = false
title = "Suojaa tuloväylät - mutta miten?"

+++

Tyypillinen web-applikaatio ottaa vastaan monenlaista palvelupyyntöä. Osa pyynnöistä tulee rekisteröityneiltä käyttäjiltä, osa vierailta, osa hakkereilta, osa sisältää dataa, osa ei.

Kaiken tämän keskellä applikaatio tulisi kehittää niin, että jokainen sisääntuloväylä on suojattu asianmukaisesti. Eli portti on kunnossa ja pysyy kiinni esim. SQL-injektioille.

Helppo, nopea tapa huolehtia suojauksesta on jokaisen tuloväylän portilla tarkistaa, että asianmukaiset paperit ovat mukana:

### versio 1

```php

// AdminController.php

public function store(Request $request) {
	$this->tarkistaAdminOikeudet();

	// Kaikki hyvin, käsittele request
}

public function index(Request $request) {
	$this->tarkistaAdminOikeudet();

	// Kaikki hyvin, käsittele request
}

public function list(Request $request) {
	$this->tarkistaAdminOikeudet();

	// Kaikki hyvin, käsittele request
}

private function tarkistaAdminOikeudet() {
	if (Auth::user()->isNotAdmin()) {
		throw new Exception("Admin-oikeudet puuttuvat!");
	}
}

```

Ylläolevasta heti nähdään, että jotain on pielessä. Sama admin-tarkistus joudutaan tekemään kolmesti eri kohdissa.

Huomattavasti paremman ratkaisun tarjoaa konstruktori-metodi, joka mahdollistaa kaikille public-metodeille *yhteisen* tarkistuksen määrittämisen. Eli:

### versio 2

```php

// AdminController.php

public function __construct(Request $request) {

	$this->tarkistaAdminOikeudet();
}

public function store(Request $request) {

	// Kaikki hyvin, käsittele request
}

public function index(Request $request) {

	// Kaikki hyvin, käsittele request
}

public function list(Request $request) {

	// Kaikki hyvin, käsittele request
}

private function tarkistaAdminOikeudet() {
	if (Auth::user()->isNotAdmin()) {
		throw new Exception("Admin-oikeudet puuttuvat!");
	}
}

```

Kuten huomaamme, duplikaatio poistui. Tarkistus tehdään vain yhdessä pisteessä.

Mutta miksi turhaan edes keksiä pyörää uudestaan? Laravel tarjoaa "Middleware"-nimisen konseptin käyttöömme. Middleware on käytännössä yksi ylimääräinen kerros internetin ja applikaatiosi välissä. Tuo ylimääräinen "rasvakerros" soveltuu hyvin admin-tarkistuksen suorituspisteeksi.

### versio 3

// Middleware/TarkistaAdmin.php

```php

class TarkistaAdmin
{

    public function handle($request, Closure $next)
    {
        if (Auth::user()->isNotAdmin()) {
        	// Ohjataan käyttäjä kirjautumissivulle.
            return redirect('kirjaudu_sisaan');
        }

        // Kaikki ok!
        // Muu applikaatio voi luottaa että käyttäjällä on tarvittavat oikeudet!

        return $next($request);
    }

}

```

```php

// AdminController.php

public function __construct(Request $request) {

	$this->middleware('tarkistaAdmin')
}

public function store(Request $request) {

	// Kaikki hyvin, käsittele request
}

public function index(Request $request) {

	// Kaikki hyvin, käsittele request
}

public function list(Request $request) {

	// Kaikki hyvin, käsittele request
}

```

Tämä on jo varsin pätevä ratkaisu. Ensinnäkin admin-tarkistuksen logiikka elää nyt poissa AdminControllerista. Tämä on ihan hyvä, sillä oletettavasti joku muukin komponentti applikaatiossa on kiinnostunut tekemään admin-tarkistuksia. Kun admin-tarkistus elää middleware-kerroksessa, se on kaikkien applikaation osasten käytettävissä.

Noin muutenkin on järkevintä tsekata admin-oikeudet mahdollisimman aikaisin. Tilanne on vastaava kuin lentokentällä - turvatarkastus tapahtuu *keskitetysti* ennen lähtöporteille siirtymistä. Millainen sotku syntyisi, jos turvatarkastus järjestettäisiin kunkin lähtöportin edessä erikseen? Aikamoinen. 

Sama konsepti pätee web-applikaatioon - mitä aiemmin tarkastukset tehdään, sitä parempi. Aikainen tarkastus selkeyttää kaikkien osapuolten toimintaa. Lentokentälläkin on helpompi kuljeskella, kun turvatarkastus on rajattu tietylle alueelle.

Laravellin middleware-konsepti lisää myös uusia mahdollisuuksia valikoimaamme. Voimme esimerkiksi määrittää suoraan konstruktorissa, mille kaikille sisääntuloväylille (eli public metodeille) haluamme middleware-suojauksen pätevän.

```php

// AdminController.php

public function __construct(Request $request) {

	$this->middleware('tarkistaAdmin', ['only' => [
		'store',
		'update'
	]]);
}

public function store(Request $request) {

	// SUOJATTU!
}

public function update(Request $request) {

	// SUOJATTU!
}

public function index(Request $request) {

	// EI SUOJATTU!
}

public function list(Request $request) {

	// EI SUOJATTU!
}

```

Kätevää, varsin kätevää. Esimerkin *only*-attribuutin lisäksi meillä on käytössämme *except*-attribuutti, joka
toimii nimensä mukaisesti - se suojaa kaikki muut väylät paitsi erikseen *except*:in perässä määritellyt.