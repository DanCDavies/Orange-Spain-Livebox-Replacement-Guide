# Substituir el Livebox 6 d'Orange Spain per un UFiber Loco i un Unifi USG-3P

## Per què substituir el Livebox?

El Livebox 6 d'Orange Spain és un router tancat i controlat per l'ISP. No pots configurar VLANs, establir regles de firewall personalitzades, executar bloqueig de publicitat a nivell de xarxa, ni gestionar la teva xarxa com a sistema unificat. Substituir-lo et dona control total sobre la teva xarxa amb regles de firewall adequades, segmentació per VLANs, DNS personalitzat i un únic pla de gestió per a tota la xarxa.

En el moment d'escriure això, l'USG-3P és End-of-Life. Tot i així, continua sent assequible i fàcilment accessible a Wallapop i altres revenedors de hardware de segona mà. Especialment combinat amb un controller auto-allotjat en una Raspberry Pi o un altre mini PC, és un mètode de baix cost i baixa barrera d'entrada per substituir l'equipament existent.

**Important:** Aquesta guia es va escriure utilitzant un Unifi Controller amb la versió 6.0.28, degut a l'ús de hardware EoL a la xarxa de l'autor. Si esteu executant una versió més actualitzada, haureu de (a) assegurar-vos que el vostre controller suporti el Gateway que voleu utilitzar, i (b) comprovar les ubicacions equivalents dels paràmetres en versions més noves del controller, ja que aquesta guia fa referència a ubicacions de la versió 6.0.28.

### Per què no utilitzar el mode ONT?

El Livebox d'Orange té un mode 'Modem Only', que en teoria permet tractar-lo com un simple ONT i utilitzar un router comercial o opnSense per al routing real (inclòs l'USG-3P), saltant-se completament el Nano Fiber Loco / Nano G.

Segons la meva experiència, això no va funcionar sota cap circumstància ni combinació de hardware. El vostre cas pot ser diferent, i si ja teniu un router existent, potser voleu provar-ho primer, ja que aquesta guia és bastant extensa.

## Requisits

Com a mínim, necessitareu el següent (o els seus equivalents) per seguir aquesta guia:

| Element                                                      | Propòsit                                                  | Notes                                                        |
| ------------------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| **Ubiquiti UFiber Loco** (UF-LOCO) o **UFiber Nano G** (UF-NANO) | GPON ONT — substitueix la funció de mòdem de fibra del Livebox | El Loco és més barat i suficient. El Nano G té WiFi integrat. Tots dos haurien de funcionar de manera idèntica per a aquest propòsit. |
| **Ubiquiti USG-3P** (UniFi Security Gateway)                 | Router/firewall — substitueix la funció de routing del Livebox | EOL des de novembre de 2024, però encara funciona amb versions del controller fins a 8.x. Vegeu [Notes sobre fi de vida](#notes-sobre-fi-de-vida). |
| **UniFi Controller**                                         | Software per gestionar l'USG (i opcionalment APs/switches) | Es recomana auto-allotjat via Docker. Vegeu [Configuració del Controller](#configuracio-del-controller). |
| **Un ordinador amb Python 3 i SSH**                          | Per a la clonació del número de sèrie i la configuració   | Windows, macOS o Linux.                                      |
| **Un cable ethernet**                                        | Connexió directa al Loco durant la configuració           | Haureu de desconnectar-vos de la vostra xarxa temporalment.  |
| **Les vostres credencials del contracte d'Orange Spain**     | Contrasenya PLOAM i número de sèrie del Livebox           | Extretes al [Pas 1](#pas-1-extreure-les-credencials-del-livebox). |

### Opcional però molt recomanat

- **Un switch gigabit no gestionat barat** — útil si necessiteu mantenir la connectivitat local entre dispositius durant la migració.
- **Un telèfon amb dades mòbils** — útil per crear un hotspot mòbil o consultar referències durant les etapes on no hi ha internet disponible.

## Visió general de l'arquitectura

```
┌─────────────────┐     SC/APC fibre      ┌──────────────┐
│  Orange Spain   │◄────────────────────► │  UFiber Loco │
│  Rosetta.       │     GPON              │  (ONT/modem) │
│                 │                       └──────┬───────┘
└─────────────────┘                          Ethernet
                                                │
                                          ┌─────┴──────┐
                                          │  USG-3P    │
                                          │  WAN: eth0 │◄── VLAN 832 (DHCP)
                                          │  LAN: eth1 │
                                          └─────┬──────┘
                                                │
                                          ┌─────┴──────┐
                                          │  Switch    │
                                          │  + APs     │
                                          │  + Devices │
                                          └────────────┘
```

El Loco gestiona l'autenticació GPON amb l'OLT d'Orange i passa ethernet sense processar a l'USG. L'USG etiqueta el tràfic a la VLAN 832, obté una IP pública via DHCP i encamina la vostra LAN. El controller UniFi gestiona la configuració de l'USG.

## Crític: Orange Spain ≠ Orange France

Moltes guies en línia (especialment en francès) descriuen com substituir routers d'Orange utilitzant **DHCP Option 90** per a l'autenticació, **vendor-class-identifier "sagem"**, **cadenes user-class** i **PPPoE**. **Res d'això s'aplica a Orange Spain.**

Orange Spain utilitza (en el moment d'escriure aquesta guia, maig de 2026):

- **VLAN 832** amb **DHCP** estàndard (sense PPPoE)
- **Sense Option 90**
- **Sense vendor-class-identifier**
- **Sense user-class**
- **Sense cadenes d'autenticació RFC3118**

Si veieu qualsevol guia que us digui de generar una cadena hexadecimal a partir de credencials `fti/xxxxxxxxx` o establir vendor-class a "sagem", aquesta guia és per a **Orange France** i **impedirà activament** que la vostra connexió funcioni a Espanya.

L'única autenticació que importa és a la **capa GPON**: la contrasenya PLOAM i el número de sèrie de l'ONT.

------

## Pas 1: Extreure les credencials del Livebox

Abans de desconnectar res, inicieu sessió al vostre Livebox a `http://192.168.1.1` i anoteu aquests valors:

### 1.1 Contrasenya PLOAM (contrasenya de l'ONT)

Navegueu a la secció d'informació/GPON del panell d'administració del Livebox. Busqueu la **contrasenya PLOAM** o **contrasenya de l'ONT**. Serà una cadena alfanumèrica de 10 caràcters (p. ex., `A5XK92WR7D`).

### 1.2 Número de sèrie

Trobeu el **número de sèrie GPON**. Tindrà un aspecte com `SMBS4A8F7C12` — un identificador de fabricant de 4 caràcters seguit de 8 caràcters hexadecimals. Anoteu:

- **Sèrie complet**: p. ex., `SMBS4A8F7C12`
- **Identificador de fabricant**: els primers 4 caràcters (p. ex., `SMBS`)
- **Sèrie del dispositiu**: els 8 caràcters restants (p. ex., `4A8F7C12`)

### 1.3 Adreça MAC

Trobeu l'**adreça MAC WAN** del Livebox (no la MAC del WiFi ni la MAC de la LAN). Format: `XX:XX:XX:XX:XX:XX`.

### 1.4 Taula resum

Deseu-ho en un lloc accessible sense connexió — ho necessitareu quan estigueu desconnectats d'internet:

| Paràmetre                   | El vostre valor | Exemple             |
| --------------------------- | --------------- | ------------------- |
| Contrasenya PLOAM           |                 | `A5XK92WR7D`        |
| Sèrie complet               |                 | `SMBS4A8F7C12`      |
| Identificador de fabricant  |                 | `SMBS`              |
| Sèrie del dispositiu        |                 | `4A8F7C12`          |
| Adreça MAC WAN              |                 | `DC:15:C8:1A:E3:4B` |
| IP per defecte del Loco     | `192.168.1.1`   | —                   |
| Credencials per defecte del Loco | `ubnt` / `ubnt` | —              |

------

## Pas 2: Preparar el UFiber Loco

### 2.1 Accés inicial

1. Desconnecteu el vostre PC de la vostra xarxa.
2. Connecteu el port ethernet del vostre PC directament al **port LAN** del Loco.
3. Alimenteu el Loco mitjançant el seu adaptador PoE de 24V o una font d'alimentació micro-USB de **5V/1A o superior**. No confieu en un port USB 2.0 del PC — pot no subministrar prou corrent (el Loco consumeix fins a 3,5W = 700mA a 5V).
4. Configureu el vostre PC amb una IP estàtica: `192.168.1.100`, màscara de subxarxa `255.255.255.0`, gateway `192.168.1.1`.
5. Navegueu a `https://192.168.1.1` (nota: **HTTPS**, no HTTP — el Loco redirigeix a HTTPS i alguns navegadors rebutgen la redirecció silenciosament). Accepteu l'avís del certificat.
6. Inicieu sessió: `ubnt` / `ubnt`.

> **Error comú:** Si el navegador mostra "unable to connect", proveu tant `http://` com `https://`. Si cap dels dos funciona, verifiqueu que la vostra IP estàtica és realment `192.168.1.100` i no, p. ex., `192.168.100.1` (un error de transposició fàcil a Windows). Executeu `ipconfig` per confirmar-ho.

> **Usuaris d'Hyper-V / WSL:** Si teniu Hyper-V o WSL habilitat a Windows, un adaptador de xarxa virtual pot interceptar el tràfic del vostre port ethernet físic. Desactiveu l'adaptador vEthernet temporalment o utilitzeu una màquina diferent.

### 2.2 Actualitzar el firmware

Aneu a **System** → **Upload Firmware** i actualitzeu a la darrera versió. En el moment d'escriure, el firmware **4.4.6** funciona. El dispositiu es reiniciarà (espereu 3-5 minuts). Hi ha còpies del firmware d'Ui.com disponibles a la carpeta \Firmware d'aquest repositori. Tot el firmware és sense modificar i és propietat de Ubiquiti.

### 2.3 Configurar els paràmetres GPON

1. Aneu a la pestanya **GPON**.
2. Establiu **OLT Profile**: proveu primer el **Profile 3** (FiberHome). Vegeu [Selecció de perfil](#seleccio-de-perfil) per a més detalls.
3. Establiu la **contrasenya PLOAM**: la vostra contrasenya del Pas 1 (p. ex., `A5XK92WR7D`).
4. **Format hexadecimal**: **DESACTIVAT** (desmarcat).
5. Deixeu els camps d'autenticació LOID als valors per defecte (`ubnt` / `ubntubnt`) — Orange Spain no utilitza LOID.
6. Feu clic a **Save**.

### 2.4 Canviar la IP de gestió

**Això és crític.** L'USG també utilitzarà `192.168.1.1` com a IP de la LAN. El Loco ha d'estar en una adreça diferent.

1. Aneu a la pestanya **Network**.
2. Canvieu la IP del dispositiu a alguna cosa fora del vostre rang DHCP, p. ex., `192.168.1.15` o `192.168.100.1`.
3. Deseu. Haureu de reconnectar-vos a la nova IP.

### 2.5 Habilitar SSH

Aneu a **System** i assegureu-vos que **SSH** està habilitat al port 22. Ho necessitareu per a la clonació del número de sèrie.

------

## Pas 3: Clonar el número de sèrie (Crític)

Aquest és el pas que la majoria de guies no mencionen, i és la causa més comuna de fallada d'autenticació GPON. El Loco ve de fàbrica amb l'identificador de fabricant d'Ubiquiti (`UBNT`) i un número de sèrie aleatori. L'OLT d'Orange espera veure el número de sèrie i l'identificador de fabricant del vostre Livebox.

### 3.1 Instal·lar dependències

Al vostre PC (mentre encara teniu internet, o descarregat amb antelació):

```bash
pip install paramiko scp
```

### 3.2 Descarregar l'eina de modificació del número de sèrie

Descarregueu des de: https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack

Cloneu el repositori o descarregueu el ZIP i extreu-lo.

### 3.3 Convertir el vostre número de sèrie a format hexadecimal

El número de sèrie GPON és de 8 bytes: 4 bytes per a l'identificador de fabricant (ASCII) + 4 bytes per al sèrie del dispositiu (hex).

Per exemple, `SMBS4A8F7C12`:

- `SMBS` → hexadecimal ASCII: `53:4D:42:53`
- `4A8F7C12` → bytes hexadecimals: `4A:8F:7C:12`
- **Paràmetre de sèrie complet**: `53:4D:42:53:4A:8F:7C:12`

Per convertir el vostre identificador de fabricant a hexadecimal, utilitzeu una taula ASCII o una eina en línia:

| Char | Hex  | Char | Hex  | Char | Hex  | Char | Hex  |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| A    | 41   | H    | 48   | O    | 4F   | V    | 56   |
| B    | 42   | I    | 49   | P    | 50   | W    | 57   |
| C    | 43   | J    | 4A   | Q    | 51   | X    | 58   |
| D    | 44   | K    | 4B   | R    | 52   | Y    | 59   |
| E    | 45   | L    | 4C   | S    | 53   | Z    | 5A   |
| F    | 46   | M    | 4D   | T    | 54   |      |      |
| G    | 47   | N    | 4E   | U    | 55   |      |      |

La part del sèrie del dispositiu (els 8 caràcters després de l'identificador de fabricant) s'ha de tractar com a **bytes hexadecimals** — cada parell de caràcters és un byte.

### 3.4 Verificar abans d'escriure (només lectura)

Connecteu el vostre PC al Loco via ethernet (IP estàtica a la mateixa subxarxa), després (utilitzant la IP que hàgiu configurat al Pas 2.4):

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Contrasenya: `ubnt` (o la que hàgiu configurat).

Haureu de veure una sortida com:

```
[~] vendor_id:        "UBNT"
[~] serial_id:        "a9e0dd3c"
[~] base_mac_addr:    ac:8b:a9:e0:dd:3c
[~] gpon_password:    "A5XK92WR7D"
```

Confirmeu que la contrasenya GPON coincideix. L'identificador de fabricant i el número de sèrie són els valors per defecte d'Ubiquiti — els canviarem ara.

### 3.5 Escriure el número de sèrie clonat

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 \
  --serial 53:4D:42:53:4A:8F:7C:12 \
  --mac DC:15:C8:1A:E3:4B \
  --insecure
```

Substituïu el sèrie i la MAC pels **vostres** valors del Pas 1.

> **El flag `--insecure` és obligatori per al UFiber Loco.** L'script es queixarà de diferències en el checksum del firmware als dispositius Loco sense aquest flag. Això és normal i segur.

### 3.6 Verificar l'escriptura

Executeu la comanda de només lectura un altre cop i confirmeu que tots els valors coincideixen amb els del vostre Livebox:

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Sortida esperada:

```
[~] vendor_id:        "SMBS"
[~] serial_id:        "4A8F7C12"
[~] base_mac_addr:    dc:15:c8:1a:e3:4b
[~] gpon_password:    "A5XK92WR7D"
```

------

## Pas 4: Provar l'autenticació GPON

1. Desconnecteu el cable ethernet del Loco.
2. Desconnecteu el cable de fibra SC/APC del Livebox. Manipuleu-lo amb cura — no toqueu la fèrrula.
3. Connecteu la fibra al **port GPON** del Loco.
4. Observeu els LEDs. Espereu fins a 60 segons.

### Indicadors LED

- **Barres verdes** = Autenticació GPON satisfactòria
- **Parpelleig taronja/ambre** = Intent de registre
- **Barres blanques/blaves fixes** = Encesa, sense autenticació GPON

> **Important:** Els LEDs poden ser enganyosos en algunes versions de firmware. El Loco pot mostrar només barres blanques fins i tot quan l'autenticació GPON ha tingut èxit. **El dashboard és l'autoritat.** Per comprovar-ho correctament: reconnecteu l'ethernet al Loco, navegueu a `https://192.168.1.15` i mireu el dashboard. Si mostra **STATE: CONNECTED** amb una potència RX al voltant de **-15 a -20 dBm**, l'autenticació ha tingut èxit independentment del que mostrin els LEDs. El meu propi Fiber Loco, per exemple, no mostra mai barres verdes, tot i funcionar perfectament.

### Selecció de perfil

Si el Profile 3 no s'autentica, proveu els altres perfils:

| Perfil    | Fabricant de l'OLT | Notes                                                        |
| --------- | ------------------- | ------------------------------------------------------------ |
| Profile 1 | Ubiquiti OLT        | **Salteu-vos aquest** — és per a l'equipament OLT propi d'Ubiquiti. No exposa PLOAM ni paràmetres d'autenticació de tercers. |
| Profile 2 | Huawei              | Comú per a Orange Spain                                      |
| Profile 3 | FiberHome           | Comú per a Orange Spain — proveu aquest primer               |
| Profile 4 | ZTE                 | Menys comú però funciona en algunes zones                    |

Per canviar de perfil: aneu a `https://192.168.1.15` → pestanya GPON → canvieu el desplegable OLT Profile → Save. El dispositiu es reinicia. Reconnecteu la fibra i comproveu de nou.

Segons la nostra experiència, **els Profiles 2, 3 i 4 van funcionar tots** — l'OLT era flexible. El vostre cas pot variar depenent de l'equipament de la vostra central. Si cap dels Profiles 2/3/4 produeix un estat CONNECTED, el vostre OLT pot ser Alcatel/Nokia, contra el qual els dispositius UFiber no es poden autenticar. En aquest cas, necessitaríeu un ONT diferent (p. ex., un Huawei HG8010H amb clonació de número de sèrie).

### Marxa enrere

Si res no funciona: desconnecteu la fibra del Loco, reconnecteu-la al Livebox i enceneu el Livebox. Tornareu a la normalitat en 2 minuts. No s'ha canviat res al costat d'Orange.

------

## Pas 5: Configurar l'USG-3P

### 5.1 Cablejat físic

```
Fibre → Loco GPON port → Loco LAN port → USG WAN (eth0)
                                           USG LAN (eth1) → Switch → Devices
```

### 5.2 El fitxer config.gateway.json

L'USG necessita crear una subinterfície VLAN 832 al seu port WAN i sol·licitar una concessió DHCP a través d'ella. També necessita una regla NAT masquerade en aquella interfície VLAN (segons la meva experiència, no a `eth0` directament), i — de manera crítica — correccions de firewall i routing que el controller UniFi no aplica correctament per defecte.

Creeu aquest fitxer al vostre controller UniFi:

```json
{
    "firewall": {
        "source-validation": "loose"
    },
    "interfaces": {
        "ethernet": {
            "eth0": {
                "vif": {
                    "832": {
                        "address": ["dhcp"],
                        "description": "Orange Spain Internet",
                        "dhcp-options": {
                            "default-route": "update",
                            "default-route-distance": "1",
                            "name-server": "update"
                        },
                        "firewall": {
                            "in": {
                                "name": "WAN_IN"
                            },
                            "local": {
                                "name": "WAN_LOCAL"
                            }
                        }
                    }
                }
            }
        }
    },
    "service": {
        "nat": {
            "rule": {
                "5010": {
                    "outbound-interface": "eth0.832",
                    "type": "masquerade"
                }
            }
        }
    }
}
```

Aquest fitxer fa quatre coses:

1. **VLAN 832 amb DHCP** — crea la subinterfície `eth0.832` i sol·licita una IP pública a través d'ella.
2. **NAT masquerade a `eth0.832`** — sense això, l'USG obté una IP pública però el tràfic de la vostra LAN no passa per NAT. Les regles masquerade per defecte de l'USG apunten a `eth0`, no a `eth0.832`.
3. **Regles de firewall a `eth0.832`** — aquesta és la part crítica que la majoria de guies no mencionen. Vegeu [El forat de firewall de la VLAN 832](#el-forat-de-firewall-de-la-vlan-832) a continuació.
4. **Validació d'origen establerta a "loose"** — corregeix el filtratge de camí invers que fa que tràfic de retorn legítim sigui descartat com a "martian". Vegeu [Filtratge de camí invers](#filtratge-de-cami-invers) a continuació.

### El forat de firewall de la VLAN 832

Quan el controller UniFi crea regles de firewall WAN_LOCAL i WAN_IN (ja siguin les per defecte o les que afegiu a la UI), les aplica a `eth0` — la interfície WAN física. Però la vostra interfície real cara a internet és `eth0.832`, la subinterfície VLAN. **El controller no aplica automàticament les regles de firewall a les subinterfícies VLAN.** Això significa que sense el bloc `firewall` a la configuració anterior, el port WAN del vostre USG està efectivament sense filtrar: cada escaneig de ports, intent de força bruta SSH i sonda des d'internet arriba directament als serveis de l'USG.

En un dispositiu amb una CPU Cavium Octeon MIPS64 i 512MB de RAM, aquest bombardeig constant (que és normal — qualsevol IP pública és escanejada contínuament per bots automatitzats a tot el món) pot fer que l'USG deixi de respondre en unes hores. Els símptomes són pèrdues intermitents d'internet que requereixen un reinici elèctric per solucionar-se, sense cap error clar als logs.

Les entrades `firewall.in` i `firewall.local` a la configuració anterior indiquen a EdgeOS que apliqui les cadenes WAN_IN i WAN_LOCAL a `eth0.832`, de manera que tot el tràfic que arriba per la VLAN d'Orange està subjecte a les mateixes regles de firewall que el tràfic a la interfície física. Això és essencial per a l'estabilitat.

### Filtratge de camí invers

El paràmetre `"source-validation": "loose"` canvia el filtre de camí invers del kernel (`rp_filter`) del mode estricte (1) al mode relaxat (2). En mode estricte, el kernel descarta qualsevol paquet l'adreça d'origen del qual no seria encaminada de tornada per la mateixa interfície per la qual va arribar. Amb la VLAN 832, el tràfic de retorn legítim d'internet pot arribar per `eth0.832` però la taula de routing del kernel pot dir que l'origen hauria de ser accessible via `eth0` — la interfície pare. El mode estricte descarta això com a paquet "martian" i ho registra.

El símptoma és connectivitat intermitent: una part del tràfic funciona, una altra és descartada silenciosament, i els logs s'omplen de línies com:

```
IPv4: martian source 192.168.1.x from <external-ip>, on dev eth0.832
```

El mode relaxat continua validant que l'adreça d'origen és accessible via *alguna* interfície (no està desactivat), però no requereix que sigui la mateixa per la qual va arribar el paquet. Aquest és el paràmetre correcte per a qualsevol configuració WAN basada en VLAN.

### 5.3 On col·locar el fitxer

**Si el vostre controller és basat en Docker (auto-allotjat):**

Trobeu el punt de muntatge del volum de dades UniFi. Camins habituals:

```bash
# Check your docker-compose.yml for the volume mapping
# If: volumes: - ./config:/config
# Then the file goes at:
/opt/docker/unifi/config/data/sites/default/config.gateway.json
```

Creeu el directori `sites/default/` si no existeix:

```bash
mkdir -p /path/to/unifi/config/data/sites/default/
```

> **El nom del lloc "default"** correspon a l'identificador intern del lloc, no al nom que es mostra. Comproveu l'URL del vostre controller — mostra `manage/site/XXXXX/` on `XXXXX` és l'identificador del lloc. Utilitzeu aquest valor en lloc de `default` si és diferent.

**Si el vostre controller s'executa de forma nativa (sense Docker):**

```bash
/usr/lib/unifi/data/sites/default/config.gateway.json
```

### 5.4 Prova ràpida via SSH (Opcional, abans de l'adopció pel controller)

Si voleu provar internet abans de gestionar l'adopció pel controller, podeu configurar l'USG directament via SSH:

```bash
ssh ubnt@192.168.1.1
# Password: ubnt (factory default)

configure
set interfaces ethernet eth0 vif 832 address dhcp
set interfaces ethernet eth0 vif 832 description "Orange Spain Internet"
set interfaces ethernet eth0 vif 832 dhcp-options default-route update
set interfaces ethernet eth0 vif 832 dhcp-options name-server update
set interfaces ethernet eth0 vif 832 firewall in name WAN_IN
set interfaces ethernet eth0 vif 832 firewall local name WAN_LOCAL
set firewall source-validation loose
set service nat rule 5010 outbound-interface eth0.832
set service nat rule 5010 type masquerade
commit
save
```

Comproveu si teniu una IP pública:

```bash
show interfaces ethernet eth0 vif 832
```

Si veieu una IP pública i `ping 8.8.8.8` funciona des del vostre escriptori — el Livebox ha estat substituït amb èxit.

> **La configuració via SSH es sobreescriu quan el controller aprovisiona l'USG.** Això està bé per a proves, però necessiteu el `config.gateway.json` al seu lloc abans de l'adopció pel controller per fer que la configuració sigui persistent.

------

## Pas 6: Adopció pel controller

### 6.1 Xarxes Docker — Mode host obligatori

**Aquesta és la causa més comuna de fallada d'adopció quan s'executa el controller UniFi a Docker.**

El controller UniFi necessita connectar-se via SSH *a* l'USG per aprovisionar-lo. Si el contenidor del controller està en una xarxa bridge de Docker (la configuració per defecte), no pot encaminar cap a `192.168.1.1`. El procés d'inform (USG → controller) funciona perquè Docker mapeja el port, però l'aprovisionament (controller → USG via SSH) falla.

**Símptoma:** L'USG apareix al controller, feu clic a Adopt, entra en un bucle d'"Adopting" i mai es completa, o mostra "Disconnected" repetidament.

**Solució:** Canvieu el contenidor del controller a mode host.

Abans (mode bridge — **no funciona per a l'adopció de l'USG**):

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:version-6.0.28
    container_name: unifi-controller
    restart: unless-stopped
    ports:
      - "3478:3478/udp"
      - "10001:10001/udp"
      - "8080:8080"
      - "8443:8443"
      - "8880:8880"
      - "6789:6789"
      - "5514:5514/udp"
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

Després (mode host — **funciona**):

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:version-6.0.28
    container_name: unifi-controller
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

Elimineu tot el bloc `ports` — amb mode host, el contenidor es vincula directament als ports de la màquina amfitriona. Verifiqueu que cap altre servei de la màquina amfitriona entri en conflicte amb els ports d'UniFi (8080, 8443, 3478/udp, 10001/udp, 8880, 6789, 5514/udp).

**Important:** La resta dels paràmetres yaml són exemples del meu propi desplegament per proporcionar context. Haurieu d'utilitzar els valors correctes per al vostre propi docker-compose.yml. Si no sabeu com configurar-los, us recomano llegir sobre Docker abans d'intentar aquesta migració (https://docs.docker.com/get-started/). També podeu considerar utilitzar el Unifi Controller directament sobre el sistema operatiu en lloc de Docker, però recordeu establir la versió a la que necessiteu per donar suport a la vostra pròpia configuració de hardware.

Després:

```bash
docker compose down
docker compose up -d
```

### 6.2 Procés d'adopció

1. Assegureu-vos que el `config.gateway.json` està al seu lloc a `sites/default/`.

2. Obriu i inicieu sessió a la UI del controller a `https://<controller-ip>:8443`.

3. L'USG hauria d'aparèixer com a "Pending Adoption".

4. Feu clic a **Adopt**.

5. Si no apareix, connecteu-vos via SSH a l'USG i forceu l'inform:

   ```bash
   ssh ubnt@192.168.1.1
   set-inform http://<controller-ip>:8080/inform
   ```

   Pot ser que hàgiu d'executar `set-inform` dues vegades — un cop per presentar el dispositiu, i un altre després de fer clic a Adopt.

### 6.3 Resolució de problemes d'adopció

**L'USG queda atrapat en un bucle d'adopció:**

Si l'USG cicla repetidament entre "Adopting" i "Disconnected":

1. Comproveu la configuració de xarxa Docker (vegeu 6.1 més amunt).

2. Elimineu el dispositiu de la base de dades del controller per força:

   ```bash
   docker exec -it unifi-controller mongo --port 27117
   ```

   ```javascript
   use ace
   db.device.remove({"model": "UGW3"})
   ```

   ```bash
   docker restart unifi-controller
   ```

3. Restaureu l'USG als valors de fàbrica (manteniu premut el botó de reset 10 segons) i torneu a adoptar.

**La configuració no s'aplica després de l'adopció:**

Si l'USG s'adopta però no té la VLAN 832:

1. Verifiqueu que el fitxer és visible dins del contenidor:

   ```bash
   docker exec -it unifi-controller cat /usr/lib/unifi/data/sites/default/config.gateway.json
   ```

2. Forceu el re-aprovisionament restablint la versió de configuració:

   ```bash
   docker exec -it unifi-controller mongo --port 27117 --eval \
     'db.getSiblingDB("ace").device.update({"model":"UGW3"},{$set:{"cfgversion":"0"}})'
   ```

3. Reinicieu el controller: `docker restart unifi-controller`

------

## Pas 7: Verificació

Un cop tot funcioni:

```bash
# From the USG SSH:
show interfaces ethernet eth0 vif 832
# Should show a public IP address

# From any LAN device:
ping 8.8.8.8
ping google.com
```

### Verificar les correccions de firewall i rp_filter

Aquestes són tan importants com la comprovació bàsica de connectivitat. Després que el controller aprovisioni l'USG, connecteu-vos via SSH i confirmeu:

```bash
# Check rp_filter is set to loose (should return: 2)
cat /proc/sys/net/ipv4/conf/eth0.832/rp_filter

# Check firewall is applied to eth0.832 (should show eth0.832 → WAN_LOCAL)
sudo iptables -L VYATTA_FW_LOCAL_HOOK -nv --line-numbers
```

A la sortida de `VYATTA_FW_LOCAL_HOOK`, haureu de veure una línia que mapeja `eth0.832` a `WAN_LOCAL`. Si no hi és, la secció de firewall del vostre `config.gateway.json` no s'està aplicant — comproveu el camí del fitxer i forceu el re-aprovisionament.

### Llista de comprovació post-migració suggerida

- [ ] El dashboard GPON del Loco mostra **CONNECTED** amb bona potència RX (-15 a -20 dBm)
- [ ] L'USG té una IP pública a `eth0.832`
- [ ] Els dispositius de la LAN poden accedir a internet
- [ ] La resolució DNS funciona (`ping google.com`)
- [ ] El controller mostra l'USG com a "Connected" i gestionat
- [ ] El `config.gateway.json` està al seu lloc (la configuració sobreviu al re-aprovisionament)
- [ ] El `rp_filter` a `eth0.832` està establert a `2` (loose)
- [ ] `VYATTA_FW_LOCAL_HOOK` mostra `eth0.832 → WAN_LOCAL`
- [ ] Els punts d'accés s'han tornat a adoptar i emeten (si escau)

Si tot això funciona, felicitats! Ara podeu posar el vostre Livebox d'Orange al fons d'un armari i oblidar-vos-en fins que canvieu d'ISP i us el demanin de tornada.

Gaudiu de la vostra nova capacitat per encaminar VLANs correctament i configurar el vostre propi DNS.

------

## Notes sobre fi de vida

L'**USG-3P** va ser declarat End of Life per Ubiquiti al **novembre de 2024**:

- Les versions del controller UniFi Network **9.x i posteriors no poden adoptar dispositius USG en absolut**.
- L'USG-3P funciona amb versions del controller fins a **8.x**.
- Si auto-allotgeu el vostre controller (Docker), podeu fixar la versió i controlar el vostre propi cicle d'actualització.

Si ja teniu hardware Ubiquiti / Unifi més nou, considereu si l'USG-3P encara val la pena per al vostre cas d'ús. Si ja en teniu un, continuarà funcionant indefinidament mentre controleu la versió del vostre controller.

------

## Configuració del controller

Si encara no teniu un controller UniFi, l'enfocament més senzill per a l'auto-allotjament és Docker. Un `docker-compose.yml` mínim:

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:latest
    container_name: unifi-controller
    restart: unless-stopped
    network_mode: host  # Required for USG adoption
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

Executeu amb `docker compose up -d`. Accediu a la UI a `https://<host-ip>:8443`.

> **Fixeu la versió del controller** si teniu un USG-3P. Utilitzeu una etiqueta específica com `version-8.6.9` en lloc de `latest` per evitar actualitzacions accidentals que eliminin el suport per a l'USG.

------

## Referència de resolució de problemes

### El GPON no s'autentica

| Símptoma                                     | Causa probable              | Solució                                                      |
| -------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| Barres blanques, sense CONNECTED al dashboard | Número de sèrie no clonat   | Executeu l'eina de sèrie (Pas 3)                             |
| Barres blanques després de la clonació        | Perfil OLT incorrecte       | Proveu els Profiles 2, 3 i 4                                 |
| Sense potència RX al dashboard                | Fibra mal connectada        | Torneu a col·locar el connector SC/APC, comproveu si hi ha pols |
| Tots els perfils fallen                       | OLT Alcatel/Nokia           | UFiber no es pot autenticar contra Alcatel. Cal un ONT diferent. |

### L'USG no obté internet

| Símptoma                                       | Causa probable                                     | Solució                                            |
| ---------------------------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| Sense IP a eth0.832                            | VLAN 832 no configurada                            | Comproveu `config.gateway.json` o la configuració SSH |
| IP pública a eth0.832 però sense internet a la LAN | Falta NAT masquerade a `eth0.832`              | Afegiu la regla 5010 a config.gateway.json         |
| L'USG mostra regles per defecte només a `eth0` | Controller aprovisionat sense config.gateway.json  | Assegureu-vos que el fitxer és al camí correcte, forceu el re-aprovisionament |

### Inestabilitat de l'USG (fallades, talls d'internet que requereixen reinici elèctric)

| Símptoma                                                     | Causa probable                                     | Solució                                                      |
| ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| Internet cau després d'unes hores, reinici elèctric ho soluciona | Firewall no aplicat a `eth0.832`                  | Afegiu el bloc `firewall` a la configuració de la VLAN 832. Vegeu [El forat de firewall de la VLAN 832](#el-forat-de-firewall-de-la-vlan-832). |
| Els logs mostren força bruta SSH des d'IPs externes          | WAN_LOCAL no filtra `eth0.832`                     | Mateixa solució — el bloc de firewall aplica WAN_LOCAL a la interfície VLAN. |
| Els logs mostren "martian source" a `eth0.832`               | Filtratge estricte de camí invers                  | Afegiu `"source-validation": "loose"` a la secció firewall de config.gateway.json. Vegeu [Filtratge de camí invers](#filtratge-de-cami-invers). |
| Bloquejos aleatoris amb temps càlid, sense evidència als logs | Apagada tèrmica                                    | Vegeu [Consideracions tèrmiques](#consideracions-termiques) a continuació. |

### Consideracions tèrmiques

L'USG-3P té refrigeració passiva amb un petit dissipador d'alumini. És propens al bloqueig tèrmic, especialment en climes càlids o espais tancats. No sembla que hi hagi cap limitació gradual que jo hagi pogut determinar durant les meves proves. Quan el SoC es sobreescalfa, el dispositiu es bloqueja silenciosament i requereix un reinici elèctric. No hi ha cap avís als logs perquè la fallada és instantània.

Els factors que hi contribueixen inclouen DPI (Deep Packet Inspection), IDS/IPS i una alta rotació de la taula de connexions — tots els quals augmenten significativament la càrrega de la CPU i la generació de calor al SoC MIPS64. DPI i IDS/IPS no haurien d'estar habilitats a l'USG-3P en cap cas degut a la seva potència de processament limitada, però també empitjoren la situació tèrmica. Si necessiteu aquest tipus de capacitat, haureu de considerar si l'USG-3P és adequat per a vosaltres — un router opnSense en un dispositiu com un Lenovo m920q podria ser més adequat.

Si experimenteu bloquejos inexplicables que correlacionen amb la temperatura ambient, assegureu-vos que l'USG està en una zona ben ventilada amb flux d'aire adequat, no dins d'un armari tancat ni apilat amb altres equips que generin calor. Un ventilador actiu de 80mm o 120mm dirigit al dispositiu pot marcar una diferència significativa. Espereu que l'USG tingui dificultats a temperatures ambient sostingudes per sobre dels ~30°C si no té refrigeració activa.

### L'adopció pel controller falla

| Símptoma                                     | Causa probable                    | Solució                                                      |
| -------------------------------------------- | --------------------------------- | ------------------------------------------------------------ |
| Bucle d'adopció (Adopting → Disconnected)    | Xarxa bridge de Docker            | Canvieu a `network_mode: host`                               |
| L'USG queda com a "Disconnected"             | Entrada de dispositiu obsoleta    | Elimineu via MongoDB, restauració de fàbrica de l'USG        |
| SSH a l'USG rebutjat després de l'adopció    | El controller ha canviat les credencials | Utilitzeu les credencials de Settings → Site → Device Authentication |

------

## Crèdits i referències

- [mordor.world — Sustituir Livebox 6 de Orange por UFiber Loco y un router neutro](https://www.mordor.world/2023/04/26/sustituir-livebox-6-de-orange-por-ufiber-loco-y-un-router-neutro/) — Guia en castellà que va ser el punt de partida per a aquesta migració.
- [v-a-c-u-u-m/ufiber_nano_serial_hack](https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack) — Eina Python per clonar números de sèrie als dispositius UFiber Loco/Nano G.
- [Ubiquiti Community Forums](https://community.ui.com/) — Diversos fils sobre autenticació GPON i configuració de UFiber.

------

## Llicència

Aquesta guia es proporciona tal qual sota la llicència MIT. Utilitzeu-la sota la vostra responsabilitat. Desconnectar equipament de l'ISP pot infringir el vostre contracte de servei — reviseu el vostre contracte.
