+++
date = "2017-01-11T19:03:44+02:00"
draft = false
title = "Amazonin kartoitus #1: Polly"

+++

> Tämä bloggaus aloittaa Nollaversio IT:n blogiin uuden artikkelisarjan, jossa käyn lävitse yksi kerrallaan Amazon AWS-ekosysteemin tarjoamia palveluita. Keskityn artikkelisarjassa palveluiden hyödyntämiseen osana web-palveluiden rakentamista.

# Amazon Polly

[Amazon Polly](https://aws.amazon.com/polly/) on yksi AWS-tuoteperheen uusimmista lisäyksistä. Polly täyttää varsin konkreettisen tarpeen; *se mahdollistaa tekstin kääntämisen puheeksi*. 

> Pollyn vastapari AWS-perheessä on [Amazon Lex](https://aws.amazon.com/lex/), joka kääntää puheen tekstiksi. Tutustutaan Lexiin myöhemmin.

Polly siis ottaa vastaan tekstiä ja puhuu suunsa puhtaaksi - aivan kuten tavallinen ihminen lukisi pätkän tekstiä mikrofoniin. Pollyn tapauksessa puhumisen hoitaa tietokone-algoritmi. Algoritmipuhe tuottaa äänitiedoston, esim. mp3-tiedoston.

> Polly on suoranainen kielivirtuoosi. Se höpöttää englannin lisäksi ainakin ruotsia, venäjää ja saksaa. **Valitettavasti suomi ei ole joukossa mukana, ainakaan vielä.**

### Soveltuvuus

Pollyn kaltaisen palvelun sisällyttäminen osaksi web-applikaatiota vaatii hiukka järkeilyä. 

Mitä lisäarvoa puhuttu puhe tuottaa verrattuna näyttöpäätteeltä luettuun tekstiin? Valtaosassa web-applikaatioita ei yhtään mitään - kirjoitettu teksti on helppokäyttöisempää kuin luettu puhe.

Yksi selkeä käyttötarkoitus on applikaatioissa, joissa käyttäjä ei ole näyttöpäätteen äärellä jatkuvasti. Applikaatio voi tällöin muuntaa esim. sisääntulevan viestin puheeksi, joka soitetaan käyttäjän kaiuttimista. Tällä tavoin viesti saadaan ihmiskäyttäjälle perille vaikka hän ei olisi läsnä näyttöpäätteen äärellä. Riittää, että hän on kaiuttimien äänen kantaman saavutettavissa.

Tämä ensimmäinen käyttötarkoitus perustuu ajatukseen siitä, että ääniviestiä on vaikeampi olla huomaamatta kuin visuaalista viestiä.

Toinen käyttötarkoitus on muuntaa tekstidokumentteja puheeksi esim. matkakuuntelua varten.

Ilmiselvä käyttötarve on esim. kirjan muuntaminen mp3-muotoon ja audiomuodossa matkalle mukaan ottaminen.

Toinen, vähemmän ilmiselvä käyttöpotentiaali, löytyy sähköpostiviestien käsittelystä. Auton ratissa on mahdoton käyttää silmiä sähköpostiviestien yms. dokumenttien lukemiseen. Tämä on fakta, jota moni on yrittänyt uhmata henkensä hinnalla. 

Mutta entä jos sisääntuleva sähköpostiviesti luettaisiin ääneen auton kaiuttimista? 

Arkkitehtuuri voisi toimia seuraavasti.

### Malli-arkkitehtuuri: sähköpostit auton kaiuttimista.

Arkkitehtuurin hardware vaatii älypuhelimen 3G- ja Bluetooth-yhteyksillä sekä autosoittimen Bluetooth-yhteydellä. Lähes kaikki modernit älypuhelimet tukevat 3G + Bluetooth -yhdistelmää, ja valtaosa uusista autoista sisältää Bluetooth-soittimien sisäänrakennettuna auton audiojärjestelmään.

Automatkan alkaessa kuski avaa applikaation (joko web-appi tai mobiiliappi) "EmailitPuheeksi". Applikaatiosta hän valitsee audiokytkennän auton audiojärjestelmään.

Applikaatio kysyy käyttäjän sähköpostitilin tietoja. Tiedot syötettyään applikaatio jää kuuntelemaan sähköpostiliikennettä; aina kun email lävähtää käyttäjän sähköpostilaatikkoon, EmailitPuheeksi-appi saa siitä kopion käyttöönsä.

Tämän email-kopion applikaatiomme lähettää Amazonin rajapintaan. Amazon herättää Pollyn kauneusunilta, ja käännös "teksti -> puhe" suoritetaan. Käännöksen suoritus saattaa kestää useita sekunteja, joten Amazon ampuu käännetyn email-puhetiedoston AWS:n omaan jonopalveluun.

> Amazonin ei tarvitse toimia oikean ihmisen tavoin, eli lukea tekstiä sana kerrallaan. Koska puheenmuodostus tapahtuu algoritmisesti, on puhe mahdollista tuottaa *paralleelisti* - alkuperäinen teksti pätkikään osiin ja kukin osa käännetään puheeksi erikseen. 

Jonopalvelusta applikaatiomme sitten käy nappaamassa puhetiedoston, ja sen saatuaan soittaa tiedoston. Koska applikaation ääni-output on kytketty auton audiojärjestelmään, sähköposti luetaan ääneen auton kaiuttimista.

### Koodiesimerkki

Pollyn käyttö on helppoa. AWS tarjoaa rajapintapalvelunsa API Gatewayn liitettäväksi Pollyn kylkeen; tällöin HTTP-pyyntö voidaan lähettää rajapintaan, joka sitten parsii siitä tarvittavat tiedot (hyödyntäen *AWS Lambda*-funktiota!) ja lähettää ne Pollyn luettavaksi.

> Polly edustaa teknologisen kehityksen terävintä kärkeä. Lambdan hyödyntäminen Pollyn käytössä edustaa tuon kehityksen terävimmän kärjen ylintä atomia. Vielä kuukausi sitten - joulukuussa 2016 - Lambdaa ei oltu päivitetty sisältämään rajapintatoimintoja Pollyn suuntaan. Nyt (11.01.17) päivitys on tehty, joskin sen deploymentti läpi AWS valtavan infrastruktuurin on vielä kesken. Lisätietoja [täältä.](https://forums.aws.amazon.com/thread.jspa?threadID=244156)  

Polly on sen verran uusi palvelu, että netistä ei löydy käytännössä lainkaan esimerkkejä sen käytöstä. Mutta joltain tämänkaltaiselta se näyttää:

```javascript

// Tuodaan HTTP-kirjasto käyttöön
var request = require('request');
// Tuodaan jokin soitin, ei tarkemmin määritelty.
var soitin = require('mp3soitin');

// Kaikki authentikaatio on skipattu.

// Tämä käännetään äänitiedostoksi Pollyn avulla.
var text = "Translate this!";

var options = {
  uri: 'http://aws.polly.com/v1/speech',
  method: 'POST',
  json: {
   	"OutputFormat": "mp3",
   	"Text": text,
   	"TextType": "text",
   	"VoiceId": "Emma"
  }
};

// Tehdään kutsu AWS:n Polly-rajapintaan.

request(options, function (error, response, body) {
  // Virheiden tarkistus tähän..

  // Napataan audio.
  var audio = response.AudioStream;

  // Lähetetään audio johonkin soittimeen
  soitin.play(audio);

});

```

Amazonin suuntaan lähtevän HTTP POST-kutsun tärkein osa on tämä:

```javascript

 {
   	"OutputFormat": "mp3",
   	"Text": text,
   	"TextType": "text",
   	"VoiceId": "Emma"
  }

``` 

Tuossa objektissa määritämme mm. luettavan tekstin, lukijaäänen (ihana Emma, jolla on brittiaksentti) ja ääniformaatin. Muitakin asetuksia on laitettavissa, mm. samplaus-rate.

### Hinta

Polly on hinnoiteltu - kuten käytännössä kaikki AWS:n palvelut - naurettavan halvaksi. 

Esimerkiksi 24 tunnin kestoinen puhe maksaa neljä dollaria. 

Tuntipalkkaa Polly perii siis huimat n. 20 senttiä. Mikä pahinta, Polly-parka kituuttaa nollasopimuksella; kk-maksuja ei ole lainkaan ja kaikki veloitus menee suoraan käytön mukaan.

### Summarum

> Amazon Polly tarjoaa tekstin kääntämisen puheeksi mukavan kivuttomasti. Palvelu on upouusi, joten käyttökokemukset siitä ovat vähissä. Edelläkävijälle palvelu tarjoaa spesifiin käyttötarpeeseen optimaalisen täsmäratkaisun, joka yksinkertaisesti toimii. Tai ainakin lupaa toimivansa.
>
> Huonona puolena on, että - vielä toistaiseksi - suomen kieltä ei ole mukana.







