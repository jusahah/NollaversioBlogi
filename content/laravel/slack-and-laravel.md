+++
date = "2016-08-19T06:26:46+03:00"
draft = false
title = "Slack and Laravel"

+++

Uuden Laravel 5.3 ohjelmistokehyksen avulla web-applikaation integrointi Slackin kanssa on naurettavan helppoa. Otetaan esimerkkinä tapaus, jossa haluamme lähettää tiedoksiantoja Slackin suuntaan. 

Sanotaan vaikka, että meillä on Slack-käyttäjänä bisnespersoona Jari Sarasvuo. Applikaatiomme haravoi internettiä etsien blogimainintoja hänen firmastaan Trainer's House. Aina kun joku bloggari kirjoittaa blogiinsa postauksen, jossa termi 'Trainer's House' mainitaan, applikaatiomme tuottaa Slack-viestin ja lähettää sen Sarasvuon Slack-tilille.

Ylimmällä tasolla applikaatiomme toimii esim. näin:

```php

// BlogiController.php

use App\Notifications\SlackViesti

$this->blogs = collect([/* lista blogeja */]);

public function tarkistaBlogit() {
  
  $maininnat = $this->blogs->map(function(blogi) {
    // Tsekkaa blogi-objektia käyttäen jos uusi maininta havaittu
    if ($blogi->uusiMainintaHavaittu()) return $blogi->haeMaininta();
    return null;   	
  })->filter(function($maininta) {
    // Filteröi nullit pois
    return $maininta !== null;
  });

  // Haetaan tietokannasta Sarasvuon käyttäjä-objekti.
  $sarasvuo = User::where('nimi', 'Jari Sarasvuo')->first();

  // Ilmoitetaan Sarasvuon Slack-tilille.
  $sarasvuo->notify(new SlackViesti($maininnat));

}

```

Ylimmällä tasolla Slack-viestin lähettäminen on juurikin noin helppoa kuin yllä. Toki tarvitsemme vielä lisäksi pari luokkaa:

```php

// App\Notifications\SlackViesti.php

class SlackViesti {

  protected $maininnat;

  public function __construct($maininnat) {
    // Talletetaan maininnat jotta voidaan käyttää sitä myöhemmin
    $this->maininnat = maininnat;
  }

  // Kehys kutsuu tätä metodia kun tiedoksianto luodaan ja lähetetään
  // Parametrinä sisään tulee tässä tapauksessa Sarasvuon käyttäjä-objekti.	
  public function via($sarasvuo) {
    // Täällä päätämme mitä tiedoksiantokanavaa haluamme käyttää.
    return ['slack'];

  }

  public function toSlack($user) {
    // Hoidetaan Slack-viestin luonti.
    // Kehys hoitaa loput.

    $mainintaTeksti = $this->maininnat->reduce(function($teksti, $maininta) {
      return $teksti . $maininta->url . ", ";
    }, 'Blogimaininnat: ')

    // SlackMessage on Laravel-kehyksen sisäinen apuluokka.
    return (new SlackMessage)
      ->line('Uusia Trainers House mainintoja')
      ->line('Firmasi mainittiin blogeissa ' . $mainintaTeksti);
  }


}

```

```php

// User.php

class User extends Authenticatable {

  // Mahdollistaa tiedoksiantojen lähetyksen käyttäjälle.	
  use Notifiable;

  // Mahdollistaa Slack-viestien lähettämisen tiettyyn Slack-endpointiin.
  public function routeNotificationForSlack() {
    // Sarasvuo on luonut itselleen HTTP-endpointin Slack-appin puolella.
    // Tässä tapauksessa kirjoitetaan testi-endpoint suoraan lähdekoodiin.
    // Oikeassa applikaatiossa haluamme tallentaa tuo endpointin tietokantaan.

    return 'https://hooks.slack.com/services/T00000000/B00000000/1234abcd';
  }
}

```

Ylläolevan koodin kautta Sarasvuo saa suoraan Slackiin ilmoituksia tyyliin:

> Uusia Trainers House mainintoja
>
> Firmasi mainittiin blogeissa: http://www.kakkumaakari.fi, http://nollaversio.fi


Toimii kuin unelma. Laravel tekee tässäkin tapauksessa koodarin elämästä lähes laittoman helppoa.





