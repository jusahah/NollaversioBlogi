+++
date = "2016-10-06T18:14:55+03:00"
draft = false
title = "Jonotettu työvaihe ja debuggaus"

+++

Yksi Laravellin monista hienoista ominaisuuksista on kyky jonottaa. Siis laittaa työtehtäviä jonoon myöhemmin suoritettavaksi.

Laravel tarjoaa kaikki tarvittavat komponentit jonotuksen toteuttamiseksi ns. "out-of-the-box". Kaikki vain toimii.

Itse jonotuksen saloista olen puhunut aiemminkin [täällä](http://www.nollaversio.fi/blog/public/laravel/queue-worker/), mutta yksi hauska twisti jonon kautta ajetulla koodilla on.

Se on tämä: *koska jonotettu koodinpätkä ajetaan erillisessä prosessissa, se ei voi palauttaa selaimelle debug-tekstiä ohjelmoijan tarkasteltavaksi*.

Kun PHP-koodi ajetaan tavanomaisesti osana selaimelta lähtöisin olevaan kutsua, PHP voi aina palauttaa tarvittavan tekstijonon ohjelmoijan käyttöön. PHP-koodin puolella tämä onnistuu esimerkiksi komennoilla *echo* tai *var_dump*. 

Tuo palautettu tekstijono printataan selaimen toimesta suoraan näyttöpäätteelle. 

Mutta kun PHP-koodi ajetaan *jonotetun työvaiheen* kautta, ei ole mitään selainta jolle palauttaa mitään! Jonotettu työvaihe ajetaan nimittäin jono-managerin toimesta, joka siis käskyttää erillistä käyttöjärjestelmän prosessia ajamaan PHP-koodin. Tuo jono-manageri ei saa yhteyttä selaimeen.

Joten miten debugata jonotetun työvaiheen sisällä ajettavaa koodia? 

En tiedä oikeaa vastausta itsekään. Pitäisi varmaan kysellä. Yksi ok tapa on logata debug-viestejä Laravellin lokiin. Jonotetulla prosessilla on luonnollisesti kyky kirjoittaa lokitiedostoihin, joten tämä onnistuu.

> Jonotettu työvaihe ajetaan irrallaan perinteisestä *selain->palvelin->selain* -viestienvaihdosta. Tämä on koko jonotuksen ydinpointti (selain saa vastauksen nopeasti, ja raskas työvaihe voidaan jonottaa myöhempään ajankohtaan), mutta sen heikkous on, että debuggaus hiukan monimutkaistuu.
>
> Yksi varteenotettava ratkaisu on debugata kirjoittamalla Laravellin omiin lokitiedostoihin, esim. komennolla *\Log::info('debug-viesti');*

