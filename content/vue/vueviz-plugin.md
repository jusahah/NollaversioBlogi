+++
date = "2018-04-27T04:48:27+03:00"
draft = false
title = "Vue komponenttipuun visualisaatio (plugin)"

+++

### Vue komponenttipuun visualisaatio

![Vueviz osana kehitystyötä kuva 2](/blog/public/img/vueviz-2.png)

Kehitin alkuvuodesta Vue pluginin erääseen tärkeään tarpeeseen. Monissa yhteyksissä oli tarve nähdä Vue-appia kehitettäessä paitsi komponenttipuun rakenne, myös komponenttipuun *diff*, eli mitkä komponentit ovat juuri päivittyneet tai tulleet luoduiksi.

Tällä tavalla on helpompi varmistaa, että kun klikkaan vaikka nappulaa "Avaa tuotehierarkia", suunnilleen oikeat komponentit päivittyvät ja oikea setti uusia komponentteja ilmestyy puuhun.

Tarvetta varten kehitin [Vueviz pluginin](https://github.com/jusahah/VueViz). Se kuuntelee koko appin komponenttipuussa tapahtuvia muutoksia, laskee diffin edelliseen puuversioon nähden, ja printtaa ruudulle tiedot muutoksesta. Tiedot muutoksista kommunikoidaan värikoodeilla; sininen merkkaa päivittynyttä komponenttia, vihreä on uusi komponentti.

![Vueviz osana kehitystyötä](/blog/public/img/vueviz-1.png)



