+++
date = "2016-09-05T07:38:13+03:00"
draft = false
title = "Laravel 5.3: ilmoitukset"

+++

Laravellin uusin versio (5.3) tekee web-ohjelmoinnista taas laittoman helppoa. Ikäänkuin se ei olisi jo sitä ollut.

Uusi versio tuo mukanaan *ilmoituksen* (engl. notification) konseptin, jonka avulla ns. domain-koodista pystyy ampumaan ilmoituksia suoraan domain-objektien suuntaan. Laravel-kehys sitten hoitaa loput.

Tyypillinen tapa ilmoitttaa jotain on ampua ilmoitus *User*-objektin suuntaan. Homma toimii äärimmäisen yksinkertaisesti:

```php

$matti->notify(new LaskuEraantynyt());

```

Ylläoleva koodi kertoo Matille, että hänen laskunsa on erääntynyt.

Pinnan alla tapahtuu ylläolevan koodinajon jälkeen vielä hiukka asioita. Ensiksi tarvitsemme *User*-luokkaan ($matti on User-luokan objekti) metodin nimeltä *routeNotificationForSlack*. 

Tämä routeNotificationForSlack-metodi määrittelee mihin "postilaatikkoon" lähetämme laskun erääntymisestä kertovan ilmoituksen. Se **ei** tee itse ilmoitusta, vaan ainoastaan kertoo mihin tuo ilmoitus ohjataan.

```php

// User.php

public function routeNotificationForSlack() {
  // Tässä määritetään Matin Slack-tilin endpoint joka vastaanottaa viestit.
  // Oletetaan että Matti on rekisteröinnin yhteydessä antanut endpoint-URL:n.
  // Tuo Slack-URL on sitten tallennettu osaksi Matin käyttäjätietoja tietokantaan.
  return $this->slack_url;	
}

```

Lisäksi tarvitsemme vielä LaskuEraantynyt-viestiluokan. Koska Laravel 5.3 vakiona tukee Slackkia, voimme luoda tuon luokan helposti.

Tarvitsemme ensinnäkin *via*-metodin, joka määrittää mitä ilmoitustapaa käytämme. Voimme käyttää esim. SMS-viestiä tai email-viestiä. Tässä esimerkissä tyydymme Slackin käyttöön.

Lisäksi tarvitsemme *toSlack*-metodin, joka luo Slackia varten uuden viestin. Tätä metodia tarvitsemme ainoastaan lähettäessämme ilmoituksen Slackiin. 

Jos lähettäisimme ilmoituksen emaililla, käyttäisimme metodia *toMail*. Koska lähetämme Slackiin, käytämme metodia *toSlack*. Suorastaan johdonmukaista.

```php

// Notifications/LaskuEraantynyt.php

class LaskuEraantynyt extends Notification {

  public function via($notifiable) {
    // Laskuilmoitukset lähetetään asiakkaiden Slack-kanaviin.
    return ['slack'];	
  }

  public function toSlack($notifiable) {
    // Kehys kutsuu tätä metodia kun Slack-viestiä luodaan/lähetetään.
    // SlackMessage on Laravellin sisäinen apuluokka.
    return (new SlackMessage)->content('Maksa heti!');

  }
	
}

```

Muuta ei tarvita (paitsi Guzzle, lue loppukaneetti). 

On syytä nopeasti katsoa miten Laravel-kehys hoitaa lähetyksen pinnan alla:

1. Kutsumme domain-koodissa User-objektin *notify*-metodia. Parametrinä sisään pyyhältää uunituore LaskuEraantynyt-objekti.
2. Laravel selvittää User-objektin *via*-metodilla, että haluttu viestiväylä on Slack.
3. LaskuEraantynyt-objektin *toSlack*-metodi palauttaa SlackMessage-viestiobjektin.
4. SlackMessage-viestiobjekti ohjataan User-objektin *routeNotificationForSlack*-metodin palauttamaan URL-osoitteeseen. Teknisesti tuon ohjauksen hoitaa Guzzle, joka kutsuu Slackin rajapintaa HTTP POST-pyynnön turvin.

> Loppukaneetti: Slack-viestin lähettäminen vaatii Guzzle-lisäosaa, joka ottaa yhteyden Slackin HTTP-rajapintaan. 





