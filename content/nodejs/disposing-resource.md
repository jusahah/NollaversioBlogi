+++
date = "2016-09-28T10:08:17+03:00"
draft = false
title = "Älä kuole ääneti"

+++

Monet ohjelmointikielet sisältävät tärkeän konseptin nimeltä *garbage collection", suomeksi siis roskienkeruu. Tuo konsepti tarkoittaa yksinkertaisesti sitä, että ohjelmointiympäristö automaattisesti huolehtii ohjelman ajon aikana luotujen *objektien* tuhoamisesta. 

Alimmalla raudan tasolla tämä tuhoamisesta huolehtiminen tarkoittaa sitä, että keskusmuistista vapautetaan tilaa uusia objekteja varten. 

Myös Javascript noudattaa garbage collection-periaatetta. Kun tietystä objektista tulee tarpeeton, Javascriptin runtime-ympäristö hoksaa vapauttaa objektin varaamaan muistitilan. Se kuinka tuo *hoksaaminen* käytännössä tapahtuu ei ole oleellista ohjelmoijan kannalta; oleellista on vain se, että *ohjelmoijan ei tarvitse asiasta välittää*. Ohjelmointikielen taustalla pyörivä runtime-alusta toimii roskakuskina.

> Niille jotka ovat kiinnostuneita roskienkeruun teknisestä toteutuksesta, seuraava linkki auttaa: [http://stackoverflow.com/questions/10112670/when-are-javascript-objects-destroyed](http://stackoverflow.com/questions/10112670/when-are-javascript-objects-destroyed)

Asiassa on kuitenkin yksi mutta.

Entä jos roskakoriin päätyvä objekti on varannut olemassaolonsa ajaksi käyttöönsä jonkin *ulkoisen resurssin*? Kun Javascript objekti tulee elinkaarensa päähän, runtime-alusta viskaa sen roskakoriin. Mutta miten käy tuon objektin omistaman resurssin?

Tosimaailman esimerkki selventää.

Kuvitellaan, että varaan liput teatteriesitykseen huomisillalle. Ikäväkseni kuitenkin käy niin, että saan kohtalokkaan sydänkohtauksen tänä iltana, ja siirryn ajasta ikuisuuteen. 

Vielä tämän päivän puolella eloton ruumiini käydään noukkimassa ruumishuoneelle ("roskien keruu").

Vaan miten käy teatterilippujeni? Olen varannut liput huomisen esitykseen. Se, että menin kuolemaan tässä välissä, *ei automaattisesti peruuta varaustani huomisen teatteriesitykseen.*

Kuolleena en valitettavasti pääse paikalle teatteriin, mutta teatteri ei myöskään voi antaa paikkaa kellekään toiselle, sillä teatteri ei tiedä kuolemastani.

Ongelman ydin on siinä, että *kuollessani kukaan ei peruuta paikkavaraustani*. 

Mutta entä jos toimisin seuraavasti; vielä kun olen elävien kirjoissa, raapustan post-it-lapulle tekstin "peruuta paikkavaraus teatteriin mikäli olen kuollut". Asetan lapun lompakkooni ajokortin oheen.

Kuka ikinä elottoman ruumiini löytää, löytää myös tuon lapun. Hän voi toimia lapun ohjeiden mukaan. Teatteri saa tiedon siitä, etten pääse paikalle esitystä seuraamaan. Täten teatteri voi myydä paikkani jollekin toiselle.

Ylläolevan esimerkin logiikkaa seuraten voimme myös toteuttaa *resurssin vapautuksen* resurssia hallinnoivan objektin kuollessa. Vai voimmeko? Riippuu ohjelmointikielestä.

C++ -kielessä on konsepti nimeltä "destructor", joka mahdollistaa juurikin tuollaisen post-it-lapun luomisen. *Objektin destructor kutsutaan juuri ennen objektin kuolemaa*. Tällä tavoin destructor-metodi voi ajaa tarvittavan koodin, jolla huolehditaan että *objekti ei jätä keskeneräisiä velvoitteita peräänsä kuollessaan*.

Esimerkiksi teatteriesityksen tapauksessa:

(HUOM! C++ koodia)

```

class Katsoja {
public:
   Katsoja(char* nimi, Teatteriesitys *esitys); 
   ~Katsoja();
private:
  char *nimi;
  Teatteriesitys *esitys;
};

Katsoja::~Katsoja() {
  // Ilmoitetaan teatterille, että tämä katsoja
  // ei pääse paikalle; hän kun on kuolemaisillaan.
  esitys->vapautaPaikka(this);
}

// jne. muut metodit

```

Ylläolevassa koodissa objekti *ilmoittaa kaikille kiinnostuneille osapuolille* että hän on kuolemassa. Tämän ilmoituksen hän tekee *juuri ennen* kupsahtamistaan.

> HUOM! C++ suorittaa automaattisen roskien keruun ainoastaan ns. lokaaleille objekteille. Tälläisiä objekteja ovat ne, jotka luodaan suoraan funktion sisälle lokaaliin käyttöön (ns. "stäkkimuuttujat").

C++:n puolella ylläoleva konsepti "*kerro omasta kuolemastasi juuri ennen kuin kuolet*" toimii erinomaisesti. Konseptin ja stäkkimuuttujien automaattisen destruktion varaan on rakennettu erittäin vahva patteri nimeltä **RAII* ("resource acquisation is initialization").

Mutta Javascriptin puolella konsepti ei toimi, sillä Javascript ei tunne *destructorin* käsitettä lainkaan. 

Tämä destructorin puute on ongelmallista. Kun roskakuski nappaa turhaksi käyneen objektin, objekti ei voi ilmoittaa viimeisenä äännähdyksenään muulle maailmalle että "hei, se on menoa nyt!".

Eritoten Javascript-objekti ei kuolemansa hetkellä ajaa koodia, joka vapauttaa objektin omistamat resurssit (esim. teatterivarauksen).

Käytännössä tämä tarkoittaa, että koodarin täytyy vastaava logiikka ohjelmoida itse ja huolehtia visusti, että objekti *tapetaan eksplisiittisesti*; ts. objekti tapetaan ohjelmoijan kirjoittaman koodin toimesta. 

Myöhempi automaattinen roskien keruu on typistyy kuolleen ruumiin siivoamiseksi pois kadulta.

```javascript

function Katsoja(esitys) {
  
  this.kuole = function() {
    // Kerro teatterille että kuolema iski päälle.
    esitys.vapautaPaikka(this);
  }	

  // jne...
}

var esitys = new TeatteriEsitys('Mielensäpahoittajan paluu');
var katsoja = new Katsoja(esitys);

// jne...

katsoja->kuole();

// Muuttuja "katsoja" kerätään roskiin kunhan se menee out-of-scope.

```

Yllä Javascript-koodissa määritämme *kuole*-metodin. Metodi on pitkälti vastaava kuin C++:n *~Katsoja*-metodi. 

**Merkittävä ero on, että C++:ssa tuo metodi kutsutaan automaattisesti, Javascriptissä meidän tulee kutsua metodia itse!**

> Resurssien hallinta on tärkeä osa ohjelmointia. Hallinta pohjimmiltaan typistyy kysymykseen: "kuinka *varmistua* siitä, ettei kuollut objekti vahingossa *unohda* vapauttaa omistamaansa resurssia".
>
> Mikäli objektit unohtavat vapautuksen, järjestelmä pikkuhiljaa syö kaikki resurssit. Tosimaailmassa vastaava ilmiö tapahtuisi mikäli kuolleet ihmiset eivät menettäisi omistusoikeuttaan esim. kiinteistöihinsä kuolemansa hetkellä.
>
> Koska kuolleet eivät voi niitä myydä (kuolleelta on pirun vaikea saada allekirjoitusta kauppakirjaan), ne olisivat kuolleiden omistuksessa *ikuisesti*.
>
> Ajan mittaan Suomen kaikki rakennukset olisivat kuolleiden ihmisten omistuksessa. Tälläistä ilmiötä kutsutaan ohjelmoinnin parissa nimellä "resource depletion". Yksi jos toinenkin (päin mäntyjä ohjelmoitu) applikaatio kärsii ongelmasta.





