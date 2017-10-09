+++
date = "2017-10-08T08:19:43+03:00"
draft = false
title = "Yksi tunniste, monta käyttöä"

+++

Yksi erinomainen tapa kytkeä front-end applikaatio rajapintaan, joka vaatii kirjautumisen/tunnistautumisen, on käyttää nk. API-avainta. 

> API-avain on vähän vastaava asia kuin ranneke kesäfestivaaleilla. Kun festivaalien vierailija ensi kertaa astuu festivaalialueelle, häneltä kysytään lippua, mahdollisesti myös henkilökorttia. Lipun antaessaan vierailijalle lätkäistään käteen ranneke. Jos vierailija myöhemmin poistuu festivaalialueelta, hän voi palata sinne takaisin ranneketta (API-avaimen) näyttämällä. Jos rannekkeessa on RFID-siru, rannekkeella voidaan yksilöidä kävijä helposti. Myös API-avain yksilöi käyttäjänsä. Käyttäjän tarkka yksilöinti on valinnainen "lisäpalvelu"; joissain käyttötarkoituksissa riittää tietää, että kävijällä on *oikeus nähdä tiedot* ilman tarvetta tietää *kuka haluaa tiedot nähdä*. Useimmiten API-avain kuitenkin yksilöi käyttäjän.

API-avaimen saa antamalla rajapinnalle validin tunnus+salasana-yhdistelmän. Tällä tavoin rajapinta tietää, että API-avaimen vastaanottava taho on ihan oikea ~~poika~~ palveluun rekisteröitynyt käyttäjä.

API-avain on yleensä voimassa siihen asti, kunnes käyttäjä erikseen kirjautuu ulos palvelusta (rajapinnasta). Vaihtoehtoisesti tunniste voi olla voimassa vain tietyn ajan. 

Tyypillisessä arkkitehtuurissa rajapinnasta saatu API-avain talletetaan käyttäjän tietokoneen kovalevylle talteen. Tällä tavoin käyttäjä pysyy automaattisesti kirjautuneena rajapintaan, vaikka sulkisi tietokoneen välillä. 

> Automaattisesti kirjautuneena pysyminen tässä kohtaa tarkoittaa, että frontend-applikaatio hoitaa kovalevyltä ladatun API-avaimen avulla tunnistautumisen; ihmiskäyttäjän ei tarvitse syöttää salasanaa. Oikeasti käyttäjä ei pysy kirjautuneena yhtään mihinkään. Pinnan alla joka ikisen rajapintakutsun yhteydessä kirjautuminen suoritetaan uusiksi juurikin API-avaimen avulla. Ihmiskäyttäjä ei tätä prosessia näe.

API-avaimenin ominaisuuksiin myös kuuluu useimmiten, että jos käyttäjä tarjoaa validin tunnus+salasana-yhdistelmän vaikka hänellä on (tai pitäisi olla!) hallussaan API-avain, rajapinta generoi uuden API-avaimen. Vanha API-avain lentää roskakoriin. Näin on pakko olla; muuten käyttäjä hävittäessään API-avaimensa ei enää koskaan pääsisi sisälle rajapintaan. Ja kuinka API-avain voi hävitä? Esimerkiksi käyttäjän tyhjentäessä web-selaimen välimuistin.

Tämä malli toimii erinomaisesti. Jos kovalevyltä ei API-avainta löydy, käyttäjän on pakko syöttää salasana. Salasanan (mieluiten oikean) syötettyään käyttäjä saa API-avaimen, jonka voi tallettaa kovalevylleen.

Useimpien web-applikaatioiden yhteydessä 'kovalevy' on synonyymi web-selaimen localStorage:lle. 

## Yksi monen puolesta

Mutta entä jos yhtä rajapintaa käyttää kaksi erillistä web-applikaatiota? Tälläinen tilanne syntyy herkästi nk. micro service -arkkitehtuurissa sovellettuna fronttipuolelle. Yksi rajapinta tarjoaa palvelut monelle web-applikaatiolle, jotka yhdessä muodostavat tuoteperheen. 

Esimerkkinä vaikkapa applikaatiokokonaisuus, jossa yksi web-app huolehtii lomakedatan käsittelystä, ja toinen web-app huolehtii lomakkeiden luonnista (lomake-editori). Molemmat web-appit ovat osa samaa kokonaisuutta, jota kutsuttakoon vaikka "liidien hallinnaksi". 

Kutsutaan applikaatioita vaikka nimillä "Lotus Lomakekäsittely" ja "Lotus Lomake-editori".

On luontevaa, että applikaatiokokonaisuuden tilaava taho saa käyttöön yhdet admin-tunnukset, joilla kirjautua molempiin applikaatioihin sisään.

Mutta jos orjallisesti seuraamme yllä kuvattua API-avaimen käyttömallia, olemme pian dilemman edessä.

## Dilemma

Ongelmaksi muodostuu kysymys siitä, minne tallennamme käyttäjän API-avaimen? Se siis tallennetaan käyttäjän laitteelle. Mutta kumman applikaation alaisuuteen?

Jos tallennamme API-avaimen **Lotus Lomakekäsittelyn** alaisuuteen, **Lomake-editori** ei pääse siihen käsiksi.

Jos tallennamme API-avaimen **Lotus Lomake-editorin** alaisuuteen, **Lomakekäsittely** ei pääse siihen käsiksi.

> Syy siihen mikseivät eri web-applikaatiot (teknisesti eri **web-domainien** alaisuudessa elävät verkkosivut) näe toistensa API-avaimia on tietoturva. Rajoitus estää yhtä web-applikaatio näkemästä dataa, jota joku toinen web-applikaatio tallentanut käyttäjänsä päätelaitteelle.

Tässä kohtaa saattaa nousta ihmetys, että miksi molempien tarvitseekaan päästä yhteen ja samaan API-avaimeen käsiksi? Kuten aiemmin jo mainittua, uuden API-avaimen saa rajapinnasta pyytämällä.

Ongelman ydin on siinä, että kun Lomake-editori pyytää uuden API-avaimen, rajapinta resetoi nykyisen API-avaimen. Lomake-editori ei ole moksiskaan; se halusi uuden tokenin ja sai sen. 

Mutta Lotus Lomakekäsittelylle tilanne on pirullisempi. Sen API-avain on nyt *väärä*. Siis vanhentunut. Vielä hetki sitten sillä oli hallussaan täysin käyttökelpoinen API-avain. Mutta sitten Lomake-editori meni pyytämään itselleen uutta avainta, ja näin toimiessaan Lomake-editori resetoi ja generoi uuden API-avaimen.

Lotus Lomakekäsittelyn avain on siis väärä, joten mitä se tekee? Se tietenkin hakee itse uuden API-avaimen rajapinnasta. Näin toimiessaan Lotus Lomakekäsittely puolestaan invalidoi Lotus Lomake-editorin juuri saadun API-avaimen.

> Lotus Lomakekäsittelyn ja Lotus Lomake-editorin siirtyvät pelaamaan API-pingistä. Kumpikin vuorollaan invalidoi toisen API-avaimen. Ikuinen noidankehä on valmis. 

Mikä avuksi?

## Ratkaisut

Ongelmaan on monta ratkaisua.

### Ratkaisu 1

Yksi ilmiselvä ratkaisu on välttää ongelma kokonaan laittamalla eri applikaatiot saman domainin alle. Jos sekä Lotus Lomake-editori että Lotus Lomakekäsittely elävät samassa valtakunnassa, ne voivat jakaa yhteisen API-avaimen. Tällöin jokainen API-avaimen hakureissu on *yhteinen*. Tosin ikinä nämä applikaatiot eivät tee itse hakureissua yhtäaikaa. Yksi osapuoli hakee, ja palatessaan kiltisti jakaa saadun aarteen toisen osapuolen kanssa.

Ratkaisun ongelma on siinä, että mikäli web-applikaatioiden lähdekoodi elää eri palvelimilla, voi olla ikävän työlästä saada ne saman domainin alaisuuteen.

### Ratkaisu 2

Toinen ratkaisu on tallentaa rajapintaan useampi API-avain. Jos API-avaimia on yksi per applikaatio, ei eri applikaatioiden tarvitse keskenään tapella avaimen herruudesta. Tämä on varsin OK vaihtoehto, mutta loogisesti hiukka luonnottoman tuntuinen. Jos eri web-applikaatioiden käyttöoikeus on selkeästi yhden käyttäjätilin (admin) alaisuudessa, niin loogista olisi, että yksi API-avain kävisi kaikkialle. 

Toinen ongelma on, että jos admin haluaa kirjautua kaikista tuoteperheen applikaatioista ulos, hänen täytyy käydä suorittamassa kirjautumiset yksitellen. Ellei sitten rajapinta sisällä toiminnallisuutta, jolla kaikki API-avaimet voi resetoida kerralla. Niin tai näin, menetelmä tuntuu fundamentaalisesti väärältä.

### Ratkaisu 3 (paras?)

Kolmas ratkaisu on luoda isäntä-renki -hierarkia eri web-applikaatioiden välille. Yksi applikaatio on isäntä, muut renkejä.

Pointti on, että ainoastaan isäntä-applikaatio voi resetoida olemassaolevan API-avaimen. Renki-applikaatiot voivat hakea API-avaimen, mutta eivät resetoida. Tämä ratkoo aiemmin mainitun noidankehän. Kun Lotus Lomakekäsittely ("isäntä") hakee uuden API-avaimen, se samalla resetoi Lotus Lomake-editorin käyttämän API-avaimen. Tämän seurauksena Lomake-editori hakee uuden avaimen. Mutta Lomake-editorin haku ei generoi uutta API-avainta. Rajapinta yksinkertaisesti palauttaa aiemmin isäntä-applikaation toimesta generoidun avaimen. Noidankehän katkeaa; molemmat applikaatiot käyttävät samaa, käyttökelpoista avainta.

> Noheva lukija saattaa nyt miettiä, että eikö koko ruljanssin voisi välttää yksinkertaisesti pitämällä API-avain **aina samana**. Tällöin ei tarvita isäntä-renki -hierarkiaa, sillä kaikki web-applikaatiot ovat renkejä; yksikään ei voi pyytää rajapintaa generoimaan uutta API-avainta.
>
> Yksi ongelma on, että mitä uloskirjautuminen tarkoittaa tapauksessa, jossa API-avain on ikuinen ja koskematon? Uudelleen generoitavan API-avaimen tapauksessa uloskirjautuminen tuhoaa sen hetkisen API-avaimen. Uloskirjautumisen aikana käyttäjällä ei ole lainkaan API-avainta. Kun seuraavan kerran käyttäjä haluaa kirjautua sisään, hänen on pakko syöttää tunnus+salasana.
>
> Tämä on eri tilanne kuin aiemmin mainitussa kahden web-applikaation noidankehässä. API-noidankehässä yksi applikaatio tuhoaa API-avaimen, mutta rajapinta generoi samantien uuden avaimen. Konseptuaalisesti käyttäjällä on siis joka hetkellä aktiivinen API-avain olemassa. 
>
> Mutta jos API-avainta ei koskaan tuhottaisi, niin miten käyttäjä voisi koskaan kirjautua ulos?
>
> Toinen, huomattavasti vakavampi ongelma tässä skenaariossa on, että jos API-avain edes yhden kerran päätyy vääriin käsiin, admin-tunnarit ovat pysyvästi mennyttä. Niihin ei voi enää luottaa. Tämä on valtava tietoturvariski. Siksi API-avaimet resetoidaan jokaisen uloskirjautumisen yhteydessä. Jos hakkeri saa sinun API-avaimen käsiins, riittää että menet pää yhtenä jalkana web-applikaation kirjautumissivulle syöttämään oman tunnus+salasana -yhdistelmän. Yhdistelmän syöttäminen regeneroi uuden API-avaimen, samalla tuhoten hakkerin haltuunsa saaman avaimen.

> API-avainten käyttö on joidenkin mielestä täysin väärin. He suosivat hienompia lähestymistapoja, kuten OAuth. API-avain on kuitenkin yksinkertaisuudessaan ylivertainen ratkaisu, ja maalaisjärkeä käyttämällä varsin tietoturvallinen. Tärkein elementti API-avaimen ja tietoturvan kannalta on SSL-yhteyden käyttö web-applikaation ja rajapinnan välisessä yhteydenpidossa.


