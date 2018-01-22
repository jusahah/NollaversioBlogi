+++
date = "2018-01-22T15:11:33+02:00"
draft = false
title = "Laravel ja välimuistin testaus"

+++

Moderni Laravel-pohjainen weppi-appi käyttää usein hyväkseen välimuistia (cache). Välimuistia hyödyntämällä vältetään joko turhat HTTP-kutsut (frontend-välimuisti) tai turhat tietokantahaut (backend-välimuisti). Tällä tavalla applikaation suorituskyky paranee, toivottavasti.

## Frontti-välimuisti

Fronttipuolella välimuisti liittyy rajapintakutsujen välttämiseen. Toimintamalli tällöin on, että kun tietty rajapintakutsu on tehty, sen tulos tallennetaan lokaalisti, ja tulevaisuudessa rajapintakutsun sijasta käytetään tallennettua tulosta. 

Fronttipuolen välimuistilla on käyttönsä, mutta HTTP-kutsun skippaamisella on varjopuolensa; on vaikea tietää hetkeä, jolloin lokaali välimuisti on vanhentunut. Eli hetkeä, jolloin täytyy tehdä uusi HTTP-kutsu ja päivittää välimuistin sisältö tuoreella datalla.

> Yksi keino on käyttää jonkinlaista *subscriber*-systeemiä, jossa backend puskee komennon tyhjentää välimuisti fronttiin. Komento voidaan toimittaa vaikka Pusherin kaltaisen järjestelmän kautta. Toimintamalli on kuitenkin varsin monimutkainen saavutettavaan hyötyyn nähden.

Fronttipuolella välimuisti soveltuu parhaiten tapauksiin, joissa palvelimelta haettava data muuntuu aniharvoin jos koskaan. Tällöin voidaan esim. kirjautumisen yhteydessä tehdä yksi HTTP-kutsu, ja tämän jälkeen tallettaa saatu data pysyvästi välimuistiin. 

Tyypillisesti paras vaihtoehto on yksinkertaisesti välttää frontti-välimuistin käyttöä kokonaan, ja tehdä HTTP-kutsu palvelimelle joka kerta kun dataa tarvitaan.

## Backend-välimuisti

### Erillinen välimuisti-layeri

Backendin puolella yksi mahdollisuus on käyttää jonkinlaista erillistä välimuisti-layeriä. Tälläinen layer on kokonaan erillisellä palvelimella, ja toimii täysin erillään varsinaisesta business-backendistä. Esimerkiksi Varnish tarjoaa ratkaisun tähän.

> Erillisen välimuistipalvelimen voi valjastaa myös muihin käyttötarkoituksiin, esimerkiksi kuorman tasaukseen.

Erillisen välimuisti-layerin käyttö törmää samaan ongelmaan - joskin hiukan helpommassa muodossa - kuin frontti-välimuistin; *kuinka tyhjentää välimuisti ja pakottaa tuoreen datan haku*.

Applikaatiosta riippuen voimme skipata *erillisen tyhjennyskomennon* kokonaan, ja tyytyä *aikaperusteiseen tyhjennykseen*. Aikaperusteisessa tyhjennyksessä välimuisti tyhjentyy esimerkiksi viiden minuutin välein itsestään. Tällöin loppukäyttäjä saa haltuunsa pahimmillaan 5 minuuttia vanhaa dataa. 

Toinen vaihtoehto on turvautua Laravellin omaan välimuisti-ratkaisuun.

### Laravel Cache

Laravellin oma välimuistiratkaisu siirtää välimuistin samalle palvelimelle (default-asetuksilla, tätäkin voi toki kustomoida!) itse business-koodin kanssa. Loppuosa artikkelista tutkii tätä toimintamallia.

#### Käyttö

Laravellin oman välimuistiratkaisun käyttö onnistuu - oman kokemukseni perusteella - parhaiten suoraan HTTP-layeriltä käsin, eli siis *Controller*:eista. 

Tässä toimintamallissa välimuistiin talletetaan HTTP-endpointtien palauttama sisältö.

> Toinen vaihtoehto on toteuttaa välimuisti syvemmälle applikaation sisuksiin, esimerkiksi suoraan yksittäisten domain-objektien sisälle. Tällöin välimuistiin talletetaan tietokannasta saatu data jonkinlaisessa tekstimuodossa.
>
>
>
> Ero nk. *domain-välimuistin* ja *Controller-välimuistin* välillä on hienojakoisuudessa; Controllereiden hallitsema välimuisti tallentaa endpointin lopullisen palautusarvon (joka usein sisältää *useamman* domain-objektin sekä lisäksi mahdolliset *transformaatiot*, joita domain-objekteille on tehty). Domain-välimuisti taas tallettaa yksittäisen domain-objektin kerrallaan sellaisena kuin se tietokannasta pötkähtää ulos.
>
>
>
> Domain-välimuisti on teoriassa suorituskykyisempi ja konseptuaalisesti "oikeampi" malli, mutta myös vaikeampi toteuttaa. Huonosti toteutettuna domain-välimuisti on ensiaskel tiellä kohti BBoM\*-helvettiä. Käytännössä se on valtava *overkill*. 

Alla esimerkki Controllerista, joka hyödyntää Controller-välimuistia:

```php

class VihannesController extends Controller {
  
  public function all() {
  
    if (Cache::has('vihannekset')) {
      return Cache::get('vihannekset');
    }
    // Ei vihanneksia välimuistissa -> haetaan tietokannasta.
    $vihannekset = Vihannes::all();

    // Ajetaan fractalin transformaatiot, jotka rakentavat meille
    // lopullisen vastaus-objektin palautettavaksi HTTP-vastauksena.
    $responseData = $this->transformCollection($vihannekset, new VihannesTransformer);
    
    // Lisätään välimuistiin 10 minuutiksi.
    Cache::put('vihannekset', $responseData, 10);
    // Palautetaan kutsujalle
    return $responseData;
  }

  protected function transformCollection($collection, $transformer) {
    //... muunna objektit front-endin odottamaan formaattiin
  }
}

```

Ylläoleva Controller lisää vihannekset välimuistiin 10 minuutin ajaksi. Eli 10 minuutin ajan tietokantaan ei tarvitse tehdä hakuja vihannesten osalta. Tämä säästää kivasti tietokannan hermoja.

**Mutta entä jos ennen 10 minuutin aikarajan umpeutumista joku lisää uuden vihanneksen tietokantaan?**

Laravel tarjoaa toiminnon tyhjentää ("forget") välimuisti halutun avain-arvon osalta. Tällä tavalla voimme pakottaa uuden vihannekset-haun tietokannasta joka kerta, kun uusi vihannes lisätään.

```php

class Vihannes extends Model {
  
  public static function flushCache() {
    Cache::forget('vihannekset');
  }

  public static function create(array $data = []) {
    // Uusi vihannes lisätään tietokantaan, 
    // tyhjennä välimuisti lisäyksen jälkeen.
    $model = parent::create($data);
    static::flushCache();
    return $model;

  }

}

```

> Huom! Ylläoleva koodi ei toimi Laravel 5.4 tai tuoreemmilla versioilla, koska create-metodia on muutettu frameworkin konepellin alla.

Yllä teemme *overriden* Eloquentin create-metodille. Overriden sisällä kutsumme Eloquent-metodia, joka lisää vihanneksen tietokantaan, ja kutsun jälkeen tyhjennämme välimuistin.

> Yllä on tehty välimuistin tyhjennys vain create-metodin kohdalle. Vastaavat overridet tarvitaan myös metodeille, jotka muuttavat vihanneksen dataa tai tuhoavat vihanneksia.
>
>
>
> Metodien overraidaamisen sijaan voisimme käyttää *tapahtumakuuntelijaa* (model listener), jolla kuuntelisimme esimerkiksi *saved*-eventtejä vihannesten osalta. Tällöin saamme enkapsuloitua välimuistin tyhjennyksen yhteen paikkaan, eli tapahtumakuuntelijan sisälle.
>
>
>
> Valinta näiden kahden vaihtoehdon välillä on ensisijaisesti makukysymys. Itse suosin eksplisiittistä koodia ja usecase-arkkitehtuuria (josta lisää seuraavassa kappaleessa), ja siksi vältän sekä tapahtumakuuntelijoita että välimuisti-kontrollin sijoittamista domain-luokkien (kuten *Vihannes*) sisälle.

Ylläoleva koodi toimii (tietenkin), mutta olen itse päätynyt viimeisimmässä applikaatiossani malliin, jota kutsun "*get from controller, flush from usecase*" -malliksi. 

### Get from Controller, flush from Usecase

Mallin nimi kertoo kaiken oleellisen siitä, mistä koodipohjan osasta käsin kukin operaatio suoritetaan. Controller-puolen käsittelimme jo. Mutta mitä tarkoittaa "flush from usecase"?

Usecase-arkkitehtuurissa kukin applikaatiolle suoritettava toimenpide muodostaa erillisen usecase-luokan. Tämä usecase-luokka kontrolloi toimenpiteen suorittamista, ja tarjoaa oivallisen sijainnin kaikelle toimenpiteen *oheis*-koodille. Tälläistä oheiskoodia on mm. virhehallintakoodi sekä tässä käsiteltävä välimuistin kontrolloimiseen liittyvä koodi. 

Usecase-luokan instanssi on tyypillinen manager-objekti; sen ydintehtävä on *koordinoida toimenpiteen suorittaminen, ei niinkään suorittaa itse toimenpidettä*. Usecase on siis työnjohtaja, joka valvoo työtehtävien suoritusta. Ero on hienovarainen, mutta konseptina hyödyllinen. 

Välimuistin tyhjennykselle usecase on mainio paikka, koska mikäli applikaatio rakennetaan oikein, *yksikään muutos tietokantaan ei tapahdu ilman usecase-objektin antamaa käskyä*. Alla esimerkki usecasesta, joka tekee vihannessopan sille annetuista vapaavalintaisista vihanneksista (Vihannes-luokan instanssit):

```php

class ValmistaSoppa extends Usecase {

  public function execute($soppaVihannekset) {
    // Soppavihannekset koostuu Vihannes-objekteista, jotka
    // käytetään sopan valmistukseen. 

    // Aloitetaan sopan valmistus, mieluiten transaktion sisällä
    // jotta emme töpeksi tietokantaa mikäli jotain menee päin hönkiä.
    DB::transaction(function() {
      
      // Luodaan soppakattila
      $soppaKattila = new SoppaKattila;
      // Siirretään vihannekset yksi kerrallaan kattilaan
      $ainesosat = $soppaVihannekset->each(function($vihannes) use ($soppaKattila) {
        // Siirrä vihannes kattilaan
        $soppaKattila->lisaaKattilaan($vihannes);
        // Vihannes on nyt käytetty, tuhotaan se tietokannasta.
        $vihannes->delete();
      });

    });

    // Vihanneksia on poistettu tietokannasta, joten välimuisti tyhjennettävä.
    Cache::forget('vihannekset');

    // Kiehauta ja suolaa
    $soppaKattila->kiehauta();
    $soppaKattila->lisaaSuola();

    // Soppa on valmis, palauta kutsujalle joka voi
    // kaataa liemen lautasille ym.
    return $soppaKattila;

  }
  
}

```

Ylläolevassa usecasessa valmistamme vihannessopan. Koko usecasen koodi on selkeästi step-by-step -muodossa; tee näin, sitten tee näin, sitten tee näin. Tämä hienosti tarjoaa meille selkeän paikan, jonne tunkea välimuistin tyhjennys. Vasta kun vihannekset on poistettu tietokannasta - ja poisto tapahtuu vain mikäli transaktio onnistuu -, tyhjennämme välimuistin.

### Välimuistin testaamisesta kehityksen aikana

Vihdoin otsikon aiheeseen, eli kuinka testata välimuistin kontrolloimista devauksen aikana.

Ensinnäkin ilmiselvä fakta: välimuistin käytön merkitys korostuu applikaatioissa, jotka ovat joko datamäärältään tai käyttäjämäärältään suuria. 

Kehitystyön aikana voi olla vaikea simuloida tarpeeksi suurta data-/käyttäjämäärää, jotta välimuistin tuomasta performanssi-hyödystä pääsee jyvälle. Siksi olen päätynyt seuraavanlaiseen toimintatapaan: *joka kerta kun datahaku palvelimelle menee välimuistin ohi, palvelupyynnön suoritusaikaan lisätään 5 sekuntia luppoaikaa.*

Tällä tavalla on applikaatiota testatessa esim. fronttiappista käsin helppo omin silmin erotella *välimuisti-hitit* (cache hit) ja *välimuisti-missit* (cache miss) toisistaan. 

Ohessa muokattu Controllerin koodi:

```php

class VihannesController extends Controller {
  
  public function all() {
  
    if (Cache::has('vihannekset')) {
      return Cache::get('vihannekset');
    }

    if (env('APP_ENV' === 'development')) {
      // Välimuisti ohitettu! Simuloidaan tietokannan hitautta
      // odottamalla viisi sekuntia.
      sleep(5);     
    }

    // Ei vihanneksia välimuistissa -> haetaan tietokannasta.
    $vihannekset = Vihannes::all();

    // Ajetaan fractalin transformaatiot, jotka rakentavat meille
    // lopullisen vastaus-objektin palautettavaksi HTTP-vastauksena.
    $responseData = $this->transformCollection($vihannekset, new VihannesTransformer);
    
    // Lisätään välimuistiin 10 minuutiksi.
    Cache::put('vihannekset', $responseData, 10);
    // Palautetaan kutsujalle
    return $responseData;
  }

  protected function transformCollection($collection, $transformer) {
    //...
  }
}

```

Odotamme siis 5 sekuntia mikäli dataa ei löydy välimuistista. Tällä tavalla simuloimme tilannetta, jossa kutsu ylikuormitettuun tietokantaan kestää pienen ikuisuuden. Luonnollisesti haluamme tehdä simulaation vain kehitysympäristössä.

Suuressa applikaatiossa odottelu-koodi kannattaa enkapsuloida joko traitin sisään, tai siirtää yläluokkaan. Esimerkiksi:

```php

class VihannesController extends CacheController {
  
  public function all() {
  
    if (Cache::has('vihannekset')) {
      return Cache::get('vihannekset');
    }

    static::cacheMiss();

    // jne...
}

```

```php

class CacheController extends Controller {
  
  protected static function cacheMiss() {
    if (env('APP_ENV' === 'development')) {
      // Välimuisti ohitettu! Simuloidaan tietokannan hitautta
      // odottamalla viisi sekuntia.
      sleep(5);     
    }    
  }
}

```

Viiden sekunnin odottelu voi jossain kohtaa kehitystyötä alkaa rasittaa. Toisaalta haluamme silti tietää, milloin välimuistiin on osuttu ja milloin ei. Voimme palauttaa tiedon headerissa, ja fronttiappi voi lukea headerin ja ilmoittaa välimuistiosuman/-ohituksen devaajalle visuaalisesti:

```php

class CacheController extends Controller {
  
  protected static function cacheMiss() {
    if (env('APP_ENV' === 'development')) {
      // Välimuisti ohitettu! Lisätään tieto headeriin.
      Response::header('cache-miss-occurred', 1);   
    }    
  }
}

```

Ja frontissa jotain tähän tyyliin:

```javascript

axios.get(baseUrl + '/vihannekset')
.then((response) => {
  if (response.headers['cache-miss-occurred']) {
    // Hienovaraisesti ilmoita käyttäjälle
    setInterval(() => { alert('Cache miss!')}, 1);
  }

  return response;
})
.then(/*...*/);

```

Summa summarum: harkitse välimuistin toteuttamista sanahirviön *get from controller, flush from usecase* eli GCFU-mallin mukaisesti. Devauksen aikana kehitä käyttöliittymää siten, että välimuistiin osumisen/missauksen seuraukset näkee saman tien.


* \* BBom = Big Ball of Mud = hirveä sekasotku *
