+++
date = "2018-01-31T11:37:33+02:00"
draft = false
title = "Komponenttipuun kontrollointi URL:n timestampilla"

+++

Männä päivänä syntyi tarve ladata Vue:n reitittimeen kytketty Vue komponentti uusiksi ilman, että reitittimen toimintaa ohjaava *route* muuttuu. 

> Löyhä määritelmä: route on kytkös URL:n eli selaimen www-osoitteen ja käyttöliittymässä aktiivisena olevan näkymän välillä.

Tyypillisestihän Vuen reititin (router) automaattisesti päivittää komponentti-puun ajan tasalle mätsäämään sen hetkistä URL-rakennetta. 

Esimerkiksi:

*www.appi.fi/#/sahkopostit/{id}/liitteet* tuottaa komponenttipuun: **App.vue > Sahkoposti.vue > Liitelista.vue**

> App.vue on ylimmän tason *container*-tyyppinen komponentti, jonka sisään applikaation business-näkymät rakentuvat. Kaikki mahdolliset komponenttipuut sisältävät App-komponentin esi-isänään.

Kun nyt käyttäjä klikkaa applikaation menuvalikosta painiketta "Profiili", URL päivittyy:

*www.appi.fi/#/profiili*, joka tuottaa komponenttipuun **App.vue > Profiili.vue**.

Kun URL päivittyy, Vue automaattisesti hoitaa lifecycle-kontrollin *poistuville* ja *ilmestyville* komponenteille. Esimerkin tapauksessa poistuvia komponentteja ovat *Sahkoposti* ja *Liitelista*, ja ilmestyvä komponentti on *Profiili*. Osana tätä lifecycle-kontrollia komponenttien lifecycle-hooksit ajetaan, mikä mahdollistaa uuden komponentin populoinnin esimerkiksi palvelimelta haetulla datalla.

```javascript

// Sahkoposti.vue

<template>
  <div v-if="email">
    <h1>{{email.subject}}</h1>
    <span>{{email.content}}</span>
  </div>
  <h3 v-else>"Ladataan..."</h3>
</template>

<script>

import API from '@/api'

export default {
  
  name: 'Sahkoposti',
  props: ['id'],
  data() {
    email: null,
  },
  created() {
    return API.emails.single(id)
    .then((email) => {
      this.email = email;
    });
  }
}
</script>

```

Ylläoleva soveltuu hyvin datalle, joka ei koskaan muutu. Tällöin komponentin alustuksen aikana tehty hakureissu serverille riittää populoimaan komponentin sen *koko elinkaaren ajaksi*.

Mutta entä jos meillä on seuraavanlainen URL-reitti ja siihen kytketty komponentti?

*www.osakeseuranta.fi/#/osakekurssit/{id}*

```javascript

// Osakekurssi.vue

<template>
  <div v-if="osakekurssi !== null">
    <h1>Kurssi: {{osakekurssi}}</h1>
  </div>
  <h3 v-else>"Ladataan..."</h3>
</template>

<script>

import API from '@/api'

export default {
  
  name: 'Osakekurssi',
  props: ['id'],
  data() {
    osakekurssi: null,
  },
  created() {
    return API.osakekurssit.single(id)
    .then((osakekurssi) => {
      this.osakekurssi = osakekurssi;
    });
  }
}
</script>

```

Haemme jälleen osakekurssidatan komponentin alustuksen yhteydessä, mutta ongelmana on, että osakekurssilla on tapana muuttua. Varsin nopeasti ja usein. Haettu osakekurssi on tarpeeksi hyvä approksimaatio *oikeasta osakekurssista* ehkä muutaman sekunnin ajan; sen jälkeen se on *vanhentunutta* tietoa. Vanhentunut tieto on arvotonta tietoa.

Yksi tapa on tehdä jonkinlainen push-notifikaatioihin perustuva järjestelmä, jossa uusin osakekurssi ammutaan fronttiin aina parin sekunnin välein:


### Ratkaisu 1: Push-notifikaatiot

```javascript

<script>

import PusherWrapper from '@/inbound/PusherWrapper'

export default {
  
  name: 'Osakekurssi',
  props: ['id'],
  data() {
    osakekurssi: null,
    cb: null
  },
  created() {
    this.cb = (viimeisinKurssi) => {
      this.osakekurssi = viimeisinKurssi;
    }.bind(this);

    return PusherWrapper.subscribe('osakkeet', id, this.cb);
  },
  beforeDestroy() {
    PusherWrapper.unsubscribe(this.cb);
  }
}
</script>

```

Toimiakseen komponentti vaatii tietenkin taustajärjestelmien olemassaolon, kuten (esimerkin tapauksessa) Pusher-tilin. Myös osakekurssidatan sisältävän palvelimen täytyy muuttua; sen täytyy ampua tietoa Pusherin suuntaan harvasen sekunti.

Ratkaisu on varsin monimutkainen. Mikäli esimerkiksi viiden sekunnin viive datan saannissa on OK, seuraava ratkaisu on parempi:

### Ratkaisu 2: Polling API

```javascript

<template>...</template>

<script>

import API from '@/api'

export default {
  
  name: 'Osakekurssi',
  props: ['id'],
  data() {
    osakekurssi: null,
    // Polling-järjestelmän apumuuttujat
    poller: null,
    // Viimeisimpänä lähteneen requestin järjestysnumero
    requestNum: 0,
    // Suurin järjestysnumero jolle saatu vastaus palvelimelta.
    latestResponse: 0
  },
  created() {
    // Ensihaku
    this.haeData(this.requestNum++);
    // Luo looppi joka hakee dataa 5 sek välein
    this.poller = setInterval(() => {
      this.haeData(this.requestNum++); 
    }, 5000);
  },
  methods: {
    haeData(currentRequestNum) {
      API.osakekurssit.single(id)
      .then((osakekurssi) => {
        if (this.latestResponse > currentRequestNum) {
          // Responsella kesti liian pitkään saapua. 
          // Tuoreempaa dataa on jo ehtinyt saapua, joten
          // tällä datalla emme tee yhtikäs mitään.
          return;
        }
        this.osakekurssi = osakekurssi;
      });
    }
  },
  beforeDestroy() {
    clearInterval(this.poller);
    this.poller = null;
  }
}
</script>

```

Ylläoleva ratkaisu on siitä hyvä, että se ei vaadi muutoksia palvelimen puolelle eikä push-notifikaatioita. Yksinkertaisesti häiriköimme rajapintaa viiden sekunnin välein.

Huono puoli on, että viiden sekunnin intervalli on arbitraali, ja ei anna käyttäjälle mahdollisuutta vaikuttaa päivitystahtiin. Osakekurssien tapauksessa kustomoitua päivitys-kontrollia ei juuri tarvita (koska osakekurssit tuppaavat päivittymään orjallisen tasaisesti), mutta muissa yhteyksissä päivitystahdin parempi kontrollointi voi olla tarpeen. 

Lisäksi kiveen hakattu päivitystahti on suorastaan käyttäjän *ehdollistamista avuttomuuteen*. Kuin Albert Camuksen novellissa, käyttäjästä tulee passiivinen seuraaja, joka ei voi vaikuttaa mihinkään.

> Sidenote: vastakohta Albert Camuksen masennuskeskeiselle maailmankuvalle on Viktor Hugo, jonka novelleissa päähenkilöt omaavat vapaan tahdon, ja voivat aktiivisesti vaikuttaa oman elämänsä kulkuun.

### Ratkaisu 3: Datan uudelleenlataus nappia painamalla

Tämä ratkaisu sisältää vihdoin otsikon mukaisen ongelman. Esimerkin tapauksessa voimme tehdä *nappia painamalla refreshin* seuraavasti:

```javascript

<template>
  <div v-if="osakekurssi !== null">
    <h1>Kurssi: {{osakekurssi}}</h1>
    <button v-on:click="haeData">Päivitä</button>
  </div>
  <h3 v-else>"Ladataan..."</h3>
</template>

<script>

import API from '@/api'

export default {
  
  name: 'Osakekurssi',
  props: ['id'],
  data() {
    osakekurssi: null,
  },
  created() {
    this.haeData();
  },
  methods: {
    haeData() {
      API.osakekurssit.single(id)
      .then((osakekurssi) => {
        this.osakekurssi = osakekurssi;
      });
    }
  }
}
</script>

```

On kuitenkin huomattava, että käyttämämme esimerkki on nk. *toy example*. Oikeassa applikaatiossa päivitettävä komponentti saattaa sisältää valtavan alipuun täynnä muita komponentteja. Lisäksi päivitysnappula voi sijaita ihan toisaalla kuin itse osakekurssi-komponentin sisällä. 

Se mitä haluaisimme tehdä on hyödyntää olemassaolevaa Vue:n reititintä. Se tietää tismalleen mitä tehdä; kun saavumme esimerkiksi osoitteeseen *www.osakeappi.fi/#/osakekurssit/5*, reititin osaa rakentaa koko komponenttipuun osakekurssille, jolla on ID 5. 

> Yleisemmin: koska meillä on jo tapa *rakentaa koko komponenttipuu ja populoida* se reititintä käyttäen, olisi fiksua tehdä myös *puun uudelleenrakennus + uudelleenpopulointi* reititintä käyttäen. 

Törmäämme kuitenkin ongelmaan: kun vapaan tahdon omaava käyttäjä klikkaa osakekurssi-painiketta menuvalikossa, Vue oikeaoppisesti siirtyy uuteen URL:iin ja rakentaa uuden komponenttipuun. Mutta kun käyttäjä klikkaa samaa valikkopainiketta uudestaan, mitään ei tapahdu. 

Ongelma on, että Vuen reititin päivittää komponenttipuun *vain mikäli reitittimen havaitsema route muuttuu*. Käyttäjän klikatessa valikkonappulaa toisen kerran route ei muutu. Käyttäjän tavoitteena on saada samalle osakekurssille tuore kurssidata. Tuore data ja vanha data käyttävät kuitenkin samaa URL:iä, joten reititin ei ymmärrä päivittää komponenttipuuta, eikä täten created-hookit tule kutsutuksi.

Yksi ratkaisu on kutsua *$vm.forceUpdate()* jossain kohtaa, mutta näin toimiessa olemme matkalla kohti spagettikoodia; Vue reititin seuraa URL:ia, ja forceUpdate-ratkaisu sivuuttaa reitittimen toimintamekanismin. Oikeaoppisessa web-applikaatiossa *URL toimii käyttöliittymän näkymää kontrolloivana tilamuuttujana*, ja mieluiten *ainoana* näkymää kontrolloivana tilamuuttujana.

> Kyseessä on filosofinen ero *datan* ja *näkymän dataan* välillä. Ero on vastaava kuin vaikkapa tähden ja kaukoputken välillä. Matemaattisesti ero on vastaava kuin *X*:n ja *F(X)*:n välillä; ensimmäinen on objekti, jälkimmäinen on transformaatio eli funktio.
>
>
>
> Oikeaoppisessa web-applikaatiossa URL on tilamuuttuja sille *mitä/miten dataa näytetään*. URL on harvoin tilamuuttuja itse datalle. Ihan jo siksi, että suuren määrän dataa mahduttaminen selaimen osoiteriville on mahdotonta.
>
>
>
> Itse datan tilamuuttujana toimii useimmiten joko palvelin-rajapinta, lokaali tietovarasto (esim. local storage) tai yksinkertaisesti globaali Javascript-objekti.

Tämän periaatteen sivuuttaminen saattaa johtaa ongelmiin pitkässä juoksussa. Tai sitten ei johda. Mutta meillä on parempikin tapa, joten...

### Ratkaisu 4: Komponenttipuun kontrollointi URL:n timestampilla

Paras(?) ratkaisu ongelmaan on muuttaa URL:ia. Mutta miten? Emme voi muuttaa täysin arbitraalisti, koska osakekurssidatan sisältävä komponentti näkyy vain URL:in *www.osakeappi.fi/#/osakekurssit/{id}* alaisuudessa. Mitä siis voimme muuttaa?

Entä jos teemme näin:

*www.osakeappi.fi/#/osakekurssit/{id}?ts=772272727*

Lisäämällä query stringin URL:iin saamme muutettua URL:ia ilman, että itse komponenttipuun rakennetta ohjaava osuus URL:ista muuttuu. Nyt sitten ylätasolla (App.vue) teemme seuraavasti:

```javascript

// App.vue

<template>
  <router-view :key="$route.fullPath"></router-view>
</template>

<script>...</script>

```

```javascript

// Menu.vue

<template>
  <ul v-for="osake in osakkeet">
    <li v-on:click="openOsake(osake.id)">{{osake.name}}</li>
  </ul>
</template>

<script>
  export default {
    //...
    methods: {
      openOsake(id) {
        // Generoimme timestampin URL:in mukaan!
        // Koska timestamp muuttuu millisekunnin välein,
        // URL ei ole koskaan sama!
        this.$router.replace({ 
          name: "osakekurssi", 
          params: {id: id}, 
          query: {ts: Date.now()} 
        })
      }
    }
  }
</script>

```

Yllä oleelliset osuudet ovat App-komponentin *:key=$route.fullPath* ja Menu-komponentin *query: {ts: Date.now()}*. 

Ensin mainittu kytkee router-viewin kuuntelemaan URL:n muutoksia. Joka kerta kun URL *mukaan lukien query string* muuttuu, komponentin key-attribuutti muuttuu, ja se pakottaa komponentin uudelleenlatauksen.

Jälkimmäinen huolehtii, että joka kerta kun valikkopainiketta klikataan, generoitava URL on uniikki.

Yksi hyvä puoli ratkaisussa on, että nyt voimme automaattisesti kontrolloida päivitystahtia **suoraan URL:n tasolla**! Tämä on suorastaan upeaa, fantastista.

Ylläolevassa ratkaisussa URL muuttuu millisekunnin välein (koska Date.now palauttaa aikamääreen millisekunnin tarkkuudella). Mutta voimme yhtä hyvin tehdä URL muutoksen yhden sekunnin välein. Tällä tavalla käyttäjä ei voi ampua kuin maksimissaan yhden HTTP-requestin per sekunti vaikka kuinka näpyttelisi hurmiossa samaa valikkolinkkiä.

Toteutimme kuin vahingossa automaattisen *rate limitin*, siis.

```javascript

// Menu.vue

<template>
  <ul v-for="osake in osakkeet">
    <li v-on:click="openOsake(osake.id)">{{osake.name}}</li>
  </ul>
</template>

<script>
  export default {
    //...
    methods: {
      generateTs() {
        var now = Date.now();
        // Pyöristä sekunnin tarkkuuteen -> URL muuttuu keskimäärin
        // sekunnin välein vaikka kuinka naputtaisi li-elementtiä.
        return Math.round(now / 1000);
      },
      openOsake(id) {
        this.$router.replace({ 
          name: "osakekurssi", 
          params: {id: id}, 
          query: {ts: this.generateTs()} 
        })
      }
    }
  }
</script>

```

Tässä saavumme ihanteelliseen kompromissiin monen asian suhteen. Ensinnäkin käyttäjä olettaa, että kun hän kerran klikkaa osakkeen nimeä ja kurssi päivittyy, myöhemmät klikkaukset tekevät samoin. Tämä antaa käyttäjälle kontrollin. Samaan aikaan saamme kuitenkin toteutettua rate limiitin; koska kurssidataa ei ole järkeä hakea useammin kuin kerran sekunnissa, voimme *URL:n koostumusta* kontrolloimalla kontrolloida päivitystahtia. Kolmanneksi... koska URL:n muutos uudelleengeneroi koko router-viewin alaisen komponenttipuun, kaikki alikomponentit ladataan puhtaalta pöydältä. Tämä helpottaa synkronisaatio-ongelmia ison komponenttipuun eri haarojen välillä; kaikki haarat generoidaan uusiksi, ja vanha data häviää bittiavaruuteen.

Kaiken kaikkiaan toimintamalli muistuttaa jumalallista *RAII*-patternia C++:n puolelta. 

> RAAI = Resource acquisation is initialization 

> Opetus: pidä komponenttipuun rakenne ja päivityssykli riippuvaisena URL:ista, käyttäen tarvittaessa query stringiä generoimaan uniikki URL.