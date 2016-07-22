+++
date = "2016-07-14T19:51:05+03:00"
draft = true
title = "Koneoppimiseen erikoistunut komponentti"

+++

Koneoppiminen (engl. Machine learning) on kovaa kamaa. Vaan kuinka hyödyntää koneoppimista ihan ihkaoikeassa koodipohjassa? Kuinka rakentaa applikaatio, joka *oppii*?

Otetaan esimerkki.

### Turistinähtävyys-luokitin

Suomen ulkoministeriö on tehnyt tilauksen. He haluavat järjestelmän, joka kykenee turistipuodista ostetun postikortin kuvasta päättelemään mikä nähtävyys on kyseessä. Onko siinä Stockmannin kello? Vaiko Tuomiokirkko? Linnanmäen maailmanpyörä? Kenties Kolera-allas?

Järjestelmä on tarkoitus ottaa käyttöön Postin ulkomaanlähetysten osastolla. Joka kerta kun Suomesta lähetetään turistipostikortti ulkomaille, järjestelmä selvittää mitä turistinähtävyyttä kortin kansikuva kuvaa.

Tällä tavoin Suomen valtio saa ensiluokkaista tietoa siitä, mikä nähtävyys on eniten ulkomaalaisten turistien mieleen. 

### Toteutus

Oletetaan, että Posti on hankkinut neliväriskannerin, jonka läpi jokainen ulkomaille matkaava postikortti kulkee. Tuo skanneri lähettää kopion postikortin kansikuvasta suoraan järjestelmäämme. 

Ainoaksi haasteeksi jää täten järjestelmämme puolella selvittää, mitä nähtävyyttä kortti kuvaa.

Lähdetään liikkeelle yksinkertaisesti. Oletetaan, että meillä on funktio, jota Postin skanneri kutsuu aina kun postikortti on skannattu:

```javascript

// TuristinahtavyysLuokitin.js

module.exports = function(scannedImage) {
	// Koodia tänne
}

```

Se miten kuva saadaan Postin järjestelmästä tuotua meidän järjestelmään on suht irrelevanttia. Yksinkertainen ratkaisu on HTTP-rajapinta, joka määrittää HTTP-päätepisteen. Postin skanneri voi sitten tehdä vaikka POST-pyynnön tuohon päätepisteeseen, lähettäen kuvadatan siinä samalla.

```javascript

// TuristinahtavyysRajapinta.js

var luokitin = require('TuristinahtavyysLuokitin');
var Hapi = require('hapi');

var server = new Hapi.Server();
server.connection({ port: 80 }); // Vaatii sudo-oikeudet

server.route({
    method: 'POST',
    path: '/scannedimages',
    handler: function (request, reply) {
        // Hanki kuvadata request-objektista jotenkin
        var korttiKuva = getScannedImage(request);

       // Lähetä kuva luokittimeen
       luokitin(korttiKuva);

       // Ilmoita Postille että kaikki meni hyvin.
       reply("Kuva vastaanotettu, kiitosta vain.");

    }
});

```

Mitä Turistinähtävyys-luokittimen tulisi tehdä kuvalla? 

Ihan ensinnäkin meidän pitäisi jotenkin kyetä *tarkastelemaan* kuvaa. Ihmissilmällä tuo tarkastelu on helppoa. Mutta mites tietokoneen kannalta?

Mistä ihmeestä lähdemme edes liikkeelle? 

Mitä jos tutkisimme kuvan pikseli pikseliltä ja laskisimme tutkiskelun perusteella joitain tilastoja kuvaan liittyen. Siinä voisi olla järkeä. 

Voimme esimerkiksi tsekata sisältääkö kuva merkittävän määrän sinisen värisävyn pikseleitä. Jos sisältää, niin todennäköisesti kuvassa on meri tai taivas. Jos siniset pikselit ovat pääasiassa kuvan ylälaidassa, lienee kyseessä taivas. Jos alalaidassa, lienee meri.

Varmoja emme tietenkään voi olla. Eikä tarvitsekaan. Koneoppiminen perustuu todennäköisyyksiin.

Yksinkertaistetaan ongelmaa hiukan. Oletetaan naiviuspäissämme, että japsituristit ostavat vain postikortteja, joissa on joko a) Stockmannin tai b) Kiasman kuva. Tällä tavalla saamme rajattua vaihtoehdot kahteen.

Mielessämme meillä on kuvat sekä Stockmannin että Kiasman. Mikä näitä kahta erottaa? No, Stockmann on rakennettu paljon aiemmin.

Voiko rakennusvuotta päätellä kuvasta? Ei. Rakennusvuoden eroavaisuus on järjestelmämme kannalta turha.

Toinen erottava asia on, että Kiasman pintamateriaali on valkoista. Stockmannin pintamateriaali on tumman ruskea.

Eikun siis toimeen:

```javascript

// TuristinahtavyysLuokitin.js

module.exports = function(scannedImage) {
	if (kuvanKeskimaarainenVaaleus(scannedImage) > 150) {
		return 'Kiasma';
	} else {
		return 'Stockmann';
	}
}

function kuvanKeskimaarainenVaaleus(image) {
	var pikselit = image.getPixels();

	// Pikseleiden valoisuuksien summa
	var summa = _.reduce(pikselit, function(s, pikseli) {
		return s + pikseli.brightness; // Brightness 0 -> 255
	}, 0);

	// Keskiarvo
	return summa / pikselit.length; 
}

```

Avot - meillä on nyt hyvin alkeellinen tapa erotella Stockmann-postikortit Kiasma-korteista.

Painotus sanalla "alkeellinen". Luokittimemme ei valitettavasti osaa huomioida sitä, että aurinkoisena päivänä valokuvan väriskaala on erilainen kuin pilvisenä päivänä.

Puhumattakaan siitä, että jos otamme kuvan talvella, luokittimemme luulee kaikkia rakennuksia Kiasmaksi. Luminen maisema tekee kuvasta niin valkean, että pieni pläntti tummanruskeaa Stockmannin julkisivua ei riitä.

Ja tästä pääsemmekin pirulliseen ongelmaan. Kiasmasta on niin kovin monenlaisia postikorttikuvia. Yksi on otettu yöllä, toinen Hotelli Tornin huipulta, kolmas sammakkoperspektiivistä, neljäs helikopterista.

Miten ihmeessä valokuvan värimaailmaa tutkiva luokittimemme pystyy ottamaan kaikki mahdolliset kuvakulmat, kuvausetäisyydet ja valaistusolosuhteet huomioon?

Ei mitenkään. Ei yksinkertaisesti ole mahdollista. Luokittimemme on liian suppea - se perustaa päätöksensä liian rajattuun tietoaineistoon.

Tilanne on tismalleen sama kuin jos sinä valitsisit aviovaimon yksinomaan hänen vasemman jalan pottuvarpaan nähtyäsi. Pottuvarvas voi olla vaikka kuinka sutjakka, mutta leidi itse saattaa olla kärttyinen haahka. 

Tietoa ei yksinkertaisesti ole tarpeeksi.

Joten korjataan ongelma - hankitaan lisää tietoa! Luokittimemme ei suinkaan ole pakotettu tyytymään valokuvan värimaailman tutkimiseen.

Entä jos luokittimemme tutkisi valokuvasta myös rakennuksen muotoa? Kokeillaan jotain uhkarohkeaa.

```javascript

// TuristinahtavyysLuokitin.js

module.exports = function(scannedImage) {
	var vaaleus = kuvanKeskimaarainenVaaleus(scannedImage);
	var muoto   = kuvanRakennuksenLaatikkomaisuus(scannedImage);

	return stockmannVaiKiasma(vaaleus, muoto);
}

function kuvanKeskimaarainenVaaleus(image) {
	var pikselit = image.getPixels();

	// Pikseleiden valoisuuksien summa
	var summa = _.reduce(pikselit, function(s, pikseli) {
		return s + pikseli.brightness; // Brightness 0 -> 255
	}, 0);

	// Keskiarvo
	return summa / pikselit.length; 
}

function kuvanRakennuksenLaatikkomaisuus(image) {
	// Oletetaan, että kuvan yläreuna on taivasta

	var taivasVaaleus = image.getPixel(0,0).brightness;

	// Ammutaan vertikaalinen säde kuvan yläreunasta 
	// alareunaa kohti ja tsekataan missä kohtaa kuvaa
	// säde törmää ensimmäiseen "ei-taivas"-pikseliin.
	// Kun näin tapahtuu, oletetaan rohkeasti että kyseessä 
	// on rakennuksen katto (eikä esim. harhaileva varis).

	// Tehdään ylläoleva jokaiselle yläreunan pikselille

	var pikseliSateetMaara = image.getWidth();
	var taivasKattoEtaisyydet = [];

	_.times(pikseliSateetMaara, function(etaisyysVasenReuna) {
		var sateenPituus = ammuSadeAlas(etaisyysVasenReuna, taivasVaaleus, image);
		taivasKattoEtaisyydet.push(sateenPituus);
	});

	// Nyt meillä on etäisyydet kuvan yläreunasta kattoon koko kuvan matkalta.
	// Jos talo on laatikkomainen, nuo etäisyydet ovat lineaarisessa suhteessa toisiinsa.
	// Jos talo on epämuodostuneen näköinen, lineaarista suhdetta ei ole 
	// Näin esim. silloin jos katto tekee kumpuja kuten Tuomiokirkon suhteen.

	if (merkittavaLineaarisuus(taivasKattoEtaisyydet)) {
		return 'Stockmann';
	} else {
		return 'Tuomiokirkko';
	}

}

// Apufunktio laskemaan säteen pituus
function ammuSadeAlas(etaisyysVasenReuna, vaaleusRaja, kuva) {
	
	var kuvanKorkeus = kuva.getHeight();

	_.times(kuvanKorkeus, function(etaisyysYlareunaan) {
		var pikseli = kuva.getPixel(etaisyysVasenReuna, etaisyysYlareunaan);

		// Kaikki taivaan pikselit eivät ole tismalleen mallipikselimme
		// kaltaisia, joten ota pieni virhemarginaali huomioon!
		if (pikseli.brightness < (vaaleusRaja - 25)) {
			return etaisyysYlareunaan;
		}
	});

	// Säde osui alareunaan asti -> koko matkan ajan oli taivasta!
	return kuvanKorkeus;
}

function merkittavaLineaarisuus(luvut) {
	
	// Onko parametrin lukujen välillä lineaarinen riippuvuus?
	// Tai edes "jokseenkin" lineaarinen riippuvuus...

	// Selvitä asia jotenkin (muutos peräkkäisten lukujen välillä aina sama?)
	return lineaarisuudenAste;
}

```

Noin. Nyt hieno luokittimemme ottaa kaksi eri ominaisuutta valokuvasta huomioon. Värimaailman ja kuvassa olevan rakennuksen muodon.

### Ja sitten itse aiheeseen - koneoppiminen

Kaikki edellä oli johdantoa, jolla ei ollut koneoppimisen kanssa juuri mitään tekemistä. Kunhan vain edellä totesimme, että perinteinen ongelmanratkaisu ei sovellu. Emme pysty luomaan kaikenkattavaa algoritmia, joka pystyisi valokuvasta kuin valokuvasta erottamaan Stockmannin Kiasmasta.

Se, että me emme siihen pysty ei tietenkään tarkoita, etteikö *joku muu* siihen pystyisi.

Kuka tuo kuuluisa joku muu on? Tietokone itse.


