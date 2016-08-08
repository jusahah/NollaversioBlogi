+++
date = "2016-08-08T06:45:45+03:00"
draft = false
title = "Kolme tapaa lukea tiedosto (Node.js)"

+++

Tiedoston lukeminen on varsin yleinen toimenpide Node.js-applikaatiossa. Alla esittelen lyhyesti kolme tapaa hoitaa luku-urakka.

### Synkronoitu, bufferoitu

Synkronoitu ja bufferoitu tiedostoluku tapahtuu seuraavanlaisesti:

```javascript

var fs = require('fs')

var tiedostonSisalto = fs.readFileSync('tiedosto.txt', 'utf8');

```

Ylläoleva koodi on äärimmäisen yksinkertainen. Luemme tiedoston nimeltä *tiedosto.txt* ja tallennamme sen sisällön *tiedostonSisalto*-muuttujaan. Määritämme erikseen vielä, että tiedoston on enkoodattu utf8-merkistöllä, jotta voimme käsitellä sisältöä oikein.

Ongelmaksi ylläolevassa muodostuu se, että lukeminen on **synkronoitu** toimenpide. Toisin sanoen, koko Nodejs:n runtime seisoo tyhjän panttina sen aikaa, kun tiedoston lukeminen kovalevyltä on käynnissä. Näin ei tarvitsisi olla, mutta ylläolevassa näin on. 

Toimintojen suoritusjärjestys on kutakuinkin seuraava:

1. Node.js-prosessi pyytää käyttöjärjestelmää avaamaan tiedoston.
2. Käyttöjärjestelmä pyytää kovalevyä (sen ajureita) suorittamaan luvun.
3. Tiedoston luku tapahtuu
4. Käyttöjärjestelmä informoi Node.js-prosessia onnistuneesta luvusta.
5. Node.js-prosessi jatkaa suoritustaan.

Ongelman ydin on siinä, että vaiheiden 2-4 ajan Node.js-prosessi seisoo paikallaan.  

### Asynkronoitu, bufferoitu

Ylläoleva ongelma ratkeaa suorittamalla tiedostonluku **asynkronoidusti**.

```javascript

var fs = require('fs')

fs.readFile('tiedosto.txt', 'utf8', function(err, tiedostonSisalto) {
	console.log(3)
});

console.log(1);
console.log(2);

```
Sen sijaan, että määrittäisimme paluuarvon sisälleen ottavan muuttujan, syötämme tiedoston lukemisesta vastaavaan Node.js-funktioon kyytipojaksi callback-funktion. Callback-funktio nimensä mukaisesti sitten joskus tulee kutsutuksi, saaden parametrinään tiedoston sisällön.

Huomioitavaa on, että nyt Node.js-runtime **ei seiso** tyhjän panttina tiedostoluvun aikana. Sen sijaan tiedoston lukeminen fyysiseltä kovalevyltä tapahtuu yhtäaikaa Node.js-prosessin koodinajon kanssa. 

Ylläolevassa koodissa *console.log()*-komennot kuvaavat eri koodirivien suoritusjärjestystä. Callback-funktion sisällä oleva lokkaus tapahtuu viimeisenä.

Asynkronoitu ja bufferoitu ratkaisu on pätevä, ja yleisesti käytössä. Mutta entä jos emme halua bufferoida koko tiedostoa keskusmuistiin? Jos tiedoston koko on esimerkiksi 20 gigatavua, meillä ei riitä keskusmuistissa edes tila:

```javascript

FATAL ERROR: CALL_AND_RETRY_0 Allocation failed - process out of memory

```

Ratkaisu on striimata tiedosto pienissä paloissa.


### Asynkronoitu, streamattu

Kolmas ja viimeinen tapa hoitaa tiedoston lukeminen on *striimata* tiedoston sisältö pienissä pätkissä kerrallaan. Tällöin kerrallaan keskusmuistissa on vain pieni osa tiedostoa; kun tuo osa on käsitelty, seuraava osa voi ottaa sen paikan.


```javascript

var fs = require('fs');
var readStream = fs.createReadStream('tiedosto.txt');

readStream.on('data', function(chunk) {
	console.log(chunk);
});

```
Ylläolevassa ratkaisussa määritämme callback-funktion, jonka lähetämme kyytipojaksi tiedoston lukemisesta vastaavaan järjestelmäfunktioon. Tässä suhteessa ratkaisu on identtinen #2 ratkaisun kanssa.

Mutta ero #2 ratkaisuun tulee siinä, mitä tuo callback ottaa parametrinään sisälle. Kakkosratkaisussa koko tiedoston sisältö tuli parametrinä sisään. Nyt tulee vain *pieni osa* tiedostoa - sen sijaan callback-funktiota kutsutaan **uudelleen ja uudelleen* niin kauan, kunnes koko tiedosto on pala palalta käsitelty.

Striimauksen ongelma on, että saamme palat yksitellen. Jos siis haluamme **uudelleenrakentaa** tiedoston sisällön eheänä kokonaisuutena, meidän täytyy erikseen yhdistää nuo palat yhteen. 

> Loppukaneetti: yllä kolme yleisintä tapaa hoitaa tiedoston lukeminen ja käsittely Node.js-applikaatiossa. Eri tavat sopivat eri ongelmiin:

> 1. Synkronoitu ja bufferoitu sopii hyvin Node.js-applikaation initialisaatiovaiheeseen (eli kun applikaatio käynnistyy). Initialisaation aikana applikaatio rakentuu keskusmuistiin, ja varsinaisia loppukäyttäjien HTTP-kutsuja ei vielä käsitellä. Tästä syystä synkronoidun kutsun negatiiviset vaikutukset ovat vähäiset. Myöhemmin applikaation normaalin toiminnan aikana synkronoitu kutsu hidastaa merkittävästi applikaation vasteaikaa.

> 2. Asynkronoitu ja bufferoitu sopii erinomaisesti pienten tiedostojen lukemiseen applikaation varsinaisen toiminta-ajon aikana. Applikaatio ei jumahda paikalleen, vaan pysyy käyttökelpoisena ja suorituskykyisenä.

> 3. Asynkronoitu ja striimattu sopii suurten tiedostojen lukemiseen. Se sopii myös tapauksissa, joissa tiedoston sisältö voidaan pala palalta lähettää eteenpäin, esim. loppukäyttäjän HTTP-yhteyteen. 




