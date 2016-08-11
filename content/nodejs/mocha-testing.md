+++
date = "2016-08-11T06:38:25+03:00"
draft = false
title = "Asynkronoidun koodin testaus (Mocha)"

+++

Rakensin eilen *PromiseMonopoly*-nimistä ohjelmistokehystäni jälleen hiukan eteenpäin. Kehys on siinä pisteessä, että on syytä kirjoittaa muutamia yksinkertaisia automatisoituja testejä sille.

Piskuiseksi ongelmaksi muodostui, että koska kutsut kehyksen sisälle ovat *asynkronoituja* - eli palauttavat lupauksen -, testaaminen täytyy myös tehdä asynkronoidusti.

Maanmainio [Mocha](https://mochajs.org/) tuli tässä kohtaa apuun.

### Async-testi Mochalla

Kirjoitin kehykselleni allaolevan testin:

```javascript

describe('Phase', function() {
  describe('onEnter + onExit', function() {
    it('Phase with empty subphases goes correctly', function(done) {
      var tracking = [];
      var testiphase = new Phase('testi', {loop: false}, []);
      testiphase.onEnter = function() {
        tracking.push("START");
      }

      testiphase.onExit = function() {
        tracking.push("STOP");
      }
      testiphase.__initialize({}, [new Player(whiteUser), new Player(blackUser)]);

      testiphase.__start()
      .then(function() {
        expect(tracking).to.deep.equal(['START', 'STOP']);
        done();
      })
    })
  })
})	

```

Ylläoleva testi varmistaa, että *onEnter*- ja *onExit*-kutsufunktiot tulevat kutsutuksi kehyksen toimesta oikeassa järjestyksessä. Eli kutsuessamme **testiphase.__start()**, myöhemmin meillä on tracking-listassa viestit "START" ja "STOP" peräkkäin.

``` expect(tracking).to.deep.equal(['START', 'STOP']); ```

Asynkronoidun testauksen ytimessä Mochalla on tämä koodirivi:

 ```javascript

 it('Phase with empty subphases goes correctly', function(done) {

 ```

 Oleellista ylläolevassa rivissä on **done**-parametri, jonka testiajon suorittava funktio ottaa vastaan. Mikä tuo mystinen **done** sitten on? *Se on parametri on funktio, jota kutsumalla testi julistetaan suoritetuksi.*

 Toinen tärkeä on tämä rivi:

 ```javascript

 expect(tracking).to.deep.equal(['START', 'STOP']);

 ```

**Expect**-kutsulla suoritamme varsinaisen testin, eli varmistamme, että ohjelma-ajon tuottama tulos on haluttu. 

On syytä huomata, missä tämä expect-kutsu sijaitsee; se on lupausketjun viimeisen *then*-metodin sisällä! Tämä tarkoittaa, että varsinainen testaus suoritetaan vasta kun lupausketju on siirtynyt viimeiseen vaiheeseensa. Muita vaihtoehtoja suorittaa testaus ei ole, sillä testauksen kannalta relevantit operaatiot suoritetaan lupausketjun aiemmissa vaiheissa.

Seuraava esimerkillinen testi EI toimi kuten haluamme:

```javascript

// Ei toimi, async ja sync sekoitettuna!

describe('Matikka', function() {
  describe('Yhteenlaskut', function() {
    it('2+2=4', function() {
      var summa = laskeAsync(2, 2);
      expect(4).to.equal(summa);
    })
  })
})	

```

Ylläoleva ei toimi juuri siksi, että **laskeAsync** on (nimensä mukaisesti) asynkronoitu funktio. Se ei voi palauttaa haluttua lukua, sillä asynkronoidut funktiokutsut eivät tiedä lopputulosta ajoissa. Tässä tapauksessa oletamme, että **laskeAsync** suorittaa yhteenlaskun vaikkapa kysymällä Googlen serveriltä lopputulosta. Tuo lopputulos saapuu sitten joskus, riippuen nettiyhteyden nopeudesta.

Eli ongelma on, että muuttuja *summa* ei ole ajoissa tiedossa.  

Ongelma on helppo korjata, ja muuntaa testaus asynkronoituun muotoon:

```javascript

// Toimii, 100% async!

describe('Matikka', function() {
  describe('Yhteenlaskut', function() {
    it('2+2=4', function(done) {
      laskeAsync(2, 2).then(function(summa) {
        expect(4).to.equal(summa);
        done();
      })
			
    })
  })
})	

```

Nyt homma pelittää virheettömästi. Kutsumme **laskeAsync**-funktiota, jota palauttaa lupauksen. Kun tuo lupaus täyttyy (*then()*), meillä on haluamamme *summa* saatavilla ja voimme varmistaa **expect**-kutsun avulla, että tuo summa on neljä.

Suoritettuamme **expect**-testin kutsumme funktiota *done*, joka ilmoittaa Mochalle, että testaus on tältä osalta valmis. Miksi tuota done-funktiota pitää erikseen kutsua? 

Synkronoidussa versiossa ei tarvitse. Tämä siksi, että Mocha voi olettaa testauksen olevan valmis heti kun kooditiedosto on ajettu kerralla loppuun. Eli siis ollaan saavuttu viimeiselle koodiriville. 

Mutta asynkronoidussa versiossa Mocha ei voi tehdä tuollaisia rämäpäisiä oletuksia. Osa testauskoodista saattaa odottaa vuoroaan. Meidän esimerkissämme näin tekee Googlen palvelimelta yhteenlaskun tulosta odottava koodipätkä. Tällöin Mocha ei voi vain julistaa testejä suoritetuksi heti kun testitiedoston viimeinen koodirivi on nähty ja ajettu; testit ajetaan *myöhemmin* ja on syytä jäätä odottamaan testien tuloksia. Done-funktion käyttö mahdollistaa odotuksen - kukin yksittäinen testi ilmoittaa oman done-funktionsa kautta milloin se on valmis.








