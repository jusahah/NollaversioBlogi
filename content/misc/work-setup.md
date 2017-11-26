+++
date = "2017-11-26T08:31:56+02:00"
draft = false
title = "Kahden näytön työ-setup"

+++

Ammattimaisen koodaamisen perusedellytys on, että tukitoiminnot ja työkalut varsinaista koodin kirjoittamista ajatellen ovat kunnossa. Koodarin tärkein työkalu on luonnollisesti laitteisto, jolla koodia kirjoitetaan. Siis fyysinen tietokone ja jonkin sortin näppäimistö.

Oma työkaluni on vanha kunnon pöytäkone, joka hurisee hiljaa työpöydän alla. Koneen speksit eivät ole tärkeät, etenkään web-koodauksen puolella. Vanhakin prosessori riittää oikein hyvin, ja näytönohjain tarvitaan lähinnä usean näytön tukea varten (useimmissa web-sovelluksissa itse graafiikka on yksinkertaista eikä vaadi näytönohjaimilta suuremmin tehoja).

Tärkein osa laitteistokokonaisuutta on näyttöpäätteet, ja niiden konfigurointi maksimaalista tuottavuutta ajatellen. Seuraavassa oma ratkaisuni.

### Kaksi fyysistä näyttöpäätettä, kahdeksan virtuaalista näyttöä

Työpöytäni näyttää tältä:

![Työpöytä](/blog/public/img/work-setup.png)

Kaksi näyttöä vierekkäin, joista toinen on perinteinen vaakasuuntainen, toinen käännetty pystyyn. 

Miksi toinen on vaaka-asennossa, toinen pystyasennossa? Näytöt palvelevat eri tarpeita. Vaakasuuntainen näyttö sisältää kivasti vaakasuuntaista tilaa, joten siihen sopii hyvin selainikkuna, tarvittaessa vaikka kaksi vierekkäin. 

Pystysuuntainen näyttö taas sisältää rutkasti tilaa pystysuunnassa. Koodieditori soveltuu tälle näytölle mainiosti, sillä koodia kirjoittaessa on tärkeämpää *nähdä monta koodiriviä kerrallaan* kuin *nähdä yhden pitkän koodirivin koko teksti*. 

Toisin sanoen, koodieditori puolella vertikaalinen tila on tärkeämpää kuin horisontaalinen. Pystynäytöllä saa nopeasti kokonaiskuvan isosta palasesta koodia, ja esimerkiksi moni yksittäinen kooditiedosto mahtuu näyttöruudulle kokonaisuudessaan, jolloin ei tarvitse skrollata. Horisontaalisesti tilaa on vähemmän, mutta koodirivit tuppaavat olemaan horisontaalisesti lyhyitä, joten tämä ei ole ongelma.

Näyttöpäätteiden tarjoama tila puolestaan jakautuu seuraavasti (per näyttöpääte):

![Näyttöjen jaottelu](/blog/public/img/workspaces-work-setup.png)

Koodieditori valtaa kokonaan pystynäytön. Vaakanäytöllä puolestaan on niin paljon horisontaalista tilaa, että olen laittanut vasempaan reunaan komentorivikehoitteen (siis terminaalin), ja oikealle laidalle selainikkunan. 

> Kuvasta asiaa ei näe, mutta itse asiassa selainikkuna jakautuu vielä kahteen osaan: itse varsinaiseen työskentelyalueeseen ("webbisivu-näkymään") ja työkalupalkkiin (Chrome Dev Tools). Tämäkin jaottelu on horisontaalinen.

Tällä tavoin saan kahden näytön turvin luotua setupin, jossa pystyn näkemään koodieditorin ja koodattavan applikaation yhtäaikaisesti. Editori vasemmalla näytöllä, applikaatio oikealla näytöllä.

Mutta tämä on vasta alkua, sillä useimmat applikaatiot koostuvat sekä frontend-koodipohjasta että backend-koodipohjasta. Nämä kaksi koodipohjaa ovat erilliset, eivätkä millään mahdu yhteen koodieditoriin. Mikä avuksi?

### Virtuaaliset näytöt (workspaces)

Ubuntussa on kiva konsepti nimeltä "workspace". 

> Ubuntun help-sivuston kuvaus workspacesta: *Workspaces refer to the grouping of windows on your desktop. You can create multiple workspaces, which act like virtual desktops. Workspaces are meant to reduce clutter and make the desktop easier to navigate.*

Useamman kuin yhden Workspacen käyttö mahdollista ikäänkuin *fyysisten näyttöjen monistamisen virtuaalisiksi näytöiksi*. 

Toistaiseksi olemme olettaneet, että käytössä on yksi workspace. Mutta Ubuntu sallii jopa neljän workspacen käytön. Tälläisessä tilanteessa meillä on kahdeksan virtuaalisen näyttöpäätteen verran tilaa. 

Vertauskuvallisesti voimme ajatella, että saamme kolme uutta kopiota koko työpöydästä (siis siitä puisesta työpöydästä, jolla fyysiset näyttöpäätteet seisovat) käyttöömme.

Tämä mahdollistaa asetelman, jossa yhden applikaation jokainen "osa-applikaatio" elää omassa workspacessaan. Ohjelmoija voi sitten pomppia workspacejen välillä nopeasti *Ctrl+Alt+nuolinäppäin* -komennolla.

Esimerkkinä oma tyypillinen workspace-struktuurini, kun kehitän vaativaa web-applikaatiota:

![Kahdeksan virtuaalinäyttöä](/blog/public/img/workspaces-logical-setup.png)

Yksi *virtuaalinen näyttöpari* on varattu backend-koodille ja tietokantanäkymälle (esim. phpmyadmin). Toinen on varattu frontend-koodin käyttöön. Kolmas on varattu Slackille (mikäli koodaus vaatii muiden koodareiden kanssa kommunikointia; muussa tapauksessa koko workspace on tyhjä). Neljäs on varattu kaikelle ylimääräiselle hölynpölylle, kuten Youtube-näkymälle, josta kuunnella - fiiliksestä riippuen - vaikka [huuhkajan huhuilua](https://www.youtube.com/watch?v=ih4_1FyVjaY) tai [ammattilaiskäyttöön soveltuvaa koodausmusiikkia](https://www.youtube.com/watch?v=mpbDlp_gk6M). 



