+++
date = "2016-07-22T16:14:13+03:00"
draft = false
title = "Tarkastuslista uutta Laravel-projektia aloittaessa"

+++

Olen ihastanut suuresti <a href="https://en.wikipedia.org/wiki/The_Checklist_Manifesto" target="_blank">checklist-manifestoon.</a> Manifeston hengessä loin alkukesästä itselleni muistilistan asioista, joita uutta Laravel-projektia aloittaessa tulee ottaa huomioon.

Monet listan kohdista pätevät yleisesti kaikkiin ohjelmistoprojekteihin.

### Laravel-checklist

#### Vaiheet 1-3: Projektikansion valmistelu, projekti-boilerplate, etc

+ 1. Alusta Git-repo projektikansioon, luo Github-repo, kytke yhteen.
+ 2. Lataa Composer.phar projektikansioon
+ 3. Kloonaa Laravel-boilerplate
+ 4. Muokkaa hakemisto-oikeudet (mm. Laravellin storage-kansio)
+ 5. Luo uusi Sublime-projekti

#### Vaiheet 6-9: Tietokannan luonti, valmistelu, tietokantayhteys, email-testaus

+ 6. Luo uusi tietokanta (esim. phpMyAdmin:in kautta)
+ 7. Päivitä projektitiedostoihin tietokannan käyttäjätunnus + salasana.
+ 8. Aseta email-ajuri osoittamaan testitiedostoon (loki).
+ 9. Luo "finnish"-kielitiedosto.

#### Vaiheet 10-12: Ensimmäiset tietokantataulut, relaatiot, mallit (models)

+ 10. Suorita 'php artisan make:auth', joka luo käyttäjähallinnan tietokantaan.
+ 11. Luo mallit kuvaamaan domain-käsitteitä. Tässä vaiheessa riittää tyhjä tiedosto kullekin mallille.
+ 12. Luo applikaation migraatiot (yksi per malli). Hahmottele kunkin mallin tietorakenne.

#### Vaiheet 13-15: Seeders, tehtaat, migraatioiden toiminnan varmistus

+ 13. Luo seeder-tehtaat (seeder factories) kullekin mallille.
+ 14. Luo seeder-tehtaiden avulla (feikki)käyttäjiä ym. domain-objekteja.
+ 15. Testaa, että migraatiot toimivat ja että relaatiot eri mallien välillä ovat kunnossa.

Tähän muistilistani päättyy. Tästä eteenpäin alkaa ns. raaka työ, eli itse applikaation toimintalogiikan ja käyttöliittymän ohjelmointi. 

Tämä on se pisin ja uuvuttavin vaihe projektissa. *Vaiheet 1-15 ovat verrattavissa arkkitehdin työhön. Vaiheet 16-20 ovat verrattavissa kirvesmiehen työhön.*

#### Vaiheet 16-20: Toteuta logiikka, käyttöliittymä, jne.

+ 16. Hahmottele, koodaa, testaa, näpyttele sormet kipeäksi.
+ 17. Kiroile, paisko pari hiirtä tusinan päreiksi, harkitse puutarhurin uraa.
+ 18. Onnistu lopulta ratkomaan ongelmat.
+ 19. Juhlista valmista applikaatiota.
+ 20. Aloita seuraava projekti.

Ylläoleva checklist on osoittanut hyödyllisyytensä useammassa omassa projektissani. Kun on muistilista, jota seurata orjallisesti, pysyy laatu tasaisena ja työtahti tiiviinä.







