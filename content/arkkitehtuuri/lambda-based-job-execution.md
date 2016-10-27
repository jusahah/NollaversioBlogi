+++
date = "2016-10-27T16:58:00+03:00"
draft = false
title = "Lambda-pohjainen arkkitehtuuri"

+++

Amazonilla on palvelu nimeltä AWS Lambda. Tuo palvelu suorittaa datan prosessoinnin pilvipalvelun muodossa.

Käytännössä se toimii siten, että ulkopuolinen ohjelmisto kutsuu Amazonin rajapintaa. Tuo rajapinta on Amazonin hallintapaneelissa (tms.) kytketty haluttuun Lambda-funktioon. Rajapinnan kutsu tällä tavoin *laukaisee* Lambda-funktion suorittamisen.

Oleellinen myyntiargumentti Lambdan kohdalla on, että loppukäyttäjän ei tarvitse välittää tuon taivaallista palvelinten ylläpidosta. Ei edes virtuaalipalvelinten. 

Loppukäyttäjä vain kutsuu Amazonin rajapintaa, ja Amazon hoitaa loput. Käytännössä Amazon valitsee valtavasta rauta-arsenaalistaan sopivan palvelimen, jonka suoritettavaksi loppukäyttäjän työvaihe annetaan. 

Loppukäyttäjältä - eli web-palveluiden ohjelmoijalta, tyypillisesti - jää täten yksi huolenaihe vähemmän. Hänen ei tarvitse pelätä palvelimen kaatumista jouluyönä kello 3.00, sillä *ei ole mitään palvelinta, joka voisi kaatua*. 

## Mihin käyttötarkoituksiin Lambda soveltuu?

Lambda-funktio noudattaa *fire-and-forget*-mallia. Jokainen Lambda-funktion kutsu on erillinen - yksi kutsu ei pysty jättämään post-it-lappuja toiselle kutsulle muuten kuin tietokannan tai vastaavan *ulkoisen* kiintopisteen kautta. 

Tämän rajoitteen (ominaisuuden?) vuoksi Lambda soveltuu huonosti esimerkiksi moninpelipalvelimeksi, sillä moninpelipalvelimen luonteeseen kuuluu, että palvelin ylläpitää pelitilaa yksittäisten siirtojen/kutsujen välillä. 

Lambda ei voi ylläpitää pelitilaa keskusmuistissaan, sillä yksittäisen Lambda-kutsun maksimisuoritus aika on muutamia minuutteja. 

Käytännössä loppukäyttäjä voi ajatella Lambda-palvelua ikäänkuin palvelimena, joka kaatuilee parin minuutin välein. Jos toiminto vaatii yli parin minuutin suoritusajan tai tilamuuttajan ylläpidon, Lambda ei sovellu tarkoitukseen.

Se mihin Lambda soveltuu erinomaisesti on *dataa sisään -> dataa ulos* -tyylisten itsenäisten työvaiheiden suorittamiseen. 

Hyvä esimerkki on vaikkapa tekstidokumentin kääntäminen suomesta englanniksi. Tälläinen operaatio on luonteeltaan itsenäinen; tarkoittaen, että operaatio ottaa vastaan dataa, ajaa tietyn pätkän koodia, ja palauttaa ulos uutta dataa. 

### Malliarkkitehtuuri

Seuraavassa kokonaisvaltainen korkean tason arkkitehtuuri, joka hyödyntää Lambdaa.

Oletetaan dokumenttien kääntämiseen erikoistunut web-palvelu. Tyypillinen käyttötarkoitus on, että asiakas antaa web-palvelulle kasan asiakirjoja, jotka haluaa käännettäväksi suomesta englanniksi. Web-palvelu kääntää dokumentit omalla ajallaan, ja kun **kaikki** käännökset ovat valmiita, asiakkaalle lähetetään sähköpostilla tiedoksianto.

Heti alkuun nähdään, että kokonaisarkkitehtuurissa *käännökset suorittava ohjelma* on järkevä eristää *dokumentit asiakkaalta vastaanottavasta ohjelmasta*. Ne siis ovat kaksi erillistä palapelin palasta osana kokonaisarkkitehtuuria.

#### Käännösohjelma

Käännöksien suorittamisesta vastaava ohjelma ajetaan Amazonin Lambda-palvelussa. Miksi? Koska sen käyttötarkoitus soveltuu mainiosti Lambdan päälle. 

Toinen syy on, että on luontevaa suorittaa käännökset *dokumentti kerrallaan*, mutta *samanaikaisesti*. Tällä tarkoitan, että yksi Lambda-funktion kutsu ottaa käännettäväkseen tasan yhden dokumentin*, mutta *kullakin ajanhetkellä useampi Lambda-funktio tekee käännöstyötään*. 

> Periaate on sama kuin pankissa - kukin pankkivirkailija palvelee tasan yhtä asiakasta kerrallaan, mutta useita pankkivirkailijoita on yhtäaikaisesti töissä.

Sanotaan esimerkin vuoksi, että asiakas syöttää web-palveluumme 1000 kpl asiakirjoja. Yhden dokumentin kääntäminen tekoälyn turvin vie 10 sekuntia. Tuhannen dokumentin kääntäminen perätysten veisi 1000 * 10 sekuntia, eli noin kolme tuntia. 

Mutta jos ajamme samanaikaisesti 1000 kpl Lambda-funktioita, koko urakka kestää 10 sekuntia.

Käännösohjelman kannalta valitsemamme *samanaikaisesti x määrää dokumentteja kääntävä* palvelumme ei aiheuta ongelmia, sillä kuten mainittua, käännösohjelman koodi vastaanottaa vain yhden dokumentin. Koodia ajetaan tuhannella eri palvelimella samanaikaisesti, mutta koodi ei välitä - se huolehtii vain yhden dokumentin kääntämisestä.

Samanaikaisuus aiheuttaa hienoisia vaikeuksia arkkitehtuurimme toisessa palasessa, mutta nekin ratkeavat helposti.

### Dokumenttien vastaanotto -ohjelma

