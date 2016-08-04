+++
date = "2016-08-04T04:12:30+03:00"
draft = false
title = "Näkymämalli (view model) helpottaa elämää"

+++

Laravellin kaltainen laittoman hieno ohjelmistokehys hoitaa valtavan määrän abstraktioita koodarin puolesta. Sisääntulevan palvelunpyynnön hallinta, tietokantayhteyden hallinta, jne... kaikki on valmiiksi pureskeltu, jotta ohjelmoijaparan ei tarvitse vaivata liiaksi päätään.

Mutta jotkin asiat Laravel jättää ohjelmoijan omien abstraktiovalintojen armoille. Yksi tälläinen on näkymämallin (engl. view model) konsepti.

### Näkymämalli vs. malli?

Ennenkuin keskitymme näkymämalliin, on syytä kerrata ns. "tavallisen mallin" - eli yksinkertaisesti "mallin" - olemassaolon tarkoitus.

*Malli* edustaa yksittäistä domain-tason objektia. Domain-tason objekti on yksinkertaisesti jokin applikaation ydintehtävän kannalta oleellinen objekti; esimerkiksi nettipankin taustajärjestelmässä tuollainen domain-objekti voisi olla *pankkitili*.

Yksittäinen malli on ikäänkuin rakennepiirros (engl. blueprint) tuosta objektista; miltä objekti näyttää, mitä toimintoja se sisältää ja jne. 

> Englanniksi termi "model" tarkoittaa yleensä laajempaa kokonaisuutta kuin yksittäisen objektin rakennepiirrosta. Tässä yhteydessä käytämme käännöstermiä "malli" tarkoittamaan juurikin yksittäisen objektin "mallia", eli rakennepiirrosta.

Tavallinen malli siis edustaa domain-objektia. Se kuvaa yksityiskohtaisesti, kuinka *ympäröivä applikaatio* voi vuorovaikuttaa objektin kanssa. Esimerkiksi *pankkitili*:

```php

// Malli nimeltä "Pankkitili"

// App/Models/Pankkitili.php

class Pankkitili extends Model {
	
	public function haeSaldo() {};
	public function talleta() {};
	public function nosta() {};
	public function tilinOmistaja() {};

	// jne..
}

```

*Malli* on useimmiten paras suunnitella niin, että se on ainoastaan kiinnostunut domain-tason asioista. Mitä tarkoitan tällä? Tarkoitan, että **mallin tulisi olla autuaan tietämätön käyttöliittymän olemassaolosta.**

Jos malli on autuaan tietämätön käyttöliittymän olemassaolosta, malli EI saa sisältää seuraavanlaisia metodeja:

```php

// Malli nimeltä "Pankkitili"

// App/Models/Pankkitili.php

class Pankkitili extends Model {
	// Seuraavat metodit liittyvät käyttöliittymään!
	// Mallin EI tulisi sisältää seuraavia metodeja, sillä ihannearkkitehtuurissa
	// malli ei tiedä käyttöliittymän olemassaolosta hölkäsen pöläystä.

	// Etunimi + sukunimi + asiakasnumero
	public function printtaaOmistajanTiedot() {};
	// Jos negatiivinen saldo, väri = punainen, muuten väri = vihreä
	public function varitaSaldo() {};
}

```

Ylläolevan mallin metodit *printtaaOmistajanTiedot* ja *varitaSaldo* ovat nk. käyttöliittymämetodeja. Tarkoittaen, että niiden olemassaolon syy on yksinomaan tarjota *ihmiskäyttäjälle* monipuolisempi ja visuaalisempi käyttöliittymä.

**Itse applikaation ydintarkoituksen kannalta em. metodeilla ei ole osaa eikä arpaa.** Pankkijärjestelmä itsessään ei ymmärrä miksi ihmeessä negatiivinen saldo tulisi olla punaisella fontilla - vain ihmissilmä ymmärtää punaisen värin tarkoituksen.

Siksi metodit *printtaaOmistajanTiedot* ja *varitaSaldo* on syytä abstraktoida ulos *mallista* ja siirtää *näkymämallin* sisälle.

### Näkymämalli huolehtii datan muokkauksesta ihmissilmälle sopivaksi

Näkymämallin tarkoitus on juurikin ottaa vastuulleen *mallin* sisältämän datan muokkaus ihmissilmälle sopivaan muotoon. Kun näkymämalli vastaa visuaalisesta representaatiosta, varsinainen *malli* voi keskittyä omaan ydintehtäväänsä, eli itse applikaation kanssa vuorovaikutukseen. Eli lyhyesti:

1. *Malli* keskittyy vuorovaikuttamaan applikaation kanssa.
2. *Näkymämalli* keskittyy vuorovaikuttamaan ihmiskäyttäjän kanssa.

Jatketaan pankkiesimerkkiämme. Malli on edelleen tämä:

```php

// Malli nimeltä "Pankkitili"

// App/Models/Pankkitili.php

class Pankkitili extends Model {
	
	public function haeSaldo() {};
	public function talleta() {};
	public function nosta() {};
	public function tilinOmistaja() {};

	// jne..
}

```

Luodaan mallin oheen näkymämalli, joka vastaa mm. saldon värittämisestä punaiseksi mikäli tili paukkuu pakkasella.

Näkymämallin nimeämisessä ohjenuorana on, että mallin nimen perään lisätään "Presenter". Täten pankkitilin näkymämalli on "PankkitiliPresenter".

```php

// Näkymämalli nimeltä "PankkitiliPresenter"

// App/ViewModels/PankkitiliPresenter.php

class PankkitiliPresenter extends Model {
	
	public function printtaaOmistajanTiedot() {}	
	public function varitaSaldo() {}

	// jne..
}

```

### Käytännön toteutus - miten näkymämalli saa tietoonsa mallin?

Yllä loimme pohjustukset kahdelle eri konseptille - malli ja näkymämalli. Loimme mallin nimeltä **Pankkitili**, ja tuota mallia vastaavan *näkymämallin* nimeltä **PankkitiliPresenter**.

Seuraavaksi näkymämalli tulee kytkeä yhteen mallin kanssa. **Kytkentä on yhdensuuntainen.** Pankkitilin ei tarvitse tietää PankkitiliPresenterin olemassaolosta, mutta PankkitiliPresenterin tulee saada käyttöönsä Pankkitili. 

Jos PankkitiliPresenter ei tiedä Pankkitilin olemassaolosta mitään, se ei myöskään voi kutsua Pankkitili-objektin metodeja. Ja PankkitiliPresenterin on pakko kutsua Pankkitilin metodeja, sillä esimerkiksi saldon väritys onnistuu vain jos tuo saldosumma on tiedossa.

Yksi hyvä tapa hoitaa kytkös on seuraava:

```php

// Malli nimeltä "Pankkitili"

// App/Models/Pankkitili.php

class Pankkitili extends Model {
	
	public function haeSaldo() {};
	public function talleta() {};
	public function nosta() {};
	public function tilinOmistaja() {};

	public function present() {
		return new PankkitiliPresenter($this);
	}
}

```
```php

class PankkitiliPresenter extends Model {

	protected $tili;

	public function __construct(Pankkitili $tili) {
		$this->tili = $tili;
	}
	public function printtaaOmistajanTiedot() {
		// Varmista että nimet isolla alkukirjaimella.
		return $this->capitalize($this->tili->tilinOmistaja());
	}	
	public function varitaSaldo() {
		$saldo = $this->tili->haeSaldo();
		// Lisää väritys
		if ($saldo < 0) {
			return '<div class="red">' . $saldo . '</div>';
		} else {
			return '<div class="green">' . $saldo . '</div>';
		}

	}

	// jne..
}

```

Ylläolevassa arkkitehtuurissa Pankkitili-malli sisältää erillisen *present*-metodin. Tuo metodi palauttaa PankkitiliPresenter-objektin kutsujan käyttöön. 

PankkitiliPresenter-objektia käyttämällä kutsuja saa luotua helposti HTML-koodin pätkän, joka sisältää saldosumman ja tarvittavan HTML-syntaksin tuon saldosumman värittämiseksi joko vihreäksi tai punaiseksi.

On huomattavaa, että esimerkiksi *varitaSaldo*-metodissa PankkitiliPresenterin tulee kutsua Pankkitilin metodia. Tästä syystä PankkitiliPresenterillä tulee olla aina käytettävissään Pankkitili-objekti.

Valitsemassamme ratkaisussa tuo Pankkitili-objekti annetaan parametrinä PankkitiliPresenterin konstruktoriin. 

### Näkymämallin käyttö

Ylläolevan ratkaisumme käyttö on helppoa. Aina kun saatavillamme on Pankkitili, on saatavillamme myös PankkitiliPresenter, sillä Pankkitili-malli sisältää metodi PankkitiliPresenter-objektin luomiseen.

```php

// views/Saldoikkuna.php

// Oletetaan, että käytössämme on Pankkitili-objekti nimeltä $pankkitili.
// Esim. Controllerissa olemme avanneet näkymän kutsulla: 
// view('saldoikkuna')->with('pankkitili', $pankkitili);

<h1>Tämän hetkinen saldosi</h1>
<p><?php echo $pankkitili->present()->varitaSaldo() ;?></p>

```

Ylläoleva koodi toimii mainiosti. Aina kun haluamme kutsua jotain PankkitiliPresenterin metodia, käytämme muotoa:

```php

$pankkitili->present()->metodi();

```

> Loppukaneetti: näkymämallin käytön koko ydinajatus on, että applikaation kannalta oleelliset toiminnot ja käyttöliittymän kannalta oleelliset toiminnot erotetaan toisistaan. Applikaatiota ei kiinnosta se, millä värisävylle negatiivinen saldo näytetään ihmissilmälle. Ihmissilmää tuo asia kiinnostaa.









