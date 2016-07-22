+++
date = "2016-07-18T17:11:43+03:00"
draft = false
title = "Raskas koodi erillisessä säikeessä? Lupaus auttaa."

+++

Javascriptin hauska puoli on, että se ajaa itseään mukavasti yhdessä säikeessä. Tämä tarkoittaa, että kaikki
koodi ajetaan *perätysten*, kiltisti jonossa. 

Eli kun funktio **A** aloittaa ajonsa, funktio **B** ei voi aloittaa ennenkuin **A** on valmis. Täydellinen metafööri Javascriptille onkin McDonaldsin autokaistan jono - jos yksi autoilija jää suustansa kiinni noutotiskille, yksikään takana olevista autoista ei liiku senttiäkään.

Ohjelmoinnin maailmassa tämä tarkoittaa, että jos yksi funktio rohmuaa prosessorin ajoaikaa viisi sekuntia, kaikki muut ajovuoroa odottavat koodinpätkäset joutuvat vähintään tuon viisi sekuntia odottamaan.

Tämä kylmä totuus pätee sekä selaimen puolella että serverimaailmassa (Node.js).

Yksi tapa ratkoa jonotuksen tuomat haasteet on pitää huoli, että jono liikkuu vauhdilla. Mäkkärikin tekee näin - he pitävät huolen, ettei yksittäinen asiakas tuki koko autokaistaa, vaan siirtyy sutjakasti elämässään eteenpäin. Koodin puolella tämä on tehtävissä ohjelmoijan maalaisjärjen käytöllä - ohjelmoija arvioi parhaan kykynsä mukaan kuinka kauan kunkin koodinpätkän ajo kestää. 

Jos ajo kestää kaksi mikrosekuntia, ei ongelmia. 

Jos ajo kestää kaksi sekuntia, niin pulassa ollaan.

Mikä avuksi tilanteisiin, joissa yksittäinen koodinajo on pitkäkestoinen?

### Luo uusi säie, joka tekee raskaat työt

Ratkaisu on yksinkertainen - uusi työmyyrä (säie), joka ottaa kontolleen raskaan työurakan. Selaimessa Web Worker-standardi mahdollistaa säikeen luomisen. Muita *järkeviä* vaihtoehtoja ei juuri ole.

Serveripuolella (Node.js) on enemmän vaihtoehtoja. Yksi vaihtoehto on ajaa raskas koodi kokonaan uudessa Node.js-instanssissa. Eli uudessa käyttöjärjestelmän prosessissa, joka pyörittää Node.js-koodia.

Se voi olla ihan hyväkin idea, mutta aika raskas, sillä koko Node.js-moottori täytyy ladata uusiksi tätä uutta "säiettä" varten. Jos koodinajo on pitkäkestoinen, tällä ei ole juuri merkitystä.

Toinen vaihtoehto olisi käyttää "kevyempää säiettä", joka ajetaan jo olemassaolevan Node.js-prosessin alaisuudessa. Tällöin käyttöjärjestelmän ei tarvitse synnyttää uutta prosessia, vaan uusi prosessi syntyy kivasti käyttöjärjestelmän tietämättä asiasta mitään. 

Valitaan kuitenkin vaihtoehto yksi ihan siksi, että yksi parhaista *säiekirjastoista* turvautuu siihen.

### Threads -kirjasto ja lupaukset

Oletetaan, että meillä on koodinpätkä, joka etsii kaikki suomalaiset erisnimet tekstidokumentista. Skripti toimii seuraavasti:

```javascript

// etsiErisnimet.js

var ERISNIMET = ['Aado', 'Aamu', 'Aapo', ... , 'Yrjö'];

module.exports = function(dokumentti) {
	
  var sanat = dokumentti.split(" "); // Erottele välilyönnillä

  var nimet = _.filter(sanat, function(sana) {
    return ERISNIMET.indexOf(sana) !== -1; // Löytyikö sana nimiluettelosta?
  })

  // Poista duplikaatit
  // ['Mikko', 'Mikko', 'Matti'] -> ['Mikko', 'Matti']
  return _.uniq(nimet);
}

```

Algoritmi on kompleksisuudeltaan about *O(nk)*, jossa *n* kuvaa tekstin pituutta ja *k* etunimien lukumäärää. 
Ei ehkä ihan optimialgoritmi, mutta what the hell. Käyttö on helppoa.

```javascript

// Testi.js
var nimiEtsinta = require('etsiErisnimet');

var nimet = nimiEtsinta('Pirkko ja Ville menivät kalaan.');
console.log(nimet) // ['Pirkko', 'Ville']

```

Seuraavaksi katsotaan, miten tuo algoritmi saadaan ajettua erillisessä säikeessä.

Ensinnäkin tarvitsemme säiekirjaston. Sen voi asentaa *npm install threads --save* -komennolla komentorivillä.
Tämän lisäksi on syytä tehdä pieni muutos etsiErisnimet.js-tiedostoon, jotta se pystyy toimimaan threads-kirjaston kanssa. Muuta ei tarvita. 

Sitten itse koodi. Huomattavaa on, että *paketoimme* aiemman erisnimien etsintään erikoistuneen koodin siten, että se voidaan ajaa säikeen sisällä.

```javascript

// etsiErisnimetThreaded.js

// Tämä moduuli toimii yksinomaan wrapperinä.

var threads = require('threads'); // Säiekirjasto
var Promise = require('bluebird'); // Lupauskirjasto

module.exports = function(dokumentti) {

  return new Promise(function(resolve, reject) {
    // Luodaan uusi säie
    // Spawn-funktio ottaa parametrikseen sen moduulin nimen, 
    // jonka koodin säie ottaa ajaakseen.
    var thread = threads.spawn('etsiErisnimet');

    // Säie on luotu pinnan alla ja valmis toimimaan.
    // Lähetetään säikeelle viesti
    thread.send(dokumentti)
    // ...ja jäädään kuuntelemaan vastausta
    .on('message', function(loydetytErisnimet) {
      // Resolvoidaan lupaus saaduilla tuloksilla.
      return resolve(loydetytErisnimet);
    })
    .on('error', function(error) {
      // Rejektoidaan lupaus
      return reject(error);
    });

  });

}

```

```javascript

// etsiErisnimet.js

// Aiempi erisnimien etsintä toimii kuten ennenkin, mutta
// tarvitsemme hiukan lisäkoodia hallitsemaan tiedonvaihtoa
// säikeiden välillä.

var ERISNIMET = ['Aado', 'Aamu', 'Aapo', ... , 'Yrjö'];

module.exports = function(dokumentti, takaisinlahetys) {

  var sanat = dokumentti.split(" "); // Erottele välilyönnillä

  var nimet = _.filter(sanat, function(sana) {
    return ERISNIMET.indexOf(sana) !== -1; // Löytyikö sana nimiluettelosta?
  })

  // Poista duplikaatit
  // ['Mikko', 'Mikko', 'Matti'] -> ['Mikko', 'Matti']
  takaisinlahetys(_.uniq(nimet));
}

```

Ylläoleva koodi on kaikki mitä tarvitsemme. Nyt voimme suorittaa erisnimietsinnän täysin erillään omassa säikeessään!

```javascript

// RikosSeurantaApplikaatio.js

var Promise = require('bluebird');
var nimietsinta = require('etsiErisnimetThreaded');

function vastaanotaDokumentti(dokumentti) {
	
  nimietsinta(dokumentti)
  .then(function(loydetytNimet) {
    return tarkistaEpailyttavatNimiParit(loydetytNimet);
  })
  .catch(function(error) {
    console.log("Nimien etsintä epäonnistui");
    console.error(error);
  })
}

function tarkistaEpailyttavatNimiParit(nimet) {
  if (_.intersection(['Ilkka', 'Kanerva'], nimet).length === 2) {
    // Sekä Ilkka että Kanerva löytyivät, soita Karhuryhmä.
  }
}

```

Tällä tavoin olemme kivasti paketoineet säikeidenhallinnan ikävät sivuseikat lupausta tarjoavat abstraktion taakse. *RikosSeurantaApplikaation* ei tarvitse välittää tuon taivaallista säikeiden olemassaolosta - riittää, että se kutsuu tarjottua rajapintaa ja ottaa vastaan *lupauksen*. 

Tuo lupaus sitten jonain kauniina aamuna realisoituu todeksi, ja kaikki ovat tyytyväisiä.



