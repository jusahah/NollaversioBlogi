+++
date = "2016-08-09T06:07:23+03:00"
draft = false
title = "Kuinka CSRF toimii?"

+++

Lomakkeiden lähetys ja vastaanotto ovat tyypillisen web-applikaation tärkeimpiä vastuutehtäviä.

Lomakkeiden ja niiden datalähetysten suojaus on tärkeä aspekti turvallisen web-applikaation kannalta. Keskitytään tässä postauksessa yhteen suojamuuriin; CSRF-suojaukseen.

### CSRF

CSRF tulee sanoista "Cross-Site Request Forgery". Sanahirviö tarkoittaa yksinkertaisesti tilannetta, jossa *rikollinen käyttäjä* huijaa web-applikaatiota luulemaan, että viesti tulee *rehelliseltä* käyttäjältä. 

Erityisesti tämä puijaus kohdistuu lomakkeiden lähetyksiin. Tyypillisellä web-applikaatiolla on oltava jokin tapa mahdollistaa käyttäjiensä tallettaa/muokata sisältöä. 

Tuo sisältö voi olla blogipostauksia, pankkimaksuja, lentovarauksia, jne. 

Useimmiten uuden sisällön luomista varten web-applikaatio tarjoaa lomakkeen, jonka täyttämällä ja lähettämällä sisällön luonti tapahtuu.

Otetaan esimerkkinä pankkisuorituksia hallinnoiva sivusto. Sivustolla voi tehdä tilisiirron täyttämällä lomake:

> Lähettäjän tili:
>
> Saaja:
>
> Saajan tili:
>
> Summa:
>
> Viesti:
>
> Eräpäivä:
>

Lomakkeen alla on "Maksa"-nappula, jota painamalla lähetys lähtee liikkeelle. 

Ongelmaksi ylläolevassa muodostuu, että kuka tahansa voi luoda ylläolevan kaltaisen datapaketin, ja lähettää sen nettipankkiapplikaation suuntaan. Esimerkiksi minä voisin luoda seuraavanlaisen lähetyksen:

> Lähettäjän tili: Kimi Räikkönen FI00112233-4
>
> Saaja: Jussi Hämäläinen
>
> Saajan tili: FI22334455-6
>
> Summa: 1000
>
> Viesti: Jäätelöraha
>
> Eräpäivä: 9.8.2016
>

Nettipankkiapplikaatio vastaanottaa ylläolevan lomakelähetyksen. Mitä tapahtuu vastaanoton jälkeen?

Ei mitään, sillä käyttäjä "Kimi Räikkönen" ei ole kirjautunut sisään. Eli tilisiirtoa ei tapahdu. Huomioitavaa on, että Räikkösen sisäänkirjautuminen ei vaikuta CSRF-suojauksen toimintaan. 

Vain sisäänkirjautuneet käyttäjät voivat luoda tilisiirtoja, joissa "Lähettäjän tili" on oma tili.

Mutta mennään askel pidemmälle. Kuvitellaan, että olen *jotenkin* onnistunut injektoimaan skriptin nettipankin käyttöliittymään. 

Tällä tarkoitan, että kun nettipankin käyttöliittymäsivu rakennetaan HTML-koodista, olen jollain tavalla onnistunut työntämään tuohon rakennusvaiheeseen palan painikkeeksi haluamaani koodia. 

Ilmiöstä käytetään nimitystä XSS (Cross-Site Scripting).

XSS:n avulla kykenen toteuttamaan seuraavanlaisen tempun. Seuraavan kerran kun Kimi Räikkönen - siis oikea Kimi, joka tietää omat pankkitunnuksensa - loggautuu nettipankkijärjestelmään sisään ja siirtyy maksusuoritusten luomissivulle, **minun** määrittämä koodinpätkäni suoritetaan Räikkösen tietokoneen web-selaimessa.

Mitä tuo minun määrittämä koodinpätkä tekee? 

Se lähettää lomakelähetyksen (kuten yllä, jossa Räikkönen vippasi minulle tonnin jäätelörahaa) nettipankin suuntaan.

Lomakelähetys saapuu nettipankin rajapintaan. Ja nyt tullaan tärkeään vaiheeseen: **koska Kimi Räikkönen on kirjautunut sisään omilla oikeilla tunnuksillaan, nettipankki luulee, että Räikkönen toden totta on tuon tilisiirron takana**. Ja miksi ei luulisi? 

Tilisiirron tiedot sisältävä lomakelähetys lähti liikkeelle Räikkösen tietokoneelta. Nettipankkiapplikaatio ei tiedä, että lähetyksen liikkeellelähdön sai aikaan *minun* ohjelmoimani skripti, joka *ajettiin* Räikkösen www-selaimen sisällä.

Nettipankille tilanne näyttää siltä, että Räikkönen täytti lomakkeen ja klikkasi "Maksa"-nappulaa.

Joten nettipankilla ei ole mitään syytä epäillä, etteikö lomakelähetys olisi aito. Siispä se tekee tilisiirron ja raha vaihtaa omistajaansa.

### Miten CSRF-suojaus auttaa?

CSRF-suojauksen ydinajatus on, että kun käyttäjälle tarjotaan lomaketta täytettäväksi, tuo lomake yksilöidään jollain tunnisteella. Myöhemmin web-applikaatio kykenee tämän yksilöidyn tunnisteen avulla varmistamaan, että saapuva lomakelähetys (esim. tilisiirron tiedot) on luotu asianmukaisesti.

Eli aiempi datalähetys

> Lähettäjän tili: Kimi Räikkönen FI00112233-4
>
> Saaja: Jussi Hämäläinen
>
> Saajan tili: FI22334455-6
>
> Summa: 1000
>
> Viesti: Jäätelöraha
>
> Eräpäivä: 9.8.2016
>

menee nyt muotoon

> _CSRF: ejse72Hja7299391Jkla28
>
> Lähettäjän tili: Kimi Räikkönen FI00112233-4
>
> Saaja: Jussi Hämäläinen
>
> Saajan tili: FI22334455-6
>
> Summa: 1000
>
> Viesti: Jäätelöraha
>
> Eräpäivä: 9.8.2016
>

Kuten huomaamme yltä, lomakedatan yhteyteen on lisätty "_CSRF"-niminen lomakekenttä. 
Käytännössä web-applikaatio siis lähettää lomakesivun mukana CSRF-tunnisteen, ja myöhemmin vastaanottaa datan sisältäen CSRF-tunnisteen. **Vain jos nämä kaksi CSRF-tunnistetta täsmäävät, applikaatio hyväksyy vastaanotetun datan**.

Jos ne eivät täsmää, applikaatio kieltäytyy toimimasta.

Miksi CSRF-tunnisteiden täsmääminen ratkoo aiemman jäätelörahahuijauksen?

Yksikertaisesti siksi, että nettipankki osaa yhdistää tarjotun lomakkeen ja vastaanotetun lomakedatan toisiinsa. Täten jos minä XSS:n kautta lähetän tilisiirtolähetyksen (Räikkösen koneelta käsin, kiitos XSS:n), niin nettipankkiapplikaatio tekee seuraavan tarkistukset:

1. Tilisiirtolähetys lähti liikkeelle sisäänkirjautuneelta käyttäjältä - > check!
2. Tilisiirtolähetyksen CSRF-tunniste täsmää applikaation tallentaman tunnisteen kanssa -> fail!

CSRF-tunniste ei täsmää, sillä *minun etukäteen tuottamani* lomakelähetys ei tiedä tuota tunnistetta. Tunniste luodaan jokaiselle lomakkeelle erikseen, ja se on satunnainen merkkijono. 

Lopputulos siis on, että nettipankkiapplikaatio **ei** tee tilisiirtoa, ja jään ilman jäätelörahaa.

> Loppukaneetti: XSS:n avulla saattaa teoriassa olla mahdollista *selvittää* CRSF-tunniste kesken hyökkäyksen. Tällöin CSRF-suojaus menettää tehonsa. Tämä vaatii, että XSS-hyökkääjällä on mahdollisuus ajaa mielivaltaista Javascript-koodia uhrinsa www-selaimessa. 

> Jos tätä mahdollisuutta ei ole, CSRF-suojaus toimii ja estää useimmat muunlaiset hyökkäysyritykset; esim. linkki-injektion, jossa rikollinen on *jotenkin* saanut nettipankin käyttöliittymään lisättyä linkin, jota klikkaamalla etukäteen suunniteltu lomakedata lähtee salaa liikkeelle. Koska tuo etukäteen suunniteltu lomakedata ei voi mitenkään tietää ETUKÄTEEN sen hetkisen CSRF-tunnisteen merkkijonoa, CSRF-suojaus toimii ja rikollinen jää nuolemaan näppejään.


 

