+++
date = "2017-10-29T05:04:20+02:00"
draft = false
title = "Kaikki tapahtumat vievät try-catchiin"

+++

Tyypillinen UI-lähtöinen web-applikaatio perustuu nk. event-driven paradigmaan. Tämä tarkoittaa, että applikaation oleelliset toiminnallisuudet suoritetaan *tapahtumien* (events) seurauksena. 

Esimerkkinä: kun käyttäjä klikkaa hiirellä ikonia, syntyy tapahtuma. Tuo tapahtuma aiheuttaa jonkin toiminnallisuuden suorittamisen applikaation sisällä. Kun toiminnallisuus on suoritettu, applikaatio menee horrostilaan odottamaan seuraavaa tapahtumaa.

Tapahtumakeskeiset applikaatiot tupataan koodaamaan *tapahtumakuuntelijoiden* ympärille. Tyypillinen UI-applikaatio on pohjimmiltaan pelkkä kasa kuuntelijoita, jotka suorittavat toimintoja. Tyypillinen ylätason arkkitehtuuri on seuraavanlainen:

```javascript

// App contains all the business code and logic.
var app = new App();

var listeners = {

	onEventX: app.api.doSomething,
	onEventY: app.api.doSomethingElse,
	onEventZ: app.api.doThirdThing,
	//...
}

// Bind listeners to device, allowing User to interact 
// with our App by pressing buttons on the device etc.
device.registerListeners(listeners);

```

Ylläoleva on karkea kuvaus siitä, miten käytännössä kaikki graafisen käyttöliittymän omaavat applikaatiot toimivat.

Entä miltä näyttää tuollaisen applikaation suoritus-/ajohistoria? Tapahtumia odottaville applikaatiolle on nyrkkisääntönä tyypillistä, että ne kirjaimellisesti **odottavat** valtaosan ajasta. Tämä johtuu siitä, että tyypillinen applikaatio käsittelee sisääntulleen tapahtuman silmänräpäyksessä. 

> Esimerkiksi tyypillinen tekstieditori - sanotaan vaikka Microsoftin Notepad - istuu ja odottaa vähintään 99% elinkaarestaan toimettomana. Joka kerta kun tekstieditorin käyttäjä - siis ruudun edessä istuva ihminen - painaa näppäimistöllä nappulaa, tekstieditori herää ruususen unestaan ja suorittaa toimenpiteen. Tekstieditorin tapauksessa toimenpide on useimmiten käyttäjän näppäimistöllä painaman kirjaimen tallentaminen keskusmuistiin ja piirtäminen ruudulle. Aikaa tuohon kuluu ehkä parisenkymmentä *mikro*sekuntia (sekunnin miljoonasosa!), jonka jälkeen tekstieditori siirtyy takaisin unten maille.

Ajohistorian toinen hauska piirre on, että kaikki suoritusajot lähtevät liikkeelle tapahtumahallinnasta. Tämä on väistämätöntä, sillä juuri tapahtumahallinta vastaanottaa sisääntulleen tapahtuman ja kutsuu applikaation varsinaisen bisneslogiikan sisältämiä funktioita.

Tämä "tapahtumalähtöisyys" antaa mainion tavan organisoida loki- ja virhehallinta! Koska kaikki suoritusajot lähtevät liikkeelle tapahtumien kautta, voi näppärä koodari luoda *putken*, jonne kaikki tapahtumat ajetaan. 

Putken toisessa päässä odottaa itse applikaatio. Kun putkeen työntää *tapahtuman*, se hetkeä myöhemmin tömähtää toisesta päästä ulos ja herättää horrokseen vaipuneen applikaation.

Ensimmäistä koodiesimerkkiä muokkaamalla:

```javascript

////////////////////////
//// EVENTS.JS /////////
////////////////////////

// App contains all the business code and logic.
var app = new App();

var listeners = {

	onEventX: app.eventBus.bind(app, 'eventX'),
	onEventY: app.eventBus.bind(app, 'eventY'),
	onEventZ: app.eventBus.bind(app, 'eventZ'),
	//...
}

// Bind listeners to device, allowing User to interact 
// with our App by pressing buttons on the device etc.
device.registerListeners(listeners);


////////////////////////
/////// APP.JS /////////
////////////////////////

function App() {

	this.api = new Api();
	
	this.eventBus = function(eventTag) {
		// eventTag on joko eventX, eventY tai eventZ.

		if (eventTaget === 'eventX') {
			this.api.doSomething();
		} else if (eventTaget === 'eventY') {
			this.api.doSomethingElse();
		} else if (eventTaget === 'eventZ') {
			this.api.doThirdThing();
		}
	}
}

```

Ylläolevan ero verrattuna ensimmäiseen koodiesimerkkiin on, että nyt kaikki tapahtumat saapuvat *yhden* linkkipisteen kautta. Tuo linkkipiste on eventBus-metodi. 

Tämä on käytännössä ainoa ero näiden kahden koodiesimerkin välillä; applikaatiota ajaessa ne toimivat tismalleen samoin. Miksi siis luoda yksittäinen linkkipiste?

Periaate on sama kuin vaikkapa Suomen rajalla. Sen sijaan, että ulkomaalaisten annettaisiin hyppiä Suomen maaperälle mistä kohdin tahansa, kaikki maahantulot ohjataan *raja-asemalle*. Tuolla raja-asemalla voidaan **keskitetysti** suorittaa tietyt toimenpiteet, kuten passin tarkastus.

Siirtämällä esimerkkiapplikaatiomme käyttämään keskitettyä linkkipistettä, mekin voimme nyt suorittaa keskitetysti avustavia toimenpiteitä. 

```javascript

function App() {

	this.api = new Api();
	
	this.eventBus = function(eventTag, event) {
		// eventTag on joko eventX, eventY tai eventZ.

		// Kirjaa lokitietoihin uuden tapahtuman käsittely
		this.log('Tapahtuma ' + eventTag + ' saapunut'); 

		if (eventTaget === 'eventX') {
			this.api.doSomething();
		} else if (eventTaget === 'eventY') {
			this.api.doSomethingElse();
		} else if (eventTaget === 'eventZ') {
			this.api.doThirdThing();
		}
	}
}

```

Yllä kirjasimme lokiin tiedon tapahtuman saapumisesta. Koska kaikki tapahtumat tulevat sisään eventBus-metodin kautta, kaikki tapahtumat myös tulevat kirjatuksi lokiin!

> Lisäsimme myös eventBus-metodiin toisen parametrin nimeltä *event*. Applikaatiosta riippuen tätä parametriä tarvitaan tai ei tarvita. Se sisältää itse *tapahtuman*, jonka applikaation alta löytyvä laitteisto synnytti. Ensimmäinen parametri (eventTag) sisältää vain tiedon *minkälainen* tapahtuma on kyseessä; toinen parametri sisältää itse tapahtuman. Kuten sanottua, joskus (usein) riittää tietää millainen tapahtuma on kyseessä; tällöin itse tapahtuma-objektia ei tarvita lainkaan.
>
>
>
> Silloin kun tapahtuma-objekti tarvitaan, se sisältää kaiken tapahtumaan liittyvän informaation. Esimerkiksi klikatessa hiirellä ikonia tuo parametri *event* sisältää tiedon siitä, mitä ikonia klikattiin. Tai vaihtoehtoisesti se voi sisältää tietokoneen näyttöpäätteen koordinaatit (x/y), jossa klikkaus tapahtui.

Ylläoleva on ihan kiva, mutta todellinen hyöty syntyy virhehallinnan puolella. Kuten useaan otteeseen todettu, tyypillisessä UI-applikaatiossa kaikki toimenpiteet lähtevät liikkeelle tapahtumahallinnasta. Sama hiukka teknisemmin todettuna: yksittäinen suoritusajo muodostaa itsenäisen call stackin, jossa ylimpänä funktiokutsuna on tapahtumahallinta, meidän esimerkin tapauksessa *eventBus*.

Esimerkkinä applikaation call stack, joka muodostuu vaikkapa Photoshopissa kun käyttäjä klikkaa hiirellä työkalupalkista "Pensseli-työkalua".

```
	eventBus
	  api.handleClick
	    drawTools.handleClick
	      drawTools.setPensseliAsNewTool
	  
```

Kun käyttäjä painaa Photoshopin teksti-objektin ollessa valittuna näppäintä "s", syntyy puolestaan seuraava call stack:

```
	eventBus
	  api.handleKeyPress
	    canvas.handleKeyPress
	      textObject.handleKeyPress
	        textObject.updateText
	  
```

Ylläolevan sisäkkäisten funktiokutsujen sarjan perusteella Photoshop päivittää teksti-objektin sisältämän tekstin. Jos aiemmin ruudulla luki "Kaamo", nyt siinä lukee "Kaamos".

Yhteistä kahdelle edeltävälle call stackille on, että eventBus on molempien lähtöpiste. Tämä antaa mahdollisuuden seuraavanlaiseen virhehallintaan.


```javascript

function App() {

	this.api = new Api();
	
	this.eventBus = function(eventTag, event) {
		// eventTag on joko eventX, eventY tai eventZ.

		// Kirjaa lokitietoihin uuden tapahtuman käsittely
		this.log('Tapahtuma ' + eventTag + ' saapunut'); 

		try {
			if (eventTag === 'eventX') {
				this.api.doSomething();
			} else if (eventTag === 'eventY') {
				this.api.doSomethingElse();
			} else if (eventTag === 'eventZ') {
				this.api.doThirdThing();
			}
		}

		catch (e) {
			// Jotain meni pieleen tapahtumaa käsitellessä/suorittaessa

		}

	}
}

```

Wrappasimme **koko event-dispatchin** (tuon ison if-else-lausekkeen) try-catchin sisälle. Tämä tarkoittaa, että kaikki virheet, jotka tapahtuvat alempana call stackissa, napataan viimeistään eventBus-metodin sisällä kiinni. Tämä on keskitettyä virheiden hallintaa parhaimmillaan.

Myös virheiden raportointia on helppo kehittää:

```javascript

function App() {

	this.api = new Api();
	
	this.eventBus = function(eventTag, event) {
		// eventTag on joko eventX, eventY tai eventZ.

		// Kirjaa lokitietoihin uuden tapahtuman käsittely
		this.log('Tapahtuma ' + eventTag + ' saapunut'); 

		try {
			if (eventTag === 'eventX') {
				this.api.doSomething();
			} else if (eventTag === 'eventY') {
				this.api.doSomethingElse();
			} else if (eventTag === 'eventZ') {
				this.api.doThirdThing();
			}
		}

		catch (e) {
			// Jotain meni pieleen tapahtumaa käsitellessä/suorittaessa
			
			// Lähetetään virheilmoitus Bugsnag-palveluun.

			// Kerrotaan ensin minkä tapahtuman käsittelyssä virhe syntyi...
			Bugsnag.leaveBreadcrumb("Virhe syntyi käsitellessä tapahtumaa " + eventTag);
			// ...sitten lisätään lähetyspakettiin itse Exception-objekti.
			Bugsnag.notifyException(e);

			// Kehittäjien ja sidosryhmien informointi on suoritettu!

			// Yritetään korjata tilanne palauttamalla aiempi tila. Tällä tavoin
			// käyttäjän on mahdollista jatkaa ohjelman käyttämistä virheestä huolimatta.
			this.resetPreviousState();
		}

	}
}

```

Yllä lähetämme virheilmoituksen mainioon Bugsnag-palveluun. Tuon palvelun kautta ilmoitus päätyy applikaation kehittäjille, parhaimmillaan jopa reaaliajassa.

Tämän lisäksi yritämme palauttaa applikaation aiempaan, varmuudella toimivaan tilaan. Yksi ikävä piirre virhetilanteissa noin yleensä on, että ne sotkevat applikaation sisäiset tilamuuttujat. Näin ei ole pakko tapahtua; on vallan mahdollista, että virhe tapahtuu *turvallisesti*, jolloin se jättää jälkeensä siistin, toimivan applikaation. Mutta monet ennakoimattomat virheet tapahtuvat nk. kriittisellä hetkellä, jolloin ne sotkevat applikaation. 

> Tilanne on vähän vastaava kuin vaikka laskiessa säästöpossun kolikoita. Jos kesken laskusuorituksen menet yhtäkkiä laskuissa sekaisin (= aivojesi virhetilanne), ei sinulla ole muuta vaihtoehtoa kuin aloittaa alusta. Virhe tapahtui kriittisellä hetkellä, tässä tapauksessa laskennan ollessa käynnissä. 
>
>
>
> Ei-kriittinen virhetilanne syntyy jos kesken laskutoimituksen vahingossa pudotat kädessä olevan kolikon lattialle. Tämä on ilmiselvä käsiesi virhetilanne; et varmastikaan tarkoittanut pudottaa kolikkoa. Mutta kyseessä on ei-kriittinen virhe siksi, että voit nostaa kolikon lattialta ja jatkaa laskutoimitusta siitä mihin jäit. No harm done.

Metodikutsumme resetPreviousState antaa applikaatiolle käskyn palauttaa aiempi, toimivaksi todettu tila. Tämän toiminnallisuuden toteuttaminen olisi toisen postauksen aihe; tässä kohtaa riittää, että oletamme aiemman tilan palauttamisen olevan mahdollista.

Koodia voi vielä hiukan siistiä siirtämällä varsinaisen dispatch-osuuden erikseen avustavista toimenpiteistä (raportointi, recovery-toimenpiteet):

```javascript

function App() {

	this.api = new Api();
	
	this.eventBus = function(eventTag, event) {
		// eventTag on joko eventX, eventY tai eventZ.

		// Kirjaa lokitietoihin uuden tapahtuman käsittely
		this.log('Tapahtuma ' + eventTag + ' saapunut'); 

		try {
			this.handleEvent(eventTag, event);
		}

		catch (e) {
			// Jotain meni pieleen tapahtumaa käsitellessä/suorittaessa
			
			// Lähetetään virheilmoitus Bugsnag-palveluun.

			// Kerrotaan ensin minkä tapahtuman käsittelyssä virhe syntyi...
			Bugsnag.leaveBreadcrumb("Virhe syntyi käsitellessä tapahtumaa " + eventTag);
			// ...sitten lisätään lähetyspakettiin itse Exception-objekti.
			Bugsnag.notifyException(e);

			// Kehittäjien ja sidosryhmien informointi on suoritettu!

			// Yritetään korjata tilanne palauttamalla aiempi tila. Tällä tavoin
			// käyttäjän on mahdollista jatkaa ohjelman käyttämistä virheestä huolimatta.
			this.resetPreviousState();
		}

	}

	// HandleEvent-metodi keskittyy yksinomaan valitsemaan oikean toimenpiteen saamansa
	// tapahtuman (tai tapahtumatagin) perusteella.
	this.handleEvent = function(eventTag, event) {
		if (eventTag === 'eventX') {
			this.api.doSomething();
		} else if (eventTag === 'eventY') {
			this.api.doSomethingElse();
		} else if (eventTag === 'eventZ') {
			this.api.doThirdThing();
		}		
	}
}

```

Thats it! Koodi näyttää selkeältä, ja eri vastuualueet on selkeän visuaalisesti erillään koodipohjassa.
