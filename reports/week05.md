# Viikko 5 – Wireshark

## Johdanto

Tällä viikolla harjoiteltiin verkkoliikenteen kaappausta ja analysointia Wiresharkilla paikalliselta työasemalta käsin. Tehtävässä selvitettiin pilvipalvelussa sijaitsevan webbipalvelimen (yle.fi) maantieteellistä sijaintia GeoIP-tietojen avulla kahdella eri suodatustekniikalla, sekä tutkittiin DNS-kyselyä ja TLS-kättelyä opiskelijaportaalin (learn.hamk.fi) kirjautumisen yhteydessä. Osat 1 ja 2 on tehty ja raportoitu tässä dokumentissa; osa 3 (HTTP Basic Auth -liikenteen kaappaus Docker-kontista sekä yle.fi:n GeoIP-tarkistus Endpoints-ikkunasta) on vielä tekemättä ja lisätään raporttiin myöhemmin.

## Osa 1 – yle.fi:n GeoIP-sijainnin selvittäminen

Tavoitteena oli selvittää, missä Ylen verkkosivuston taustalla oleva webbipalvelin pilvipalvelussa sijaitsee, kokeilemalla sekä capture filtteriä että display filtteriä. Koska Ylen sivut on hajautettu sisällönjakeluverkkoon (CDN), eri kaappauskerrat saattavat osua eri palvelinosoitteisiin — tämä näkyi myös omissa tuloksissa.

### Tapa A: Capture filter

Kaappaus rajattiin jo ennen tallennuksen aloittamista komennolla:

```text
host yle.fi and port 443
```

Capture filter suodattaa vain kyseiseen nimeen (DNS:n kautta selvitettyyn osoitteeseen) ja porttiin 443 kohdistuvan liikenteen, joten tallenteeseen jää vain yksi palvelinosoite.

**Ensimmäinen kokeilu**

- Palvelimen IP-osoite: 13.249.8.4
- Havainto: liikenne oli QUIC-protokollaa (HTTP/3), eli sisältö oli salattua jo kuljetuskerroksella eikä varsinaista sovellustason dataa päässyt näkemään.

**Toinen kokeilu**

- Palvelimen IP-osoite: 13.249.8.116
- GeoIP: United States, ei kaupunkia, AS 16509 Amazon.com, Inc.

![Endpoints-ikkuna capture filterillä host yle.fi and port 443 — palvelin 13.249.8.116](images/WiresharkOsa1TapaA.png)

Molemmilla kerroilla osoite kuului Amazonin AS 16509 -verkkoon, mikä kertoo Ylen käyttävän AWS:ää pilvialustanaan. GeoIP-tietokanta ilmoitti maaksi Yhdysvallat, mutta kaupunkia ei määritelty ja koordinaatit (37,751° / -97,822°) ovat Yhdysvaltojen maantieteellinen keskipiste eli tietokannan oletusarvo silloin, kun tarkempaa sijaintia ei tunneta. Tämä ei kuitenkaan tarkoita, että palvelin fyysisesti sijaitsisi Yhdysvalloissa: pingin vasteaika oli keskimäärin noin 15 ms, mikä ei ole mahdollinen lukema Atlantin yli. Todennäköisin selitys on, että kyseessä on AWS:n sisällönjakeluverkon (esim. CloudFront) solmu, joka palvelee liikennettä lähialueelta, mutta jonka IP-osoite on rekisteröity Amazonin Yhdysvaltalaiseen organisaatioon — GeoIP näyttää siis osoitteen rekisteröintimaan, ei laitteen todellista fyysistä sijaintia.

### Tapa B: Display filter

Toisella kerralla kaappaus tehtiin ilman capture filteriä, jolloin kaikki sivulatauksen aikana syntynyt liikenne tallentui. Rajaus tehtiin vasta jälkikäteen display filtereillä:

```text
tls.handshake.extensions_server_name contains "yle.fi"
tls.handshake.extensions_server_name == "yle.fi"
```

![Display filter tls.handshake.extensions_server_name contains "yle.fi" — useita eri yle.fi-alipalvelimia](images/WiresharkOsa1TapaB1.png)

Ensimmäinen rivi (Client Hello / SNI) paljasti heti, ettei sivu lataudu yhdeltä palvelimelta: selain avasi yhteyden kymmeneen eri osoitteeseen, joiden palvelinnimet (Server Name -sarake) olivat mm. www.yle.fi, yle.fi, design-system.cdn.yle.fi, locations.api.yle.fi, login.api.yle.fi, player-v2.yle.fi, img-cdn.yle.fi, analytics-sdk.yle.fi, player.api.yle.fi ja haavi.yle.fi. Osa yhteyksistä oli QUIC:ia, osa TLS 1.3:a TCP:n päällä.

![Endpoints-ikkuna display filterin jälkeen — kaikki kymmenen yle.fi-palvelinta ja niiden GeoIP-tiedot](images/WiresharkOsa1TapaB2.png)

Kaikki kymmenen palvelinta kuuluivat samaan AS 16509 (Amazon.com, Inc.) -verkkoon, eli koko sivuston alipalvelut pyörivät saman pilvipalveluntarjoajan infrastruktuurissa. Yhdeksän osoitetta näytti samat Yhdysvaltojen oletuskoordinaatit kuin tapa A:ssa, joten niihin pätee sama CDN-tulkinta. Yksi osoite, 99.81.134.100 (haavi.yle.fi), erosi joukosta: GeoIP ilmoitti maaksi Irlannin ja kaupungiksi Dublinin, ja koordinaatit olivat selvästi tarkempia kuin muualla. Tämä viittaa siihen, että kyseessä on kiinteä palvelin Amazonin Dublinin konesalialueella eikä jakeluverkon solmu — GeoIP-tietokanta tuntee sille tarkan sijainnin, koska osoite ei ole dynaamisesti jaettu CDN-reuna.

Endpoints-taulukon sarakkeista on hyvä huomata, että Tx Packets oli palvelimilla 0: suodatin (tls.handshake...) näyttää vain Client Hello -paketit, jotka kulkevat aina selaimelta palvelimelle päin, joten vastaukset eivät osuneet samaan suodattimeen. Oma kone 192.168.1.38 lähetti 13 Client Helloa, ja koko kaappauksessa oli yhteensä 1678 pakettia. Palvelin 13.249.8.86 erottui 111 paketin kokonaismäärällä — sama 13.249.8.-alkuinen osoitealue kuin tapa A:n palvelimella, joten kyseessä on todennäköisesti pääsivun palvelin.

### Vertailu

| | Capture filter (tapa A) | Display filter (tapa B) |
|---|---|---|
| Rajaus tehdään | ennen kaappausta | kaappauksen jälkeen |
| Tallenteessa | vain yle.fi:n liikenne, 1 palvelin | kaikki liikenne, 1678 pakettia |
| Näkyi | pääsivun palvelin | pääsivu ja 9 muuta yle.fi-alipalvelinta |
| Sopii tilanteeseen | kun tiedetään tarkasti, mitä etsitään | kun halutaan tutkia ja vaihtaa näkökulmaa |

Ero tekniikoiden välillä näkyi konkreettisesti: capture filter tallensi vain yhden palvelimen liikenteen, koska rajaus tehtiin jo ennen kaappausta eikä muuta liikennettä tallentunut lainkaan. Display filter sen sijaan paljasti koko sivulatauksen rakenteen, koska kaikki liikenne oli tallessa ja sitä pystyi suodattamaan jälkikäteen eri näkökulmista. Capture filter on kevyempi ja kohdennetumpi, kun kohde tiedetään etukäteen; display filter taas soveltuu paremmin tutkivaan analyysiin, jossa ei vielä tiedetä, mitä kaikkea liikenteessä on.

## Osa 2 – learn.hamk.fi: DNS-kysely ja TLS-kättely

Kaappaus aloitettiin ennen osoitteen learn.hamk.fi avaamista selaimessa ja pysäytettiin onnistuneen kirjautumisen jälkeen.

### DNS-kysely

Kysymykset selvitettiin suodattimella `dns.qry.name == "learn.hamk.fi"`.

![DNS A-kyselyn vastaus, paketti 896: learn.hamk.fi = 195.148.239.84, TTL 3030 s](images/WiresharkOsa2DNS.png)

| Kysytty | Vastaus | Suodatin |
|---|---|---|
| IP-osoite | 195.148.239.84 | `dns.qry.name == "learn.hamk.fi"` |
| TTL | 3030 s (50 min 30 s) | sama |
| Auktoritatiiviset palvelimet | ns1, ns2, ns3.hamk.fi sekä ns-secondary.funet.fi | `dns.qry.type == 2` |

TTL-arvo 3030 sekuntia ei ole tasaluku, mikä kertoo, että vastaus tuli resolverin välimuistista — TTL laskee koko ajan sen jälkeen, kun tietue haettiin ensimmäisen kerran auktoritatiiviselta palvelimelta. Alkuperäinen arvo on todennäköisesti tasaluku 3600 s (1 tunti), mikä vahvistui myöhemmin NS-kyselyn vastauksesta, jossa TTL oli tasan 3600 s.

Valitussa A-tietueen vastauspaketissa Authority RRs -kenttä oli 0, eli paketti ei sisältänyt suoraan nimipalvelintietoa. Toisessa samaan aikaan lähetetyssä kyselyssä selain kysyi myös HTTPS-tyyppistä tietuetta, jota ei ollut olemassa, joten vastauksena palautettiin vyöhykkeen SOA-tietue (Authoritative nameservers -kohdassa), joka nimeää ensisijaiseksi palvelimeksi ns1.hamk.fi:n.

![SOA-tietue (Authoritative nameservers), paketti 897: primary name server ns1.hamk.fi](images/WiresharkOsa2DNS2AuthNameserver.png)

Täydellisen nimipalvelinlistan selvittämiseksi ajettiin erillinen, lyhyt kaappaus komennolla:

```powershell
nslookup -type=NS hamk.fi
```

ja suodatettiin vastaus display filterillä `dns.qry.type == 2`.

![NS-kyselyn vastaus, paketti 122: neljä auktoritatiivista nimipalvelinta](images/WiresharkOsa2DNS3Nimipalvelimet.png)

Vastauksessa (paketti 122) oli neljä NS-tietuetta toimialueelle hamk.fi: ns1.hamk.fi, ns2.hamk.fi, ns3.hamk.fi ja ns-secondary.funet.fi. Kolme ensimmäistä ovat HAMKin omia nimipalvelimia, ja neljäs on Funetin eli Suomen korkeakoulujen yhteisen tietoverkon palvelin, joka toimii varapalvelimena HAMKin oman verkon ulkopuolella siltä varalta, että HAMKin oma verkko olisi tavoittamattomissa. Additional records -kohdasta löytyivät myös nimipalvelimien IP-osoitteet (esim. ns1.hamk.fi = 195.148.152.2), jotta kysyjän ei tarvitse hakea niitä enää erikseen.

Kaikkiin DNS-kyselyihin vastasi osoite 192.168.1.1, eli kotireitittimeni (Zyxel). Se ei ole toimialueen auktoritatiivinen nimipalvelin vaan DNS-resolveri, joka välittää kyselyn eteenpäin ja vastaa jatkossa välimuistista — tämä näkyy sekä Flags-kentässä (palvelin ei ole auktoriteetti) että erittäin nopeassa vastausajassa (n. 2,5 ms), joka olisi mahdoton, jos kysely olisi kiertänyt oikeasti HAMKin nimipalvelimille asti.

### TLS-kättely

TLS-kättely tutkittiin suodattimilla:

```text
tls.handshake.type == 1 && tls.handshake.extensions_server_name == "learn.hamk.fi"
ip.src == 195.148.239.84 && tls.handshake.type == 2
```

(Tehtävänannon vihjeenä annettu `ssl.handshake.ciphersuite` on Wiresharkin nykyversioissa nimetty uudelleen muotoon `tls.handshake.ciphersuite`.)

![Client Hello, SNI = learn.hamk.fi, TLS 1.3, 16 ehdotettua cipher suitea](images/WiresharkOsa2TSL.png)

Selain (Client Hello) ehdotti yhteensä 16 cipher suitea, joista yksi on GREASE-arvo (selaimen tapa testata, ettei palvelin kaadu tuntemattomasta arvosta, ei oikea algoritmi). Todellisia ehdokkaita oli 15: kolme TLS 1.3:n cipher suitea ja 12 vanhempaa TLS 1.2 -yhteensopivaa suitea.

![Client Hello -paketin cipher suite -lista kokonaisuudessaan](images/WiresharkOsa2CipherSuites.png)

![Client Hellon supported_versions-laajennus: TLS 1.3 ja TLS 1.2 tuettuina](images/WiresharkOsa2ExtensionsSupported_versions.png)

Palvelin vastasi Server Hellolla, jossa se ilmoitti käyttävänsä protokollaversiota TLS 1.3 (supported_versions-laajennus) ja valitsi cipher suiteksi TLS_AES_128_GCM_SHA256 (0x1301) — listan ensimmäisen varsinaisen (ei-GREASE) vaihtoehdon, eli selaimen omaa ensisijaista ehdotusta.

![Server Hello, paketti 874: TLS 1.3, valittu cipher suite TLS_AES_128_GCM_SHA256](images/WiresharkOsa2ExtensionsSupported_versions2.png)

| Kysytty | Vastaus | Mistä luettu |
|---|---|---|
| TLS-versio | TLS 1.3 | Server Hello → supported_versions |
| Selaimen ehdotus | 15 algoritmia (3× TLS 1.3, 12× TLS 1.2) + GREASE | Client Hello → Cipher Suites |
| Palvelimen valinta | TLS_AES_128_GCM_SHA256 | Server Hello → Cipher Suite |

## Osa 3 – (vielä tekemättä)

Osa 3 (Basic Auth -tunnusten kaappaus Docker-kontista ja yle.fi:n GeoIP-tarkistus Endpoints-ikkunasta) toteutetaan ja raportoidaan myöhemmin.

## Yhteenveto

Osissa 1 ja 2 harjoiteltiin kahta eri tapaa rajata Wireshark-kaappauksesta kiinnostava liikenne — capture filter ennen kaappausta ja display filter sen jälkeen — ja havaittiin, että molemmilla on oma käyttötarkoituksensa: capture filter kun kohde tiedetään tarkasti etukäteen, display filter kun halutaan tutkia tallennettua liikennettä laajemmin ja vaihtaa näkökulmaa. GeoIP-tietojen tulkinnassa opittiin, että tietokannan ilmoittama maa ja koordinaatit kertovat usein vain IP-osoitteen rekisteröintiorganisaation sijainnin (tässä tapauksessa Amazon, Yhdysvallat) eivätkä laitteen todellista fyysistä sijaintia — tätä voi arvioida esimerkiksi vasteajan (ping) perusteella. DNS-osiossa opittiin erottamaan tavallinen resolveri (kotireititin) auktoritatiivisesta nimipalvelimesta TTL-arvojen ja Flags-kentän avulla, sekä se, miten SOA- ja NS-tietueista löytää toimialueen vastuulliset nimipalvelimet. TLS-osiossa havainnollistui käytännössä, miten Client Hello ja Server Hello -paketit paljastavat, mitä salausalgoritmeja selain ehdottaa ja minkä niistä palvelin lopulta valitsee — ja että nykyaikainen learn.hamk.fi-yhteys käyttää TLS 1.3:a vahvalla AEAD-salauksella (AES-GCM).
