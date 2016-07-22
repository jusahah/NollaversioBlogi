+++
date = "2016-07-19T14:44:27+03:00"
draft = true
title = "Usecase - palvelupyyntö elää rinnakkaistodellisuuksissa"

+++

Usecase-arkkitehtuuri (en keksinyt hyvää käännöstä suomeksi) on eräs tapa järjestää Laravel-pohjaisen tietokoneohjelman control flow. 

Mitä usecase-arkkitehtuuri painottaa? Nimensä mukaisesti se pyrkii abstraktoimaan koodin erillisiin käyttötarkoituksiin. 

Käyttötarkoitus on esim. "nosta rahaa pankista".

Otetaan esimerkki. Kuvitellaan järjestelmä, jossa loppukäyttäjä voi ryhtyä haluamansa pankin asiakkaaksi. Pankkeja on useita, ja asiakas voi yhden järjestelmän kautta hallita asiakkuuksiaan kussakin pankissa.

### Usecase - rahan nosto

```php

// NostaRahaa_useCase.php

public function nostaRahaa(int $pankkiID, int $asiakasID, int $summa) {

	// Alkuvalmistelut, eli varmistetaan että asiakas-ID on olemassa
	$asiakas = Asiakas::findById($asiakasID); // Throws "EiOlemassa"
	// Varmistetaan, että pankkiID on olemassa
	$pankki = Pankki::findById($pankkiID); // Throws "EiOlemassa"

	// Usecasen tunnusmerkkejä on, että siinä tietyt toimenpiteet
	// suoritetaan järjestyksessä, ja tällä tavoin saavutetaan
	// haluttu lopputulos.

	// Tässä tapauksessa vaiheet ovat:
	// 1. Varmista asiakkuus
	// 2. Nosta rahat
	// 3. Lähetä ilmoitus nostosta asiakkaalle 

	// Virheet napataan kiinni ylempänä call stäkissä.
	
	// #1
	// Throws "EiAsiakkuutta" mikäli varmistus epäonnistuu.
	$pankki->varmistaAsiakkuus($asiakas); 

	// #2
	// Throws "EiKatetta" mikäli ei tarpeeksi rahaa tilillä.
	$nostettuSumma = $pankki->nostaTililta($asiakas, $summa); 

	// #3
	// Onnistuu aina (oletamme)
	$asiakas->lahetaSMS('nostoHyvaksytty', [
		'summa' => $nostettuSumma,
		'ajankohta' => Carbon::now(),
	]);

}  

```
```php

// PankkiController.php

public function nostaRahaa(Request $request, int $pankkiID) {
	// parametrit tulevat IOC-containerin kautta

	// Validation sisääntullut request jotenkin
	try {
		$this->validateNostoRequest($request); // Throws "ValidaatioVirhe"		
	} catch (ValidaatioVirhe $vv) {
		return Response::error('Nosto epäonnistui - tarkista tiedot');
	}

	$asiakasID = $request->get('asiakasID');
	$summa     = $request->get('summa');

	// Kutsutaan usecasea!
	try {
		$nosto = (new NostaRahaa_useCase())->nostaRahaa($pankkiID, $asiakasID, $summa);
	} catch (Error $e) {
		// Ei tarvetta tietää mikä tietty virhe tapahtui, sen sijaan ilmoitetaan
		// virheviesti käyttäjälle. Viesti kertoo kaiken oleellisen.
		return Response::error($e);
	}

	// Kaikki meni oikein mukavasti.
	return Response::success('nostoOnnistui', $nosto);

}


```

Ylläoleva usecase-arkkitehtuuri erottelee *sisääntulevan palvelupyynnön* käsittelyn ja *itse toiminnon läpiviemisen* toisistaan. On syytä muistaa, että rahan nostaminen pankista on palvelupyyntö asiakkaalta pankille. Jotta tuo palvelupyyntö voidaan viedä läpi, täytyy asiakkaan tietokoneen lähettää tekninen palvelupyyntö järjestelmän palvelimelle. Tässä on kaksi fundamentaalista konseptia:

#1 ) Palvelupyyntö siinä mielessä, että tosimaailmassa minä pyydän sinua tekemään jotain
#2 ) Palvelupyyntö siinä mielessä, että kasa bittejä siirtyy tietokoneelta toiselle.

**Jälkimmäinen on pelkkä bittimaailman kuvaus ensimmäisestä**. Täydellisessä maailmassa jälkimmäiselle konseptille ei olisi lainkaan tarvetta. Mutta meidän maailmassamme on - tieto rahan nostosta täytyy jotenkin välittää kotikoneelta palvelimelle. Se ei välity telepatialla, joten joudumme turvautumaan *teknisen palvelupyynnön* lähettämiseen.

Usecase-arkkitehtuuri mahdollistaa näiden kahden konseptin erottelun *kauas* toisistaan. Siis kauas siinä mielessä, että ne sijaitsevat eri tiedostoissa. Tässä on suuri vahvuus. 

Usecase-tiedoston ei tarvitse välittää siitä, millä tavoin asiakkaan kotikone ilmaisi palvelimen suuntaan halunsa nostaa rahaa.

Sen sijaan Controller-tiedosto (PankkiController.php) välittää tuommoisista alhaisen tason detaljeista. Controller ottaa sisään teknisen palvelupyynnön (siis #2 äskeisessä listassamme!), ja luo sen pohjalta oikean palvelupyynnön (#1 listassamme). Usecase-tiedosto ei koskaan edes tiedä #2 olemassaolosta - se välittää vain #1 käsittelystä.

Itse asiassa Usecase-tiedosto ei edes tiedä, että se on osa internet-applikaatiota. Sillä kaikki internet-liikenteeseen liittyvä logiikka elää Controller-tiedostossa.

Lisätään järjestelmään toinen usecase. Mitä muuta haluamme pankkijärjestelmältämme kuin nostaa rahaa? No, ainakin siirtää rahaa yhdeltä tililtä toiselle.

Oletetaan, että rahan siirron voi tehdä miltä tahansa tililtä mille tahansa tilille. Ainoa vaatimus on, että siirron tekevä asiakas omistaa lähtötilin.

### Usecase - tilisiirto

```php

// SiirraRahaa_useCase.php

public function siirraRahaa(
	int $lahtoPankkiID, // Mistä pankista rahat lähtevät?
	int $tuloPankkiID, // Mihin pankkiin rahat saapuvat?
	int $lahettajaID,    // Kenen tili lähtöpankissa?
	int $vastaanottajaID,  // Kenen tili tulopankissa?
	int $summa
) {
	// Tässä oletetaan, että jokaisella asiakkaalla voi olla max. yksi tili per pankki.
	// Täten yhdistelmä {pankki, asiakasID} kuvaa yksilöllisesti pankkitilin.
	// Oikeassa maailmassa käyttäisimme tietenkin *tilinumeroa*, mutta tämä järjestelmä
	// ei sellaista konseptia tunne.

	// Alkuvalmistelut, eli varmistetaan että lähettäjä ja vastaanottaja ovat olemassa.
	$lahettaja = Asiakas::findById($asiakasID); // Throws "EiOlemassa"
	$vastaanottaja = Asiakas::findById($vastaanottajaID); // Throws "EiOlemassa"

	// Varmistetaan, että molemmat pankit ovat olemassa.
	$lahtoPankki = Pankki::findById($lahtoPankkiID); // Throws "EiOlemassa"
	$tuloPankki = Pankki::findById($tuloPankkiID); // Throws "EiOlemassa"	

	// Tämän usecasen vaiheet ovat:
	// 1. Varmista asiakkuudet
	// 2. Nosta summa lähettäjän tililtä
	// 3. Lisää summa vastaanottajan tilille
	// 4. Lähetä ilmoitus nostosta lähettäjälle 
	// 5. Lähetä ilmoitus saapuneesta rahasummasta vastaanottajalle

	// Virheet napataan kiinni ylempänä call stäkissä.
	
	// #1 Varmista asiakkuudet
	// Throws "EiAsiakkuutta" mikäli varmistus epäonnistuu.
	$lahtoPankki->varmistaAsiakkuus($lahettaja); 
	$tuloPankki->varmistaAsiakkuus($vastaanottaja); 

	// Koska nosto yhdeltä tililtä ja talletus toiselle tilille
	// ovat toisistaan *riippuvaisia* operaatioita - eli joko
	// molemmat onnistuvat tai ei kumpikaan - meidän tulee
	// turvautua transaktioon.


	DB::transaction(function () use ($lahtoPankki, $tuloPankki, $lahettaja, $vastaanottaja, $summa) {

		// #2 Nosta summa lähettäjän tililtä
		// Throws "EiKatetta" mikäli ei tarpeeksi rahaa tilillä.
		$nostettuSumma = $lahtoPankki->nostaTililta($lahettaja, $summa); 

		// #3 Lisää summa vastaanottajan tilille
		$tuloPankki->talletaTilille($vastaanottaja, $nostettuSumma);
	});

	// #4 Lähetä ilmoitus nostosta
	$lahettaja->lahetaSMS('siirtoHyvaksytty', [
		'summa' => $summa,
		'ajankohta' => Carbon::now(),
	]);

	// #5 Lähetä ilmoitus saapuneesta rahasummasta
	$vastaanottaja->lahetaSMS('siirtoSaapunut', [
		'summa' => $summa,
		'ajankohta' => Carbon::now(),
	]);
}  

```
```php

// PankkiController.php

public function nostaRahaa(Request $request, IPankki $pankki) {
	// Kuten ennenkin
	// ...
}

public function siirraRahaa(Request $request, IPankki $lahtoPankki) {
	// Parametrit IOC:in kautta
	// Miksi otamme IOC:n kautta $lahtoPankin, mutta emme $tuloPankkia?
	// Koska lähettäjä operoi omalla selaimellaan *tietyn* pankin käyttöliittymässä, 
	// ja kaikki lähettäjän tekemät palvelupyynnöt tehdään tietyn pankin suuntaan.
	// Toisin sanoen, kaikki sisääntulevat palvelupyynnöt tehdään URL:ään, jonka rakenne
	// on seuraavanlainen:

	/*
		http://pankkijarjestelma.fi/pankki/pankkiID/operaatio
	*/

	// Validation sisääntullut request jotenkin
	try {
		$this->validateSiirtoRequest($request); // Throws "ValidaatioVirhe"		
	} catch (ValidaatioVirhe $vv) {
		return Response::error('Rahan siirto epäonnistui - tarkista tiedot');
	}

	// Varmistetaan, että vastaanottava pankki on olemassa
	$tuloPankkiID = $request->get('tuloPankkiID');
	$tuloPankki = Pankki::findById($tuloPankkiID); // Throw "EiPankkia";

	// Haetaan muut siirtoon liittyvät tiedot.
	$lahettajaID = $request->get('lahettajaID');
	$vastaanottajaID = $request->get('vastaanottajaID');
	$summa     = $request->get('summa');

	// Kutsutaan usecasea!
	try {
		(new SiirraRahaa_useCase())->siirraRahaa(
			$lahtopankki, 
			$tuloPankki,
			$lahettajaID,
			$vastaanottajaID, 
			$summa
		);

	} catch (Error $e) {
		// Ei tarvetta tietää mikä tietty virhe tapahtui, sen sijaan ilmoitetaan
		// virheviesti käyttäjälle. Viesti kertoo kaiken oleellisen.
		return Response::error($e);
	}

	// Kaikki meni oikein mukavasti.
	return Response::success('siirtoOnnistui');	


}


```

Tässä vaiheessa on hyvä mainita eräästä seikasta.

Kuten huomaamme, sisääntulevan datan validaatio on jaettu kahteen osaan. Esimerkiksi vastaanottajaID:

1) Ensin validoimme, että vastaanottajaID on mukana sisään tulevassa palvelupyynnössä. Tämä validointi tapahtuu ```$this->validateSiirtoRequest($request)``` rivillä. Millainen tuo metodi on? Esimerkiksi seuraavanlainen:

```php

protected function validateSiirtoRequest(Request $request)
{
	// Throws "ValidaatioVirhe"
    $this->validate($request, [
        'tuloPankkiID' => 'required|int',
        'lahettajaID' => 'required|int',
        'vastaanottajaID' => 'required|int',
        'summa' => 'required|int',
    ]);

    // Kaikki kunnossa
    return true;
}

```

Huomioitavaa on, että tämä tarkistus/validatointi tapahtuu *controllerin* puolella.

2) Myöhemmin validoimme/tarkistamme - että kunkin ID:n takaa löytyy oikea, aito objekti. Eli jos pankkiID on 15, järjestelmässämme on olemassa Pankki, jonka ID on 15.

Tämä tarkistus tapahtuu *usecasen* puolella.


### Controller-validaatio vs. usecase-validaatio

Miksi validaatio on jaettu kahteen paikkaan? Eikö olisi selkeämpää, jos molemmat validaatiot tehtäisiin yhdessä ja samassa paikassa?

Ei.

On syytä huomata, että nämä kaksi validaatiota tarkistavat *eri* asioita.

Controller-validaatio tarkistaa, että sisääntulevat ID:t ovat *numeroita*. Ne eivät saa olla esimerkiksi JPG-kuvia - on vaikea etsiä pankkia JPG-kuvan kautta.

Usecase-validaatio tarkistaa, että *ID-numero* (ja usecasen kohdalla me jo varmuudella tiedämme, että ID on numero, kiitos Controller-validaation!) vastaa jotakin järjestelmässä sijaitsee pankkia. On mahdollista, että palvelupyynnön mukana tullut ID-numero ei vastaa yhtäkään pankkia. Pankkeja ei kuitenkaan ole rajatonta määrää, numeroita sen sijaan on.

Tässä on ero. **Controller validoi, että sisääntuleva data on oikeanmuotoista. Usecase validoi, että sisääntuleva data on järjellistä järjestelmän kannalta.**

### Summa summarum

Usecase-arkkitehtuurin vahvuus piilee juuri edellisessä huomiossa. Voimme käsitellä "ylätason toimintoja" selkeinä kokonaisuuksina, eli usecasenaina, käyttötarkoituksina. Samaan aikaan usecase on *irrallaan* kaikesta siitä ikävästä, mutta pakollisesta säläkoodista, joka liittyy internet-applikaation tekniseen toteutukseen. Eli HTTP-pyyntöjen hallinnasta, jne.

Hyvässä web-applikaatiossa päteekin, että itse applikaation ydinkoodi - tässä tapauksessa se koodi, joka suorittaa siirtoja ja nostoja pankkien välillä - ei edes tiedä asuvansa osana web-applikaatiota. Se tietää asuvansa osana *applikaatiota*, mutta webin olemassaolosta se on onnellisen tietämätön.

