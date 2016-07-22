+++
date = "2016-07-06T06:28:46+03:00"
draft = false
title = "Laravel vinkit #1"

+++

**Kuinka lisätä uusi raportointitoiminnallisuus ilman muutoksia vanhaan koodipohjaan?**

*Lyhyt vastaus: luomalla palveluntarjoaja, joka määrittää tapahtumakuuntelijan.*

Laravellilla rakennettu järjestelmä on helposti laajennettavissa palveluntarjoajien kautta. Palveluntarjoajan mahdollistavat arkkitehtuurin rakentamisen siten, että uudet toiminnallisuudet elävät täysin erillään ns. ydinkoodista. Ydinkoodin ei edes tarvitse tietää uuden toiminnallisuuden olemassaolosta.

Paras tapa toteuttaa tämä erilläänolo on käyttää palveluntarjoajia (Service Provider).

------------------------------

Käytetään esimerkkinä Nordean pankkijärjestelmää. 

Oletetaan, että Nordean mahtava nettipankki sallii asiakkaidensa lisätä pankkitileilleen rahaa. Ahneuspäissään pankkiväki ei ole lisännyt mahdollisuutta nostaa rahaa tililtä - ainoastaan talletus on mahdollista. 

Ihana nettipankki toimii kuin unelma, kunnes tulee lakimuutoksen myötä lisävaatimus: jokaisen pankkitilille tehtävän talletuksen jälkeen järjestelmän tulee ilmoittaa viranomaisille ko. pankkitilin uusi saldo. 

Oletetaan, että tässä esimerkissä viranomaisilla on upea HTTP-rajapinta nimeltä "Pankkipoliisi". Rajapinnan tarkempi toiminta ei ole oleellista, joten oletetaan, että pintaa voidaan kutsua tyyliin: 

```Pankkipoliisi::ilmoita($asiakasID, $uusiSaldo)```

Miten nykyistä Nordean pankkijärjestelmää tulee muuttaa, jotta lain vaatimus täyttyy?

------------------------------

Tarvittava muutos järjestelmän on yksinkertainen. Vanha ydinkoodi - joka huolehtii pankkitilin hallinnasta - voi pysyä tismalleen identtisenä.

Vaatimuksen täyttöä varten lisäämme järjestelmään palveluntarjoajan, joka puolestaan lisää *tapahtumakuuntelijan*. Tuo tapahtumakuuntelija on kaiken A ja O - se kuuntelee järjestelmän tuottamia tapahtumia ja valikoi niistä jatkokäsittelyyn itselleen mieluisat.

Käytännössä homma toimii siten, että **vanha ydinkoodi tuottaa tapahtumia**, ja **uusi koodipohja reagoi noihin tapahtumiin**.

Tämä patterni on yleismaailmallinen ja soveltuu moniin käyttötarkoituksiin: olemassaoleva järjestelmäkomponentti tuottaa informaatiota, uusi komponentti reagoi tuotettuun informaatioon.

Kaiken pohjalla toimii oletus siitä, että vanha komponentti ei tiedä uuden komponentin olemassaolosta mitään. Parhaimmillaan myöskään uusi komponentti ei havaitse vanhaa komponenttia. Kaikki informaatio kulkee tapahtumien muodossa. 

Hieman karrikoiden; vanha komponentti "*ampuu tapahtumia kohti tyhjyyttä*", uusi komponentti "*vastaanottaa tapahtumia tyhjyydestä*".

Tämä on koko arkkitehtuurin perimmäinen ajatus - vanha ja uusi koodipohja elävät täysin omissa maailmoissaan tietämättä mitään toisistaan.

Uusi koodipohja vain ottaa sopivat tapahtumat kiinni.

Vanha ydinkoodi:

```php

// Models/Pankkitili.php

public function saveCash(ICurrency $amount) {

	// Ei tarvetta transaktiolle kun lisätään rahaa -> menee aina läpi.

	// Lasketaan uusi saldo
	$this->balance = $this->balance + $amount->convert($this->accountCurrency);
	// Päivitetään muutos tietokantaan. 
	// Luo ja ampuu tapahtuman "pankkitili päivitetty!".
	$this->save();

	return true;
	
}

```

Yllä siis ydinkoodipohja. Koodi ei missään sisällä ekspliittistä käskyä ampua tapahtumaa. Tapahtuman luonti ja välitys järjestelmän muille komponenteille tapahtuu implisiittisesti, pinnan alla, Laravellin toimesta. 

Kun lisätoiminnallisuutta järjestelmään lisätään, *ylläolevaan koodipohjaan ei tarvitse koskea*. Tämä on koko hajautetun, tapahtumien välitykseen perustuvan arkkitehtuurin keskeisin pointti.

Toteutetaan uusi raportointitoiminnallisuus lisäämällä palveluntarjoaja, jonka vastuulla on napata lennosta sopivat tapahtumat ja reagoida niihin.

```php
// Providers/RaportointiPankkipoliisille.php

public function boot() {	

  // Ilmoitetaan halustamme kuunnella Pankkitiliin liittyviä "updated"-tapahtumia.
  Pankkitili::updated(function($tili) {
    // Tämä klosuuri ajetaan aina kun tilin saldo on päivittynyt.
    // Klosuurin sisällä käytössämme on $tili-objekti
    // $tili edustaa sitä Pankkitiliä, johon päivitys kohdistui.

    // Selvitetään uusi saldo lähettääksemme sen viranomaisille.
    $uusiSaldo = $tili->getBalance();

    $poliisi = new Pankkipoliisi(/*api-tunnukset tähän*/);

    // Ilmoitetaan käyttäjän uusi saldo, eli lähetetään käyttäjä-ID ja saldosumma.
    try {
      $poliisi->ilmoita(\Auth::user()->id(), $uusiSaldo);
    } catch (\Exception $e) {
      Log::warning('poliisi_ilmoitus_fail', $e);
    }
  });
}

```

Pankkipoliisin asetukset täytyy määritellä jossain config-tiedostossa, mutta periaate on ylläolevan mukainen. 

Tiivistettynä:

1. Vanhan koodipohjan (*Pankkitili.php*) ei tässä esimerkissä tarvinnut muuttua kun halusimme lisätä uuden toiminnallisuuden järjestelmään.
2. Uusi palveluntarjoaja (*RaportointiPankkipoliisille.php*) lisäsi **laajennuksen** olemassaolevaan järjestelmään.
3. Lisätyn laajennuksen määrittämä *tapahtumakuuntelija* nappaa sopivat tapahtumat kiinni ja reagoi niihin lähettämällä viestin viranomaisille.


