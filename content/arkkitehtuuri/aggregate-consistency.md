+++
date = "2016-09-12T08:10:14+03:00"
draft = false
title = "Sisäinen eheys vs. ulkoinen eheys"

+++

Sain yhden perustavanlaatuisimmista oivalluksistani liittyen Domain-Driven Designiin pdf-dokumentista *Domain-Driven Design Reference: Definitions and Pattern Summaries*. Tuossa Eric Evansin (se "sinisen kirjan" guru) rustaamassa dokkarissa on elintärkeä lause piilotettuna tekstin joukkoon:

> Within an aggregate boundary, apply consistency rules **synchronously**. Across boundaries, handle updates **asynchronously**.

Tummennukset allekirjoittaneen.

Vapaasti suomennettuna ja hieman yksinkertaistettuna lausahdus menee muotoon:

> yhden aggregaatin *sisäinen* eheys hoidetaan transaktioiden avulla, useamman eri aggregaatin *ulkoinen* (tai "välinen") eheys hoidetaan muulla tavoin.

### Aggregaatti? Sisäinen eheys vs. ulkoinen eheys?

Ensiksi määritetään aggregaatti. Aggregaatti on entiteetti, joka on jaettavissa pienempiin osiin. Mutta nuo pienemmät osat ovat nähtävissä vain *sisältä käsin*; ulkoa katsottuna aggregaatti on eheä ja atominen palanen.

Esimerkiksi lentokone voidaan nähdä aggregaattina. Ulkoapäin katsottuna lentokone näyttää yksittäiseltä objektilta. Kun minä katson Espoon Vanttilan yli pyyhältävää Finnairin matkustajajettiä, näen yksittäinen objektin. 

Minun näkökulmastani katsottuna tuo kilometrin korkeudessa pyyhältävä lentokone on eheä kokonaisuus, joka ei ole jaettavissa pienempiin osiin.

Lentokoneen sisällä reissatessa taas huomaa selvästi, että lentokone on jaettavissa pienempiin osiin. Penkit, ovet, ruuma, cockpit, suihkumoottorit - tästä sisäisestä näkökulmasta asiaa tarkastellessa huomaa, että lentokone on *aggregaatti*; objekti, joka koostuu valtavasta määrästä muita objekteja.

Jatketaan esimerkkiä. Sanotaan, että tehtävämme on kehittää tietojärjestelmä, joka mallintaa lentokoneiden liikennöintiä Helsinki-Vantaan ilmatilassa. Järjestelmä mallintaa koneiden toimintaa mahdollisimman yksityiskohtaisella tasolla, esim. yksittäisen lentokoneen suihkumoottoreiden toiminta mallinnetaan.

Tämä järjestelmä koostuu ilmiselvästi objekteista - tai paremminkin *entiteeteistä* - jotka ovat tyyppiä "lentokone". Jokainen lentokone on järjestelmän sisällä itsenäinen entiteetti. 

Samaan aikaan jokainen lentokone on myös aggregaatti, joka koostuu siivistä, suihkumoottoreista, navigointilaitteista, yms.

### Sisäinen eheys

Nyt tässä kontekstissa sisäinen eheys tarkoittaa, että kukin lentokone on kunakin ajan hetkenä sisäisesti eheässä tilassa. Toisin sanoen, jokainen lentokoneen omat alikomponentit ovat keskenään johdonmukaisessa tilassa. 

Millainen olisi sisäisesti ei-johdonmukainen tila? Esimerkiksi sellainen, jossa lentokoneen kerosiinitankki olisi typötyhjä, mutta polttoainemittari näyttäisi 100%.

Tai sellainen, jossa koneen laskeutumistelineet olisivat visusti ylhäällä, mutta cockpitin infonäyttö näyttäisi niiden olevan alhaalla.

Sanomattakin selvää, että yllämainitun kaltaiset *epäjohdonmukaisuustilat* ovat hengenvaarallisia lentoturvallisuuden suhteen. Siksi on elintärkeää, että lentokone ei koskaan päädy niihin. **Lentokoneen tulee siis olla sisäisesti johdonmukaisessa tilassa kaikkina ajan hetkinä**. Jos löpömittari näyttää 100%, tankissa on oltava polttoainetta piri pintaan asti.

Samaan aikaan kun jokainen lentokone on sisäisesti johdonmukaisessa tilassa, tulee järjestelmän olla kokonaisuutena johdonmukainen. 

Tämä tarkoittaa, että eri lentokoneiden tulee olla *toisiinsa nähden* johdonmukaisessa tilassa.

### Ulkoinen eheys

Millainen olisi ulkoisesti epäjohdonmukainen tila? 

Esimerkiksi sellainen, jossa kaksi lentokonetta laskeutuisi yhdelle samalle kiitoradalle tismalleen samaan aikaan. Järjestelmän oikean toiminnan kannalta on elintärkeää, että yhdelle kiitoradalle laskeutuu vain yksi lentokone kerrallaan.

Sisäinen eheys on siis lentokoneen sisäisen tilan johdonmukaisuus.

Ulkoinen eheys on eri lentokoneiden johdonmukaisuus toisiinsa nähden.

### Järjestelmän toiminta ja eri eheyksien varmistaminen?

Palataan postauksen alun kultaiseen lausahdukseen:

> Within an aggregate boundary, apply consistency rules **synchronously**. Across boundaries, handle updates **asynchronously**.

Esimerkissämme lentokone on "aggregate boundary". Lausahduksen mukaan meidän tulee lentokoneen sisäinen eheys varmistaa *synkronoidusti*. Synkronoitu tarkoittaa tässä tapauksessa sitä, että muun järjestelmän kannalta lentokoneen tulee olla *kaikkina ajanhetkinä* sisäisesti eheässä tilassa.

Tämä onnistuu transaktioita käyttämällä. Kun lentokone laskee laskutelineensä, tarvitsemme transaktion, joka huolehtii että *laskutelineiden laskeminen* ja *cockpitin telinemittarin päivitys* joko onnistuvat tai epäonnistuvat yhdessä.

Toisin sanoen, missään välissä ei saa olla tilannetta, jossa *laskutelineiden asento* ja *laskutelinemittariston väittämä asento* eivät täsmäisi.

Transaktion tehtävä on huolehtia, että tuollaista epäjohdonmukaisuutta ei pääse syntymään. 

Sitten siirrytään huomattavasti mielenkiintoisempaan kakkosvaatimukseen:

> Across boundaries, handle updates **asynchronously**.

Palataan laskeutumisesimerkkiin. Helsinki-Vantaan ilmatilaan on saapumassa Air Francen Airbus. Samaan aikaan Finnairin DC-10 on parhaillaan kiitoradan #1 alkupäässä odottamassa nousulupaa.

Lennonjohto päättää, että Airbus saa välittömän laskeutumisluvan kiitoradalle #1, ja että DC-10 käyttäköön kiitorataa #2. Mutta DC-10 on iso kone, ja sillä kestää pari minuuttia poistua kiitoradalta #1.

Nyt jos järjestelmä vaatisi eri lentokoneiden välille ("across boundaries") *synkronoitua* eheyttä, ei missään välissä saisi tulla tilannetta, jossa Airbus yrittäisi laskeutua kiitoradalle, jolla on toinen lentokone. Toisin sanoen, synkronoidun eheys vaatimus vaatii, että lennonjohto ensin varmistaa kiitoradan #1 olevan typötyhjä, ja sitten antaa Airbus-koneelle laskeutumisluvan.

Asynkronoidun eheys tapauksessa teemme löysennyksen ylläolevaan: sallimme, että **hetkellisesti** järjestelmä voi olla epäjohdonmukaisessa tilassa. 

Esimerkkimme tapauksessa se tarkoittaa, että Airbus saa laskeutumisluvan kiitoradalle #1 vaikka tuolla kiitoradalla seisoo DC-10 odottamassa nousulupaa. Tämä tilanne aiheuttaa sen, että järjestelmä on hetkellisesti ristiriitaisessa tai epäjohdonmukaisessa tilassa; järjestelmän perussääntö on, että kaksi lentokonetta ei voi käyttää samaa kiitorataa samanaikaisesti.

Huomionarvoista on termi "hetkellinen". Järjestelmän on huolehdittava, että epäjohdonmukaisuus on väliaikainen. Toisin sanoen lennonjohdon on pidettävä huoli, että DC-10 poistuu kiitoradalta ennenkuin Airbus laskeutuu sille.

Asynkronoitu tuo siis mukaan ajallisen ulottuvuuden. Kaksi lentokonetta voi olla toisiinsa nähden epäjohdonmukaisessa tilassa jos a) tuo epäjohdonmukaisuus kestää vain hetken ja b) tuon hetken aikana ei ehdi tapahtua mitään katastrofaalista.

Oikean elämän lennonjohto toimii juurikin asynkronoituun johdonmukaisuuteen perustuen. Kaksi lentokonetta voi olla hetkellisesti suoralla törmäyskurssilla toisiinsa nähden. Riittää, että lennonjohto muuttaa jomman kumman koneen kurssia hyvissä ajoin ennen törmäystä.

### Mitä seurauksia tekniseen toteutukseen?

Asynkronoidun ja synkronoidun johdonmukaisuuksien erottaminen toisistaan antaa meille lisämahdollisuuksia järjestelmän teknisen toteutuksen kannalta.

Synkronoitu johdonmukaisuus täytyy kyetä hoitamaan yhden ja saman transaktion sisällä. Käytännössä tämä tarkoittaa, että transaktion tulee elää yksittäisen tietokoneen (siis ihan fyysisen palvelinraudan) sisällä. 

Asynkronoitu johdonmukaisuus sallii tilanteen, että järjestelmä on hetkellisesti epäjohdonmukaisessa tilassa. Riittää, että ennen pitkään järjestelmä tila palaa johdonmukaiseksi. Tämä sääntökevennys sallii viestittelyn esim. tietoverkkoa pitkin. Järjestelmän yksi osanen voi tehdä omaan tietokantaansa muutoksen, lähettää *sen jälkeen* viestin järjestelmän toiselle osaselle, joka tekee vastaavan muutoksen omaan tietokantaansa. 

Viestin liikkuminen tietoverkon lävitse kestää hetken aikaa; tuon hetken ajan järjestelmä on epäjohdonmukaisessa tilassa. Kun viesti lopulta saapuu vastaanottavaan osaseen, järjestelmä palautuu johdonmukaiseen tilaan.

> Asynkronoidun johdonmukaisuuden vaatimus on löysempi kuin synkronoidun johdonmukaisuuden vaatimus. Synkronoidusti johdonmukainen järjestelmä ei voi olla hetkeäkään epäjohdonmukaisessa tilassa (esim. tilassa, jossa kaksi laskeutuvaa lentokonetta suuntaa kohti samaa kiitorataa). Asynkronoidusti johdonmukainen järjestelmä *voi olla* hetkellisesti epäjohdonmukaisessa tilassa; riittää, että epäjohdonmukaisuus *poistuu* ennenkuin mitään peruuttamatonta vahinkoa ehtii syntymään.




