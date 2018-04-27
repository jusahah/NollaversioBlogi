+++
date = "2018-02-12T13:20:21+02:00"
draft = false
title = "Käännä kutsujärjestys mixinin avulla"

+++

### Vue and Mixins (osa 2): Vaihda kutsujärjestys

Viime viikolla käsittelin blogipostauksessa Vuen mixineitä. Ne mahdollistavat hienoja asioita, kuten useaa komponenttia koskevan koodin abstraktoinnin ikäänkuin yläluokkaan.

Vue mixinin yksi hienoimmista ominaisuuksista on lifecycleen liittyvien hooks-kutsujen kutominen; komponentin oman hookin lisäksi Vuen moottori kutsuu myös mixinin saman nimisen hookin (mikäli sellainen on määritelty).

Lähtökohtaisesti kutsujärjestys on mixin ensin, komponentti sitten. 

> Tämä tarkoittaa siis, että useaa komponenttia koskeva *yhteinen* koodi ajetaan ensin, ja komponentin oma *spesifi* koodi ajetaan jälkeen. 

Valittu järjestys on yhtäältä looginen, toisaalta epälooginen. 

Yhtäältä järjestys on äärimmäisen looginen. Ajamalla mixinin hooksin ensin saamme tehtyä vakioalustuksen ensin; mikäli komponentti on tyytyväinen vakioalustukseen, all is fine ja komponentin oman hooksin ei tarvitse tehdä mitään. Mikäli komponentti puolestaan tarvitsee "ylhäältä määrätyn" vakioalustuksen lisäksi jotain omaa, se voi turvallisesti määrittää omat data-muuttujansa luottaen siihen, että mahdolliset nimi-konfliktit ratkotaan komponentin omien määritysten hyväksi! 

Tämä luottamus toimii juuri siksi, että komponentin oma hook ajetaan viimeisenä, jolloin komponentti voi vapaasti ylikirjoittaa haluamansa data-muuttujan.

> Vastaava periaate on keskiössä kaikissa OOP-ohjelmointikielissä. Periaatteen ydinajatus on, että mitä *ylemmällä* abstraktion tasolla jokin luokkamuuttuja on määritelty, sitä *alempi* sen prioriteetti on. Eri tasojen eri prioriteetit varmistavat sen, että konfliktit on aina mahdollista ratkaista. 

Toisaalta järjestys on epälooginen. Entä jos haluamme luoda mixinin, joka loggaa komponentin alkutilan? Koska mixinin created-hook ajetaan ensin, ei mixinillä ole mitään logattavaa; komponentin oma created-hook ei ole vielä ehtinyt alustaa data-muuttujia haluttuihin alkuarvoihin, joten koko loggaus on yhtä tyhjän kanssa. 

Jotta loggaus toimisi oikein, on meidän saatava mixinin hook ajetuksi komponentin hookin *jälkeen*, ei ennen. Tällä tavalla mixinin created-hook näkee komponentin tilamuuttujan juuri sellaisina kuin mihin komponentin oma created-hook ne alusti.

Ongelmaan on ratkaisu:

### Vaihda kutsujärjestystä -mixin

```javascript

// CreationLogger.js

function privCreated() {
    console.log(JSON.stringify(this.$data)); 
}

export default {
  created: function () {
    setTimeout(privCreated.bind(this), 0);
  }
}

```

Ratkaisu perustuu siihen, että varsinainen toiminnallisuus siirretään pois mixinin created-hookista. Sen sijaan created-hook - joka ajetaan ennen komponentin created-hookia - kutsuu mixinin privaattia funktiota, joka sisältää varsinaisen business-koodin. Ja oleellista on, että kutsu ajastetaan tehtäväksi *Javascriptin event-loopin seuraavalla pyörähdyksellä (tick)*; tällä tavoin komponentin oman created-hook ehtii ajaa välissä. 

Kutsujärjestys ajon aikana siis on:

1. Mixinin created()
2. Komponentin created()
3. Mixinin privCreated()

Mixinin käyttöönotto tapahtuu normaaliin tapaan:

```javascript

import CreationLogger from '@/mixins/CreationLogger'
import Gen from '@/chess/PositionGenerator'

export default {
  name: 'ChessGame',
  props: ['players'],
  mixins: [CreationLogger],
  data() {
    return {
      
      position: '',
      startingTimes: null
    }
  },
  created() {

    this.position = Gen.getRandomStartPosition();
    this.startingTimes = [15, 15] // Minuuttia
    // ...
  }
}

```

Tärkeintä tässäkin ratkaisussa on, että itse komponentin ei tarvitse tietää hölkäsen pöläystä mixinin poikkeavasta toimintatavasta. 

> Loppuhuomautus: idea koodinajon siirtämisestä seuraavalle event-loopin pyörähdykselle on erittäin monikäyttöinen. Se on itse asiassa yksi tärkeimmistä (*lexical scoping*:in ja *prototyyppi*-ketjujen ohella) tekijöistä, jotka tekevät Javascriptista Javascriptin.
