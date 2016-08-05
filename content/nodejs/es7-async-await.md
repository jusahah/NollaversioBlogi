+++
date = "2016-08-05T06:02:06+03:00"
draft = false
title = "Async ja Await"

+++

Kokeilin eilen ensimmäistä kertaa *async-await* -funktion kirjoittamista Javascriptilla. Kyseessä on uusi ja mullistava tapa ohjelmoida asynkronoidusti toimivia funktioita.

Vastaava tapa on ollut esim. C#-kielessä jo kauan, mutta Javascriptiin ominaisuus on vasta tuloillaan. Se ei ole vielä virallisesti osana Javascript-kieltä. Mutta [Babelin](https://babeljs.io/) kaltaisten koodimuuntajien avulla tuota ominaisuutta pääsee testaamaan jo tänään.

> Async-await ei ole virallisesti tuettu ominaisuus. Tuota ominaisuutta voidaan kuitenkin *simuloida*. Simulointi onnistuu yksinkertaisesti siten, että async ja await-avainsanoja sisältävä Javascript-koodi käännetään koodiksi, joka ei sisällä async ja await-avainsanoja, mutta toteuttaa vastaavat toiminnot muulla tavoin.

### Minkä ongelman async-await ratkoo?

Kerrataanpa tuikitavallisen funktion määritys ja kutsuminen:

```javascript

function hae(hetu) {
  // Meillä on jossain muualla määritelty 'henkiloTietokanta'
  // niminen Javascript-objekti.
  return henkiloTietokanta[hetu];
}

var henkilo = hae('010787-111A');
console.log(henkilo.nimi); // Jaakko Jantunen

```

Ylläoleva on perinteinen, synkronoitu Javascript-funktio. Koodi määrittää funktion, ja tämän jälkeen kutsuu tuota funktiota. Funktiokutsun seurauksena saadaan takaisin *Henkilö-objekti*, jonka attribuutti *nimi* printataan käyttäjälle.

Vastaavat funktiot ovat arkipäivää kaikissa yleisimmissä ohjelmointikielissä. 

Nyt kysymys kuuluu: entä jos *hae*-funktio joutuisikin hakemaan henkilön tiedot tietokoneen kovalevyltä?

Vaatimus ei ole poikkeuksellinen, päinvastoin. Henkilötietokanta sisältää miljoonia rivejä tietoa. Tuollaisen tietomäärän pitäminen yksinomaan keskusmuistissa (=Javascript-objektin sisällä) on kutakuinkin mahdotonta. Entä jos palvelin joku kaunis päivä kaatuu? Kaikkien henkilöiden tiedot katoaisivat savuna ilmaan!

Meidän on siis pakko tallettaa henkilötiedot kovalevylle.

Nyt naivisti voisimme yrittää seuraavaa:

```javascript

// HUOM! Tämä yritelmä ei toimi halutusti!

function hae(hetu) {
  // Meillä on nyt kovalevyllä 'henkiloTietokanta' tiedosto!
  // Ladataan ensin tietokanta keskusmuistiin.
  var tietokanta = kovalevy.lue('henkiloTietokanta');
  // Haetaan haluttu henkilö. Ei toimi kuten haluamme!
  return tietokanta[hetu];
}

var henkilo = hae('010787-111A'); //Error! Cannot read property of undefined!

```

Ylläoleva koodi ei toimi. Miksi? **Koska tiedoston lukeminen kovalevyltä keskusmuistiin on asynkronoitu operaatio.** Tämä tarkoittaa, että tiedoston lukeminen keskusmuistiin vie sen verran kauan aikaa, että Javascript-koodiajo siirtyy eteenpäin.

> Vastaava tilanne syntyy meidän ihmisten elämässä usein. Esimerkkinä pankissa asiointi. Astut sisälle pankkiin ja otat vuoronumeron. Edelläsi jonossa on viisitoista henkilöä. Jäätkö kiltisti odottamaan pankin odotustilaan vai käytkö välillä vaikka lounaalla? Jos odotat varvastakaan liikuttamatta, **toimit synkronoidusti**. Jos käyt muilla asioilla ja palaat paikalle omaa vuoroasi varten myöhemmin, **toimit asynkronoidusti**.
>
> Jotta ylläoleva analogia toimisi vielä täsmällisemmin, pankin tulisi *ilmoittaa* sinulle kun jono etenee vuoronumerosi kohdalle. Tälläinen järjestelmä on käytössä Helsingin Grand Casinolla - aloittaessasi jonotuksen pokeripöytään, saat taskuusi mukaan piipparin, joka ilmoittaa heti kun pöydässä on tilaa. Jonotuksen ajan voit huoletta törsätä pikkurahat kolikkopeleihin.

Koska Javascript-koodiajo siirtyy eteenpäin, tarkoittaa tämä, että seuraava rivi

``` return tietokanta[hetu]; ```

 on ongelmallinen. Yritämme etsiä tietokanta-objektista henkilöä hetun perusteella. Mutta juuri yllä totesimme, että tiedoston luku kovalevyltä vie muutaman tovin aikaa, ja koodinajo ei jää odottamaan. Toisin sanoen, tietokanta-objekti ei voi sisältää haluttua henkilötietokantaa. 

 Tilanne on sama kuin jos yrittäisit asua talossa, jonka rakentaminen on aloitettu viisi minuuttia sitten. Taloa ei yksinkertaisesti ole olemassa, joten etpä siinä voi asustellakaan.

### Mikä on ratkaisu?

Ongelman ydin on siis siinä, että kovalevyltä tiedoston lukeminen on asynkronoitu operaatio, ts. se suoritetaan *sitten joskus*. Koodiajo ei jää odottamaan tuota operaatiota, vaan pyyhältää surutta eteenpäin.

Tästä syystä tarvitsemme konseptin *lupaus*, joka palauttaa kovalevy.lue-kutsusta *lupausobjektin*. 

Tuo lupausobjekti sisältää tarvittavat mekanismit henkilötietokannan käyttöä varten *sitten joskus*.

```javascript

function hae(hetu) {
  // Meillä on nyt kovalevyllä 'henkiloTietokanta' tiedosto!
  // Ladataan tietokanta keskusmuistiin.
  // Koska kyseessä on asynkronoitu operaatio, palautamme lupausobjektin.
  var lupaus = kovalevy.lue('henkiloTietokanta');

  // lupaus-objekti saa tulevaisuudessa käyttöönsä henkilötietokannan
  // Kun näin vihdoin tapahtuu, haluamme etsiä henkilön hetun perusteella!
  return lupaus.then(function(tietokanta) {
    return tietokanta[hetu];
  });
	
}

hae('010787-111A').then(function(henkilo) {
  console.log(henkilo.nimi); // Jaakko Jantunen
})

```

Nyt kaikki toimii jälleen. Olemme turvautuneet lupausobjektin käyttöön. Lupausobjekti tarjoaa kutsuttavaksemme *then()*-metodin, johon voimme tarjota parametriksi funktion. Tuo funktio saa *sitten joskus* funktiokutsun yhteydessä sisäänsä tietokannan. Myöhemmin teemme toisen *then()*-kutsun, jonne työnnämme sisään henkilön nimen printtaavan funktion. 

Tässä on koko lupauskonseptin viehätysvoima - voimme ketjuttaa asynkronoituja funktiokutsuja lähes samaan tapaan kuin synkronoituja funktiokutsuja:

```javascript

// Synkronoidut funktiokutsut
a(b(c(d(e()))));

// Asynkronoidut funktiokutsut

Promise.resolve(e())
.then(d)
.then(c)
.then(b)
.then(a)

```

Vieläkään emme ole aivan päässet itse postauksen aiheeseen, async-await-kuvioon.

### Async-await

Ylläoleva henkilön haku toimii mainiosti lupausobjekteja ketjuttamalla. Mutta konsepti silti vaatii *lupaus.then()*-ketjuihin perustuvan koodaustyyliin. Tälläinen koodityyli ei välttämättä ole yhtä intuitiivinen kuin perinteinen synkronoitu koodaus.

**Async-await tarjoaa tavan toteuttaa lupauksiin perustuva koodinajo tavalla, joka visuaalisesti muistuttaa tavanomaista synkronoitua koodaustapaa.**

Tämä on (ymmärtääkseni) async-awaitin ainoa etu. Se ei tuo mitään uusia maagisia ominaisuuksia - se vain helpottaa koodinkirjoitusta silloin, kun käytämme asynkronoituja funktiokutsuja.

Katsotaan miten henkilötietojen haku onnistuu async-await-kuvion avulla:

```javascript

async function hae(hetu) {
  // Meillä on nyt kovalevyllä 'henkiloTietokanta' tiedosto!
  // Ladataan tietokanta keskusmuistiin.
  var tietokanta = await kovalevy.lue('henkiloTietokanta');
  // Haetaan henkilö
  return tietokanta[hetu];
	
}

hae('010787-111A').then(function(henkilo) {
  console.log(henkilo.nimi); // Jaakko Jantunen
})

```

Kuten yltä huomaamme, *hae*-funktio muistuttaa *synkronoitua* funktiota. Ainoa ero on **async** ja **await** avainsanojen käyttö.

```async function hae(hetu)```
```await kovalevy.lue('henkiloTietokanta')```

Async merkitsee, että kyseinen funktio palauttaa lupausobjektin. Await puolestaan merkkaa paikan, jossa *koodi stoppaa* - eli koodinajo jämähtää kuin seinään siksi aikaa, kunnes await-avainsanan perässä oleva lauseke on valmis.

Tässä esimerkissä *await*-termin perässä oleva lauseke on juurikin tiedoston lukeminen kovalevyltä. Toisin sanoen, *await* kiltisti odottaa, että kovalevyltä lukeminen on valmis. Vasta lukemisen onnistuttua koodinajo siirtyy eteenpäin kohti riviä *return tietokanta[hetu]*.

Async-await-kuvion todellinen upeus piilee tilanteessa, jossa saman funktiokutsun sisällä on useita *await-stoppauksia*:

```javascript

async function ostaVerkkokaupasta(ostaja, ostajanVerkkopankki, myyjanVerkkopankki, tuote) {
  // Varmistetaan ostajan tiedot ottamalla yhteys YTJ:n yritystietopalveluun
  var vahvistettu = await ytj.vahvistaYritys(ostaja);
  // Varmistetaan tuotteen saatavuus (joku varastolla kipaisee katsomaan).
  var tuoteSaatavilla = await varasto.tarkistaVarastaSaldo(tuote);

  if (vahvistettu && tuoteSaatavilla) {
    // Ostajan tiedot kunnossa ja tuote saatavilla!

    // Nostetaan rahat ostajan verkkopankista.
    var rahat = await ostajanVerkkopankki.nosta(hinta);
    // Annetaan rahat myyjälle
    var maksunTila = await myyjanVerkkopankki.talleta(rahat);

    if (maksunTila === true) {
      // Tilisiirto onnistui.
      // Haetaan tuote ja annetaan s asiakkaalle
      // (Haku tarkoittaa, että joku taas kipaisee varastolle.)
      return await varasto.haeTuote(tuote);
    } 

  }

}
var Nordea = /* rajapinta Nordean verkkopankkiin */
var OP = /* rajapinta OP:n verkkopankkiin */

ostaVerkkokaupasta('Nollaversio IT', Nordea, OP, 'Poravasara')
.then(function(poravasara) {
  if (!poravasara) {
    return console.log("Jäi saamatta.")
  }
  console.log("Poravasara saapunut ja käytettävissä");
})

```

Yllä funktion *ostaVerkkokaupasta* sisällä käytämme **await**-termiä viidesti. Jokaisen awaitin kohdalla koodinajo stoppaa ja jää odottamaan asynkronoidun operaation valmistumista. Funktio etenee askel kerrallaan, kunnes lopulta *sitten joskus* saamme (jos saamme) käyttöömme poravasaran.

> Loppukaneetti: async-await-kuvio on varsin vahva lisäys Javascriptiin. Tällä hetkellä async- ja await-avainsanoja ei voi vielä käyttää ilman Babelin kaltaista koodimuuntajaa. Noheva Javascript-koodari kuitenkin jo etukäteen tutustuttaa itsensä noiden termien käyttöön, sillä mitä todennäköisimmin tulevaisuuden Javascript perustuu paljolti niiden pohjalle.





