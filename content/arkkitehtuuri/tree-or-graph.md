+++
date = "2017-10-15T10:08:29+03:00"
draft = true
title = "Onko ohjelmasi puu vai graafi?"

+++

Tässä kätevä ajatusmalli: ennenkuin ohjelmoit riviäkään koodia, päätä onko ohjelmasi (tai ohjelmasi osa!) puu vai graafi!

Ai mikä ihmeen "puu vai graafi"?

Puumalli ja graafi ovat tapoja organisoida objektit, joilla on liitoksia muihin objekteihin. Puumallin spesialiteetti on, että objektit on organisoitu kuusipuun näköiseksi struktuuriksi. Kuusipuun keskiossä on runko, josta lähtee oksia. Jokainen oksa haarautuun pienempiin oksiin, ja jokainen pienempi oksa haarautuu vielä pienempiin oksiin. 

Puumallin ydinominaisuus on, että jokaisella *lapsi-oksalla* on tasan yksi *äiti-oksa*. Oikeassakin kuusipuussa jokainen uusi oksa haarautuu tasan yhdestä oksasta. 

Graafi puolestaan organisoi objektit vailla em. *äiti-lapsi*-hierarkiaa. Esimerkiksi Helsingin tieristeykset noudattavat graafi-mallia. Kuhunkin risteykseen yhtyy useampi tie, ja yksikään risteys ei ole *äiti* jollekin toiselle risteykselle.

Ohjelmoinnissa näiden kahden mallin ero näkyy esim. Laravellin Model-layerin ja Vuen view-layerin välissä.

> Laravel ja Vue ovat vain esimerkkiteknologioita. Fundamentaalisemmin voisi sanoa, että ero näkyy domain-driven-design -periaatteen mukaisen Domain-layerin ja XML-pohjaisen elementtihierarkian välillä.

Laravellin Model-layer saattaa näyttää esim. tältä.

[Kuva tähän]

Vuen view-layer puolestaan saattaa näyttää tältä.

[Kuva tähän]

Ylläolevat kaksi erilaista tapaa strukturoida applikaatio vaativat erilaiset ratkaisut. Esimerkiksi Vue:n ratkaisussa (= puumalli) huomaamme, että jos tuhoamme objektin nimeltä *Profiili* (kts. kuva), objektit *Tallenna-nappi* ja *Profiilin kentät* putoavat "tyhjyyteen". Ne ovat erillään jäljellejäävästä Vue-puusta. Mitä tälläisille erakoille tulisi tehdä? Käytännössä kaksi vaihtoehtoa; joko *liitämme* ne takaisin puuhun, tai *tuhoamme* ne.

Takaisin puuhun liittäminen on vaikeaa, sillä mistä tiedämme mihin nuo kaksi erakkoa liitämme? Yksi looginen ajatus olisi liittää ne siihen objektiin, joka oli äskettäin tuhoamamme *Profiilikentät*-objektit äiti.

Tätä mallia käytetään paljon. Monissa käyttötarkoituksissa tämä on **tismalleen** oikea tapa toimia. 

Mutta monissa muissa käyttötarkoituksissa tuo ei ole oikea tapa toimia.

Ajatellaan vaikkapa tavanomaista sukupuuta, jossa suvun viimeisin jäsen on ylimpänä (root, juuri). Tämä käännetty sukupuu on *binaaripuu*; jokaisella objektilla on tasan kaksi *lapsi-objektia*. Ironisesti, nuo kaksi lapsi-objektia ovat objektin kuvaaman henkilön vanhemmat.

Jos tästä puusta poistetaan yksi objekti, niin meidän on pakko poistaa kaikki hänen aiemmat esi-isänsäkin. Muuten puu ei enää olisi luotettava.

Puumallin ohjelmoinnissa on muutamia muitakin erityisseikkoja:

1. Puumalli on helppo käydä läpi. Lähtee juuresta liikkeelle, ja kiertää koko puun. Puumallin hienous on, että kun loogisesti seuraa liitoksiä yksi kerrallaan esim. vasemmalta oikealle, ei koskaan saavu samaan objektiin kahdesti.

2. Objekteilla on selkeä hierarkia. Juuri tästä syystä puumalli sopii niin hyvin esimerkiksi käyttöliittymän taustalla olevaksi datastruktuuriksi. Tyypillinen käyttöliittymä on pohjimmiltaan pelkkä iso pino *sisäkkäisiä suorakulmioita*. Koska suorakulmiot ovat sisäkkäisiä, ne sopivat mainiosti puumalliin. Yhdellä suorakulmiolla on aina tasan yksi äiti; suorakulmio ei voi olla yhtäaikaisesti kahden eri suorakulmion sisällä siten, että nuo kaksi muuta suorakulmiota eivät ole keskenään äiti-lapsi -hierarkiassa.

3. Hierarkia tekee objektien välisestä kommunikaatiosta helpompaa. Puumallin hieno ominaisuus on, että pohjalta lähtiessä ylöspäin päätyy **aina** juuri-objektiin. Tämäkin seikka on mukava käyttöliittymän kannalta. Moni käyttöliittymä reagoi eventteihin (tapahtumat) siinä objektissa, missä ne alunperin tapahtuvat. Esimerkkinä vaikka hiiren klikkaus. Kun käyttäjä klikkaa hiirellä "Tallenna-nappia" (kts. aiempi puumalli-kuva), klikkaus rekisteröidään vastaanotetuksi "Tallenna-nappi"-objektissa. 

Mutta entä jos haluamme tietää ylimmällä tasolla (juuri-objektissa), että nappulaa on klikattu? Monissa käyttöliittymissä juuri-objekti edustaa *ikkunaa*. 

Usein haluamme klikkauksen - tapahtui se klikkaus missä kohtaa ikkunaa tahansa - seurauksena aktivoida ikkunan. Tämä aktivointi tehdään ikkuna-objektissa. Mutta itse klikkaus voi tapahtua missä tahansa objektissa, vaikka kuinka "syvällä" puumallin pohjamudissa tahansa. Miten ikkuna-objekti saa tiedon klikkauksesta? Helposti, sillä puumallin ominaisuus on, että liikkumalla puussa ylöspäin päätyy ennen pitkään väistämättä juureen. Tässä tapauksessa siis ikkuna-objektiin. 

> Voit testata saman lähimetsässä. Valitse iso puu. Valitse satunnaisesti mikä tahansa sen oksa. Kiipeä valitsemaltasi oksalta ylöspäin. Ennen pitkään saavut puun latvaan. Maagisinta on, että saavut samaan latvaan riippumatta siitä, miltä oksalta kiipeämisesi aloitit.

Tätä eventin liikuttelua kohti juurta kutsutaan nimellä "event bubbling".

Graafi tähän. 
