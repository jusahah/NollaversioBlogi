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

Samanaikaisuus aiheuttaa hienoisia vaikeuksia arkkitehtuurimme toisessa palasessa, mutta probleemat ovat ratkottavissa.

### Dokumenttien vastaanotto -ohjelma

Vastaanotto-ohjelman tehtävä on ottaa dokumentit käyttäjältä vastaan. Käytännössä tämä tarkoittaa jonkinlaista www-sivua, jossa on lomake, jota käyttäen loppuasiakas lataa dokumentit sisään. Tuhannen asiakirjan upload saattaa toki kestää hetken, mutta ei takerruta siihen (loppuasiakas voi lähettää zip-paketin joka sisältää kaikki asiakirjat).

Vastaanotto-ohjelma pyörii tuikitavallisella web-palvelimella. Se ei siis pyöri Lambdan päällä ihan siksi, että se joutuu *pitämään kirjaa* käännetyistä dokumenteista.

Käytännössä asiakirjojen vastaanotto loppuasiakkaalta toimii näin:

1. Web-rajapinta vastaanottaa zip-paketin ja purkaa sen.
2. Kukin asiakirja kirjataan saapuneeksi. Palvelinohjelmisto tällä tavoin tietää, montako asiakirjaa lähetys sisälsi. 
3. Kukin asiakirja lähetetään Amazonin rajapintaan.

Amazonin puolella kukin asiakirja kääntyy pikkuhiljaa itsestään. Mutta miten Amazon saa palautettua tulokset takaisin vastaanotto-ohjelmallemme?

Yksi todella huono tapa olisi se, että vastaanotto-ohjelma lähettää asiakirjan Amazonille HTTP-kutsuna, ja jää odottamaan tuon kutsun vastausta. Ongelmaksi muodostuu se, että jos käännös kestää vaikka 60 sekuntia, HTTP-yhteys Amazonin suuntaan on 60 sekuntia auki. Tämä ei ole ideaaliratkaisu.

Parempi ratkaisu on, että vastaanotto-ohjelma ampuu asiakirjan Amazonin suuntaan HTTP-kutsulla, ja Amazon vastaa HTTP-kutsuun *välittömästi*. Amazonin antama vastaus ei sisällä käännöstä, vaan kuittauksen tyyliin *käännöstyö vastaanotettu, ilmoitamme erikseen kun käännös on valmiina*.

> Keskustelun voi kuvata näin:
>
>
>
>*(yhteys aukeaa)*
>
>**Vastaanotto-ohjelma**: hei Amazon, tässä sinulle työtehtävä...
>
>**Amazon**: selvä pyy, ilmoitan sitten kun on valmista! 
>
>*(yhteys sulkeutuu)*

Entä miten Amazon palauttaa vastauksen takaisin vastaanotto-ohjelmalle? Se ottaa itsenäisesti uuden HTTP-yhteyden! Tämä on mahdollista suorittaa suoraan Lambda-funktion sisältä. Keskustelu jatkuu kutakuinkin näin:

>*(yhteys aukeaa)*
>
>**Amazon(Lambda):** hei kaveri, muistatko antamasi työtehtävän? Tässä tulokset siitä!
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(yhteys sulkeutuu)*

Tässä kohtaa vastaanotto-ohjelma on saanut yhden käännöstuloksen takaisin. Käännöksiä lähti alunperin liikkeelle 1000 kpl, joten tämä yksi on vasta alkua. Käytännössä seuraavat pari minuuttia (tai sinnepäin) vastaanotto-ohjelma saa 999 uutta yhteydenottoa:

>*(yhteys aukeaa)*
>
>**Amazon(Lambda):** tässä tulokset...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
>*(toinen yhteys aukeaa)*
>
>**Amazon(Lambda) #2:** tässä tulokset...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(toinen yhteys sulkeutuu)*
>
> --------------------------
>
>*kolmas yhteys aukeaa)*
>
>**Amazon(Lambda) #3:** tässä tulokset...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(kolmas yhteys sulkeutuu)*
>
> --------------------------
>
> ...
>
> --------------------------
>
>*999s yhteys aukeaa)*
>
>**Amazon(Lambda) #999:** tässä tulokset...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(999s yhteys sulkeutuu)*

Kun kaikki 1000 käännöstä ovat saapuneet, koko urakka on vihdoin valmis! Mutta ennen sitä on syytä miettiä seuraavaa: Amazonilla saattaa olla kullakin ajan hetkellä *usean eri loppuasiakkaan käännösurakat pyörimässä*. 

Eli edellinen keskustelu olikin VALTAVA yksinkertaistus, sillä siinä oletettiin, että kaikki käännöstulokset kuuluivat yhdelle ja samalla ihmisasiakkaalle. Oikeasti keskustelu näyttää tältä:

>*(yhteys aukeaa)*
>
>**Amazon(Lambda) #3829:** tässä käännös Matin dokumenttiin nro 12...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
>*(yhteys aukeaa)*
>
>**Amazon(Lambda) #115:** tässä käännös Pirkon dokumenttiin nro 821...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
>*(yhteys aukeaa)*
>
>**Amazon(Lambda) #8008:** tässä käännös Pirkon dokumenttiin nro 822...
>
>**Vastaanotto-ohjelma:** kiitos, otan talteen!
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
> //jne. jne

Yllä näemme toisen tärkeän konseptin; kukin dokumentti on yksilöity järjestysnumerolla. Tämä järjestysnumero mahdollistaa sen, että lähtevä suomenkielinen dokumentti voidaan myöhemmin mätsätä eli yhdistää sisääntulevaan englanninkieliseen käännökseen. 

Tällä tavoin tiedämme, mitkä dokumentit on käännetty ja mitkä ovat vielä prosessoitavana.

### Käännökset saapuneet, yksi urakka valmis!

Kun vastaanotto-ohjelma on saanut kaikki käännökset haltuunsa, se voi vihdoin lähettää tiedon ja käännökset ihmiskäyttäjälle. Ensin 1000 kpl käännöksiä pakataan zip-pakettiin. Sen jälkeen vastaanotto-ohjelma (joka tässä vaiheessa toimii enemmänkin "lähetysohjelmana") ottaa yhteyden SMTP-rajapintaan. 

Tuonne rajapintaan pusketaan zip-paketti ja ihmiskäyttäjän email-osoite. SMTP-palvelin hoitaa loput, ja hetken kuluttua ihmiskäyttäjän sähköpostilaatikko kilahtaa. 

### Entä jos vastaanotto-ohjelma kaatuu kesken käännösten odottelun?

Mietitäänpä seuraavaa tilannetta. Matti lähettää 1000 kpl dokumentteja web-palveluumme. Vastaanotto-ohjelma lähettää ne kaikki Amazonin suuntaan. Amazon ehtii kääntämään ja palauttamaan 500 kpl, kunnes jotain menee pieleen: **vastaanotto-ohjelmamme kaatuu.**

Näin voi käydä esimerkiksi siinä tapauksessa, että fyysinen palvelin simahtaa pois päältä. Ehkä palvelinsalin siivooja sattui kippaamaan Fairyt tuuletinaukosta sisään. 

> Muista, että vastaanotto-ohjelma pyörii ihan tavallisella palvelimella. Ainoastaan Amazonin pääty pyörii ulkoistetun pilvipalvelun varassa. 
>
> Jos Amazonin päädyssä yksittäinen palvelin sattuu tekemään itsemurhan, Amazon hoitaa korjaustoimenpiteet osana palvelulupaustaan. Jos vastaanotto-ohjelman päädyssä palvelin posahtaa, se on **ohjelmoijan** ongelma. Eli siis minun ongelma, joka ylläpidän käännöspalvelua.

Niin tai näin, koko vastaanotto-ohjelman keskusmuistitila nollaantuu palvelimen käynnistyessä uudestaan. Tämä nollaantuminen on hiukan ongelmallista, sillä vastaanotto-ohjelma piti keskusmuistissaan kirjaa dokumenteista, jotka olivat parhaillaan prosessoitavina Amazonin päädyssä.

Auts. Se siitä kirjanpidosta. Mites nyt suu pannaan?

#### Kovalevy avuksi

Ongelmaan on helppo ratkaisu. Sen sijaan, että vastaanotto-ohjelma pitää kirjanpitoa keskusmuistin sijaan kovalevylle. Kovalevyn hyvä puoli on, että palvelimen sipatessa tieto ei katoa mihinkään. Kun palvelin buuttaa itsensä ja vastaanotto-ohjelma palaa linjoille, se voi kovalevyltä tarkistaa kirjanpidon. Ongelma ratkaistu!

Mutta valitettavasti kirjanpidon pöllähtäminen taivaan tuuliin ei ollut ainoa ongelmamme. Sillä mietipä seuraavaa. Sanotaan, että vastaanotto-ohjelmamme kaatuu kahdeksi minuutiksi (tuon ajan fyysisellä palvelimella kestää buutata itsensä). Tällä välin Amazonin pääty on saanut käännöksen valmiiksi. Miltä keskustelu näyttää?

>*(yhteys aukeaa)*
>
>**Amazon(Lambda) #3829:** tässä Matin käännös dokumentti nro 12...
>
>**...**
>
>**Amazon(Lambda) #3829:** haloo, onko ketään kotona...?
>
>**...**
>

Ongelman ydin on yksinkertainen: vastaanotto-ohjelma on poissa langoilta, joten Amazon ei saa siihen yhteyttä! 

Ongelma on pirullinen ratkaista. Naivi, ihanan sinisilmäinen ratkaisuehdotus on *pakottaa* Amazonin Lambda-funktio odottamaan kunnes vastaanotto-ohjelma on taas takaisin elävien kirjoissa.

Tämä "ratkaisu" on erittäin huono. Sen surkeuden voi paljastaa yhdellä kysymyksellä: **entä jos vastaanotto-ohjelma ei ehdi palaamaan linjoille ennen Lambda-funktion maksimielinajan ylittymistä?**

Tälläisessä tilanteessa käännöstyön tulokset häviävät pysyvästi bittiavaruuteen.

### Kolmas osapalanen

Paras ratkaisu on lisätä kokonaisarkkitehtuuriimme kolmas elementti: *käännöstöiden tulokset vastaanottava jono*.

Tämä jono on esimerkiksi Amazonin SQS jonopalvelu. Jonon ydinidea on, että *se ei ole koskaan poissa linjoilta*. Voimme siis luottaa, että Amazonin Lambda saa *aina* yhteyden Amazonin jonoon.

Jonon toinen ydinidea on, että se pitää tuloksia hallussaan siihen asti, kunnes vastaanotto-ohjelma käy ne hakemassa itselleen.

Tällä tavoin ongelma ratkeaa. Vastaanotto-ohjelman ollessa alhaalla Amazonin pääty lähettää tulokset jonoon. Kun vastaanotto-ohjelma sitten joskus herää kuolleista, se käy hakemassa tulokset tuolta samasta jonosta.

Itse asiassa jono mahdollistaa vielä paremman yksinkertaistuksen: Amazon Lambda lähettää käännösten tulokset jonoon riippumatta siitä onko vastaanotto-ohjelma elossa vai ei! Tällä tavoin Lambdan ei tarvitse milloinkaan ottaa suoraa yhteyttä vastaanotto-ohjelmaan. 

Tässä uudessa, parannellussa mallissamme keskustelun kulku menee kutakuinkin näin. Käydään keskustelu yhden käännettävän dokumentin näkökulmasta:

>*(yhteys aukeaa)*
>
>**Vastaanotto-ohjelma**: hei Amazon, tässä sinulle työtehtävä...
>
>**Amazon**: selvä pyy, ilmoitan sitten kun on valmista! 
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
>*(yhteys aukeaa)*
>
>**Amazon Lambda**: hei jono, tässäpä tulokset...
>
>**Jono**: kiitos, pistän talteen
>
>*(yhteys sulkeutuu)*
>
> --------------------------
>
>*(yhteys aukeaa)*
>
>**Vastaanotto-ohjelma**: hei jono, onko mitään uutta?
>
>**Jono**: kyllä on, tässä uudet tulokset!
>
>*(yhteys sulkeutuu)*
>

On tärkeä ymmärtää syyt miksi tämä *kolmen osapuolen* arkkitehtuuri on valtava parannus alkuperäiseen *kahden osapuolen* arkkitehtuuriin verrattuna. Kerrataan siis:

Alkuperäisessä mallissa vastaanotto-ohjelmalla oli **kaksi(!)** vastuualuetta mitä tulee tulosten vastaanottamiseen:

1. Vastaanottaa tulokset (d'oh)
2. Pysyä hengissä

Listan kakkoskohta saattaa kuulostaa hupaisalta, mutta datan katoamisessa bittiavaruuteen ei ole mitään hupaisaa.

Uudessä, kolmen osapuolen arkkitehtuurissa vastaanotto-ohjelmalla on vain **yksi** vastuualue:

1. Hakea tulokset jonosta

Kyseessä on valtava yksinkertaistus ihan siitä syystä, että palvelinohjelmiston ylläpitäminen 100% luotettavuudella pystyssä on helvetinmoinen haaste. Sen lisäksi että sähköt saattavat katketa, käytännössä kaikki ohjelmistot sisältävät bugeja. 

Hyvä nyrkkisääntö palvelinpuolen koodauksessa onkin seuraava:

> Ennemmin tai myöhemmin jokainen palvelinohjelmisto kaatuu bugin seurauksena.

Ja mitä monimutkaisempi ohjelma, sitä todennäköisemmin se pölläyttää savut pihalle. Tässä mielessä yksi vastuualue on parempi kuin kaksi.

Joko vihdoin olemme kuivilla vesillä kokonaisarkkitehtuurin suhteen?

### Entä jos vastaanotto-ohjelma kaatuu otettuaan jonosta tulokset?

Palvelinohjelmistojen ohjelmointi on saatanallista ongelmanratkontaa. Emme suinkaan ole vielä paratiisin ovilla. Seuraava ratkaistava ongelma on tämä:

**Entä jos vastaanotto-ohjelma kaatuu heti sen jälkeen, kun se on hakenut uusimmat tulokset jonosta?** 

Se siis hakee uusimmat tulokset jonosta, joka luonnollisesti unohtaa nuo tulokset. Mutta ennenkuin vastaanotto-ohjelma ehtii lähettää tulokset ihmiskäyttäjälle, palvelin kohtaa sähkökatkon.

Tulokset eivät ole enää jonossa, mutta ne eivät ole enää vastaanotto-ohjelman keskusmuistissakaan - ohjelma kun kaatui. Bittiavaruus ja niin edelleen.

#### Ratkaisuehdotus #1

No, ratkaisuhan on ilmiselvä? Kun vastaanotto-ohjelma saa tulokset jonosta itselleen, se *ensitöikseen tallentaa ne kovalevylle*. Ratkaisu on siis sama kuin aiemmassa ongelmassamme käännöstöiden kirjanpidon suhteen.

Paitsi että pieleen meni. Sillä entä jos vastaanotto-ohjelma kaatuu *juuri ennenkuin* se ehtii kirjata tulokset kovalevylle? Se siitä, bittiavaruus kohtalona jälleen.

#### Ratkaisuehdotus #2

Oikea ratkaisu on hoitaa asia niin, että *jono unohtaa tulokset vasta kun sille annetaan lupa*. Keskustelu vastaanotto-ohjelman ja jonon kanssa näyttää tältä:


>*(yhteys aukeaa)*
>
>**Vastaanotto-ohjelma**: hei jono, onko mitään uutta?
>
>**Jono**: kyllä on, tässä uudet tulokset!
>
>**Vastaanotto-ohjelma**: ok, kiva, odotapa pojka hetki...
>
>**...**
>
>**Vastaanotto-ohjelma**: voit unohtaa nuo antamasi tulokset!
>
>**Jono**: gone and gone! ensi kertaan!
>
>*(yhteys sulkeutuu)*
>


(Teknisesti tuota viestinvaihto ei käydä yhden ja saman yhteyden - ei varsinkaan HTTP-yhteyden - sisällä, mutta yksinkertaistus sallittakoon...)

### Maali

Nyt olemme saaneet ratkaistua suurimmat ongelmamme. Muutamia vielä jäin, joihin en jaksa puuttua kuin lyhyesti ja summittaisesti:

1. Entä jos vastaanotto-ohjelma kaatuu juuri kun ihmiskäyttäjä on lähettänyt zip-paketin?
2. Entä jos Amazonin Lambda-funktio jostain syystä ei saa suoritettua käännöstä (kenties teksti on liian sotkuista)? Kelle se ilmoittaa epäonnistumisestaan?
3. Entä jos asteroidi syöksää ihmiskunnan kivikaudelle?

Nopeat vastaukset:

1. Vastaanotto-ohjelma ensitöikseen tallentaa zip-paketin kovalevylle.
2. Ehkä Lambdan ei tarvitse ilmoittaa kellekään. Jos käännöstä ei saada tehtyä, sitä ei saada tehtyä, ja sillä selvä. Vastaanotto-ohjelman puolella voi olla jokin aikamääre määriteltynä, jonka sisällä kukin käännöstyö tulee saada valmiiksi. Jos käännös ei valmistu aikamääreen sisällä, se katsotaan epäonnistuneeksi, ja hylätään. Lopullinen, ulos lähtevä zip-paketti on tällöin pienempi kuin sisääntullut zip-paketti.
3. "Päivitä Windows 10 uusimpaan versioon".








