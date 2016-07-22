+++
date = "2016-07-18T18:02:21+03:00"
draft = false
title = "Lupausketju liukuhihnana - then vs. tap"

+++

Lupausten maailma on kaunis paikka. Pitkä, hyvin abstraktoitu lupausjono on Scarlett Johanssonin vartaloon verrattavissa oleva jumalallisen kauneuden symboli.

Mutta joskus tulee vastaan ongelmia, joihin lupausjono ei luontevasti sovellu. Tai ainakin voisi äkkiseltään *luulla*, että lupausjono ei toimi halutusti. Yksi tälläinen on seuraava.

```javascript

// Lupaus jono alkaa
haeTennisTuloksetPalvelimelta()
.then(lajittelePelaajittain)
.then(printtaaFedererinTulokset)
.then(laskeKunkinPelaajanVoittoprosentti)

```

Hyvin kaunis lyhyt lupausjono, jonka asynkronoituun tyyliin etenee askel askeleelta. Koko lupausjono on kuin yksi iso liukuhihna. Kunkin *then()*-funkkarin kohdalla liukuhihnalla kulkeva tavara ohjataan *apufunktioon* (esim. lajittelePelaajittain). 

Apufunktio voidaan ajatella koneena, joka jollain tavalla *muuttaa* tai *transformoi* saamansa esineen. Muutoksen/transformaation jälkeen tavara etenee kohti seuraavaa apufunktiota/käsittelypistettä. 

Toimii kuin unelma. Mutta katsotaanpa mitä kunkin askeleen apufunktio palauttaa jonon *seuraavalle* kaverille tässä meidän tennistuloksia hallinnoivassa esimerkissämme.

Katsotaan vaiheittain:

### haeTennisTuloksetPalvelimelta 

*null -> tulokset*

Tämä on selvä peli - se hakee tulokset serveriltä ja työntää ne liukuhihnalle. Tämä vaihe istuu liukuhihnan alussa, joten se ei saa syötettä sisäänsä lainkaan. Sen sijaan se aloittaa hihnan toiminnan puskemalla erikseen palvelimelta haetut tulokset hihnalle.

---

### lajittelePelaajittain

*tulokset -> niputetut tulokset*

Tämä myös helppo - se ottaa vastaan tulokset, ja lajittelee ne pelaajittain. Eli esimerkiksi Rafael Nadalin kaikki ottelutulokset paketoidaan kivasti yhteen nippuun siten, että myöhemmin niitä on helppo käsitellä erillään muista pelaajista. Tehtyään niputuksen tämä vaihe siirtää tuotetut niput takaisin liukuhihnalle kohti seuraavaa vaihetta.

---

### printtaaFedererinTulokset

*niputetut tulokset -> niputetut tulokset*

Mutta entä tämä? Mikä on tämän käsittelyvaiheen tarkoitus? Nimensä mukainen. Vaihe etsii juuri niputetuista (pelaajittain!) tuloksista Roger Federerin tulokset, ja printtaa ne paperille. Miksi juuri Federer? En tiedä, eikä se ole oleellista.

Mikä on oleellista on se, että tämä vaihe EI transformoi/muunna koko sisääntulevaa datapakettia johonkin uuteen muotoon. 

Joten mitä tämä vaihe sitten palauttaa liukuhihnalle? Me tiedämme alkuperäistä lupausjonoa tarkastelemalla, että seuraava vaihe (*laskeKunkinPelaajanVoittoprosentti*) odottaa saatavakseen niputetut tulokset. Toisin sanoen, *seuraava* vaihe odottaa *edellisen* vaiheen syötettä.

Tämä tarkoittaa, että *printtaaFedererinTulokset* vaihe on ikäänkuin tyhjänpäiväinen läpikulkupiste. Kuin tyhjä putki. Kuin kone, joka ei tee mitään. Huomioitavaa on, että kone tekee kyllä jotain (printtaa paperille Federerin tulokset), mutta liukuhihnan syötteen näkökulmasta mitään ei tapahdu. 

**Syöte vain kulkee läpi muuntumatta lainkaan!**

```javascript

// printtaaFedererinTulokset.js

module.exports = function(sisaantulevaData) {
	
	var federerTulokset = sisaanTulevaData['Federer'];

	// Emme ole kiinnostuneita tulostuksen onnistumisesta yms.
	// Kunhan kutsumme tulostusfunktiota ja jatkamme elämäämme eteenpäin.
	printtaa(federerTulokset);

	// Palautetaan saatu syöte identtisenä takaisin hihnalle.
	return sisaantulevaData;
}

```

---

### laskeKunkinPelaajanVoittoprosentti 

*niputetut tulokset -> voittoprosentit pelaajittain*

Taas selvä peli - tämä steppi ottaa sisäänsä niputetut tulokset ja aggregoi kunkin pelaajan osalta ne yhteen laskien voittoprosentin.

Ja avot - kaikki toimii. 

---

---

Mutta.

Jokin häiritsee *printtaaFedererinTulokset*-vaiheessa. Tuo vaihe ottaa sisäänsä dataa ja puskee saman datan *identtisenä* ulos. Mitä järkeä tässä on? 

**Oleellinen huomio on, että noin maalaisjärjellä ajateltuna *printtaaFedererinTulokset* ei ole osa liukuhihnaa**. Tai siis että se ei ole mikään *käsittelyvaihe* lainkaan! Se on enemmänkin vain liukuhihnan päällä sijaitseva tuntoanturi - kun paketti kulkee sen ylitse, jotain tapahtuu jossain. Tässä tapauksessa tuo "jotain" on, että printteri alkaa sylkemään aanelosta ulos. 

Tuntoanturi ei muunna pakettia mitenkään.

Joten kauniin koodin nimissä olemme pakotettuja muokkaamaan lupausjonoa. PrinttaaFedererinTulokset ei saa olla käsittelyvaihe, sen tulee olla anturi.

# Tap-funktio on anturi

Hoidetaan homma ottamaan käyttöön *tap*-funktio osana lupausketjua (liukuhihnaa). Tap-funktio on juuri tähän tarkoitukseen soveltuva - se ottaa sisäänsä edellisen vaiheen tuottamaan syötteen, mutta *ei tuota mitään ulosmenevää tavaraa*!

Toisin sanoen, tap-funktion käyttö mahdollistaa, että tap-funktiota seuraava vaihe saa syötteen sisään tap-funktiota edeltävältä vaiheelta.

Tässä tapauksessa *laskeKunkinPelaajanVoittoprosentti* saa syötteensä *lajittelePelaajittain*-vaiheelta. Tämä on juuri mitä haluammekin.

Eli lopullinen muoto.


```javascript

// Lupaus jono alkaa
haeTennisTuloksetPalvelimelta()
.then(lajittelePelaajittain)
.tap(printtaaFedererinTulokset)
.then(laskeKunkinPelaajanVoittoprosentti)

```

Tap tap. Kaunista ja toteuttaa täydellisesti SINGLE RESPONSIBILITY-prinsiippiä. PrinttaaFedererinTulokset saa sisäänsä tarvittavan datan, mutta sen ei tarvitse huolehtia ulosmenevästä tavarasta lainkaan. Itse vaihe on nyt yhden rivin lyhyempi:


	```javascript

	// printtaaFedererinTulokset.js

	module.exports = function(sisaantulevaData) {
		
		var federerTulokset = sisaanTulevaData['Federer'];

		// Emme ole kiinnostuneita tulostuksen onnistumisesta yms.
		// Kunhan kutsumme tulostusfunktiota ja jatkamme elämäämme eteenpäin.
		printtaa(federerTulokset);

		// Ei tarvitse palauttaa mitään!
	}

	```


> Loppukaneetti: On tietenkin selvää, että useimmissa projekteissa tap vs. then -funktion käyttö on aika irrelevantti seikka. Tässä esimerkissä saimme säästettyä yhden rivin koodia, ja hitusen selvennettyä lupausketjun logiikkaa (kokenut koodari huomaa yhdellä silmäyksellä liukuhihnan toimintalogiikan). Hyöty on silti aika minimaalinen ja lähinnä kosmeettinen.
>
> Tap-funktion suurin hyöty tulee silloin, kun joudumme lupausketjun osana kutsumaan koodia, jota emme hallitse. Kuvitellaan, että *printtaaFedererinTulokset* sijaitsee osana valtavaa, minimoitua JS-kirjastoa. Tuon kirjaston kirjoittaja on oikeaoppisesti koodannut funktion siten, että se ei palauta mitään ulos. Me emme pysty asiaan vaikuttamaan. Joudumme täten tilanteeseen, jossa emme voi käyttää pelkistä *then()*-vaiheista koostuvaa ketjua - printtaaFedererinTulokset-vaihe rikkoisi tuon ketjun.
>
> Tässä tapauksessa *tap-funktio* pelastaa päivän suorastaan naurettavan helpolla. Kutsumme *printtaaFedererinTulokset*-kirjastofunktiota tap-funktion sisältä, ja kaikki toimii. 




