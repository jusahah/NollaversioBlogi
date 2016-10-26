+++
date = "2016-10-26T17:24:15+03:00"
draft = false
title = "Laravel: viivyttelyn taito"

+++

Laravel-kehyksen yksi sisäänrakennetuista ominaisuuksista on *jono*. Laravel mahdollistaa tehtävien puskemisen jonoon, ja suorittamisen erillisessä käyttöjärjestelmän prosessissa.

Tällä tavoin käyttäjän palvelupyyntöä käsittelevä prosessi pääsee helpommalla. Sen ei tarvitse hoitaa kuin tehtävien assignointi, ei itse tehtävien suoritusta.

> Jonotuksen perusteet löytyvät parhaiten aiemmasta postauksestani [täältä](http://www.nollaversio.fi/blog/public/laravel/queue-worker/)
>
> Tässä postauksessa keskitymme erityisesti *delay()*-metodin käyttöön jonotuksen yhteydessä. 

Lähtökohtaisesti jonoon työnnetyt tehtävät suoritetaan *niin pian kuin mahdollista*. Useimmiten tämä tarkoittaa, että tehtävä viettää jonossa aikaa vain muutaman sekunnin murto-osan.

On kuitenkin käyttötapauksia, joissa on ihanteellista *pakottaa* tehtävä jonottamaan vähän pidempään.

### Ajastetut tehtävät jonon kautta

Yksi yleinen toimenpide on *ajastaa* sarja tehtäviä suoritettavaksi myöhempänä ajankohtana. Usein vieläpä nuo tehtävät tulee ajastaa siten, että tehtäväsuoritusten välillä kuluu tietty aika.

Otetaan esimerkki.

### Lämpötilan mittaus tunnin välein - etukäteen ajastettuna!

Oletetaan, että meillä on applikaatio, joka mittaa ulkolämpötilaa. Se miten varsinainen mittaus suoritetaan ei ole oleellista - esimerkin kannalta oleellista on se, miten mittaukset ajastetaan.

On täysin mahdollista mitata lämpötila joka sekunti. Ulkolämpötila ei kuitenkaan mainittavasti nouse/laske sekunnin välein, joten kovin järkevää tuo ei ole. Sen sijaan mitatkaamme lämpötila kerran tunnissa.

Järjestelmän hieno ominaisuus on, että se ei mittaa lämpötiloja omin päin. Sen sijaan käyttäjä joutuu pyytämään lämpötilan mittaussarjan aloittamista. Pyynnön yhteydessä käyttäjä myös ilmoittaa montako mittaustapahtumaa hän haluaa suorittaa. Mittaustapahtumien määrä vastaa tuntien määrää, sillä mittauksia tehdään yksi tunnissa.

Kätevimmin ylläolevan kaltainen toiminnallisuus onnistuu juuri *ajastetun jonotuksen avulla*.

```
// App\Execute.php

// Se miten käyttäjältä kysytään mittaustapahtumien määrä ei ole oleellista.
// Oletetaan että kysyminen on suoritettu *jotenkin*.
$mittaustenMaara = 10;

// Carbon on erinomainen ajanhallintaan erikoistuva lisäosa!
$now = Carbon::now();

// Luodaan ja jonotetaan mittaukset
for($i=0; $i < $mittaustenMaara; $i++){
  // dispatch siirtää tehtävän jonoon
  // Huomionarvoista on *delay()*-metodin käyttö. Se 
  // antaa meille tilaisuuden määrittää ajankohdan
  // jolloin tehtävä aikaisintaan voidaan suorittaa!

  // Delay-metodin avulla voimme täten siirtää tehtävän suorituksen
  // haluttuun hetkeen tulevaisuuteen. Kullekin tehtävälle annamme
  // odotusajaksi kasvavan tuntimäärän $i.
  dispatch(new MittaaLampotila()->delay($now->addHours($i)));   
}

```

Tarvitsemme vielä tuon MittaaLampotila-luokan.

```

// App\Jobs\MittaaLampotila.php

<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

use App\Models\Mittaustulos;

class MittaaLampotila implements ShouldQueue
{
    use InteractsWithQueue, Queueable

    /**
     * Execute the job.
     *
     * @param  LampotilaRajapinta
     * @return void
     */
    public function handle(LampotilaRajapinta $rajapinta)
    {
    	// $rajapinta tulee DI-konttanerin...kontaaninerin... kautta

    	// Suoritetaan mittaus kutsumalla injektoitua rajapintaa.
        $celsius = LampotilaRajapinta->mittaa();

        // Meillä on olemassa 'Mittaustulos' Active Record-malli,
        // joka hoitaa tuloksen puskemisen tietokantaan.
        $mittaustulos = new Mittaustulos($celsius);
        $mittaustulos->save();

    }
}

```

Kun koko jono on lopulta (esimerkin tapauksessa 9 tunnin kuluttua) tyhjentynyt, tietokanta näyttää appatiarallaa tältä.

```

| celsius    | ajankohta |
| ---------- | --------- |
| 12         | 16.00     |
| 13         | 17.00     |
| 13         | 18.00     | 
| 10         | 19.00     | 

// jne. jne.

```

> Laravellin *delay()*-metodi mahdollistaa helpon tavan siirtää tehtävä kauas tulevaisuuteen. Sen lisäksi, että tehtävä *ajetaan erillisessä prosessissa* (ns. prosessi-isolaatio), tehtävä ajetaan myös *ajallisesti erillään* (ns. ajallinen isolaatio).
>
> Toinen hyvä käyttötarkoitus tälle portaalliselle ajastukselle on tehdä kutsuja johonkin rajapintaan. Sanotaan, että meillä on 1000 kpl HTTP-kutsuja tehtävänä. Jos kaikki kutsut ammutaan parin sekunnin sisällä, vastaanottava pää on käärmeissään (koska DoS-hyökkäys). 
>
> Jos taas ajastamme kutsut lähtemään aina 10 sekunnin välein, vastaanottaja on tyytyväinen.



