+++
date = "2016-08-15T06:15:21+03:00"
draft = false
title = "Emailin lähetys Laravellista"

+++

Moni web-applikaatio joutuu lähettämään sähköposteja. Tyypillinen tarve sähköpostin lähetykselle syntyy käyttäjän rekisteröityessä applikaatioon; jonkinlainen tervetuloviesti olisi mukava lähettää käyttäjän suuntaan, jotta hän tuntisi olonsa tervetulleeksi.

Laravel tekee emailin puskemisesta eetteriin erittäin helppoa. Otetaan esimerkiksi *lottoapplikaatio*, joka arpoo kerran viikossa lottovoittajan kaikkien osallistujien joukosta. (Tässä esimerkissä ei siis arvota numeroita, vaan valitaan satunnaisesti yksi voittaja suuresta määrästä osallistujia).

```php

// Lotto.php

public function valitseVoittaja(array $osallistujat) {
  // Arvo voittaja satunnaisesti
  $voittaja = $osallistujat->random();
  // Lähetä voittajalle onnittelu-sähköposti
  Mail::raw('Olet voittanut jotain!', function($email) use ($voittaja) {
    $email->from('lotto@veikkaus.fi', 'Veikkaus');
    // Voittajan email-osoite on tallennettu osaksi käyttäjä-objektia
    $email->to($voittaja->email);
  });
	
}

```

Ylläoleva koodinpätkä arpoo voittajan, ja lähettää hänelle onnitteluviestin käyttäen *Mail::raw()*-metodia. Mail::raw() yksinkertaisesti lähettää email-viestin pelkkänä leipätekstinä. Viestin voi lähettää myös HTML-muotoilun kera:

```php

// Lotto.php

public function valitseVoittaja(array $osallistujat) {
  // Arvo voittaja satunnaisesti
  $voittaja = $osallistujat->random();
  // Lähetä voittajalle onnittelu-sähköposti
  Mail::send('emails.voitto', ['voittaja' => $voittaja] function($email) use ($voittaja) {
    $email->from('lotto@veikkaus.fi', 'Veikkaus');
    // Voittajan email-osoite on tallennettu osaksi käyttäjä-objektia
    $email->to($voittaja->email);
  });
	
}


// Views/emails/voitto.blade.php

<h1>Olet voittanut jättipotin!</h1>
<p>Onnittelut {{$voittaja->etunimi}}, olet juuri rikastunut oikein urakalla.</p>


```

### Tukitoimenpide vs. ydintoimenpide

Ylläoleva koodaustyyli, jossa emailin lähetys suoritetaan suoraan arvontametodin sisältä, on ihan toimiva. Mutta on syytä tehdä pesäero ydintoimenpiteen ja tukitoimenpiteen välille.

Lottovoittajan arvonta on *ydintoimenpide*. Ilman voittajan arvontaa koko lottoapplikaatio olisi aika turha.

Sähköpostin lähettäminen voittajalle taas voidaan nähdä joko *ydintoimenpiteenä* tai *tukitoimenpiteenä*. Minä näkisin sen *tukitoimenpiteenä*. Ensinnäkin lottovoittaja tuskin on kiinnostunut siitä tavasta, jolla hänelle ilmoitetaan voitosta. Emailin lähettäminen on tässä mielessä toissijaista - oleellista on, että tieto jotenkin tavoittaa tulevan miljonäärimme.

Ylläolevat ratkaisumme emailin lähettämiseen noudattivat kutakuinkin seuraavaa kaavaa:

> Ydinmetodi
>
> -- ydintoimenpide
>
> -- tukitoimenpide

Toisin sanoen, tukitoimenpiteet on yllä *ripoteltu* ydintoimenpiteiden sekaan.

Toinenkin vaihtoehto on olemassa:

> Ydinmetodi
>
> -- ydintoimenpide
>
> Tukimetodi
>
> -- tukitoimenpide

Jälkimmäisessä ratkaisussa ydintoimenpiteet - kuten arvonta, jonka suorittaminen oikeaoppisesti on ensiarvoisen tärkeää koko lottoapplikaation toiminnan kannalta - on eroteltu tukitoimenpiteistä. Kysymykseksi jää nyt, miten ydinmetodi saa kutsuttua/ilmoitettua tukimetodille, että tietty tukitoimenpide (tässä tapauksessa sähköpostin lähetys) on syytä suorittaa.

Paras tapa lienee eristää tukitoimenpiteet *Event Listener*-objektin sisälle:

```php

/////////////////////////////
// App/Events/ArvontaSuoritettu.php

class ArvontaSuoritettu extends Event
{

    public $voittaja;

    public function __construct(User $voittaja)
    {
        $this->voittaja = $voittaja;
    }
}


/////////////////////////////
// App/Listeners/LahetaTietoVoittajalle.php

class LahetaTietoVoittajalle
{

    public function __construct()
    {

    }

    public function handle(ArvontaSuoritettu $arvontaInfo)
    {
      $voittaja = $arvontaInfo->voittaja;	
      // Lähetetään sähköposti voittajalle
      Mail::raw('Olet voittanut jotain!', function($email) use ($voittaja) {
        $email->from('lotto@veikkaus.fi', 'Veikkaus');
        // Voittajan email-osoite on tallennettu osaksi User-objektia
        $email->to($voittaja->email);
      });
        
    }
}


/////////////////////////////
// Lotto.php

public function valitseVoittaja(array $osallistujat) {
  // Arvo voittaja satunnaisesti
  $voittaja = $osallistujat->random();

  // Ilmoita muulle applikaatiolle, että voittaja on valittu!
  // HUOM! Tämä metodi ei välitä siitä, lähetetäänkö voittajalle
  // sähköposti, kirje vai vaikka savumerkki. Tämän metodin 
  // ainoa vastuualue on ilmoittaa, että voittaja on valittu.

  // Joku muu huolehtii voittajalle ilmoittamisesta.

  // Luo event ja ammu se eetteriin.
  event(new ArvontaSuoritettu($voittaja));

}

```

Ylläoleva ratkaisu on hyvin erilainen alkuperäiseen verrattuna. **Se näyttää monimutkaisemmalta, mutta ei ole.** Se on yksinkertaisempi, sillä vastuualueet elävät nyt omissa kivoissa lokeroissaan. Lottoarvonnan suorittava *valitseVoittaja*-metodi ei räpellä sähköpostien kanssa - sen sijaan se yksinkertaisesti luo ohjelmistokehyksen *sisäisen tiedoksiannon*. 

Tuo tiedoksianto kulkeutuu *LahetaTietoVoittajalle*-kuuntelijan korviin, joka tiedoksiantoon perustuen luo ja lähettää sähköpostin.

Uusi jaottelu on täten selvä; ydinmetodi huolehtii ydintoimenpiteistä, ja tukimetodi (LahetaTietoVoittajalle::handle) huolehtii tukitoimenpiteistä.

> Loppukaneetti: ydintoimenpiteiden ja tukitoimenpiteiden erottelu on usein järkevä tapa selkeyttää applikaation koodia. 

> Vaan kuinka hyödyllistä tuo jaottelu lopulta on?

> Tilanne on sama kuin yritysmaailmassa. Nollaversio IT:n kaltaisessa pienessä nakkipuljussa yksi mies voi hoitaa niin markkinoinnin, ohjelmoinnin kuin laskutuksenkin. Suuressa pörssiyhtiössä yksi henkilö ei millään kykene hoitamaan kaikkia arkirutiineja, vaan vastuualueet on jaettava usean työntekijän kesken. Yksi toteuttaa asiakasprojektit (=ydintoimenpide), toinen pyörittää lakiosastoa (=tukitoimenpide), kolmas luuttuaa toimiston lattiat (=tukitoimenpide).

> Eli mitä monimutkaisempi web-applikaatio on kyseessä, sitä tärkeämpää on tehdä pesäero ydintoimintojen ja tukitoimintojen välille.









