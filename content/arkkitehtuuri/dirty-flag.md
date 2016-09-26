+++
date = "2016-09-26T10:22:11+03:00"
draft = false
title = "Likainen lippu - vältä turhaa työtä"

+++

Törmäsin patterniin nimeltä "dirty flag". Tuo patterni on ollut käytössä itselläni useissa applikaatioissa, mutta vasta nyt tajusin että sille on annettu tarkka nimikin.

Minkä ongelman dirty flag ratkoo? 

Kuvitellaan applikaatio, joka analysoi shakkiasemia reaaliajassa. Applikaatio pitää kirjaa tietyn shakkipelin - jota kaksi ihmispelaajaa pelaa - siirroista. Applikaation kautta katsojat voivat seurata tuota peliä. Lisämausteena applikaatio tarjoaa analysointipalvelun, jonka kautta katsojat saavat tietokonearvion kulloisestakin peliasemasta.

Shakkipeliaseman tietokonearvio on aika raskas laskenta suorittaa. Luotettavan arvio tuottaminen tekoälyn turvin vie rutosti CPU-aikaa. Täten analysointi suoritetaan vain kun tarve vaatii.

Jos esimerkiksi peliä ei tietyllä ajanhetkellä seuraa yhtään katsojaa, on laskentatehon väärinkäyttöä tuottaa analysointipalvelua. Reaaliaikaisesta analysoinnista ei ole hyötyä jos kukaan ei ole sitä näkemässä.

Toinen huomioonotettava seikka on, että kukin asema on järkevää analysoida vain kerran. Kun analysointi tietylle asemalle on suoritettu, analysoinnin tulos talletetaan välimuistiin. 

Jälkimmäinen vaatimus antaa hyvän syyn käyttää *likaista lippua*. Kun katsojalta tulee pyyntö saada tuorein analysointitulos käyttöönsä, seuraava algoritmi ajetaan:

1. Jos likainen lippu olemassa, hae analysointitulos välimuistista.
2. Jos likaista lippua ei olemassa, hae tuore asema tietokannasta. Aloita sen analysointi. Aseta muuttuja ilmoittamaan analysoinnin käynnissäolo.
3. Kun analysointi valmis, talleta tulos välimuistiin ja aseta likainen lippu.

Kun taas uusi peliasema saapuu, toimimme yksinkertaisesti seuraavasti:

1. Talleta peliasema applikaation tietokantaan. Älä aloita analysointia.
2. Jos likainen lippu olemassa, tuhoa se.

Upouuden aseman saapuessa siis tuhoamme (mahdollisen) vanhan likaisen lipun. Tällä tavalla seuraavan kerran kun joku katsojista pyytää viimeisintä analyysiä käyttöönsä, applikaatio osaa hakea tuoreimman aseman tietokannasta ja aloittaa sen analysoinnin. 

Kun joku toinen katsoja tämän jälkeen pyytää analyysiä, likainen lippu on jo olemassa ja analysointi ei käynnisty. Sen sijaan viimeisin analysointitulos palautetaan välittömästi välimuistista.

Toisin sanoen likainen lippu kertoo vastauksen seuraavaan kysymykseen: *onko analysointi tuoreimmalle peliasemalle jo kertaalleen suoritettu?*.

Jos on, palauta tulos välimuistista.

Jos ei, aloita analysointi ja analysoinnin päätyttyä aseta likainen lippu.

Ja uuden aseman saapuminen luonnollisesti tuhoaa likaisen lipun; muussa tapauksessa yksi ja sama analysointitulos palautettaisiin uudestaan ja uudestaan riippumatta peliasemasta. *Koska kukin analysointitulos on järkevä vain yhden ja tietyn peliaseman yhteydessä*, täytyy analysointi suorittaa erikseen jokaiselle peliasemalle.

Yllämainitun arkkitehtuurin suuri vahvuus on, että *mikäli hetkellisesti shakkipeliä ei seuraa yhtään ainutta katsojaa, ei myöskään analysointia ajeta.* Tämä johtuu siitä tosiasiasta, että analysointi käynnistyy vain katsojan **pyytäessä** tuoreinta analyysitulosta. Jos yksikään katsoja ei ole paikalle pyyntöjä tekemässä, analyysi jää suorittamatta.

Tällä tavoin vältetään turhaa työtä.

> Dirty flag -patternin ydinajatus on välttää turhaa työtä. Ajatus on vastaava kuin inventaariota tehdessä ruokakaupassa. Inventaarion tekeminen on valtava urakka. Kun se on kerran tehty, sitä ei ole järkeä tehdä uudestaan *ennenkuin vähintään yksi tuote on saapunut/poistunut hyllyistä*. Kahden inventaarion tekeminen perätysten on järjetöntä ajanhaaskausta; ne kun tuottavat saman tuloksen. Parempi tehdä yksi inventaario, asettaa *dirty flag*, ja tehdä seuraava inventaario vasta kun tarpeeksi paljon tuotteita on liikkunut kaupasta ulos ja sisään.



