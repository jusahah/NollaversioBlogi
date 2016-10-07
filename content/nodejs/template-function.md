+++
date = "2016-10-07T16:09:02+03:00"
draft = false
title = "Lodash: template"

+++

Javascriptillä populoitavien mallipohjien (template) käyttö on etenkin frontend-koodauksessa varsin yleistä. Tyypillinen tarve mallipohjalle syntyy silloin, kun DOM-puuhun pitäisi lisätä uusi DOM-elementti, ja tuo elementti on rakennettava dynaamisesti.

> DOM-puu tulee sanoista "Document Object Model tree". DOM-puu kaikessa yksinkertaisuudessaan kuvaa hierarkisessa muodossa kaiken sen mitä nettisivu sisältää. Nettisivun tekstit, kuvat, videoelementit kaikki ovat osa tuota puurakennetta.

Elementtiä dynaamisesti rakennettaessa oleellista on, että pystymme injektoimaan haluttuun mallipohjaan sopivia tekstinpätkiä. Mallipohja sisältää näitä injektioita varten erikseen määritellyt "replace here"-kohdat.

Dynaamisen elementin rakentaminen mallipohjan pohjalta on konseptiltaan sama kuin sanaristikon täyttäminen. Ristikkointoilija täyttää ennaltamääriteltyihin laatikoihin kirjaimia. Vihjekuvat ovat aina samat - ne ovat osa mallipohjaa, tässä tapauksessa ristikkoa.

Omalla kohdallani perinteinen tapa toteuttaa HTML-mallipohjien käyttö on ollut turvautua *Handlebars*-kirjastoon. Tuo kirjasto hoitaa homman asiallisesti. Mutta jokunen aika sitten kävi ilmi, että myös Lodash-kirjasto hoitaa homman.

Ja koska käytän Lodashia muutenkin kovin runsaasti, oli suora motivaatio siirtyä heidän pariin tässäkin asiassa.

### template()-funktion käyttö

Lodashin *template()* apumetodi mahdollistaa tekstijonon tuottamisen *toisen tekstijonon pohjalta* seuraavaan tyyliin:

```javascript

var vieras1 = "Jaakko";
var vieras2 = "Kalle";

var pohja = _.template('Hei vain <%= nimi %>');

// Luodaan pohjan perusteella uusia tekstijonoja, joissa 
// nimi on dynaamisesti korvattu uudella tekstijonolla.

pohja({nimi: vieras1}); // "Hei vain Jaakko"
pohja({nimi: vieras2}); // "Hei vain Kalle"

```

Käyttö on tismalleen noin yksinkertaista. Ylläolevassa esimerkissä mallipohjan käytön hyöty ei ole merkittävä - yhtä hyvin voisimme tehdä seuraavalla tavalla:

```javascript

// Luodaan kumpikin tervehdys liimamalla tekstijonoja yhteen käsin.

"Hei vain " + vieras1; // "Hei vain Jaakko"
"Hei vain " + vieras2; // "Hei vain Kalle"

```

Tilanne muuttuu, kun mallipohjana toimiva tekstijono on pitkä, ja siihen on tehtävä useita tekstikorvauksia. Tällöin leikkaaminen + liimaaminen käsipelillä vie rutosti aikaa (ohjelmoijan aikaa, ei CPU-aikaa).

Lodashin template-metodi sisältää paljon ominaisuuksia. Se pystyy mm. tunnistamaan HTML-merkistön ja tekemään asianmukaiset merkistökoodaukset ("escape"). 

Lisätietoa löytyy doc-sivuilta: [https://lodash.com/docs/4.16.4#template](https://lodash.com/docs/4.16.4#template)





