+++
date = "2016-10-24T17:41:19+03:00"
draft = false
title = "AWS ja harva indeksi"

+++

Amazonin Dynamo-tietokantaa käytettäessä törmäsin tänään mielenkiintoiseen patterniin. Tarvitsin taululle indeksin attribuuttia varten, joka harvoin saa yhtään mitään arvoa. 

Tälläisessä tapauksessa on ikävää joutua luomaan uusi, täysimittainen indeksitaulu. 

Häh, miksi tuo on niin ikävää muka? Koska jos 99% talletettavista objekteista ei hyödy indeksistä lainkaan, niiden roikottaminen mukana indeksitaulussa on tilanhukkaa.

Otetaan konkreettinen esimerkki. Sanotaan huvin vuoksi, että meillä on seuraavanlainen tietokantataulu:

```

| nimi       | ikä | maailmanmestari |
| ---------- | --- | --------------- |
| Matti M    | 62  |        -        |
| Pekka J    | 11  |        -        |    
| Ismo P     | 16  |        -        | 
| Kimi R     | 37  |    Formula 1    | 

// jne. jne.

```

Ylläoleva taulu sisältää kaikista suomalaisista kolme tietoa; *nimi*, *ikä*, ja *minkä urheilulajin maailmanmestaruuden henkilö on voittanut*.

Sanotaan että nimi-attribuutti muodostaa ns. pääavain-indeksin. Sen tulee siis olla uniikki - täyskaimoja tietokantamme ei salli. 

> Käytännössä emme tietenkään käyttäisi nimeä pääavaimena, vaan pääavain olisi henkilötunnus. En valitettavasti satu tietämään Räikkösen Kimin hetua joten esimerkki toimii paremmin näin.

Nyt voimme luoda kaksi indeksitaulua varsinaisen taulun oheen. Yksi indeksi iälle, toinen maailmanmestaruudelle. 

Tällä tavoin nopeutamme merkittävästi hakuja, joissa ikää tai maailmanmestaruutta käytetään hakukriteerinä.

> Esimerkki ikä-attribuuttia hakukriteerinä käyttävästä hausta: *palauta kaikki henkilöt, joiden ikä on 60 ja 65 välillä*

> Esimerkki maailmanmestari-attribuuttia käyttävästä hausta: *palauta kaikki henkilöt, jotka ovat voittaneet keihäänheiton MM-kultaa* 

Kaikki hyvin. On kuitenkin huomattavast, että ikä-indeksitaulu sisältää viisi miljoonaa riviä. Tämä ihan siksi, että alkuperäinen taulu sisältää myös viisi miljoonaa riviä, ja jokainen henkilö tulee indeksoida iän perusteella, jotta ikä-indeksi toimii oikein.

### Mutta kuinka moni suomalainen on voittanut MM-kultaa yhtään missään?

Datan indeksointi ikä-attribuutin suhteen on siis varsin järkevä idea. 

Jokainen henkilö kun on *jonkin ikäinen*. 

Vaan kuinka moni on voittanut *jonkin lajin maailmanmestaruuden*? 

Ydinkysymys on tämä: kuinka suuri on maailmanmestareiden osuus on verrattuna koko väestöön?

Sanotaan esimerkin vuoksi, että mestareiden lukumäärä on 1000 henkilöä. *Eli koko kansasta 0.02%*. Tästä herää pieni suorituskyvyllinen ongelma: **jos luomme indeksin *maailmanmestari*-attribuutille, 99.98% indeksitaulun jäsenistä on siellä ihan turhaan.**

He eivät ole voittaneet mestaruutta, joten ei heitä tarvitse indeksoida. Ei ole mitään mitä indeksoida! Sama kuin yrittäisi indeksoida sosiaalidemokraattien itsekunnioitusta.

Tälläinen tuhlaus kuulostaa hirveältä: 0.02% takia 99.98% joutuu kärsimään. Siis kärsimään siinä mielessä, että heille luodaan oma turhanpäiväinen rivi indeksitauluun. 

### Harva indeksi - jätä luuserit pois alunperinkin

Harva indeksi tulee apuun. Ydinpointti on tässä: miksi emme loisi *maailmanmestari*-indeksiä siten, että se sisältää **ainoastaan** maailmanmestarit?

Ajatus on varsin luonteva, ja vain ohjelmistosuunnittelija voi ilakoida sen hoksaamisella. Mutta kuitenkin - harva indeksi on pätevä ratkaisu ongelmaamme.

> Käytännössä luomme siis "eliitti-indeksin" - vain maailmanmestarit kelpuutetaan mukaan listaukseen. Indeksi toimii ikäänkuin urheilumaailman "Kuka kukin on"-oppaana.

Harvan indeksin avulla saamme pudotettua 5 miljoonan rivin kokoisen indeksin vaivaiseksi 1000 rivin indeksiksi. Tilaa säästyy valtava määrä.

### Amazon tekee harvan indeksin ohjelmoijan puolesta

Amazonin DynamoDB:ssä harvan indeksin luonti on helppoa. Jopa niin helppoa, että se tapahtuu täysin automaattisesti järjestelmän toimesta. Ihan totta, kirjaimellisesti ohjelmoijan *ei tarvitse tehdä yhtään mitään*.

Teknisesti AWS:n toteutus toimii siten, että aina kun uutta objektia lisättäessä tuon objektin indeksoidun attribuutin jättää tyhjäksi, objektia ei indeksoida lainkaan. Henkilötaulun esimerkki yllä on suoraan siirrettävissä DynamoDB:n puolelle - lisätessämme uusia henkilöitä tietokantaan riittää, että jätämme *maailmanmestari*-kentän tyhjäksi. 

Jos emme jätä sitä tyhjäksi, henkilö on voittanut maailmanmestaruuden, ja Amazonin taustajärjestelmä indeksoi hänet oikeaoppisesti.

Kätevää.

> Miten MySQL toimii indeksoitavan kentän jäädessä tyhjäksi? [Tämän linkin](http://stackoverflow.com/questions/32217099/mysql-index-for-sparse-table) mukaan Mysql osaa ottaa asian huomioon jos asettaa kentän eksplisiittisesti arvoon *NULL*. En muista kokeilleeni asiaa käytännössä.



