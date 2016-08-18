+++
date = "2016-08-18T05:47:59+03:00"
draft = false
title = "Konsolitulosteen väritys"

+++

Löysin muutama kuukausi sitten Node.js-lisäosan nimeltä [chalk](https://github.com/chalk/chalk). Tämä chalk-kirjasto tarjoaa kivan rajapinnan *värittää* komentorivillä näkyvät console.log-tekstit. Värityksestä on paljon hyötyä tapauksissa, joissa Node.js-skripti printtaa runsaasti tekstiä komentoriville.

Käyttö on helppoa - riittää, että työntää merkkijonon chalk-kirjaston metodikutsun sisälle:

```javascript

console.log(chalk.cyan("beforeMove cb"))

```

tuottaa seuraavanlaisen lopputuleman:

![Turkoosin värinen merkkijono](/blog/public/img/console-log-cyan.png)

Käytän eri värejä *simuloimaan* eri käyttäjien kommunikaatiota Node.js-serverin kanssa. Sanotaan esimerkiksi, että meillä on kolme käyttäjää A, B ja C. Nuo kaikki kolme saavat viestejä applikaatiolta. Tuotantokäytössä nuo viestit luonnollisesti menisivät kunkin käyttäjän www-selaimeen, mutta testivaiheessa on helpompaa vain printata kunkin käyttäjän saama viesti komentoriville. Ongelmaksi muodostuu, että *komentoriviltä on visuaalisesti vaikea hahmottaa mikä viesti kuuluu millekin käyttäjälle*:

![Kaikkien käyttäjien kommunikaatio palvelimen kanssa printataan testiajossa komentoriville](/blog/public/img/console-log-users-white.png)

Chalk-kirjaston avulla voimme assignoida kullekin käyttäjälle oman värin, jolloin on visuaalisesti helppo erottaa eri käyttäjien viestit toisistaan:

```javascript

// Participant.js

var chalk = require('chalk');

// Määritä kullekin testikäyttäjälle oma väritysfunktio
var consoleColorers = {
  'A': chalk.bgGreen,
  'B': chalk.bgYellow,
  'C': chalk.bgBlue
}

function Participant(id, communicator) {
  // Unique among all participants
  this.id = id;
  // communicator is probably Socket-object, can be mocked.
  this.communicator = communicator;

  this.msg = function(msg) {
    // Väritä tämän käyttäjän saama viesti hänen omalla värillään
    // ja printtaa viesti komentoriville.
    var text = this.id + ': ' + msg.msg;
    console.log(consoleColorers[this.id](text));
  }

  // ... muut metodit

}

module.exports = Participant;

```

```javascript

// test.js

var _ = require('lodash'); // _.map-funktiota varten
var Participant = require('./Participant');

// Luo testipelaajia kolme kpl.
var players = [new Participant('A', {}), new Participant('B', {}), new Participant('C', {})];

// Lähetä kullekin pelaajalle viesti kerran sekunnissa
setInterval(function() {
  _.map(players, function(player) {
    player.msg({topic: 'testi', msg: 'Sinulle on postia'});
  })
}, 1000);


```

Ylläoleva tuottaa kauniin lopputuloksen komentoriville kun test.js-tiedosto suoritetaan:

![Kullakin käyttäjällä on oma värinsä komentoriville](/blog/public/img/console-log-users-custom-colors.png)

Pohjimmiltaan värien käyttö on tietenkin makukysymys, mutta ainakin itselläni se helpottaa testitulosteen lukemista huomattavasti.