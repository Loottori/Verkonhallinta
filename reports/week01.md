  # Viikko 1 – Verkon dokumentointi

  ## 1. Johdanto

  Tämä raportti dokumentoi  Verkonhallinta -harjoitusympäristön reititystoteutuksen ja analysoi sen toimivuutta. Ympäristö on Containerlab-työkalulla rakennettu simuloitu verkkotopologia, joka koostuu kolmesta  reitittimestä (r1, r2, r3) sekä niihin kytketyistä pääte- ja palvelinlaitteista eri aliverkoissa. Topologiaan kuuluu lisäksi hallintaverkko (management LAN) valvontatyökaluja (Prometheus, Grafana, Zabbix, cadvisor, syslog) ja automaatiota (Ansible) varten, sekä erillinen hyökkääjä-kontti (attacker) verkon tietoturvan testaamista varten.
  
  Ympäristön tarkoituksena on tarjota realistinen, mutta hallittu ja turvallinen harjoitusalusta, jossa voidaan opetella ja testata reititystä, verkon dokumentointia sekä verkkoinfrastruktuurin hallintaa käytännössä ilman fyysistä laitteistoa. Tässä raportissa keskitytään erityisesti siihen, toimiiko reititys suunnitellulla tavalla eri verkkosegmenttien välillä, ja mitä poikkeamia havaittiin testauksen aikana.

  ## 2. Verkkokaavio

  ![Verkon topologia](images/topology.png)

  ## 3. Laiteluettelo
  
| Laite | Rooli |
|---|---|
| r1 | Edge-reititin. Yhdistää User LAN:in (10.10.10.0/24, kaksi osiota: client1 ja attacker) runkoverkkoon (10.255.12.0/30 → r2). OSPF-alue 0, antaa oletusreitin (`default-information originate`) muulle verkolle, NAT ulos |
| r2 |  Core-reititin. Yhdistää serveriverkon, hallintaverkon ja r3. Kokoaa ja välittää sisäisiä verkkoja (server, management, branch)toisiinsa, mutta ei ole se piste josta mennään "ulos". |
| r3 | Branch-reititin. Yhdistää LAN:in (branch-client, 10.10.30.0/24) runkoverkkoon r2:n kautta. Ei OSPF-oletusreitti.|
| client1 | Tavallinen käyttäjätyöasema, osoite 10.10.10.101, User LAN:issa |
| attacker | Kali Linux -hyökkääjäkone |
| web1 | Web-palvelin (Server LAN) |
| db1 | Tietokantapalvelin (Server LAN) |
| branch-client | Sivutoimipisteen käyttäjä |
| ansible | Ansible-hallintanoodi. Hoitaa SSH-avaiten jaon muihin laitteisiin |
| prometheus | Metriikan keräys |
| grafana | Dashboardit/Datan visualisointi |
| zabbix | Valvonta-työkalu. Appliance-kontti. |
| cadvisor | Konttimetriikka |
| syslog | Lokipalvelin. 172.20.20.50 |
| NetBox | Erillinen Docker Compose -pino (ei containerlab-noodi). Source of Truth -dokumentaatiopalvelu IP-osoitteille ja laitteille. |
| mgmt-bp | Tekninen L2-silta-kontti |
| srv-bp | Tekninen L2-silta-kontti |

  ## 4. IP-suunnitelma

  | Verkko | Tarkoitus | Yhdyskäytävä |
  |---|---|---|
  | 10.10.10.0/24 | User LAN (client1) | r1 eth2 = 10.10.10.1 |
  | 10.10.10.0/24 (2. segmentti) | attacker-linkki | r1 eth3 = 10.10.10.254 |
  | 10.10.20.0/24 | Server LAN (web1, db1) | r2 eth2 = 10.10.20.1 |
  | 10.10.30.0/24 | Branch LAN | r3 eth2 = 10.10.30.1 |
  | 10.10.99.0/24 | Management LAN | r2 eth3 = 10.10.99.1 |
  | 10.255.12.0/30 | r1↔r2 runkolinkki | r1=.1, r2=.2 |
  | 10.255.23.0/30 | r2↔r3 runkolinkki | r2=.1, r3=.2 |
  | 172.20.20.0/24 | Containerlab-hallintaverkko (Docker), erillinen loogisesta verkosta | — |

  ## 5. Reitityksen analyysi

  Reitityksen toimivuutta testatsin ping- ja traceroute-komennoilla eri verkkosegmenttien välillä sekä tarkastelemalla laitteiden reittitauluja. Tein testit kahdesta pisteestä: toimivaksi oletetusta client1-kontista (verkossa 10.10.10.0/24) ja attacker-kontista (samassa verkossa).

Ping-testit

Client1:stä (10.10.10.101) testattiin yhteyttä r2:n verkkoon (10.10.20.101) ja r3:n verkkoon (10.10.30.101). Molemmat testit onnistuivat täydellisesti, 0 % pakettihäviöllä:

64 bytes from 10.10.20.101: icmp_seq=1 ttl=62 time=1.72 ms
--- 10.10.20.101 ping statistics --- 4 packets transmitted, 4 received, 0% packet loss

64 bytes from 10.10.30.101: icmp_seq=1 ttl=61 time=0.882 ms
--- 10.10.30.101 ping statistics --- 4 packets transmitted, 4 received, 0% packet loss

Time to live-arvot (62 ja 61) osoittavat  hyppymäärän kasvavan kauemmas mentäessä, mikä kertoo reitityksen toimivan useamman reitittimen yli suunnitellusti.

Attacker-kontista (10.10.10.200) sama testi epäonnistui kokonaan — sekä oma oletusyhdyskäytävä (10.10.10.1) että etäverkko (10.10.20.101) tuottivat 100 % pakettihäviön:

PING 10.10.10.1: 4 packets transmitted, 0 received, 100% packet loss
PING 10.10.20.101: 4 packets transmitted, 0 received, 100% packet loss

Koska attacker ei pääse edes omaan yhdyskäytäväänsä, on odotettua ettei mikään muukaan verkko ole tavoitettavissa.

Traceroute

Client1:stä r3:n verkkoon suoritettu traceroute vahvisti reitin kulkevan suunnitellusti r1 → r2 → r3 -ketjun kautta:

1  10.10.10.1       5.614 ms
2  10.255.12.2      0.249 ms
3  10.255.23.2      0.966 ms
4  10.10.30.101      1.161 ms

Kaikki hypyt vastasivat nopeasti  ilman pakettihäviötä.

Reittitauluanalyysi

Sekä client1:n että attackerin reittitaulut osoittautuivat rakenteeltaan identtisiksi.

  ## 6. Yhteenveto

  Eniten aikaa kului uuteen asiaan tutustustumiseen ja termeihin ja työkaluihin perehtymiseen ja haltuun ottoon.  Paljon aikaa vei myös yksittäisten virhetilanteiden juurisyiden jäljittämiseen, esimerkiksi attacker-kontin kanssa tuli joku yhdyskäytäväongelma ja sen  syyn löytäminen vaati tarkempaa perehtymistä kontin konfiguraatioon ja topologiatiedostoon. Samoin hallintaverkon osoitteen, tai paremminkin osoitteen puuttuminen, aiheutti pulmia ja vaati pohdintaa. Tehtävään kului minulta enemmän aikaa kuin arvioitu 4-8 tuntia ja se johtui suurelta osin siitä, että en ole aiemmin tutustunut verkon dokumentaatioihin eikä minulla siten ollut mielikuvaa, mitä olen tekemässä ja mikä on tavoitteena lopputuloksen osalta.
  
  Kattava, selkeä ja juurisyihin asti viety dokumentaatio säästää varmasti merkittävästi aikaa jatkossa. Kun havainnot on kirjattu konkreettisin komennoin ja tulostein (ping, traceroute, reittitaulut), seuraava ylläpitäjä ei joudu aloittamaan selvitystyötä tyhjästä, vaan näkee suoraan mitä on testattu, mitä löytyi ja miksi. Hyvä dokumentaatio  toimii  karttana, joka ongelman ilmentyessä ohjaa nopeasti sinne, missä ongelman juurisyy on, sen sijaan että virhettä jouduttaisiin etsimään uudelleen koko verkon laajuudelta. Hyvä dokumentaatio myös helpottaa uusien työntekijöiden perehdytystä ja nopeuttaa vikatilanteiden ratkaisua tuotantoympäristöissä.