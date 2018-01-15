+++
date = "2018-01-08T13:38:19+02:00"
draft = false
title = "Age of Empires - moninpelin arkkitehtuuri"

+++

Age of Empires 2 on yksi lempipeleistäni. Etenkin sen online-multiplayer. Vuonna 1999 ilmestynyt AoE2 sisältää jopa kahdeksan pelaajan online-pelimuodon, jossa yli tuhat eri pelaajien kontrolloimaa pelihahmoa käy massiivisia taisteluja.

Ohessa video hektisestä kahdeksan pelaajan multiplayer-pelistä: [https://youtu.be/BBsyHerdpuI?t=50m3s](https://youtu.be/BBsyHerdpuI?t=50m3s)

Sattumalta googlasin männä päivänä tietoja siitä, kuinka Aoe2 on rakentanut jo 90-luvulla näin mahtavan online-pelikokemuksen. 

Ennen googlettelua oletin, että multiplayer tapahtuu *client-server*-mallin pohjalta; yksi palvelin (joka mahdollisesti sijaitsee yhden pelaajista, nk. host-pelaajan tietokoneella!) pitää globaalia pelitilaa yllä, ja jakaa sitä N kertaa sekunnissa pelaajille. 

Pelaajat puolestaan lähettävät palvelimelle komentoja; palvelin reagoi kuhunkin komentoon, päivittää yhteisen pelitilanteen, ja lähettää päivitetyn tilan pelaajille. Yksinkertaista.

Mutta eihän se näin mennytkään; AoE2:n online-arkkitehtuuri perustuu *peer-to-peer* -malliin. 

## Peer-to-peer

Peer-to-peer -mallissa ei ole keskitettyä palvelinta, joka toimisi “single source of truth”-keskuksena pelin aikana.

Missä sitten sijaitsee tieto siitä, miltä pelimaailma näyttää kullakin ajanhetkellä? Vastaus: *kullakin pelaajalla on tuo tieto erikseen*.

Jotta koko hommassa olisi mitään järkeä, kullakin pelaajalla on oltava identtinen käsitys sen hetkisestä pelitilanteesta. Muuten koko pelissä ei olisi mitään mieltä. 

> Kuvittele esimerkiksi shakkipeli, jossa valkea pelaaja näkee laudan nappulat eri ruuduissa kuin musta pelaaja. Shakin pelaaminen olisi aika tuskallista.

Yksi tapa huolehtia siitä, että kullakin pelaajalla on sama identtinen pelitilanne tietyllä ajanhetkellä, on seuraava algoritmi:

**Pelaajan algoritmi (ajetaan kunkin pelaajan tietokoneella):**

1. Suorita pelaajan tekemä pelisiirto lokaalisti ja laske uusi pelitila.
2. Lähetä uusi lokaalisti laskettu pelitila kaikille muille pelin pelaajille.
3. Vastaanota muiden pelaajien vastaavalla tavalla laskettu uusi pelitila.
4. Yhdistä eri pelaajien pelitilat yhteen, ja laske niistä uusi yhdistetty pelitila.
5. Renderöi yhdistetty pelitila ruudulle, ja jää odottamaan uutta pelaajan komentoa/pelisiirtoa.

Ongelmana tässä algoritmissä on kohta 4, joka saattaa – pelistä riippuen – olla joko mielipuolisen vaikea tai suorastaan mahdoton suorittaa. On helppo kuvitella tilanne, jossa kahden eri pelaajan tekemät pelisiirrot ovat lokaalisti (siis yksittäin tarkasteltuna) laillisia, mutta niiden yhdistelmä on laiton. 

## RTS === vuoropohjainen?

Ratkaisu tähän “lokaalisti laillinen – globaalisti laiton” -ongelmaan on pakottaa pelaajat tekemään siirrot *vuorotellen*. 

Tai, ellei teknisesti ihan vuorotellen, niin ainakin *vuoroja hyödyntäen*. 

Tämä kuulostaa liian tiukalta vaatimukselta monelle pelityypille, esimerkiksi AoE2:n kaltaiselle real-time-strategy (RTS)-pelille. Koko RTS:n pointti kun on olla real-time; vuoropohjaisten pelien ystäville on jo Civilization-saaga.

On kuitenkin huomattava, että on kaksi eri asiaa olla *aidosti real-time* versus *näennäisesti real-time*. 

Age of Empiresin kaltainen RTS-peli käyttää konepellin alla itseasiassa diskreettejä pelivuoroja, mutta vuorojen varsinainen pituus on varsin lyhyt, ja muutamaa kikkaa hyödyntäen niiden pituus saadaan vaikuttamaan kuin vuoroja ei olisi lainkaan.

Homma toimii näin. Pelin kulku koostuu pelivuoroista, joiden aikana kukin pelaaja voi tehdä N määrän pelisiirtoja. Erona Civilization-peliin on lähinnä se, että *eri pelaajat tekevät siirtonsa saman pelivuoron aikana*. Siinä missä Civilizationissa kullakin pelaajalla on oma pelivuoronsa, jonka aikana muut pelaajat kiltisti odottavat, Aoe2-pelissä kaikki pelaajat jakavat yhden globaalin pelivuoron kerrallaan. 

Lisäksi AoE2:n pelivuoro on siitä ikävä, että se ei odota pelaajaa (toisin kuin aidoissa vuoropohjaisissa peleissä); jos pelaaja ei ehdi tekemään pelisiirtoa pelivuoron aikana, se on pelaajan oma ongelma.

AoE2:n pelivuorolla on nimittäin ajallinen pituus, joka on vakiona 200 millisekuntia. Näin lyhyt siirtovuoro on tarpeen, jotta peli saa luotua illuusion reaaliaikaisuudesta.

> Kahdensadan millisekunnin pituus on luonnollisesti muutettavissa riippuen pelaajien nettiyhteyksien nopeudesta. Arvoa voi skaalata suuntaan tai toiseen jopa yksittäisen pelin ollessa käynnissä. Toimintaperiaate muistuttaa TCP-protokollan flow-kontrollia.

Mutta hetkinen, 200 ms on siltikin järjettömän *pitkä* aika tietokonepelin kontekstissa. Jos itse peli pyörisi 200 millisekunnin render-loopilla, pelin ruudunpäivitystahti (FPS) olisi viisi. 

Siis 5 ruudunpäivitystä sekunnissa. Eli puhdas slideshow.

Mikä siis lopulta pyörii 200 millisekunnin vauhdilla? 

## Pelisiirrot vs. pelilogiikka

Ainoastaan pelivuorot. Pelilogiikan sisältävä game-loop pyörii 30 FPS:n nopeudella. 

Homma toimii suunnilleen näin: kukin pelaaja tekee annetun pelivuoron (200ms) aikana *niin monta pelisiirtoa kuin ehtii*. Kun pelivuoro päättyy, tehdyt siirrot talletetaan listaksi ja lähetetään kaikille muille pelaajille. Vastaavasti pelaaja vastaanottaa kaikkien muiden pelaajien pelisiirrot.

Kun tämä valtava – kukin pelaaja lähettää omat siirtonsa kullekin toiselle pelaajalle – lähetysoperaatio on tehty, kullakin pelaajalla on nyt identtinen lista pelivuoron aikana *globaalisti* tehdyistä pelisiirroista. Nyt seuraa paras kohta; kukin pelaaja lokaalisti päivittää oman pelitilansa annettujen pelisiirtojen perusteella.

Ja koska kaikilla pelaajilla on *identtinen lista siirtoja* ja *identtinen pelitila* ennen päivitystä, päätyvät kaikki pelaajat identtiseen pelitilaan siirtopäivitysten jälkeen mikäli pelilogiikka toimii 100% deterministisesti.

> Asian voi havainnollistaa shakkipelillä: pelaajat A ja B aloittavat shakkipelin. Pelaaja A tekee siirron ja lähettää sen B:lle. Tarvitseeko A:n lähettää siirron mukana myös uusi peliasema? Ei, sillä shakkipeli on täysin deterministinen. Ja shakkipelin alkuasema on kirjoitettu shakin sääntöihin, joten se on identtinen ja molempien pelaajien tiedossa.
>
>
>
> Shakin deterministisyys mahdollistaa mielenkiintoisia pelimuotoja, jotka eivät ole mahdollisia esimerkiksi Afrikan Tähdessä. Kaksi vahvaa shakinpelaajaa voi pelata shakkipelin ilman lautaa ja nappuloita; he sanovat vuorotellen siirrot toisilleen. Tämä on sokkoshakkia. Hurjimmat pelaavat sokkoshakkia vaikka kesken tennisottelun.
>
>
>
> AoE2:n puolella pelin alkutilanne ei ole osa pelin sääntöjä (eikä täten identtinen pelikerrasta toiseen), joten pelin alkaessa alkuasema täytyy synkronoida kaikkien pelaajien kesken. Tämä on ainoa hetki, jolloin online-moninpelin pelaajat päivittävät pelitilansa globaalia tilamuuttujaa hyödyntäen. Globaalina tilamuuttujana voi toimia joko moninpelialusta (esim. Steam tai Voobly?) tai joku yksittäinen pelaaja, joka hetkellisesti ottaa host-roolin.

Tätä mallia kutsutaan nimellä “deterministic lockstep”-malli. Mallilla on vankka teoreettinen pohja, ja se toimii kuin junan vessa. 

## Back to the earth – käytännön haasteet

Toimii kuin junan vessa teoriassa, siis. 

Käytännössä mallin saaminen toimimaan vaatii pelistä riippuen joko vähän töitä tai aivan saatanasti töitä. Shakki on esimerkki ensin mainitusta, AoE2 jälkimmäisestä. Jo pelkästään AoE2 pelivideota katsomalla huomaa, että pelissä tapahtuu valtavasti asioita.

Jotta deterministic lockstep toimii, täytyy *koko pelimekaniikan olla deterministinen*. Tämä tarkoittaa, että jokaikisen saksanhirven (AI-ohjattu) liikeradan, jokaisen keihään lentoradan, jokaisen läpi sokkeloisen metsäpolun lasketun kulkuradan (unit pathing)... kaikkien on toimittava identtisesti kaikilla kahdeksalla pelaajalla.

Satunnaislukugeneraattori lentää ensimmäisenä roskakoriin, sillä jos yksikin osa pelimekaniikasta perustuu sattumaan, koko moninpeli on pilalla. Tilalle tulee *pseudo-satunnaislukugeneraattori*, joka alustetaan pelin alussa seedillä. Kaikilla pelaajilla on luonnollisesti oltava sama seed, jotta generaattorin tuottamat “satunnaisluvut” ovat ei-satunnaisia, eli samat kullakin pelaajalla.

## Mallin edut 

### Yksinpeli vai moninpeli - who cares?

Koko pelin deterministisyyden varmistaminen on pirullisen moninmutkainen ongelma. Mutta jos ongelma ratkotaan, moni muu asia tulee ikäänkuin ilmaiseksi. 

Esimerkiksi online-moninpeli typistyy lopulta lokaaliksi moninpeliksi tai yksinpeliksi isoa joukkoa AI-pelaajia vastaan, sillä AoE2-peli-instanssin ei tarvitse välittää mistä lähteestä pelisiirrot tulevat. Kaikki pelimuodot toimivat pelimoottorin näkökulmasta identtisesti; pelimoottori ottaa vastaan siirtoja, ja thats it. Siirtojen alkuperä ei pelimoottoria kiinnosta.

### Halpa tiedonsiirto

Deterministic lockstep -mallin toinen valtava etu on, että internet-yhteyden yli siirrettävä tietomäärä on verrattaen vähäinen. 

Vahva AoE2 pelaaja ehtii yhden siirtovuoron (sanotaan vaikka tuo 200 millisekuntia) aikana tekemään ehkä 3-4 siirtoa mikäli pelitilanne on oikein hektinen. Jokainen näistä siirroista on komento, joka sisältää ainoastaan tarvittavan tiedon komennon suorittamiseksi kaikkien online-pelin pelaajien tietokoneilla. Yksinkertaisimmillaan komento voisi siis olla:

```javascript
{
  type: “move”,
  unit: “knight_8282”,
  to: {x: 672, y: 992}
}
```

Komento sisältää kaiken tarvittavan tiedon; ritarihahmo, jonka ID on knight_8282, siirtykööt lokaatioon 672,992. Tämän tiedon perusteella kukin pelaaja voi päivittää pelitilansa; kunkin pelaajan AoE2-peli laskee *unit path-algoritmin* avulla reitin ritarin nykyisestä lokaatiosta uuteen lokaatioon.

> Ja koska kaikki AoE2-peli-instanssit käyttävät luonnollisesti samaa unit path-algoritmia, on koko laskettu reitti identtinen kaikilla pelaajilla.

Komennon koko JSON-tekstinä (auttamattoman kookas dataformaatti) on hädin tuskin 50 tavua. 

Kyseessä on siis todella suorituskykyinen multiplayer-arkkitehtuuri. Hyvä niin, sillä internet-yhteydet vuonna 1999 eivät olleet kummoisia.

> Deterministic lockstep-mallin onnistunut käyttö AoE2-pelissä vaatii taustamekaniikkaa, ja tässä blogikirjoituksessa raapaistiin vain pintaa. AoE2-pelin arkkitehtuurin ydinajatukset löytyvät täältä: [https://www.gamasutra.com/view/feature/131503/1500_archers_on_a_288_network_.php](https://www.gamasutra.com/view/feature/131503/1500_archers_on_a_288_network_.php)






