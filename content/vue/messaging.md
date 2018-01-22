+++
date = "2018-01-15T17:57:17+02:00"
draft = false
title = "Komponentin datahaku alustuksen aikana"

+++

Männä päivänä syntyi seuraavanlainen tarve Vue-käyttöliittymää ohjelmoidessa; yhden komponentin tuli alustuksensa (*created*-hook) aikana saada informaatiota toiselta komponentilta, joka ei ollut suora esi-isä alustettavalle komponentille.

Ongelma ei kuulosta erityisen vaikealta - eikä sitä olekaan - mutta ohjelmoijan pääkoppa alkaa herkästi *yliratkomaan* ongelmaa.

Tyypillisestihän Vue-komponenttien välinen kommunikointi tapahtuu jommalla kummalla kahdesta seuraavasta tavasta:

### 1. Emit/props

Mikäli *toinen komponentti on toisen suora jälkeläinen*, kommunikointi tapahtuu luontevasti joko käyttäen propseja (alaspäin kommunikoidessa!) tai emittoimalla eventtejä (ylöspäin kommunikoidessa!). Tämä on luonteva tapa kommunikoida jos komponenttipuussa liikutaan vain vertikaalisesti (*isä-poika*), ei horisontaalisesti (*sisar-veli*). Ohessa esimerkki eventtien käytöstä:

```

// Parent.js


<template>
	<h3>Parent component</h3>
	<Child @viesti="viestiAlhaalta"></Child>
</template>

<script>

import Child from './Child'

export default {
	methods: {
		viestiAlhaalta(viestinSisalto) {
			console.log("Viesti alhaalta: " + viestinSisalto)
			
		}
	}
}

</script>

```

```

// Child.js

<template>
	<h3>Child component</h3>
	<button v-on:click="lahetaViesti">Lähetä</button>
</template>

<script>

export default {
	methods: {
		lahetaViesti() {
			this.$emit('viesti', 'Hei vain, isäpappa');
			
		}
	}
}

</script>

```

### 2. Erillinen Vue-instanssi

Mikäli kumpikaan komponentti ei ole toisen suora jälkeläinen, kommunikointi voi tapahtua joko **1)** yhteistä ylätason komponenttia käyttäen, joka ottaa vastaan viestin yhdestä alipuusta ja ampuu sen alas toiseen alipuuhun, tai **2)** erillistä *observer*-järjestelmää käyttäen. 

Jälkimmäinen on suositeltava ratkaisu. Ensimmäinen ratkaisu toki toimii, mutta on isossa puurakenteessa tuhoisan sotkuinen toteuttaa ja ylläpitää. 

> Ohjelmoinnin kultainen sääntöhän on, että kaikkea on *mahdollista* tehdä, mutta mitään ei ole *järkevää* tehdä. Tai ainakaan lähes tulkoon mitään. 

Eli observer-ratkaisu on parempi. Observer-radiomastona toimii luontevasti koko erillinen Vue-instanssi:

```javascript

// services/Radiomasto.js

export default new Vue({});

```

```

// Palolaitos.js


<script>

import Radiomasto from './services/Radiomasto';

export default {
	data() {
		observerCb: null
	},
	name: 'Palolaitos',
	created() {
		// Ilmoita halustasi kuunnella tiettyjä viestejä
		this.observerCb = this.halytys.bind(this);
		Radiomasto.$on('tulipalo', this.observerCb);	
	},
	beforeDestroy() {
		// Lopeta kuuntelu
		Radiomasto.$off('tulipalo', this.observerCb);
	},
	methods: {
		halytys(osoite) {
			// Lähetä palomiehet annettuun osoitteeseen
			console.log("Palomiehet paikalle!");
		}
	}
}
</script>

```

```javascript

// Puukerrostalo.js


<script>

import Radiomasto from './services/Radiomasto';

export default {
	name: 'Puukerrostalo',
	methods: {
		tulipaloHavaittu() {
			// Ilmoita palosta.
			Radiomasto.$emit('tulipalo', 'Koivukuja 2');
		}
	}
}
</script>

```

Ylläolevan ratkaisun saa tarvittaessa vieläpä siirrettyä *mixiniin*, jolloin sitä on helppo käyttää milloin tarve vaatii.

*Mutta alkuperäinen ongelmani oli saada yhdeltä komponentilta informaatiota toisen komponentin alustuksen aikana!* 

Yksikään ylläolevista vaihtoehdoista ei sovellu erityisen hyvin tämän vaatimuksen täyttämiseen. 

Ylläolevassa #2 esimerkissä viestin lähetys on *tuottaja*-lähtöistä; viestin luoja lähettää viestin haluamanaan ajanhetkenä. Mutta alkuperäisessä ongelmassa viestittely on *kuluttaja*-lähtöistä; viestin vastaanottaja määrittää ajanhetken, jolloin hän tarvitsee informaatiota käyttöönsä. Tästä syystä tarvitsemme toisen lähestymistavan.

Yksinkertaisin ratkaisu on suorastaan hupaisan... yksinkertainen. Käytetään yhteistä globaalia tietovarastoa, jonne kaikilla komponenteilla on yhteys! Joka kerta kun tuottaja-komponentti havaitsee muutoksen datassa, hän päivittää tietovaraston. Kuluttaja-komponentti voi sitten hakea haluamansa datan sopivalla hetkellä, tässä tapauksessa alustuksen aikana.

### Globaali tietovarasto

```javascript

// Tietovarasto.js

export default {
	muumitKpl: 0
}

```

```

// Muumimamma.js


<template>
	<button v-on:click="lisaaMuumi">Lisää</button>
	<button v-on:click="lisaaTuutikki">Lisää tuutikki</button>
</template>

<script>

import Tietovarasto from 'services/Tietovarasto'
import Muumi from 'entities/Muumi'
import Tuutikki from 'entities/communists/Tuutikki'

export default {
	name: 'Tuottaja',
	data() {
		olennot: [],
	},
	methods: {
		lisaaMuumi() {
			this.olennot(new Muumi());
			// Muumien määrä muuttui
			// Laske ja päivitä globaali tieto muumien määrästä
			Tietovarasto.muumitKpl = this.olennot.filter((olento) => {
				return !!olento.valkoinenJaPullea;
			}).length;
		},
		lisaaTuutikki() {
			this.olennot(new Tuutikki());

			// Muumien määrä ei muuttunut

		}
	}
}
</script>

```

```

// MuumitInfotaulu.js


<template>
	<h3>Muumeja on {{kpl}}</h3>
	<button v-on:click="paivitaMuumimaara">Päivitä</button>
</template>
<script>

import Tietovarasto from 'services/Tietovarasto'

export default {
	name: 'MuumitInfotaulu',
	data() {
		kpl: 0
	},
	methods: {
		paivitaMuumimaara() {
			// Käy hakemassa viimeisin lukumäärä
			// globaalista tietovarastosta.
			this.kpl = Tietovarasto.muumitKpl;
		}
	},
	created() {
		// Alustus
		//
		// Haetaan muumimäärä.
		this.paivitaMuumimaara();
	}
}
</script>

```

Yleisemmin: ylläoleva ratkaisu antaa mille tahansa komponentille pääsyn minkä tahansa komponentin tietoihin haluamallaan ajanhetkellä. Datan tuottajalta silti vaaditaan hiukka suostuvaisuutta; tuottajan täytyy puskea muutokset globaaliin tietovarastoon.

Tietovaraston käytön voi haluttaessa yhdistää tavanomaiseen observer-järjestelmään. Tällöin kuluttaja-komponentti hakee viimeisimmän datatiedon *alustuksensa aikana*, ja tämän jälkeen *jää kuuntelemaan* päivityksiä dataan observer-järjestelmää hyödyntäen. Tämä ratkaisu on varsin toimiva monissa yhteyksissä.

> Esimerkki yhdistetystä *haku + observer* -ratkaisusta on vaikkapa chat-palikka, joka liitetään VueJS-sivustolle. Kun käyttäjä avaa chatin, on pulikan haettava keskusteluhistoria, jotta käyttäjä pääsee kärryille mistä keskustellaan. Avauksen jälkeen puolestaan on tarve saada live-päivityksiä, jotka kertovat uusien chat-viestien saapumisesta. Eli *alustuksen aikana haku, alustuksen jälkeen kuuntelu*. Tämä on erittäin yleinen toimintamalli.

> Globaali tietovarasto on esimerkissämme rakennettu erilliseen javascript-moduuliin. Toinen vaihtoehto on rakentaa se Vuen sisälle, esimerkiksi pluginin päälle. Kumpikin tapa saavuttaa kutakuinkin saman lopputuleman.