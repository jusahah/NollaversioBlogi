+++
date = "2016-08-02T11:10:16+03:00"
draft = false
title = "Työkalupakin kätköistä - zip() ja unzip()"

+++

Tällä kertaa esittelen lyhyesti maanmainion Lodash-kirjaston apufunktiot **zip** ja **unzip**.

Kuten nimistä saattaa kyetä päättelemään, zip ja unzip tekevät päinvastaisia asioita. Ne ovat loogisesti toistensa käänteisfunktioita:

``` a === unzip(zip(a)) ```

>Käytännössä ylläoleva koodi ei toimi, sillä unzip palauttaa listan, mutta zip ei ota vastaan listaa. Teknisesti ne eivät ole täysin yksi yhteen toistensa käänteisoperaatioita, mutta loogisesti niitä voi ajatella toistensa käänteisfunktioina.

Mitä nuo funktiot saavat aikaan? 

#### Zip

Zip-funktio ottaa vastaan kasan listoja, ja luo ryhmityksen kunkin listan n:nnelle jäsenelle. Huomioitavaa on, että zip ottaa listat vastaan yksitellen omina parametreinaan. Zip-operaation sisällä kaikki listojen 'ännännet' jäsenet ryhmitellään yhteen omaksi listakseen.

Tarve zip-funktion kaltaiselle apufunktiolle ilmenee erinomaisesti seuraavasta.

Esimerkki: meillä on kolme henkilöä, esim. työpaikan työntekijöitä. Kullakin työntekijällä on pituus ja paino. Työpaikan terveystutkimuksen osana tulee selvittää pituus- ja painojakaumat. Zip-funktio mahdollistaa tämän selvityksen luomisen vaivatta.

```javascript

// Luodaan henkilöitä.
// Henkilö määritetään pituuden (cm) ja painon (kg) mukaan kahden elementin listana!
var matti = [168, 67];
var mikko = [179, 76];
var pirjo = [154, 51];

// Käytetään zip-apufunktiota, joka ryhmittelee pituudet ja painot erillisiin listoihin.
var jakaumat = _.zip(matti, mikko, pirjo);
var pituusjakauma = jakaumat[0];
var painojakauma = jakaumat[1];

```

#### Unzip

Siinä missä **zip** ryhmittelee kasan listoja kunkin listan n:nnen jäsenen mukaan, **unzip** ottaa vastaan ryhmitykset sisältävän listan ja uudelleenkokoaa alkuperäiset listat. Unzip on siis suoraan zip-funktion käänteisoperaatio. 

Esimerkki: sääasemat ympäri Suomea mittaavat lämpötilan kerran päivässä, aina klo 18.00. Kerran viikossa kukin sääasema lähettää omat mittaustuloksensa keskuspalvelimelle. Keskuspalvelimen puolella meteorologi on kiinnostunut Suomen keskilämpötilasta kunakin viikonpäivänä. Unzip-operaatiolla tuo koko maan keskilämpötila on helppo selvittää kullekin viikonpäivälle:

```javascript
// Yksittäisen aseman tulokset muotoa [ma,ti,ke,to,pe,la,su]
var mittaustulokset = [
  [12,16,16,14,18,12,12], // Muonio
  [16,16,17,17,15,16,19], // Kuopio
  [20,20,18,20,21,23,21], // Tampere
];

var lampotilatPaivittain = _.unzip(mittaustulokset);
// Käytetään _.mean-apufunktiota joka laskee listan jäsenten keskiarvon.
var keskiarvot = _.map(lampotilatPaivittain, _.mean);

// Keskiarvot sisältää nyt kunkin viikonpäivän keskilämpötilan Suomessa.
console.log(keskiarvot); // [16, 17.33, 17, ...]

```

Otetaan vielä vertailun vuoksi miltä lämpötilojen jaottelu päivälokeroihin näyttäisi *ilman* unzip-funktiota:

```javascript
/////////////////
//  Ilman zip  //
/////////////////

// Luodaan lista joka sisältää oman keruulistan kullekin viikonpäivälle.
var lampotilatPaivittain = [[], [], [], [], [], [], []];

// Kaksi sisäkkäistä for-looppia, toinen luuppaa asemia, toinen viikonpäiviä.
for (var i = 0, j = mittaustulokset.length; i < j; i++) {
  for (var i2 = 0; i2 < 7; i2++) {
    lampotilatPaivittain[i2].push(mittaustulokset[i][i2]);    	
  }
}

////////////////////
//  Zipin kanssa  //
////////////////////

var lampotilatPaivittain = _.unzip(mittaustulokset);


```
Ero on - kuten niin kovin usein ohjelmoinnin piirissä - kuin yöllä ja päivällä.

> Unzip- ja zip-funktiot ovat mukava pieni lisä ohjelmoijan työkalupakkiin. Vastaavan algoritmin kirjoittaminen käsin ei houkuta.










