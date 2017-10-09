+++
date = "2016-10-08T10:03:38+03:00"
draft = false
title = "Yksi tunniste, monta käyttöä"

+++

Yksi erinomainen tapa kytkeä front-end applikaatio rajapintaan, joka vaatii kirjautumisen/tunnistautumisen, on käyttää nk. API Tokenia. 

> API Token on vähän vastaava asia kuin ranneke kesäfestivaaleilla. Kun festivaalien vierailija ensi kertaa astuu festivaalialueelle, häneltä kysytään lippua, mahdollisesti myös henkilökorttia. Lipun antaessaan vierailijalle lätkäistään käteen ranneke. Jos vierailija myöhemmin poistuu festivaalialueelta, hän voi palata sinne takaisin ranneketta (API Token) näyttämällä. Jos rannekkeessa on RFID-siru, rannekkeella voidaan yksilöidä kävijä helposti. Myös API Token yksilöi käyttäjänsä. Käyttäjän tarkka yksilöinti on valinnainen "lisäpalvelu"; joissain käyttötarkoituksissa riittää tietää, että kävijällä on *oikeus nähdä tiedot* ilman tarvetta tietää *kuka haluaa tiedot nähdä*. Useimmiten API Token kuitenkin yksilöi käyttäjän.

API Tokenin saa antamalla rajapinnalle validin tunnus+salasana-yhdistelmän. Tällä tavoin rajapinta tietää, että API Token vastaanottava taho on ihan oikea poika palveluun rekisteröitynyt käyttäjä.

API Token on yleensä voimassa siihen asti, kunnes käyttäjä erikseen kirjautuu ulos palvelusta (rajapinnasta). Vaihtoehtoisesti tunniste voi olla voimassa vain tietyn ajan. 

Tyypillisessä arkkitehtuurissa rajapinnasta saatu API Token talletetaan käyttäjän tietokoneen kovalevylle talteen. Tällä tavoin käyttäjä pysyy automaattisesti kirjautuneena rajapintaan, vaikka sulkisi tietokoneen välillä. 

> Automaattisesti kirjautuneena pysyminen tässä kohtaa tarkoittaa, että frontend-applikaatio hoitaa kovalevyltä ladatun API Token avulla tunnistautumisen; ihmiskäyttäjän ei tarvitse syöttää salasanaa. Oikeasti käyttäjä ei pysy kirjautuneena yhtään mihinkään. Pinnan alla joka ikisen rajapintakutsun yhteydessä kirjautuminen suoritetaan uusiksi juurikin API Tokenin avulla. Ihmiskäyttäjä ei tätä prosessia näe.

API Tokenin ominaisuuksiin myös kuuluu useimmiten, että jos käyttäjä tarjoaa validin tunnus+salasana-yhdistelmän vaikka hänellä on (tai pitäisi olla!) hallussaan API Token, rajapinta generoi uuden API Tokenin. Vanha API Token lentää roskakoriin. Näin on pakko olla; muuten käyttäjä hävittäessään API Tokeninsa ei enää koskaan pääsisi sisälle rajapintaan. Ja kuinka API Token voi hävitä? Esimerkiksi tyhjentämällä web-selaimen välimuistin.

Tämä malli toimii erinomaisesti. Jos kovalevyltä ei API Tokenia löydy, käyttäjän on pakko syöttää salasana. Salasanan (mieluiten oikean) syötettyään käyttäjä saa API Tokenin, jonka voi tallettaa kovalevylleen.

Useimpien web-applikaatioiden yhteydessä 'kovalevy' on synonyymi web-selaimen localStorage:lle. 

##Yksi monen puolesta

Mutta entä jos yhtä rajapintaa käyttää kaksi erillistä web-applikaatiota? Tälläinen tilanne syntyy herkästi nk. micro service -arkkitehtuurissa sovellettuna fronttipuolelle. Yksi rajapinta tarjoaa palvelut monelle web-applikaatiolle, jotka yhdessä muodostavat tuoteperheen. 

Esimerkkinä vaikkapa applikaatiokokonaisuus, jossa yksi web-app huolehtii lomakedatan käsittelystä, ja toinen web-app huolehtii lomakkeiden luonnista (lomake-editori). Molemmat web-appit ovat osa samaa kokonaisuutta, jota kutsuttakoon vaikka "liidien hallinnaksi". 

Kutsutaan applikaatioita vaikka nimillä "Lotus Lomakekäsittely" ja "Lotus Lomake-editori".

On luontevaa, että applikaatiokokonaisuuden tilaava taho saa käyttöön yhdet admin-tunnukset, joilla kirjautua molempiin applikaatioihin sisään.

Mutta jos orjallisesti seuraamme yllä kuvattua API Tokenin käyttömallia, olemme pian dilemman edessä.

(jatkuu huomenna...)






