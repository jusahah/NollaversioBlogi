+++
date = "2016-08-26T04:30:43+03:00"
draft = false
title = "Arkkitehtuuri: ohjaa pelaajat eteenpäin"

+++

Esittelen lyhyesti arkkitehtuurin, joka sopii mainiosti Laravel + Node.js -yhteisarkkitehtuureihin. 

Tälläinen yhteisarkkitehtuuri  tyypillisesti jakautuu vastuualueisiin siten, että Node.js hoitaa reaaliaikapuolen ja Laravel hoitaa admin-toiminnot ja pitkäaikaisvarastoinnin. Node.js on erinomainen ratkaisu reaaliaikaisesta tiedonvaihdosta huolehtimiseen. PHP ja Laravel taas loistavat perinteisten ei-reaaliaikaisten web-käyttöliittymien kohdalla. Yhdessä Node.js ja Laravel tekevät ihmeitä.

Rakensin viime syksynä kokonaisarkkitehtuurin reaaliaikaisten tietovisojen luomiseen ja pelaamiseen. Palvelun kautta pelaajat voivat pelata reaaliajassa toisiaan vastaan tietovisoja. Tuon järjestelmän kokonaisarkkitehtuuri on seuraavalainen:

1. Laravel-applikaatio tarjoaa admin-käyttöliittymän, jonka kautta luoda/muokata/hallita tietovisoja.

2. Node.js-applikaatio hakee tasaisin väliajoin *pian alkavat* tietovisat Laravellista ja hoitaa niiden pyörityksen, mm. socket-yhteydet pelaajiin ja pelilogiikan etenemisen.

3. Tietovisan päätyttyä Node.js-puoli kutsuu Laravellin "tulospalvelurajapintaa", jonne syöttää tietovisan tulokset pitkäaikaistallennukseen. Tässä jälleen Laravel ja Laravellin erinomainen ORM loistavat. Pelaajat voivat jälkikäteen tarkastella tuloksia Laravellin puolella.

Kokonaisarkkitehtuuri perustuu lisäksi vielä ajatukseen, että järjestelmän pyörittämisestä vastaa *yksi Laravel-applikaatio* ja *useampi Node.js-palvelin*. Miksi näin? Node.js-palvelimen tehtävänä - kuten yllä kuvattiin - on hoitaa kaikki reaaliaikainen tiedonvaihto tietovisan pelaajien suuntaan. Tämä vastuualue vaatii poweria palvelinraudalta - kutakin pelaajaa varten täytyy varata samanaikainen Websocket-yhteys ja viestiliikenne pelaajamäärältään suuressa tietovisassa on suuri.

Laravel-puoli taas on lähinnä tietovisojen luontia ja tulospalvelun ylläpitoa varten. Kumpikaan näistä ei vaadi millisekuntien latenssia. Lisäksi tietovisoja luo huomattavasti pienempi määrä käyttäjiä kuin niitä pelaa.

### Usea peliserveri - kuinka pelaaja löytää oikean?

Kuvitellaan, että meillä on yksi Laravel-palvelin ja viisi Node.js-palvelinta. Kukin tietovisa pyörii yhdellä palvelimella. Tietovisat pyritään jakamaan tasaisesti palvelinten kesken, jotta kuormitus jakautuu mahdollisimman tasaisesti.

Loppukäyttäjän eli tietovisan osallistujan kannalta viisi palvelinta on hiukka ongelmallista - kuinka loppukäyttäjä tietää mihin palvelimeen ottaa yhteys tietovisan pelaamista varten?

Ratkaisu on, että pelaaja ottaa **ensin yhteyden Laravel-palvelimeen**, joka **kertoo pelaajalle hänen valitsemansa tietovisan Node.js-palvelimen IP-osoitteen**.

Koska Laravel-palvelimia on kokonaisjärjestelmässä vain yksi kappale, sen osoite on aina tiedossa. Tai paremminkin - tietty domain johtaa suoraan Laravel-applikaatioon.

Homma toimii siis kutakuinkin näin:

1. Ihmiskäyttäjä haluaa pelata tietovisan.
2. Hän menee osoitteeseen *www.visamestari.fi*. Tämä osoite ohjaa hänet järjestelmän Laravel-osioon.
3. Laravel-osiosta hän valitsee haluamansa *piakkoin alkavan* tietovisan, ja klikkaa "Osallistu".
4. Laravel tarkistaa tietokannasta, mille Node.js-palvelimelle tuo tietovisa on *allokoitu*, ja palauttaa tuon palvelimen IP-osoitteen.
5. Käyttäjän selain ottaa yhteyden saatuun IP-osoitteeseen, täten ilmoittaen olemassaolostaan Node.js-palvelimelle. 
6. Node.js-palvelimen ja käyttäjän välille luodaan Websocket-yhteys reaaliaikaista tiedonvaihtoa varten.

Yllä vaihe #4 edellyttää, että Laravel on etukäteen tallentanut tietokantaansa tietovisan pyörityksestä huolehtivan palvelimen IP-osoitteen. Miten ja missä vaiheessa tämä tallennus tapahtuu?

Homma menee kutakuinkin näin.

Jokainen viidestä Node.js-palvelimesta *pyytää* tasaisin väliajoin pian alkavia tietovisoja Laravel-palvelimelta. Yksittäisen Node.js-palvelimen kannalta pyyntö etenee seuraavasti: 

1. Palvelin kysyy Laravellilta 'onko uusia tietovisoja, joita voisin pyörittää?'.

2. Palvelin tarkistaa tietokannasta ja vastaa joko: 'ei' tai 'kyllä on, tässä tietovisan pyöritykseen vaadittavat tiedot'

Mitä tapahtuu Laravellin päässä kun Laravel *antaa* tietovisan pyörityksen tietyn Node.js-palvelimen kontolle? Laravel tietää IP-osoitteen, josta Node.js-palvelin otti yhteyttä. Joten pyöritysvastuun antamisen yhteydessä Laravel voi tallettaa tuon IP-osoitteen tietokantaan. 

Kun myöhemmin loppukäyttäjä saapuu Laravel-puolella ja valitsee sieltä osallistumisen tuohon tietovisaan, Laravellilla on tietokannassaan tallessa Node.js-palvelimen IP-osoite. Se voi vain palauttaa tuon IP-osoitteen loppukäyttäjälle.

Node.js:n puolella ohjelmisto vastaanottaa *piakkoin alkavat tietovisan* tiedot. Näiden pohjalta se luo Tietovisa-objektin, joka jää odottamaan rekisteröitymisiä. Tietyllä kellonlyömällä Node.js sitten käynnistää tietovisan, lähettäen jokaiselle siihen mennessä rekisteröityneelle käyttäjälle "tietovisa alkaa"-viestin Websocketin kautta. 

Tietovisan päätyttyä Node.js lähettää tulokset Laravellille. Koska Laravel-palvelimia on kokonaisarkkitehtuurissa vain yksi kappale, ei Node.js-palvelimen tarvitse huolehtia Laravel-palvelimen IP-osoitteen selvittämisestä. Tuo IP-osoite on yksinkertaisesti tallennettu Node.js:n konfiguraatiotiedostoon.

> Loppukaneetti: ylläoleva arkkitehtuuri perustuu pohjimmiltaan ajatukseen, että X määrä työläisiä kysyy tasaisin väliajoin lisätyötä. Työnantajana toimii Laravel-keskuspalvelin. Oleellista on, että **Laravel on täysin passiivinen**; se ei ikinä ota yhteyttä Node.js-palvelimiin, vaan odottaa sinnikkäästi Node.js-palvelinten yhteydenottoja, ja jakaa työtehtäviä noiden yhteydenottojen pohjalta.



