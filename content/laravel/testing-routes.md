+++
date = "2016-07-27T06:11:10+03:00"
draft = false
title = "Käyttöliittymän testaus (automatisointi)"

+++

Perinteisen web-applikaation peruspointti on tarjota käyttäjilleen mahdollisuus vuorovaikuttaa itsensä (*esim. kerätä tänään dataa talteen, ja hyödyntää kerättyä dataa huomenna*) ja/tai toistensa kanssa web-applikaation kautta. 

Jotta tämä vuorovaikutus onnistuisi, täytyy web-applikaation tarjota jonkinmoinen käyttöliittymä. 

Tyypillisessä tietokantapohjaisessa web-sovelluksesta tuo käyttöliittymä on HTML-sivu, joka sisältää linkit applikaation tarjoamiin toiminnallisuuksiin. Linkkejä klikkailemalla voi vuorovaikuttaa applikaation kanssa. Malliesimerkki tälläisestä applikaatiosta on vaikkapa Wikipedia.

Vuorovaikutus ihmiskäyttäjän kanssa on monen web-sovelluksen keskeisin huolenaihe. Toki on erikseen web-sovellukset, jotka *eivät* vuorovaikuta ihmiskäyttäjän kanssa, vaan käyvät tiedonvaihtoa toisen web-sovelluksen kanssa. Malliesimerkki tälläisestä applikaatiosta on osakepörssin rajapinta. Tuo rajapinta käy keskustelua muiden applikaatioiden - mm. uutissivustojen pörssikurssien päivityksestä vastaavien ohjelmien - kanssa.

### Voiko tietokoneohjelma simuloida ihmiskäyttäjää?

Koska useimmilla applikaatioilla kommunikaatio ihmiskäyttäjän kanssa on keskiössä, on syytä kyetä varmistamaan, että käyttöliittymä toimii kuin vettä vain. Laravellin tapauksessa tämä varmistus tarkoittaa, että kukin HTML-sivu - joka siis edustaa tiettyä käyttöliittymän osaa - sisältää tarvittavat toiminnot, jotta applikaation käyttö on sujuvaa.

Mutta miten varmistua siitä, että käyttöliittymä tarjoaa tarvittavat toiminnot? Yksi tapa on silmämääräisesti selata käyttöliittymää. Ihmisaivot tekevät automaattisesti näin saapuessaan esim. Wikipedian etusivulle - luomme ikäänkuin *mentaalisen kartan* kaikista applikaation tarjoamista mahdollisuuksista.

Homman voi tietenkin myös automatisoida, ja se kannattaa automatisoida. **Sen sijaan että silmämääräisesti tarkistaisimme käyttöliittymän, annetaan erillisen tietokoneohjelman tarkistaa käyttöliittymä.**

Tätä on automatisoitu käyttöliittymätestaus.

### Mitä testataan ja miten?

Käyttöliittymätestauksessa pääpaino on varmistaa, että applikaation käyttöliittymä tarjoaa tarvittavat toiminnot, jotta applikaation käyttö on sujuvaa. 

Tyypillisen web-applikaation käyttöliittymä koostuu isosta kasasta *HTML-sivuja*. Täten web-applikaation kohdalla käyttöliittymätestaus tarkoittaa kutakuinkin HTML-sivujen sisällön testausta. Eli varmistetaan, että kukin HTML-sivu sisältää tarvittavat *toiminnot, tiedot ja ohjeet*, jotta vuorovaikutus applikaation kanssa onnistuu odotetusti.

Kirjoitetaan ensimmäinen testi. Oletetaan, että olemme rakentamassa uutta Wikipediaa. Wikipedian keskiössä on *artikkeli*, joten on luontevaa aloittaa ohjelmoimalla tarvittavat toiminnot yksittäisen artikkelin lukemista ja ylläpitoa varten. 

Mitä toimintoja haluamme kytkeä osaksi konseptia nimeltä *artikkeli*? Ainakin mahdollisuuden *lukea* artikkeli. Lisäksi olisi kiva voida *muokata* artikkelia. Aloitetaan näistä kahdesta.

Entä millainen käyttöliittymän tulee olla, jotta *lukeminen* ja *muokkaaminen* onnistuvat?

Lukemista varten tarvitsemme jotain mitä lukea. Eli artikkelin sisällön tulee olla ihmissilmin nähtävillä.

Muokkausta varten tarvitsemme jonkinlaisen linkin tai nappulan, jonka kautta siirtyä artikkelin muokkaustilaan.

Testimme näyttää yleisilmeeltään tältä.


```php

use \Integrated\Extensions\Laravel as IntegrationTest;

class ArtikkeliTesti extends IntegrationTest {

	public function testaaLukuMahdollisuus {

	}

	public function testaaMuokkausMahdollisuus {


	}
	

}

```

Seuraavaksi on syytä miettiä, miten nuo testit suoritetaan. Käyttöliittymätestauksen koko pointti on, että testaus suoritetaan ikäänkuin ihmiskäyttäjä toimisi testaajana. Oikeasti tuon testauksen tekee tietokoneohjelma, mutta tietokoneohjelma simuloi ihmisen toimintaa.

Paras tapa suorittaa käyttöliittymätestaus on siis toistaa niitä toimintoja, joita oikea ihmiskäyttäjä tekisi mikäli käyttäisi applikaatiota.

```php

// ArtikkeliTesti.php

use \Integrated\Extensions\Laravel as IntegrationTest;

class ArtikkeliTesti extends IntegrationTest {

	public function testaaLukuMahdollisuus {
			// Simuloidaan ihmiskäyttäjää
			

			// Kirjoita selaimen osoiteriville tietyn artikkelin www-osoite
			$this->visit('/artikkelit/seppo-raty');

			// Nyt edessämme pitäisi olla Seppo Rädystä kertova artikkeli
			$this->seePageIs('seppo-raty');

			// Artikkelin tulisi mainita hänen urheilulajinsa...
			$this->see('keihäänheitto');

			// ... ja muutama kuolematon sitaatti
			$this->see('Saksa on paska maa');
			$this->see('Vittuillakseni heilutin');

			// Jos kaikki ylläolevat ehdot täyttyvät, voimme
			// luottaa, että kyseessä on Rädyn wikipedia-artikkeli.

	}

	public function testaaMuokkausMahdollisuus {
			// Simuloidaan ihmiskäyttäjää
			
			// Kirjoita selaimen osoiteriville tietyn artikkelin www-osoite
			$this->visit('/artikkelit/seppo-raty');

			// Emme ole kiinnostuneita artikkelin sisällöstä, mutta
			// olemme kiinnostuneita muokkausmahdollisuudesta.

			// Varmistetaan, että "Muokkaa"-nappula on olemassa, ja että
			// sitä klikkaamalla avautuu muokkausnäkymä!
			$this->click('Muokkaa')->seePageIs('/seppo-raty/muokkaa');

			// Jos ylläolevat ehdot täyttyvät, voimme luottaa,
			// että muokkaustoiminto on olemassa.

	}
	

}

```

Ylläolevat kaksi testiä - luku ja muokkaus - voidaan suorittaa automatisoidusti. Ihmiskäyttäjää ei tarvita. Testiä varten luotu tietokoneohjelma ajaa ylläolevat testit, ja varmistaa, että kaikki oletukset/ehdot täyttyvät. Mikäli jokin ehto ei täyty, asiasta raportoidaan eteenpäin (esim. kehittäjälle).

Ylläolevan kaltaisilla yksittäisillä testeillä voimme varmistaa pala palalta koko käyttöliittymän toiminnan. Entä jos haluamme testata toiminnon *uuden artikkelin luonti*? Se onnistuu näin.

```php

// ArtikkeliTesti.php

// Muut testit kuten ennenkin

public function testaaArtikkelinLuonti() {
	
	// Artikkelin luontia varten käyttäjälle näytetään
	// HTML-lomake, johon artikkelin tiedot täytetään.

	$this->visit('/luo-artikkeli')->andSee('artikkeliluonti');

	// Varmista, että HTML-lomake on olemassa yrittämällä täyttää se...
	$this
	->type('Nollaversio IT', '#artikkelin_nimi') // Kirjoita nimi
	->type('Ihan ok firma.', '#artikkelin_teksti') // Kirjoita sisältö
	->press('Luo artikkeli'); // Paina "Submit"-nappulaa
	->andSee('Uusi artikkeli luotu!') // Varmista luonnin onnistuminen.
	// Varmista että olemme juuri luodun artikkelin sivulla.
	->onPage('/artikkelit/nollaversio-it'); 

}

```

Ylläolevan kaltaisilla testeillä voimme testata ilman epäluotettava ihmissilmän tarvetta koko käyttöliittymämme!

> Loppukaneetti: on syytä huomata, että käyttöliittymätestaus keskittyy *olennaisten seikkojen* testaamiseen. Se ei testaa sitä, onko sivun värimaailma 'ihmissilmää miellyttävä', onko fonttikoko sopiva tai ovatko sivun eri komponentit nätisti rivissä. 

> Automatisoitu testaus keskittyy testaamaan aspekteja, jotka ovat a) ylipäätänsä testattavissa ja b) elintärkeitä applikaation toiminnan kannalta. 

> Värimaailma *ei ole* elintärkeä applikaation toiminnan kannalta, ainakaan Wikipedian tapauksessa. Sen sijaan mahdollisuus muokata artikkelia *on* elintärkeä applikaation toiminnan kannalta. 

> Jokaisella applikaatiolla on tietenkin omat reunaehtonsa sen suhteen, mitkä aspektit ovat tärkeitä ja mitkä eivät. 





