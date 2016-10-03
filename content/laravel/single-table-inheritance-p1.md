+++
date = "2016-10-03T18:12:51+03:00"
draft = false
title = "Yksi taulu, useampi objekti"

+++

Tietokantapohjaisissa web-applikaatioissa tulee käyttöön aina välillä kätevä konsepti nimeltä "Single table inheritance", eli "yhden taulun periytyvyys".

Konsepti mahdollistaa useamman eri datatyypin objektin tallennettavan yhteen tietokantatauluun. 

Lähtökohtaisesti useamman eri objektin tallennuksessa samaan tauluun *ei ole mitään järkeä*. Active Record-pohjaisissa järjestelmissä kukin ns. malliobjekti on kytketty pinnan alla yhteen tauluun, ja jos kaksi objektia kytkeytyy samaan tauluun, täytyy niillä olla samanmoiset attribuutit. Tämä siksi, että kukin tietokantataulu sisältää tietyn määrän attribuutteja (sarakkeita), ja tauluun menevän objektin tulee mukauttaa itsensä noihin attribuutteihin.

Esimerkiksi objektiluokan "Hevonen" ja "Tilisiirto" kytkeminen osaksi samaa tietokantataulua kuulostaa aika järjettömältä. Hevonen on elävä eläin, Tilisiirto on abstrakti konsepti liittyen pankkitoimintaan. Kovin paljoa yhteistä ei noilla kahdella objektilla ole.

> Tilanne on vähän sama kuin jos yrittäisit valmistaa kulkuneuvon, joka liikkuu sekä ilmojen halki että vetten alla sukelluksissa. Ehkä saisit sellaisen aikaan, mutta kovin käytännöllinen tuo vehje ei varmasti ole.

Mutta entä jos meillä on jokin abstrakti konsepti, josta on mahdollista tuottaa konkreettisia objekteja?

Esimerkkinä vaikka "Kommunikaatio". Kommunikaatio on abstrakti konsepti; se kuvaa motiivin vaihtaa informaatiota, mutta ei määrittele *miten* informaatiota vaihdetaan.

"Puhelin" puolestaan on konkreettinen objekti, joka menettelee miten informaatiota vaihdetaan.

Samoin on "Savumerkki". Samoin on "Valomerkki". Kaikki nuo tarjoavat *menetelmän* suorittaa käytännön maailmassa konsepti "Kommunikaatio".

Kuvitellaan sitten, että meillä on Kommunikaatio-niminen luokka. Tuohon luokkaan on kytketty tietokantataulu "kommunikaatiot". 

**Nyt suuri kysymys**: miten saamme järkevästi kommunikaatiot-tauluun talletettua erilaisia kommunikaatiovälineitä?

Toinen suuri kysymys: miksi haluaisimme tehdä niin? Miksi emme vain loisi uutta tietokantataulua jokaista kommunikaatiovälinettä varten? Esim. "Puhelin" objektia varten taulu "puhelimet". Savumerkkiä varten taulu "savumerkit".

Eli: **Miten yhdistämme usean eri luokan yhteen tauluun ja miksi haluamme niin tehdä?**

(Esimerkki jatkuu huomenna)





