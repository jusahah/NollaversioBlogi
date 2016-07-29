+++
date = "2016-07-28T06:39:21+03:00"
draft = false
title = "Chain() -metodi ketjuttaa funktiokutsut"

+++

[Lodash](http://www.lodash.com) on varsin hieno apukirjasto Javascriptin ohjelmointiin. Tuo kirjasto sisältää sadoittain pieniä apufunktioita, joiden avulla yleisimmät algoritmit voi toteuttaa nopeasti ja kivuttomasti.

Esimerkkinä vaikka algoritmi listan jakamisesta osiin. Ilman lodashia algoritmi näyttää tältä.

```javascript

var lista = [1,2,3,4,5,6,7,8];
var jakoluku = 3;

// Jaetaan lista osiin (jokainen osa on uusi *lista*) siten, että 
// kukin osa sisältää *jakoluvun* verran elementtejä.

var jaettulista = [];

for (var i = 0, j = lista.length; i < j; i++) {
  var elementti = lista[i];

  // Jos i on tasajaollinen jakoluvulla,
  // on aika aloittaa uusi osalista.
  if (i % jakoluku === 0) {
    // i on joko 0, 3, 6, 9, ...jne.
    // Luodaan uusi osalista ja lisätään elementti siihen
    jaettulista.push([elementti]);
  }  

  // Jos ei ole tasajaollinen,
  // lisätään elementti tuoreimpaan osalistaan.
  else {
    // Lisätään elementti olemassaolevaan osalistaan
    jaettulista[jaettulista.length-1].push(elementti);
  }	
} 

console.log(jaettulista); // [[1,2,3], [4,5,6], [7,8]]

```

Saman saa aikaan Lodashin **.chunk()** metodilla näin:

```javascript
var lista = [1,2,3,4,5,6,7,8];
var jakoluku = 3;
var jaettulista = _.chunk(lista, jakoluku);
console.log(jaettulista); // [[1,2,3], [4,5,6], [7,8]]

```

Ero on kuin yöllä ja päivällä.

Niin hieno kuin lodash onkin, siinä on puutteensa. **Tai näin minä luulin vähintään vuoden päivät**. Kunnes hoksasin dokumentaatiota lukemalla, että puute olikin vain illuusio. Löysin metodin nimeltä *chain*.

### Chain() - mihin sitä tarvitaan?

Kuvitellaanpa seuraavanlainen korkean tason algoritmi:

> Alkuasetelma: meillä on lista desimaalilukuja

> Algoritmi:
>
> 1. Pyöristä luvut tasaluvuiksi.
>
> 2. Poista kaikki nollat.
>
> 3. Kerro luvut yhteen.

Ylläoleva algoritmi näyttää lodashin avulla *naivisti* toteutettuna seuraavanlaiselta:

```javascript

var lista = [1.9, 2.0, 2.1, 0.2];

var pyoristetyt = _.map(lista, Math.round);
var nollatPois  = _.compact(pyoristetyt);
var tulo = _.reduce(nollatPois, function(t, luku) {
  return t * luku;
}, 1);

// Välivaiheiden tulokset
console.log(pyoristetyt); // [2,2,2,0]
console.log(nollatPois); // [2,2,2]
// Lopullinen tulos eli lukujen tulo
console.log(tulo); // 8

```

Ylläoleva on ihan kiva, mutta huomion arvoista on, että joudumme käyttämään paljon väliaikaisia muuttujia. Välivaiheiden muuttujat *pyoristetyt* ja *nollatPois* ovat tälläisiä - algoritmi tallentaa niihin välitulokset, mutta loppukäyttäjä on kiinnostunut vain *tulo*-muuttujasta.

Yksi ratkaisu on jättää välimuuttujat pois:


```javascript

var lista = [1.9, 2.0, 2.1, 0.2];

var tulo = _.reduce(_.compact(_.map(lista, Math.round)), function(t, luku) {
  return t * luku;
}, 1);

console.log(tulo); // 8

```

Ylläoleva lyhempää koodirivien määrää huomattavasti, mutta **vaikeuttaa koodinlukua**. Se näyttää rumalta, ja on vaikea pysyä silmämääräisesti kärryillä siitä, mitkä sulkumerkit muodostavat parin.

Eli trade-off; koodin rivimäärä pieneni, mutta koodinluku vaikeutui merkittävästi.

Mutta meillä on parempikin ratkaisu. Käytetään **chain**-apumetodia.

### Chain() - the best of both worlds

Tässä on chainin varaan tukeutuva ratkaisu:

```javascript

var lista = [1.9, 2.0, 2.1, 0.2];

var tulo = _
.chain(lista)
.map(Math.round)
.compact()
.reduce(function(t, luku) {
  return t * luku;
}, 1)
.value();

console.log(tulo); // 8

```

Ylläoleva chain-metodiin perustuva ratkaisu vaikuttaa selkeältä voitolta. Se on äärimmäisen helppolukuinen, sillä jokainen uusi metodikutsu alkaa omalta riviltään. Samaan aikaan välimuuttujia ei tarvita! Eli win-win.

Miten chain() toimii pinnan alla? Se muuntaa annetun argumentin (tässä *lista*) sellaiseen muotoon, että sitä voidaan **juoksuttaa** pitkin ketjua. Sillä chain()-metodi aloittama metodikutsujen sarja voidaan ajatella ketjuna, tai putkena. Tai liukuhihnana. Kukin metodi saa sisäänsä argumentin, muokkaa tuota argumenttia jotenkin, ja pötkäyttää ulos muokatun version argumentista. Seuraava putkenpalanen saa sisälleen tuon muokatun version, ja niin edelleen.

Putken/liukuhihnan loppupäädyssä kutsumme metodia *value()*, joka hakee lopullisen palautusarvon.

Kyseessä on erittäin vahva ja ennenkaikkea modulaarinen koodaustapa. Ketjuta funktiokutsut ja juoksuta haluamasi dataa ketjun lävitse. Yhdestä päästä menee raaka-aineet sisään, toisesta päästä tulee valmis tuote ulos.


