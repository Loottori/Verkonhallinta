  # Viikko 1 – Verkon dokumentointi

  ## 1. Johdanto

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

  ## 5. Reitityksen analyysi

  ## 6. Yhteenveto