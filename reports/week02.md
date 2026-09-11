# Viikko 2 – SNMP ja verkon perustason valvonta

## 1. Johdanto

SNMP (Simple Network Management Protocol) on standardoitu protokolla verkon laitteiden valvontaan ja tiedonkeruuseen. Sen avulla voidaan valvoa ja hallita verkon laitteita keskitetysti. SNMP:n toiminta perustuu kahteen pääkomponenttiin: valvottavassa laitteessa toimivaan **agenttiin** (esim. `snmpd`), joka vastaa kyselyihin, ja tietoa keräävään **manageriin** (esim. Zabbix). Agentti tarjoaa tietonsa jäsennellyssä muodossa MIB:n (Management Information Base) kautta, jossa jokaisella tiedolla on numeerinen osoite (OID, Object Identifier) sekä selkokielinen luettava nimi.

Tässä tehtävässä asennettiin ja konfiguroitiin SNMP-agentti kolmelle kurssiympäristön laitteelle (web1, db1, branch-client) ja kerättiin niistä tietoa komentorivityökaluilla snmpget ja snmpwalk.

## 2. Asennus

### 2.1 SNMP-agentin asennus

Asensin snmp ja snmpd -paketit web1-konttiin (apt install snmp snmpd). Agentti asennettiin kullekin kohdelaitteelle (web1, db1, branch-client) samalla tavalla:

```bash
docker exec -it clab-hamk-verkonhallinta-golden-web1 bash -c "apt update && apt install -y snmp snmpd"
```

Palvelun tilan tarkistin ohjeessa annetulal komennolla:

```bash
docker exec -it clab-hamk-verkonhallinta-golden-web1 service snmpd status
```

### 2.2 Konfigurointi

Oletuskonfiguraatioon piti tehdä kaksi muutosta, jotta yhteys saatiin toimimaan. Kumpikaan syy ei käynyt suoraan ilmi tehtäväohjeesta, vaan ne löytyivät vasta kokeilemalla. Apuna käytin tekoälyä.

Ensimmäinen ongelma liittyi siihen, mitä tietoja SNMP-kyselyillä sai näkyviin. Ubuntu 24.04:ssä tiedoston `/etc/snmp/snmpd.conf` oletusrivi on `rocommunity public default -V systemonly`, jossa määre `-V systemonly` rajaa kyselyt vain laitteen perustietoihin, kuten nimeen, kuvaukseen ja käyttöaikaan. Esimerkiksi verkkoliitäntöjen tietoja (`ifDescr`) ei tällä asetuksella pystynyt hakemaan. Rajaus poistettiin muuttamalla rivi muotoon `rocommunity public`.

Toinen ongelma oli se, että snmpd kuunteli oletuksena kyselyitä vain koneen omalta localhost-osoitteelta rivillä `agentaddress 127.0.0.1,[::1]`. Tämän vuoksi mikään verkon yli tullut SNMP-kysely ei päässyt perille lainkaan, vaan kysely aikakatkaistiin viestillä "Timeout: No Response", vaikka muu konfiguraatio oli kunnossa. Ongelma korjattiin muuttamalla rivi muotoon `agentaddress udp:161`, jolloin agentti alkoi kuunnella kaikilta verkkorajapinnoilta eikä pelkästään localhostilta.

Molempien muutosten jälkeen palvelu käynnistettiin uudelleen komennolla `service snmpd restart`, ja toimivuus varmistettiin komennolla `ss -lunp`: rivi `0.0.0.0:161` kertoi, että agentti kuunteli nyt oikein kaikilta rajapinnoilta. On hyvä huomata, että nämä tiukat oletusasetukset eivät ole ohjelmiston virhe, vaan paketin ylläpitäjien tietoinen tietoturvaratkaisu. Tähän palataan tarkemmin pohdintaosiossa (6).

### 2.3 Kyselytyökalujen asennus

Ansible-hallintanoodille asennettiin asiakastyökalut:

```bash
docker exec -it clab-hamk-verkonhallinta-golden-ansible bash -c "apt update && apt install -y snmp snmp-mibs-downloader"
```

Ensimmäinen kysely (`snmpwalk -v2c -c public web1 system`) epäonnistui virheellä `Unknown Object Identifier (Sub-id not found: (top) -> system)`. Syy: Ubuntu/Debian poistaa MIB-nimien tulkinnan käytöstä oletuksena lisenssisyistä (`/etc/snmp/snmp.conf`: rivi `mibs :`). Asensin lisäksi paketin `snmp-mibs-downloader` ja otin nimien käytön käyttöön:

```bash
sed -i 's/^mibs :/#mibs :/' /etc/snmp/snmp.conf
```

Tämän jälkeen kyselyt toimivat ja palauttivat ihmisluettavia MIB-nimiä numeeristen OID:ien sijaan.

### 2.4 Verkkoinfrastruktuurin poikkeama

Kesken tehtävää 2.5 havaittiin, ettei web1:llä (eikä db1:llä tai r2:lla) ollut lainkaan topologian mukaista data-verkon rajapintaa (`eth1`/`eth2`) — vain containerlabin oma hallintaverkon `eth0`. Tämä ei ollut SNMP-konfiguraatio-ongelma vaan koko ympäristön verkkolinkitysten katoaminen (todennäköisesti kontit olivat käynnistyneet uudelleen ilman täyttä `containerlab deploy` -sykliä). Korjaus tehtiin ajamalla ympäristö kokonaan uudelleen (`scripts/destroy.sh` + `scripts/deploy.sh`), minkä jälkeen kaikki data-verkon rajapinnat palautuivat oikein.

## 3. Kerätyt tiedot

Tässä osiossa kerättiin web1-palvelimelta kolme perustietoa: laitteen nimi, käyttöjärjestelmä ja käynnissäoloaika. Tiedot haettiin ansible-hallintanoodilta kahdella eritavalla: yhdellä `snmpwalk`-komennolla joka haki kaikki kolme kerralla, ja kolmella erillisellä `snmpget`-komennolla jotka hakivat tiedot yksi kerrallaan.

### snmpwalk system (web1)

![snmpwalk system -komennon tuloste web1:ltä](images/snmpwalk%20-v2c%20-c%20public%20web1%20system.png)

*Kuva: yhdellä `snmpwalk`-komennolla haetaan kerralla kaikki system-ryhmän tiedot, mm. laitteen nimi, käyttöjärjestelmä ja käynnissäoloaika.*

### snmpget-kyselyt (tehtävä 2.4)

![snmpget sysName.0](images/snmpget%20-v2c%20-c%20public%20web1%20sysName.0.png)

*Kuva: `snmpget sysName.0` hakee vain laitteen nimen — vastaus on "web1".*

![snmpget sysDescr.0](images/snmpget%20-v2c%20-c%20public%20web1%20sysDescr.0.png)

*Kuva: `snmpget sysDescr.0` hakee käyttöjärjestelmän kuvauksen — näkyvissä Linux-ydin, versio ja arkkitehtuuri.*

![snmpget sysUpTime.0](images/snmpget%20-v2c%20-c%20public%20web1%20sysUpTime.0.png)

*Kuva: `snmpget sysUpTime.0` hakee SNMP-agentin käynnissäoloajan sadasosasekunteina ja luettavassa muodossa.*


## 4. Verkkorajapinnat

```
$ snmpwalk -v2c -c public web1 ifDescr
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.2 = STRING: eth0
IF-MIB::ifDescr.63 = STRING: eth1
```

Web1:ltä löytyi kolme rajapintaa:

- `lo` (indeksi 1) — looppback, ei varsinainen verkkoyhteys
- `eth0` (indeksi 2) — containerlabin hallintaverkko (172.20.20.0/24)
- `eth1` (indeksi 63) — **yhdistää web1:n topologian data-verkkoon** (Server LAN, 10.10.20.101/24, r2:n kautta)

## 5. OID-analyysi

Objektien tarkoitus selvitettiin `snmptranslate -Td`-komennolla (huom: toisin kuin `snmpget`/`snmpwalk`, `snmptranslate` vaatii täyden moduulin nimen etuliitteeksi, esim. `SNMPv2-MIB::sysName.0` — pelkkä lyhyt nimi ei riittänyt).

| OID | Tarkoitus |
|------|------|
| `sysName.0` | Laitteelle hallinnollisesti asetettu nimi |
| `sysDescr.0` | Vapaamuotoinen tekstikuvaus laitteesta: laitteiston tyyppi, käyttöjärjestelmä ja sen versio sekä verkko-ohjelmisto |
| `sysUpTime.0` | Aika sadasosasekunteina siitä, kun hallintaohjelmiston (SNMP-agentin) toiminta viimeksi alustettiin uudelleen |
| `ifDescr` | Tekstikuvaus verkkorajapinnasta: valmistaja, tuotenimi ja laitteisto-/ohjelmistoversio |
| `ifOperStatus` | Rajapinnan senhetkinen toiminnallinen tila: `up(1)`, `down(2)`, `testing(3)`, `unknown(4)`, `dormant(5)`, `notPresent(6)`, `lowerLayerDown(7)` |

## Usean laitteen valvonta

SNMP-agentti asennettiin ja konfiguroitiin samalla tavalla myös db1:lle ja branch-clientille, ja kyselyt suoritettiin kaikille kolmelle laitteelle ansible-noodilta:

| Laite | Nimi | Käyttöjärjestelmä | Uptime |
|---------|---------|---------|---------|
| web1 | web1 | Linux web1 6.18.33.2-microsoft-standard-WSL2 x86_64 | 0:02:18.47 |
| db1 | db1 | Linux db1 6.18.33.2-microsoft-standard-WSL2 x86_64 | 0:01:40.66 |
| branch-client | branch-client | Linux branch-client 6.18.33.2-microsoft-standard-WSL2 x86_64 | 0:00:58.85 |

Kaikki kolme laitetta vastasivat kyselyihin onnistuneesti samasta ansible-hallintanoodista. SNMP:n yksi hallintapiste pystyy kokoamaan tiedot useasta laitteesta samalla protokollalla riippumatta laitteen roolista (web-palvelin, tietokantapalvelin, sivutoimipisteen työasema).

## 6. Pohdinta

**1. SNMP:n hyödyt verkonhallinnassa**

SNMP (Simple Network Management Protocol) on standardoitu protokolla, jonka avulla verkon laitteita voidaan valvoa ja hallita keskitetysti. 

SNMP:n hyötyjä ovat keskitetty valvonta, riippumattomuus laitevalmistajasta, automatisoitavissa olevat hälytykset, suorituskyvyn seuranta, etähallinta ja toiminnan keveys ja nopeus.

- Keskitetty valvonta: yksi hallintajärjestelmä (NMS, Network Management System) voi seurata satoja tai tuhansia laitteita samanaikaisesti.
- Laitevalmistajariippumattomuus: SNMP on avoin standardi, jota tukevat lähes kaikki verkkolaitteet (reitittimet, kytkimet, palvelimet, tulostimet jne.), joten hallinta ei ole sidottu yhteen valmistajaan.
- Automaattinen hälytys (trap/notification): laitteet voivat itse ilmoittaa hallintajärjestelmälle poikkeamista (esim. linkki katkesi), jolloin ongelmiin voidaan reagoida nopeasti.
- Suorituskyvyn ja kapasiteetin seuranta: mahdollistaa trendien seurannan ja ennakoivan kapasiteettisuunnittelun ennen kuin resurssit loppuvat.
- Etähallinta: laitteiden asetuksia voidaan lukea ja osittain myös muuttaa etänä ilman fyysistä pääsyä laitteelle.
- Kevyt ja nopea: perustuu UDP:hen, joten se kuormittaa verkkoa vähän ja soveltuu suurienkin verkkojen jatkuvaan seurantaan.

Tässä harjoituksessa käytettiin komentorivityökaluja snmpget/snmpwalk.

**2. SNMP:n avulla  kerättävät tiedot**

SNMP:n avulla luetaan tietoa laitteen MIB-tietokannasta (Management Information Base). Tyypillisiä tietoja:

- Verkkoliikennetilastot: sisään/ulos tulevien tavujen ja pakettien määrä, virheiden ja pudonneiden pakettien määrä liitäntäkohtaisesti.
- Liitäntöjen tila: portin/liitännän ylös/alas-tila, nopeus, duplex-asetus.
- Laitteiston kuormitus: suorittimen käyttöaste, muistin käyttö, levytila.
- Laitetiedot: laitteen nimi, sijainti, käyttöjärjestelmäversio, käyttöaika (uptime), sarjanumero.
- Reititys- ja kytkentätiedot: reititystaulut, VLAN-tiedot, ARP-taulut.
- Ympäristötiedot: joissain laitteissa myös lämpötila, virtalähteen tila, tuulettimien toiminta.
- Tapahtumailmoitukset (trap): esim. linkin katkeaminen, laitteen uudelleenkäynnistys, lämpötilahälytys.

Tehdyssä harjoituksessa kerättiin vain system- ja interfaces ryhmän tietoja.

**3. Yhteisöpohjaisen SNMPv2:n ongelmat**

SNMPv1 ja SNMPv2c käyttävät ns. "community string" -pohjaista todennusta, jossa on useita tietoturvaongelmia:

- Salaamaton liikenne: community-merkkijono ja kaikki data kulkevat selväkielisenä UDP-paketeissa, joten ne on helppo siepata verkkoliikenteen kuuntelulla.
- Heikko todennus: community string toimii käytännössä salasanana, mutta sitä ei ole suojattu — kuka tahansa verkkoliikennettä kuunteleva saa sen selville.
- Ei käyttäjäkohtaista tunnistusta: kaikki samaa community-merkkijonoa käyttävät laitteet/käyttäjät ovat samanarvoisia, ei voida erottaa kuka teki mitä (ei jäljitettävyyttä).
- Oletusarvot yleisiä ja tunnettuja: monissa laitteissa oletus-community-nimet ovat "public" (luku) ja "private" (kirjoitus), jotka usein jäävät vaihtamatta — helppo hyökkäyskohde.
- Ei eheyden suojausta: paketteja voidaan väärentää tai muokata matkalla, koska niitä ei ole allekirjoitettu.
- Karkea pääsynhallinta: yleensä vain kaksi tasoa (luku/kirjoitus) yhdellä yhteisellä salasanalla, ei hienojakoista käyttöoikeuksien hallintaa.

Toteutetussa harjoituksessa konkreettinen riski on muiden laitteiden kanssa samassa verkossa oleva attacker-kontti (Kali Linux). Tämä attacker-kontti pystyisi periaatteessa kuuntelemaan SNMP-liikennettä samasta verkkosegmentistä.

Toinen harjoituksessa esiin tullut rajoittava seikka oli agentaddress-asetukset. Oletuskonfiguraatiot voivat olla tiukkoja tietoturvasyistä ja niitä voi joutua löysäämään.

**4. SNMPv3:n soveltuvuus**

SNMPv3 tuo mukanaan käyttäjäkohtaisen todennuksen, salauksen ja eheystarkistuksen, joten sitä kannattaa suosia esimerkiksi:

- Julkisen internetin tai epäluotettavan verkon yli hallittaessa — kun hallintaliikenne kulkee segmenttien tai reitittimien läpi, joissa salakuuntelu on mahdollista.
- Kriittisessä infrastruktuurissa (esim. yritysverkon runkolaitteet, palomuurit, palvelinkeskukset), joissa laitteiden väärinkäyttö tai konfiguraation muuttaminen aiheuttaisi vakavaa haittaa.
- Kun vaaditaan säädöstenmukaisuutta (esim. tietoturvastandardit, sisäiset tietoturvapolitiikat, GDPR-tyyppiset vaatimukset), jotka edellyttävät salattua ja todennettua hallintaliikennettä.
- Useiden ylläpitäjien ympäristössä, kun halutaan käyttäjäkohtaiset tunnukset ja oikeudet (kuka saa lukea, kuka kirjoittaa) sekä jäljitettävyys siitä, kuka teki mitä.
- Kun laitteita hallitaan etäältä VPN:n ulkopuolella tai muuten suojaamattoman yhteyden yli, jolloin SNMPv3:n autentikointi (esim. SHA) ja salaus (esim. AES) korvaavat puuttuvan verkkotason suojauksen.

Käytännössä SNMPv2c:tä voi  hyväksyä hyvin rajatussa, fyysisesti tai VLAN-tasolla eristetyssä hallintaverkossa, mutta aina kun liikenne voi kulkea vähänkään laajemmalla tai jaetulla verkkosegmentillä, SNMPv3 on selkeästi turvallisempi valinta.
