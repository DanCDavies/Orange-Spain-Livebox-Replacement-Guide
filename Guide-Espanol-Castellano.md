# Sustitución del Livebox 6 de Orange Spain por un UFiber Loco y un Unifi USG-3P

## ¿Por qué sustituir el Livebox?

El Livebox 6 de Orange Spain es un router cerrado y controlado por el operador. No permite configurar VLANs, establecer reglas de firewall personalizadas, ejecutar bloqueo de publicidad a nivel de red, ni gestionar tu red como un sistema unificado. Sustituirlo te da control total sobre tu red con reglas de firewall adecuadas, segmentación por VLAN, DNS personalizado y un único plano de gestión para toda tu red.

En el momento de redactar esta guía, el USG-3P está en End-of-Life. Sin embargo, sigue siendo asequible y fácil de encontrar en Wallapop y otros vendedores de hardware de segunda mano. Especialmente si se combina con un controller autoalojado en una Raspberry Pi u otro mini PC, es un método de bajo coste y baja barrera de entrada para sustituir el equipamiento existente.

**Importante:** Esta guía se escribió utilizando un Unifi Controller en la versión 6.0.28, debido al uso de hardware EoL en la red del autor. Si estás utilizando una versión más reciente, necesitarás (a) asegurarte de que tu controller es compatible con el Gateway que pretendes usar, y (b) comprobar las ubicaciones equivalentes de los ajustes en versiones más recientes del controller, ya que esta guía hace referencia a ubicaciones de la versión 6.0.28.

### ¿Por qué no usar el modo ONT?

El Livebox de Orange tiene un modo 'Solo Módem' que, en teoría, permite usarlo como un simple ONT y emplear un router comercial o de opnSense para el enrutamiento real (incluido el USG-3P), prescindiendo por completo del Nano Fiber Loco / Nano G.

En mi experiencia, esto no funcionó bajo ninguna circunstancia ni combinación de hardware. Tu resultado puede variar, y si ya tienes un router, quizá prefieras probar eso primero, ya que esta guía es bastante extensa.

## Requisitos

Como mínimo, necesitarás lo siguiente (o su equivalente) para seguir esta guía:

| Componente                                                   | Función                                                   | Notas                                                        |
| ------------------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| **Ubiquiti UFiber Loco** (UF-LOCO) o **UFiber Nano G** (UF-NANO) | GPON ONT — sustituye la función de módem de fibra del Livebox | El Loco es más económico y suficiente. El Nano G tiene WiFi integrado. Ambos deberían funcionar de forma idéntica para este propósito. |
| **Ubiquiti USG-3P** (UniFi Security Gateway)                 | Router/firewall — sustituye la función de enrutamiento del Livebox | EOL desde noviembre de 2024, pero sigue funcionando con versiones del controller hasta la 8.x. Ver [Notas sobre End-of-Life](#notas-sobre-end-of-life). |
| **UniFi Controller**                                         | Software para gestionar el USG (y opcionalmente APs/switches) | Se recomienda autoalojado mediante Docker. Ver [Configuración del Controller](#configuración-del-controller). |
| **Un ordenador con Python 3 y SSH**                          | Para la clonación del número de serie y la configuración  | Windows, macOS o Linux.                                      |
| **Un cable ethernet**                                        | Conexión directa al Loco durante la configuración         | Necesitarás desconectarte de tu red temporalmente.           |
| **Tus credenciales del contrato de Orange Spain**            | Contraseña PLOAM y número de serie del Livebox            | Se extraen en el [Paso 1](#paso-1-extraer-credenciales-del-livebox). |

### Opcional pero muy recomendable

- **Un switch gigabit no gestionado barato** — útil si necesitas mantener conectividad local entre dispositivos durante la migración.
- **Un móvil con datos** — útil para crear un hotspot o consultar referencias durante las fases en las que no hay acceso a internet.

## Visión general de la arquitectura

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

El Loco gestiona la autenticación GPON con el OLT de Orange y pasa ethernet en bruto al USG. El USG etiqueta el tráfico en la VLAN 832, obtiene una IP pública mediante DHCP y enruta tu LAN. El UniFi controller gestiona la configuración del USG.

## Importante: Orange Spain ≠ Orange France

Muchas guías en internet (especialmente en francés) describen cómo sustituir routers de Orange usando **DHCP Option 90**, **vendor-class-identifier "sagem"**, **cadenas user-class** y **PPPoE**. **Nada de esto aplica a Orange Spain.**

Orange Spain utiliza (en el momento de redactar esta guía, mayo de 2026):

- **VLAN 832** con **DHCP** estándar (sin PPPoE)
- **Sin Option 90**
- **Sin vendor-class-identifier**
- **Sin user-class**
- **Sin cadenas de autenticación RFC3118**

Si encuentras alguna guía que te dice que generes una cadena hexadecimal a partir de credenciales `fti/xxxxxxxxx` o que establezcas vendor-class a "sagem", esa guía es para **Orange France** y **evitará activamente** que tu conexión funcione en España.

La única autenticación que importa es a nivel de **GPON**: la contraseña PLOAM y el número de serie del ONT.

------

## Paso 1: Extraer credenciales del Livebox

Antes de desconectar nada, accede a tu Livebox en `http://192.168.1.1` y anota estos valores:

### 1.1 Contraseña PLOAM (contraseña ONT)

Navega a la sección de información/GPON del panel de administración del Livebox. Busca la **contraseña PLOAM** o **contraseña ONT**. Será una cadena alfanumérica de 10 caracteres (p. ej., `A5XK92WR7D`).

### 1.2 Número de serie

Busca el **número de serie GPON**. Tendrá un aspecto como `SMBS4A8F7C12` — un identificador de fabricante de 4 caracteres seguido de 8 caracteres hexadecimales. Anota:

- **Número de serie completo**: p. ej., `SMBS4A8F7C12`
- **Vendor ID**: los primeros 4 caracteres (p. ej., `SMBS`)
- **Número de serie del dispositivo**: los 8 caracteres restantes (p. ej., `4A8F7C12`)

### 1.3 Dirección MAC

Busca la **dirección MAC WAN** del Livebox (no la MAC WiFi ni la MAC LAN). Formato: `XX:XX:XX:XX:XX:XX`.

### 1.4 Tabla resumen

Guarda estos datos en un sitio accesible sin conexión — los necesitarás cuando estés desconectado de internet:

| Parámetro                          | Tu valor        | Ejemplo             |
| ---------------------------------- | --------------- | ------------------- |
| Contraseña PLOAM                   |                 | `A5XK92WR7D`        |
| Número de serie completo           |                 | `SMBS4A8F7C12`      |
| Vendor ID                          |                 | `SMBS`              |
| Número de serie del dispositivo    |                 | `4A8F7C12`          |
| Dirección MAC WAN                  |                 | `DC:15:C8:1A:E3:4B` |
| IP por defecto del Loco            | `192.168.1.1`   | —                   |
| Credenciales por defecto del Loco  | `ubnt` / `ubnt` | —                   |

------

## Paso 2: Preparar el UFiber Loco

### 2.1 Acceso inicial

1. Desconecta tu PC de tu red.
2. Conecta el puerto ethernet de tu PC directamente al **puerto LAN** del Loco.
3. Alimenta el Loco mediante su adaptador PoE de 24V o una fuente de alimentación micro-USB de **5V/1A o superior**. No confíes en un puerto USB 2.0 de PC — puede no entregar suficiente corriente (el Loco consume hasta 3,5W = 700mA a 5V).
4. Configura tu PC con IP estática: `192.168.1.100`, máscara `255.255.255.0`, gateway `192.168.1.1`.
5. Navega a `https://192.168.1.1` (nota: **HTTPS**, no HTTP — el Loco redirige a HTTPS y algunos navegadores rechazan la redirección silenciosamente). Acepta la advertencia del certificado.
6. Inicia sesión: `ubnt` / `ubnt`.

> **Error frecuente:** Si el navegador muestra "no se puede conectar", prueba tanto `http://` como `https://`. Si ninguno funciona, verifica que tu IP estática sea realmente `192.168.1.100` y no, por ejemplo, `192.168.100.1` (un error de transposición habitual en Windows). Ejecuta `ipconfig` para confirmarlo.

> **Usuarios de Hyper-V / WSL:** Si tienes Hyper-V o WSL activados en Windows, un adaptador de red virtual puede interceptar el tráfico de tu puerto ethernet físico. Desactiva el adaptador vEthernet temporalmente o usa otra máquina.

### 2.2 Actualizar firmware

Ve a **System** → **Upload Firmware** y actualiza a la última versión. En el momento de escribir esto, el firmware **4.4.6** funciona correctamente. El dispositivo se reiniciará (espera 3-5 minutos). En la carpeta \Firmware de este repositorio hay copias del firmware de Ui.com. Todo el firmware está sin modificar y es propiedad intelectual de Ubiquiti.

### 2.3 Configurar los ajustes GPON

1. Ve a la pestaña **GPON**.
2. Establece **OLT Profile**: prueba primero con el **Profile 3** (FiberHome). Consulta [Selección de perfil](#selección-de-perfil) para más detalles.
3. Establece la **PLOAM Password**: tu contraseña del Paso 1 (p. ej., `A5XK92WR7D`).
4. **Hex Format**: **OFF** (desmarcado).
5. Deja los campos de autenticación LOID con los valores por defecto (`ubnt` / `ubntubnt`) — Orange Spain no usa LOID.
6. Haz clic en **Save**.

### 2.4 Cambiar la IP de gestión

**Esto es fundamental.** El USG también usará `192.168.1.1` como IP de LAN. El Loco debe estar en una dirección diferente.

1. Ve a la pestaña **Network**.
2. Cambia la IP del dispositivo a algo fuera de tu rango DHCP, p. ej., `192.168.1.15` o `192.168.100.1`.
3. Guarda. Necesitarás reconectarte a la nueva IP.

### 2.5 Activar SSH

Ve a **System** y asegúrate de que **SSH** está activado en el puerto 22. Lo necesitarás para la clonación del número de serie.

------

## Paso 3: Clonar el número de serie (Paso crítico)

Este es el paso que la mayoría de guías omite, y es la causa más habitual de fallo en la autenticación GPON. El Loco viene de fábrica con el vendor ID propio de Ubiquiti (`UBNT`) y un número de serie aleatorio. El OLT de Orange espera ver el número de serie y el vendor ID de tu Livebox.

### 3.1 Instalar dependencias

En tu PC (mientras aún tengas internet, o descargadas previamente):

```bash
pip install paramiko scp
```

### 3.2 Descargar la herramienta de clonación de número de serie

Descarga desde: https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack

Clona el repositorio o descarga el ZIP y extráelo.

### 3.3 Convertir tu número de serie a formato hexadecimal

El número de serie GPON ocupa 8 bytes: 4 bytes de vendor ID (ASCII) + 4 bytes de número de serie del dispositivo (hex).

Por ejemplo, `SMBS4A8F7C12`:

- `SMBS` → hex ASCII: `53:4D:42:53`
- `4A8F7C12` → bytes hex: `4A:8F:7C:12`
- **Parámetro de serie completo**: `53:4D:42:53:4A:8F:7C:12`

Para convertir tu vendor ID a hexadecimal, usa una tabla ASCII o una herramienta en línea:

| Char | Hex  | Char | Hex  | Char | Hex  | Char | Hex  |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| A    | 41   | H    | 48   | O    | 4F   | V    | 56   |
| B    | 42   | I    | 49   | P    | 50   | W    | 57   |
| C    | 43   | J    | 4A   | Q    | 51   | X    | 58   |
| D    | 44   | K    | 4B   | R    | 52   | Y    | 59   |
| E    | 45   | L    | 4C   | S    | 53   | Z    | 5A   |
| F    | 46   | M    | 4D   | T    | 54   |      |      |
| G    | 47   | N    | 4E   | U    | 55   |      |      |

La parte del número de serie del dispositivo (los 8 caracteres después del vendor ID) debe tratarse como **bytes hexadecimales** — cada par de caracteres es un byte.

### 3.4 Verificar antes de escribir (solo lectura)

Conecta tu PC al Loco por ethernet (IP estática en la misma subred), y luego (usando la IP que configuraste en el Paso 2.4):

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Contraseña: `ubnt` (o la que hayas configurado).

Deberías ver una salida como esta:

```
[~] vendor_id:        "UBNT"
[~] serial_id:        "a9e0dd3c"
[~] base_mac_addr:    ac:8b:a9:e0:dd:3c
[~] gpon_password:    "A5XK92WR7D"
```

Confirma que la contraseña GPON coincide. El vendor ID y el número de serie son los valores por defecto de Ubiquiti — los vamos a cambiar a continuación.

### 3.5 Escribir el número de serie clonado

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 \
  --serial 53:4D:42:53:4A:8F:7C:12 \
  --mac DC:15:C8:1A:E3:4B \
  --insecure
```

Sustituye el número de serie y la MAC por **tus** valores del Paso 1.

> **El flag `--insecure` es necesario para el UFiber Loco.** El script se quejará de diferencias en el checksum del firmware en dispositivos Loco sin este flag. Esto es normal y seguro.

### 3.6 Verificar la escritura

Ejecuta de nuevo el comando de solo lectura y confirma que todos los valores coinciden con tu Livebox:

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Salida esperada:

```
[~] vendor_id:        "SMBS"
[~] serial_id:        "4A8F7C12"
[~] base_mac_addr:    dc:15:c8:1a:e3:4b
[~] gpon_password:    "A5XK92WR7D"
```

------

## Paso 4: Probar la autenticación GPON

1. Desconecta el cable ethernet del Loco.
2. Desenchufa el cable de fibra SC/APC del Livebox. Manéjalo con cuidado — no toques la férula.
3. Conecta la fibra al **puerto GPON** del Loco.
4. Observa los LEDs. Espera hasta 60 segundos.

### Indicadores LED

- **Barras verdes** = autenticación GPON correcta
- **Parpadeo naranja/ámbar** = intentando registrarse
- **Barras blancas/azules fijas** = encendido, sin autenticación GPON

> **Importante:** Los LEDs pueden ser engañosos en algunas versiones de firmware. El Loco puede mostrar solo barras blancas incluso cuando la autenticación GPON ha sido correcta. **El dashboard es la fuente fiable.** Para comprobarlo correctamente: reconecta el ethernet al Loco, navega a `https://192.168.1.15` y mira el dashboard. Si muestra **STATE: CONNECTED** con una potencia RX de aproximadamente **-15 a -20 dBm**, la autenticación ha sido correcta independientemente de lo que muestren los LEDs. Mi propio Fiber Loco, por ejemplo, nunca muestra barras verdes, a pesar de funcionar perfectamente.

### Selección de perfil

Si el Profile 3 no autentica, prueba los distintos perfiles:

| Perfil    | Fabricante del OLT | Notas                                                        |
| --------- | ------------------- | ------------------------------------------------------------ |
| Profile 1 | Ubiquiti OLT        | **Omitir** — es para el equipamiento OLT propio de Ubiquiti. No expone PLOAM ni ajustes de autenticación de terceros. |
| Profile 2 | Huawei              | Habitual en Orange Spain                                     |
| Profile 3 | FiberHome           | Habitual en Orange Spain — prueba este primero               |
| Profile 4 | ZTE                 | Menos habitual pero funciona en algunas zonas                |

Para cambiar de perfil: ve a `https://192.168.1.15` → pestaña GPON → cambia el desplegable OLT Profile → Save. El dispositivo se reinicia. Reconecta la fibra y comprueba de nuevo.

En nuestra experiencia, **los Profiles 2, 3 y 4 funcionaron todos** — el OLT era flexible. Tu resultado puede variar dependiendo del equipamiento de tu central. Si ninguno de los Profiles 2/3/4 produce un estado CONNECTED, tu OLT puede ser Alcatel/Nokia, con el que los dispositivos UFiber no pueden autenticarse. En ese caso, necesitarías un ONT diferente (p. ej., un Huawei HG8010H con clonación de número de serie).

### Vuelta atrás

Si nada funciona: desenchufa la fibra del Loco, reconéctala al Livebox y enciende el Livebox. En 2 minutos vuelves a la normalidad. No se ha cambiado nada del lado de Orange.

------

## Paso 5: Configurar el USG-3P

### 5.1 Cableado físico

```
Fibre → Loco GPON port → Loco LAN port → USG WAN (eth0)
                                           USG LAN (eth1) → Switch → Devices
```

### 5.2 El archivo config.gateway.json

El USG necesita crear una sub-interfaz VLAN 832 en su puerto WAN y solicitar un lease DHCP a través de ella. También necesita una regla NAT masquerade en esa interfaz VLAN (en mi experiencia, no sobre `eth0` directamente), y — de forma crítica — correcciones de firewall y enrutamiento que el UniFi controller no aplica correctamente por defecto.

Crea este archivo en tu UniFi controller:

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

Este archivo hace cuatro cosas:

1. **VLAN 832 con DHCP** — crea la sub-interfaz `eth0.832` y solicita una IP pública a través de ella.
2. **NAT masquerade en `eth0.832`** — sin esto, el USG obtiene una IP pública pero el tráfico de tu LAN no será traducido (NAT) a través de ella. Las reglas de masquerade por defecto del USG apuntan a `eth0`, no a `eth0.832`.
3. **Reglas de firewall en `eth0.832`** — esta es la parte crítica que la mayoría de guías omite. Ver [El hueco de firewall en la VLAN 832](#el-hueco-de-firewall-en-la-vlan-832) más abajo.
4. **Source validation en modo "loose"** — corrige el filtrado de ruta inversa que provoca que tráfico de retorno legítimo se descarte como "martian". Ver [Filtrado de ruta inversa](#filtrado-de-ruta-inversa) más abajo.

### El hueco de firewall en la VLAN 832

Cuando el UniFi controller crea las reglas de firewall WAN_LOCAL y WAN_IN (ya sean las predeterminadas o las que añadas desde la interfaz), las aplica a `eth0` — la interfaz WAN física. Pero tu interfaz real de cara a internet es `eth0.832`, la sub-interfaz VLAN. **El controller no aplica automáticamente las reglas de firewall a las sub-interfaces VLAN.** Esto significa que, sin el bloque `firewall` en la configuración anterior, el puerto WAN de tu USG está efectivamente sin filtrar: cada escaneo de puertos, intento de fuerza bruta SSH y sondeo desde internet llega directamente a los servicios del USG.

En un dispositivo con CPU Cavium Octeon MIPS64 y 512MB de RAM, este bombardeo constante (que es normal — cualquier IP pública es escaneada continuamente por bots automatizados en todo el mundo) puede hacer que el USG deje de responder en cuestión de horas. Los síntomas son pérdidas intermitentes de internet que requieren reiniciar el dispositivo, sin ningún error claro en los logs.

Las entradas `firewall.in` y `firewall.local` en la configuración anterior indican a EdgeOS que aplique tus cadenas WAN_IN y WAN_LOCAL a `eth0.832`, de modo que todo el tráfico que llegue por la VLAN de Orange quede sujeto a las mismas reglas de firewall que el tráfico en la interfaz física. Esto es esencial para la estabilidad.

### Filtrado de ruta inversa

El ajuste `"source-validation": "loose"` cambia el filtro de ruta inversa del kernel (`rp_filter`) del modo estricto (1) al modo permisivo (2). En modo estricto, el kernel descarta cualquier paquete cuya dirección de origen no se enrutaría de vuelta por la misma interfaz por la que llegó. Con la VLAN 832, el tráfico de retorno legítimo desde internet puede llegar por `eth0.832`, pero la tabla de enrutamiento del kernel puede indicar que el origen debería ser alcanzable a través de `eth0` — la interfaz padre. El modo estricto descarta esto como un paquete "martian" y lo registra en los logs.

El síntoma es conectividad intermitente: parte del tráfico funciona, parte se descarta silenciosamente, y los logs se llenan de líneas como:

```
IPv4: martian source 192.168.1.x from <external-ip>, on dev eth0.832
```

El modo permisivo sigue validando que la dirección de origen sea alcanzable a través de *alguna* interfaz (no está desactivado), pero no exige que sea la misma por la que llegó el paquete. Este es el ajuste correcto para cualquier configuración WAN basada en VLAN.

### 5.3 Dónde colocar el archivo

**Si tu controller está basado en Docker (autoalojado):**

Localiza el volumen de datos de UniFi. Rutas habituales:

```bash
# Comprueba tu docker-compose.yml para ver el mapeo de volúmenes
# Si: volumes: - ./config:/config
# Entonces el archivo va en:
/opt/docker/unifi/config/data/sites/default/config.gateway.json
```

Crea el directorio `sites/default/` si no existe:

```bash
mkdir -p /path/to/unifi/config/data/sites/default/
```

> **El nombre de sitio "default"** corresponde al ID interno del sitio, no al nombre visible. Comprueba la URL de tu controller — muestra `manage/site/XXXXX/` donde `XXXXX` es el ID del sitio. Usa ese valor en lugar de `default` si es diferente.

**Si tu controller se ejecuta de forma nativa (sin Docker):**

```bash
/usr/lib/unifi/data/sites/default/config.gateway.json
```

### 5.4 Prueba rápida por SSH (Opcional, antes de la adopción por el controller)

Si quieres probar internet antes de lidiar con la adopción del controller, puedes configurar el USG directamente por SSH:

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

Comprueba si tienes IP pública:

```bash
show interfaces ethernet eth0 vif 832
```

Si ves una IP pública y `ping 8.8.8.8` funciona desde tu escritorio — el Livebox ha sido sustituido con éxito.

> **La configuración por SSH se sobrescribe cuando el controller aprovisiona el USG.** Esto está bien para pruebas, pero necesitas tener el `config.gateway.json` en su sitio antes de la adopción por el controller para que la configuración sea persistente.

------

## Paso 6: Adopción por el controller

### 6.1 Red Docker — se requiere modo host

**Este es el fallo de adopción más común cuando se ejecuta el UniFi controller en Docker.**

El UniFi controller necesita conectarse por SSH *al* USG para aprovisionarlo. Si el contenedor del controller está en una red Docker bridge (la configuración por defecto), no puede enrutar hacia `192.168.1.1`. El proceso de inform (USG → controller) funciona porque Docker mapea el puerto, pero el aprovisionamiento (controller → USG por SSH) falla.

**Síntoma:** El USG aparece en el controller, haces clic en Adopt, entra en un bucle de "Adopting" y nunca se completa, o muestra "Disconnected" repetidamente.

**Solución:** Cambia el contenedor del controller a red en modo host.

Antes (modo bridge — **no funciona para la adopción del USG**):

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

Después (modo host — **funciona**):

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

Elimina todo el bloque `ports` — con red en modo host, el contenedor se vincula directamente a los puertos del host. Verifica que ningún otro servicio en el host entre en conflicto con los puertos de UniFi (8080, 8443, 3478/udp, 10001/udp, 8880, 6789, 5514/udp).

**Importante:** El resto de los ajustes del yaml son ejemplos de mi propio despliegue para proporcionar contexto. Debes usar los valores correctos para tu propio docker-compose.yml. Si no sabes cómo configurarlos, te recomiendo encarecidamente leer sobre Docker antes de intentar esta migración (https://docs.docker.com/get-started/). También puedes considerar usar el Unifi Controller directamente sobre el sistema operativo en lugar de Docker, pero recuerda establecer la versión que necesites para soportar tu propia configuración de hardware.

Luego:

```bash
docker compose down
docker compose up -d
```

### 6.2 Proceso de adopción

1. Asegúrate de que el `config.gateway.json` está en su sitio en `sites/default/`.

2. Abre e inicia sesión en la interfaz del controller en `https://<controller-ip>:8443`.

3. El USG debería aparecer como "Pending Adoption".

4. Haz clic en **Adopt**.

5. Si no aparece, conéctate por SSH al USG y fuerza el inform:

   ```bash
   ssh ubnt@192.168.1.1
   set-inform http://<controller-ip>:8080/inform
   ```

   Puede que necesites ejecutar `set-inform` dos veces — una para presentar el dispositivo, y otra después de hacer clic en Adopt.

### 6.3 Resolución de problemas de adopción

**USG atascado en bucle de adopción:**

Si el USG cicla repetidamente entre "Adopting" y "Disconnected":

1. Comprueba la configuración de red de Docker (ver 6.1 arriba).

2. Elimina forzosamente el dispositivo de la base de datos del controller:

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

3. Restablece el USG a valores de fábrica (mantén pulsado reset 10 segundos), luego vuelve a adoptar.

**Configuración no aplicada tras la adopción:**

Si el USG se adopta pero no tiene la VLAN 832:

1. Verifica que el archivo es visible dentro del contenedor:

   ```bash
   docker exec -it unifi-controller cat /usr/lib/unifi/data/sites/default/config.gateway.json
   ```

2. Fuerza un reaprovisionamiento reseteando la versión de configuración:

   ```bash
   docker exec -it unifi-controller mongo --port 27117 --eval \
     'db.getSiblingDB("ace").device.update({"model":"UGW3"},{$set:{"cfgversion":"0"}})'
   ```

3. Reinicia el controller: `docker restart unifi-controller`

------

## Paso 7: Verificación

Una vez que todo funcione:

```bash
# Desde SSH en el USG:
show interfaces ethernet eth0 vif 832
# Debería mostrar una dirección IP pública

# Desde cualquier dispositivo de la LAN:
ping 8.8.8.8
ping google.com
```

### Verificar las correcciones de firewall y rp_filter

Son tan importantes como la comprobación básica de conectividad. Después de que el controller aprovisione el USG, conéctate por SSH y confirma:

```bash
# Comprobar que rp_filter está en modo loose (debería devolver: 2)
cat /proc/sys/net/ipv4/conf/eth0.832/rp_filter

# Comprobar que el firewall está aplicado a eth0.832 (debería mostrar eth0.832 → WAN_LOCAL)
sudo iptables -L VYATTA_FW_LOCAL_HOOK -nv --line-numbers
```

En la salida de `VYATTA_FW_LOCAL_HOOK`, deberías ver una línea mapeando `eth0.832` a `WAN_LOCAL`. Si no aparece, la sección de firewall de tu `config.gateway.json` no se está aplicando — comprueba la ruta del archivo y fuerza un reaprovisionamiento.

### Lista de comprobación post-migración recomendada

- [ ] El dashboard GPON del Loco muestra **CONNECTED** con buena potencia RX (-15 a -20 dBm)
- [ ] El USG tiene una IP pública en `eth0.832`
- [ ] Los dispositivos de la LAN pueden acceder a internet
- [ ] La resolución DNS funciona (`ping google.com`)
- [ ] El controller muestra el USG como "Connected" y gestionado
- [ ] `config.gateway.json` está en su sitio (la configuración sobrevive al reaprovisionamiento)
- [ ] `rp_filter` en `eth0.832` está en `2` (loose)
- [ ] `VYATTA_FW_LOCAL_HOOK` muestra `eth0.832 → WAN_LOCAL`
- [ ] Los puntos de acceso están readoptados y emitiendo (si aplica)

Si todo esto funciona, enhorabuena. Ya puedes guardar tu Livebox de Orange en el fondo de un armario y olvidarte de él hasta que cambies de operador y te lo pidan de vuelta.

Disfruta de tu nueva capacidad para enrutar VLANs y configurar tu propio DNS.

------

## Notas sobre End-of-Life

El **USG-3P** fue declarado End of Life por Ubiquiti en **noviembre de 2024**:

- Las versiones del UniFi Network controller **9.x y posteriores no pueden adoptar dispositivos USG en absoluto**.
- El USG-3P funciona con versiones del controller hasta la **8.x**.
- Si alojas tu propio controller (Docker), puedes fijar la versión y controlar tu propio ciclo de actualizaciones.

Si ya tienes hardware Ubiquiti / Unifi más reciente, considera si el USG-3P sigue mereciendo la pena para tu caso de uso. Si ya tienes uno, seguirá funcionando indefinidamente mientras controles la versión de tu controller.

------

## Configuración del Controller

Si aún no tienes un UniFi controller, el enfoque más sencillo para autoalojarlo es Docker. Un `docker-compose.yml` mínimo:

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

Ejecútalo con `docker compose up -d`. Accede a la interfaz en `https://<host-ip>:8443`.

> **Fija la versión del controller** si tienes un USG-3P. Usa una etiqueta específica como `version-8.6.9` en lugar de `latest` para evitar actualizaciones accidentales que eliminen el soporte del USG.

------

## Referencia de resolución de problemas

### La autenticación GPON no funciona

| Síntoma                                      | Causa probable             | Solución                                                     |
| -------------------------------------------- | -------------------------- | ------------------------------------------------------------ |
| Barras blancas, sin CONNECTED en el dashboard | Número de serie no clonado | Ejecuta la herramienta de clonación de número de serie (Paso 3) |
| Barras blancas tras clonar el número de serie | Perfil OLT incorrecto     | Prueba los Profiles 2, 3 y 4                                |
| Sin potencia RX en el dashboard              | Fibra mal conectada        | Recoloca el conector SC/APC, comprueba si hay polvo          |
| Todos los perfiles fallan                    | OLT Alcatel/Nokia          | UFiber no puede autenticarse contra Alcatel. Se necesita un ONT diferente. |

### El USG no consigue internet

| Síntoma                                          | Causa probable                                     | Solución                                           |
| ------------------------------------------------ | -------------------------------------------------- | -------------------------------------------------- |
| Sin IP en eth0.832                               | VLAN 832 no configurada                            | Comprueba `config.gateway.json` o la configuración SSH |
| IP pública en eth0.832 pero sin internet en la LAN | Falta NAT masquerade en `eth0.832`                | Añade la regla 5010 a config.gateway.json          |
| El USG muestra reglas por defecto solo en `eth0` | Controller aprovisionó sin config.gateway.json     | Asegúrate de que el archivo está en la ruta correcta, fuerza reaprovisionamiento |

### Inestabilidad del USG (cuelgues, caídas de internet que requieren reinicio)

| Síntoma                                          | Causa probable                                     | Solución                                                     |
| ------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| Internet se cae tras horas, reiniciar lo soluciona | Firewall no aplicado a `eth0.832`                 | Añade el bloque `firewall` a la configuración de la VLAN 832. Ver [El hueco de firewall en la VLAN 832](#el-hueco-de-firewall-en-la-vlan-832). |
| Los logs muestran fuerza bruta SSH desde IPs externas | WAN_LOCAL no filtra `eth0.832`                   | Misma solución — el bloque firewall aplica WAN_LOCAL a la interfaz VLAN. |
| Los logs muestran "martian source" en `eth0.832` | Filtrado de ruta inversa estricto                  | Añade `"source-validation": "loose"` a la sección firewall de config.gateway.json. Ver [Filtrado de ruta inversa](#filtrado-de-ruta-inversa). |
| Cuelgues aleatorios en clima cálido, sin evidencia en los logs | Apagado térmico                               | Ver [Consideraciones térmicas](#consideraciones-térmicas) más abajo. |

### Consideraciones térmicas

El USG-3P tiene refrigeración pasiva con un pequeño disipador de aluminio. Es propenso a bloquearse por temperatura, particularmente en climas cálidos o espacios cerrados. No parece haber ningún tipo de throttling progresivo según he podido determinar en mis pruebas. Cuando el SoC se sobrecalienta, el dispositivo se bloquea silenciosamente y requiere un reinicio. No hay aviso en los logs porque el fallo es instantáneo.

Entre los factores que contribuyen se incluyen DPI (Deep Packet Inspection), IDS/IPS y una alta rotación en la tabla de conexiones — todo lo cual aumenta significativamente la carga de CPU y la generación de calor en el SoC MIPS64. DPI e IDS/IPS no deberían activarse en el USG-3P en ningún caso debido a su limitada capacidad de procesamiento, pero además empeoran la situación térmica. Si necesitas este tipo de funcionalidad, deberías considerar si el USG-3P es adecuado para ti — un router opnSense en un dispositivo como un Lenovo m920q podría encajar mejor.

Si experimentas cuelgues inexplicables que correlacionan con la temperatura ambiente, asegúrate de que el USG está en una zona bien ventilada con flujo de aire adecuado, no dentro de un armario cerrado ni apilado con otros equipos que generen calor. Un ventilador activo de 80mm o 120mm dirigido al dispositivo puede marcar una diferencia significativa. El USG tendrá dificultades a temperaturas ambiente sostenidas por encima de ~30°C si no se refrigera activamente.

### La adopción del controller falla

| Síntoma                                    | Causa probable                   | Solución                                                     |
| ------------------------------------------ | -------------------------------- | ------------------------------------------------------------ |
| Bucle de adopción (Adopting → Disconnected) | Red Docker en modo bridge       | Cambia a `network_mode: host`                                |
| USG atascado como "Disconnected"           | Entrada de dispositivo obsoleta  | Elimina vía MongoDB, restablece el USG de fábrica            |
| SSH al USG rechazado tras adopción         | El controller cambió las credenciales | Usa las credenciales de Settings → Site → Device Authentication |

------

## Créditos y referencias

- [mordor.world — Sustituir Livebox 6 de Orange por UFiber Loco y un router neutro](https://www.mordor.world/2023/04/26/sustituir-livebox-6-de-orange-por-ufiber-loco-y-un-router-neutro/) — Guía en castellano que fue el punto de partida para esta migración.
- [v-a-c-u-u-m/ufiber_nano_serial_hack](https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack) — Herramienta en Python para clonar números de serie en dispositivos UFiber Loco/Nano G.
- [Ubiquiti Community Forums](https://community.ui.com/) — Diversos hilos sobre autenticación GPON y configuración de UFiber.

------

## Licencia

Esta guía se proporciona tal cual bajo la licencia MIT. Úsala bajo tu propia responsabilidad. Desconectar equipamiento del operador puede incumplir tu contrato de servicio — consulta las condiciones de tu contrato.
