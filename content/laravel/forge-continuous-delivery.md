+++
date = "2016-09-20T07:50:09+03:00"
draft = false
title = "Forge ja koodin käyttöönotto"

+++

Laravellin ekosysteemiin kuuluu oleellisena osana palvelu nimeltä [Forge](https://forge.laravel.com/). Tuo palvelu mahdollistaa Laravel-applikaatioiden devops-ylläpidon helposti suoraan esim. Linoden pilvipalvelinten päällä.

Erityisesti Forge mahdollistaa erään nykyaikaisen ohjelmistokehityksen kulmakivenä toimivan konseptin; koodin jatkuvan käyttöönoton.

### Oma kone -> Github -> Tuotantopalvelin

Homma toimii näin yksinkertaisesti. 

Sanotaan esimerkin vuoksi, että Laravel-applikaatio vaatii bugikorjauksen. Ammattimaisella kehittäjällä on kaikista Laravel-applikaatiostaan ajan tasaiset kopiot omalla työkoneellaan, joten voin lähteä saman tien bugia korjaamaan.

Korjaan bugin työkoneella olevaan Laravel-applikaatioon muutamassa minuutissa. Testaan applikaation toiminnan (yksikkötestaus + nopea smoke test riittävät, integraatiotestaus yleensä ajan tuhlausta pienissä applikaatioissa) ja kaikki toimii odotetusti.

Seuraavaksi tuo *uusi versio* applikaatiosta tulee saada tuotantopalvelimelle. Eli tuotantopalvelimella tällä hetkellä pyörivä buginen versio tulee *korvatuksi* tällä uudella, ei-bugisella versiolla.

Kuinka homma onnistuu?

Minun näkökulmasta toimenpide on naurettavan yksinkertainen. **Pusken yksinkertaisesti uuden koodipohjan Githubiin projektipuuhun.**

Tämä onnistuu luonnollisesti yhdellä komennolla:

```
git push origin master

```

Pinnan alla Forge ja Github *automaattisesti* hoitavat loput. Kas näin:

1. Pusken siis uuden koodipohjan Githubiin (koodi liikkuu työkoneeltani -> pilveen).
2. Github ilmoittaa Forgelle, että uutta koodia on tarjolla.
3. Forge ottaa homman haltuun ja siirtää Githubista uuden koodin tuotantopalvelimelle.
4. Siirron jälkeen Forge ajaa tarvittavat asennukset, skriptit, tietokanta-migraatiot yms.
5. Tuotantopalvelimella pyörii uusin versio applikaatiosta.

Syytä huomata siis, että minun vastuuni loppuu listan kohtaan #1. **Kaikki muu osa-alueet hoituvat automaattisesti.**

Tätä on moderni PHP-ohjelmistokehitys. 

> Forge on kätevä työkalu Laravel-applikaation pyöritykseen tuotantopalvelimella. Forge itsessään ei tarjoa palvelintilaa tai -ohjelmistoja, vaan se toimii ikäänkuin *kapellimestarina*; Forge käskyttää tuotantopalvelinta ja toimii yhteistyössä Githubin rajapinnan kanssa hakeakseen uusimman koodipohjan aina kun sellainen on saatavilla.
