+++
date = "2018-02-06T14:51:13+02:00"
draft = false
title = "Tapahtumakuuntelu mixinin kautta"

+++

Moni Vue-komponentti tarvitsee elinkaarensa aikana kyvyn reagoida muiden komponenttien tapahtumiin. Mikäli tapahtumiin reagoiva komponentti ja tapahtumia tuottava komponentti eivät ole suorassa parent-child -suhteessa, paras tapa välittää tietoa on erillisen Vue instanssin kautta, joka toimii keskitettynä viestikeskuksena, siis ikäänkuin radiomastona.

[Aiemmassa Vue-blogauksessani](https://www.nollaversio.fi/blog/public/vue/messaging/) annoin esimerkin komponentista (*Palolaitos*), joka kuuntelee viestikeskuksesta tulevia viestejä, ja komponentista (*Puukerrostalo*), joka ampuu viestejä viestikeskukseen. 

Tuossa esimerkissä viestinvälitysmekanismi oli koodattu suoraan komponenttien sisälle. Mutta suuressa web-applikaatiossa vastaavaa viestittelymekanismia joudutaan käyttämään useiden eri komponenttien kohdalla. 

Eli mikäli toinen komponentti haluaa ottaa vastaavan ratkaisun käyttöön, täytyy vastaava mekanismi koodata myös sinne.

Ongelma on, että kyseessä on puhdas duplikaatio; sama koodi ripotellaan kopioina eri puolelle koodipohjaa. 

> Yksi ohjelmoinnin kaikkein fundamentaalisimmista säännöistä on: älä duplikoi koodia ilman hyvää syytä.

Lähes kaikki ohjelmointikielet tarjoavat ratkaisun duplikaation torjumiseen, ja yleisin ratkaisu on funktionaalinen abstraktio; koodi laitetaan funktion sisään, ja koodin sijasta ripotellaan funktiokutsuja pitkin poikin koodipohjaa. Vue tarjoaa tähän lisämausteen mixin-konseptin avulla. 

Perusidealtaan Vuen mixin on hiukka samankaltainen kuin vaikkapa PHP:n puolella konsepti *trait*. Molemmat mahdollistavat koodin abstraktoinnin komponentin ulkopuolelle siten, että myös muut komponentit (tai luokat) voivat koodia hyödyntää. 

Yksi ero on, että siinä missä PHP:ssä trait on yksi vaihtoehto – joskus hyvä, joskus huono, useimmiten neutraali - perimiselle yläluokasta, Vuen puolella mixin on kutakuinkin ainoa järkevä tapa abstraktoida komponentin *lifecycleen liittyvä toiminta* ulos komponentista. Vuen moottori nimittäin osaa käsitellä mixineiksi merkityn koodin spesiaalilla tavalla, ja ikäänkuin *kutoa* sen yhteen komponentin oman koodin kanssa. 

> Teknisesti Vue yksinkertaisesti tunnistaa, että mixiniä käytetään, ja ajaa mixinin määrittelemät lifecycle-kuuntelijat (esim. created)juuri ennen komponentin omia lifecycle-kuuntelijoita. Huomionarvoista on, että komponentin oma kuuntelija ajetaan mixin-kuuntelijan *jälkeen*. 

Tämä “kutominen” mahdollistaa ratkaisun, joka ei yksinkertaisesti ole mahdollinen PHP:n traittia käyttäen; mixin voi määritellä metodin “created”, ja komponentti voi määritellä saman nimisen metodin “created”, ja kun komponentti ottaa mixinin käyttöön, Vue *kutoo* kaksi metodia yhteen ja ajaa metodikutsun seurauksena molempien metodien koodit. 

Tämä on fundamentaalisti eri asia kuin useimpien ohjelmointikielten (ml. Javascript, jonka päälle Vue rakentuu!) tapauksessa kun runtime-engine etsii ajettavaa metodia metodinimen perusteella. Esimerkiksi PHP:n puolella luokka ja yläluokka voivat määritellä saman nimisen metodin, mutta koodinajon aikana vain yhden metodin sisältämä koodi ajetaan. Yksi metodi voi tietenkin kutsua toista metodia – tämä on mahdollista jopa silloin kun metodien nimet ovat tismalleen samat (esim. *parent::__construct*) - ,mutta tämä kutsuminen on erikseen kirjoitettava koodiin. Vuen hienous on, että Vuen moottori tekee kutsun automaattisesti lifecycle-kuuntelijoiden kohdalla.

Yhtäkaikki, mixin näyttää tältä:


```javascript

// mixins/eventListeningMixin.js

import {EventBus} from '@/services/eventbus'
import _ from 'lodash'

export default {

  data() {
    return {
      // vakiona tyhjä objekti -> ei kuuntelijoita
      eventBusListeners: {}
    }
  },

  beforeDestroy() {
    // Komponentti tuhoutumassa, poista kuuntelijat.
    _.forOwn(this.eventBusListeners, function(fun, eventName) {
      EventBus.$off(eventName, fun);
    });  
    
  },
  created() {
    // Komponenttia alustetaan, aseta kuuntelijat
    _.forOwn(this.eventBusListeners, function(fun, eventName) {
      EventBus.$on(eventName, fun);
    })
  }
}

```

Itse EventBus - **joka on hienosti wrapattu mixinin sisälle piiloon** - on yksinkertaisesti erillinen Vue-instanssi:

```javascript

// services/eventbus.js

import Vue from 'vue';
export const EventBus = new Vue();

```

Kun olemme luoneet mixinin, mikä tahansa komponentti voi ottaa tuon mixinin käyttöönsä. Käyttöönotto on helppoa; rekisteröi mixinin ja asettaa halutun eventBusListeners-objektin, joka kytkee tapahtumanimet kuuntelijametodeihin. Oleellista on, että mixin hoitaa kaiken koordinnoin taustajärjestelmien kanssa - komponentin sisällä voimme koodata *deklaratiivisesti* eli meidän ei tarvitse huolehtia algoritmeista, joita tapahtumakuuntelu käyttää.

Esimerkkinä komponentti Dashboard.vue:

```javascript

// components/Dashboard.vue

import API from '@/api'
import eventListening from '@/mixins/eventListeningMixin'

export default {

  name: 'Dashboard',
  mixins: [
    eventListening
  ],  
  data() {
    return {        

      // Komponentin oma data
      leads: null,
      

      // Tapahtumakuuntelijat mixiniä varten
      // Tämä ylikirjoittaa mixinin oman kuuntelijaobjektin (defaulttina tyhjä objekti)
      eventBusListeners: {
        open_lead_acquired: this.eventReloadOpenLeads.bind(this),
        own_lead_closed: this.eventReloadOwnLeads.bind(this),        
      }, 

    }
  },
  created() {
    console.log("Dashboard luotu");
    return API.leads.fetch('include=events,reminders,calls,emails')
    .then((leads) => {
      this.leads = leads;
    });

    //...jne
  },
  methods: {
    eventReloadOpenLeads() {
      // Lataa avoimet liidit uusiksi rajapinnasta
    },

    eventReloadOwnLeads() {
      //... lataa omat liidit uusiksi rajapinnasta
    },

  }
}  

```


Parasta tässä on, että kiitos Vuen kutomisen, viestikuuntelija-mixiniä käyttävän komponentin ei tarvitse pelätä, että mixin vaikuttaisi itse komponentin omien lifecycle-hooksien toimintaan. Ne toimivat tismalleen samoin kuin ilman mixiniä.

> Esimerkissä emme määrittäneet varsinaisia business-metodeja mixinin sisään. Toisin kuin lifecycle-kuuntelijoiden tapauksessa, Vue ei "kudo" mixinin ja komponentin saman nimisiä business-metodeja yhteen. Sen sijaan mixinin metodi yksinkertaisesti kipataan roskiin, ja komponentin metodi jää käyttöön. 
>
>
>
> Tämä koskee siis vain tilanteita, joissa komponentin ja mixinin määrittämällä metodilla on tismalleen sama nimi.






