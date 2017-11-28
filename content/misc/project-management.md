+++
date = "2017-11-28T09:45:14+02:00"
draft = true
title = "Ohjelmistoprojektien koordinointi (osa 1)"

+++

Vaativan ohjelmistokehitys on mentaalisesti raskasta ja kuluttavaa puuhaa. Tyypillinen ohjelmistoprojekti koostuu tuhansista ja tuhansista riveistä koodia. Abstraktion tasosta ja applikaation luonteesta riippuen koodin pystyy jaottelemaan *moduuleihin ja komponentteihin*, ja tällä tavoin hahmottamaan kehitysprosessin abstraktioiden yhdistelynä ja muovaamisena.

Mutta abstraktoiminen ja "black box" -ajattelu ovat lähinnä optimisaatioita, eivät ratkaisuja. Mitä suuremmaksi ja vaativammaksi ohjelmistoprojekti paisuu, sitä enemmän se sisältää liikkuvia osia *kaikilla* abstraktion tasoilla; yksittäisten funktioiden määrä (luonnollisesti) kasvaa kasvamistaan, mutta tämä kasvu on vertailussa vähäpätöinen ongelma, sillä funktioista elää kiltisti jonkin ylemmän tason abstraktion sisällä. 

Suurempi ongelma on, että abstraktion *ylimmällä* tasolla komponentit yhä enemmän kytkeytyvät toisiinsa. Ne siis entistä tiiviimmin kiinnittävät limaiset lonkeronsa toistensa sisuskaluihin. Tämä on seurausta kahdesta erillisestä ilmiöstä:

1. **Ajallinen ulottuvuus ("hyvätkin ideat tuppaavat unohtumaan"):** Ohjelmistoprojektin alkuvaiheessa kokonaisarkkitehtuuri on tuoreena mielessä, ja koodin määrä on vähäinen, joten arkkitehtuurillisesti kauniit/järkevät ratkaisuja ovat helppoja. Mitä pidempään projekti jatkuu, sitä häilyvämmäksi applikaation arkkitehtuuri muuntuu ohjelmoijan pään sisällä. Alunperin kirkkaana ollut idea pikkuhiljaa häviää harmaan sumuverhon taakse.

2.  **Psykologinen ulottuvuus ("kuka idiootti on tämän kirjoittanut"):** Vaativan ohjelmistoprojektin hieno asia on, että se kehittää ohjelmoijaa aivan helvetisti. Kuusi kuukautta projektin aloituksen jälkeen ohjelmoija katsoo koodiaan, jonka on itse kirjoittanut kuusi kuukautta aiemmin, ja naurahtaa: *ei jumalauta, olinpa amatööri*. Tämä on tietenkin hieno tunne, mutta psykologisesti sillä on ikävä seuraus; ohjelmoija alkaa alitajuntaisesti halveksua aiempaa, amatöörimäistä koodiaan ja haluaa pysyä siitä erossa. Mutta koska projekti jatkuu ja vaatii lisäkehitystä, ohjelmoijan täytyy elää oman menneisyytensä kanssa. Tämä on psykologisesti yllättävän raskasta. Ei ihme, että ohjelmoijat ovat kovin herkkiä kirjoittamaan tarvittaessa *vaikka koko applikaation alusta asti uusiksi*, eli nk. rewrite. Kun uusi ja parempi ratkaisu on materialisoitunut ohjelmoijan pääkoppaan, on lähes mahdoton jättää vanha, huonon ratkaisun sisältävä koodi rauhaan.

Perimmäinen syy ilmiöön numero 1 on ihmisen työmuistin rajallinen koko. Ilmiön 2 taustalla taas on kaikille kunnianhimoisille ihmisille tyypillinen perfektionismi.

Ilmiö 2 on kenties suurin yksittäinen syy siihen, miksi fiksut ihmiset tuppaavat saamaan niin *vähän* aikaa työurallaan. Mutta ei siitä sen enempää. Keskitytään ilmiöön 1. 

### Abstraktion eri tasot ja työmuisti 

Työmuistin rajallinen koko aiheuttaa sen, että ohjelmoija joutuu kaikilla abstraktion tasoilla "paloittelemaan maailman" kouralliseen yksittäisiä konsepteja. 

Mitä tarkoitan tällä?

Sitä, että työmuistiin on aina mahduttava koko *tarkastelun alaisena oleva maailma* kerrallaan.

#### Komennot (alin taso)

Alimmalla abstraktion tasolla huomiokyky (ja työmuistin sisältö) on keskittynyt asettelemaan yksittäiset koodikomennot järkevästi ja siten, että ne toimivat. Epävirallisesti voimme sanoa, että yksittäiset koodikomennot ovat palasia, joista funktiot ja metodit koostuvat. Tällä tasolla ohjelmointi on lähinnä komentojen syöttämistä mikroprosessorille, ja tarkastelun alaisena oleva maailma on *yksittäisen komennon suorittaminen*.

#### Funktiot 

Ylöspäin mentäessä seuraavalla abstraktion tasolla ohjelmoija käsittelee funktioita. Jo tällä tasolla siirrytään pois raudan parista, ja käytetään näkökulmaa "mitä halutaan saavuttaa", ei "miten halutaan saavuttaa". Web-ohjelmoinnin piirissä tämä on käytännössä alin taso. 

> Web-ohjelmoija ei kerro tietokoneelle, *miten* HTML-elementti asetellaan ruudulle, vaan *minne* HTML-elementti asetellaan. Tietokone sitten ratkoo kaikki käytännön ongelmat, kuten yksittäisten pikseleiden värittämisen.

Tällä tasolla tarkastelun alaisena oleva maailma on esimerkiksi animaation pyöritys osana tietokonepeliä. Tyypillinen animaatio on kokoelma osa-animaatioita. Sanotaan vaikka, että meillä on animaation nimeltä "vieteriukon ilmestyminen". Tuon animaation osa-animaatiot ovat seuraavat: "avaa laatikon yläkansi, pompauta vieteriukko ulos"). Kumpikin noista osa-animaatioista voi puolestaan koostua alemman tason osa-animaatioista. Jossain kohtaa sitten tullaan osa-animaatioon, joka kirjaimellisesti värittää näyttöpäätteen pikseleitä 60 kertaa sekunnissa, mutta oleellista on, että ylimmällä tasolla ("vieteriukon ilmestyminen") emme välitä pikseleistä pätkän vertaa. 

Ja koska emme välitä, eivät pikselit ja niiden värityksestä huolehtiminen rasita työmuistiamme.

> Tämä on kaiken ohjelmoinnin perusta; tietyllä abstraktion tasolla *emme välitä* alemman tason toiminnoista. Otamme ne vastaan annettuina, ja sokeasti luotamme, että ne toimivat.

#### Moduulit ja komponentit

Ylöspäin mentäessä siirrytään joko moduuleiden (löyhästi *kokoelma toisiinsa liittyviä funktioita*) tai komponenttien tasolla (löyhästi *erillinen palikka, joka kykenee itsenäisesti suorittamaan vaativia tehtäviä, esim. sähköpostin lähetyksen*). Tällä tasolla syntyy ensimmäistä kertaa kokonaiskuva applikaatiosta, jota ollaan rakentamassa. Applikaatio koostuu komponenteista (jotka puolestaan pohjautuvat moduuleihin), jotka vuorovaikuttavat toistensa kanssa. Yhdistelemällä komponentteja ja rakentamalla informaatioväyliä komponenttien välille saavutetaan applikaatio.

> Komponentin ja moduulin ero on tärkeä ymmärtää; moduuli on **staattinen** kokoelma koodia, jolla on jokin yhteinen tarkoitus olla olemassa. Komponentti on **dynaaminen** palikka, joka elää ohjelman ajon aikana ja suorittaa vastuulleen kuuluvia velvollisuuksia. Komponentti on siis ohjelman ajon aikana elävä asia; moduuli puolestaan on kasa koodia, joka elää "koodieditorissa". 
>
>
>
> Ero on sama kuin Pythagoraan lauseella ja Kheopsin pyramidilla; Pythagoraan lause ei ole olemassa muuten kuin abstraktina sääntönä, jonka perusteella voidaan käsin kosketeltavia asioita (kuten pyramidit) rakentaa.

#### Osa-applikaatiot, rajapinnat ja palvelu-arkkitehtuuri

Abstraktion ylimmällä tasolla komponentit muodostavat kokonaisuuksia, joita voi kutsua "osa-applikaatioiksi". Web-applikaatioissa esim. frontend vs. backend -jaottelu on tyypillinen esimerkki osa-applikaatioista; frontend on yksi applikaatio, backend on toinen, ja yhdessä ne muodostavat halutun "kokonaisapplikaation", joka toivottavasti täyttää jonkin oikean maailman tarpeen. Useimmiten nämä osa-applikaatiot keskustelevat vastaavalla tavalla kuin me ihmisetkin; ne rimpauttavat toisilleen HTTP-protokollan (tai jonkin alemman, kuten TCP-protokollan) avulla ja kertovat kuulumisensa. Jokainen osa-applikaatio tarjoaa rajapinnan, johon muut osa-applikaatiot voivat soitella.

* Osa 2 - Jatkuu huomenna... *

