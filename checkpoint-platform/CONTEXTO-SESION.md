# Plataformado Check Point con Ansible — Contexto de sesion

**Fecha:** 2026-07-21 · **Actualizado:** 2026-07-22
**Repo:** `RodriGN99/CheckPoint-Ansible` (GitHub) · **Rama:** `main`
**Ruta del proyecto:** `checkpoint-platform/`

> El plan inicial era `sc2n` en git.gmv.es (rama `dev-checkpoint`). Se cambio a
> GitHub personal porque la VM Ansible no estaba sincronizada con el GitLab.
> El paso a GitLab se hara cuando el proyecto entre en uso real.

Continuacion de `ansible-checkpoint-contexto.md`. Este documento recoge lo
**verificado contra el lab real**, que en varios puntos corrige al anterior.

---

## 1. Alcance acordado

Solo Check Point: **SMS + ClusterXL**. Quedan fuera a proposito FortiGate,
Cisco C9K, ArubaCX y los clientes Windows/Docker.

---

## 2. Datos del lab (confirmados, no supuestos)

Origen: Gaia Portal de cada equipo + `cphaprob -a if` en CP-GW-A.

| Rol | Host | eth0 (red 90) | eth3 (sync) | eth4 (OOB) | Etiqueta EVE-NG |
|---|---|---|---|---|---|
| Management | `sms` | 192.168.90.170 | — | 192.168.92.11 *(en eth3)* | SMS |
| Miembro A | `cp-gw-a` | 192.168.90.171 | 10.255.255.1 **/29** | 192.168.92.12 | **CP Bound 2** |
| Miembro B | `cp-gw-b` | 192.168.90.172 | 10.255.255.2 **/29** *(era /30, corregido 22/07)* | 192.168.92.13 | **CP Bound 1** |
| VIP cluster | — | **192.168.90.173** | — | — | — |

Interior (bond hacia ArubaCX):

| VLAN | Red | CP-GW-A | CP-GW-B | VIP |
|---|---|---|---|---|
| 10 | 192.168.10.0/24 | .3 (`bond2.10`) | .2 (`bond1.10`) | **192.168.10.1** |
| 99 | 192.168.99.0/24 | .3 (`bond2.99`) | .2 (`bond1.99`) | **192.168.99.1** |

Puntos que despistan y conviene tener presentes:

- **`cp-gw-a` es "CP Bound 2"** y **`cp-gw-b` es "CP Bound 1"**. El orden del
  diagrama va al reves de lo que sugiere el nombre.
- La **topologia del cluster usa el naming de CP-GW-A (`bond2.x`)**, aunque
  CP-GW-B tenga fisicamente `bond1.x`. Funciona asi.
- **`eth4` existe en los equipos pero no en el SMS.** En Gaia esta configurada y
  activa (`192.168.92.12` en CP-GW-A, `192.168.92.13` en CP-GW-B, OOB), pero el
  objeto cluster del SMS **no la incluye** en su topologia. Que no aparezca en
  `cphaprob -a if` es coherente con eso. Ver punto 11.
- **SIC y gestion van por la red 90** (Management Interface = `eth0`), no por OOB.

---

## 3. Estado real del cluster

El cluster **ya existia** antes de empezar: se llama **`CP-CLUSTER-HA`**, VIP
192.168.90.173, con CP-GW-A y CP-GW-B como miembros.

Salud actual (`cphaprob state` / `-a if` / `list`): CP-GW-B **ACTIVE**,
CP-GW-A **STANDBY**, todas las interfaces UP, **sin pnotes**. Funciona.

Dos cosas mejorables, ninguna bloqueante:

1. **Licencias de CP-GW-A.** Es la causa del estado rojo de CP-CLUSTER-HA y
   CP-GW-A en SmartConsole. No es un problema de topologia.

2. **Flapeo del cluster.** Ver el punto 12, que sustituye por completo el
   diagnostico que este documento daba antes.

Desajuste menor entre miembros, sin efecto demostrado: nombres de bond distintos
(`bond2` vs `bond1`). Verificado el 22/07/2026: solo cambia el ID del bond; la
configuracion interna es identica en los dos miembros.

**Mascara de sync: RESUELTA el 22/07/2026.** CP-GW-B estaba en `/30` y CP-GW-A en
`/29`. El usuario la unifico a `/29` en ambos. Ojo: **esto NO era la causa del
flapeo** — ver punto 12, donde se descarta con evidencia.

> **LECCION IMPORTANTE — el SMS normaliza este dato.** Durante la sesion se
> "corrigio" el documento afirmando que ambos miembros eran `/29`, porque es lo
> que devuelve `cp_mgmt_simple_cluster_facts`. **Era falso:** el SMS almacenaba
> `/29` para los dos mientras CP-GW-B tenia realmente `/30` en Gaia.
>
> **Para datos de interfaz a nivel de SO, el SMS no es fuente de verdad.**
> Contrastar siempre contra el Gaia Portal o `clish`. Ver punto 11.

---

## 4. Correcciones al documento de contexto anterior

Verificado contra la documentacion oficial de `check_point.mgmt` 6.9.0 y contra
el lab. **El documento anterior tenia TRES cosas mal** (la cuarta fila resulto
ser un error de ESTE documento, no del anterior):

| Documento anterior | Realidad |
|---|---|
| `cp_mgmt_api_call` como escape genérico | **No existe** (404 en la doc oficial) |
| `cluster-xl-ls-unicast` | `cluster-ls-unicast` |
| `version:` en `simple_cluster` | `cluster_version:` |
| ~~*"En R81.20 el CCP es unicast por defecto"*~~ | ⚠️ **El documento anterior TENIA RAZON.** Ver punto 12 |

Modulos confirmados como existentes: `cp_mgmt_simple_cluster`,
`cp_mgmt_simple_gateway_facts`, `cp_mgmt_simple_cluster_facts`,
`cp_mgmt_package_facts`, `cp_mgmt_set_api_settings`, `cp_mgmt_show_commands`.

---

## 5. Decisiones de diseño tomadas

- **`ansible_network_os` se fija a nivel de play, nunca en `group_vars`.** El SMS
  se ataca de dos formas (Management API y Gaia); fijarlo global obligaria a
  duplicar hosts en el inventario.
- **`command_timeout = 300`** en `ansible.cfg`. Con los 30s por defecto, `publish`
  e `install-policy` cortan la conexion httpapi a mitad.
- **`cluster.yml` vive en `group_vars/all`**, no en `gateways`: el objeto cluster
  se crea atacando al **SMS**, y ese play necesita ver esas vars.
- ~~**`ansible-vault` desde el dia uno.**~~ **REVISADO el 22/07/2026:** se decidio
  dejar el `vault.yml` **en claro y FUERA de git** (`.gitignore`), en vez de
  cifrarlo y commitearlo. Son credenciales de lab desechables y se gestionan a
  mano en la VM Ansible. Por eso los comandos no llevan `--ask-vault-pass`.
  Si algun dia se cifra, hay que volver a añadir ese flag.
- **Orden de trabajo: descubrimiento → objetos → cluster.** Ver punto 7.

---

## 6. Errores cometidos durante la sesion (y corregidos)

Para que no se repitan:

1. **Inventé el nombre del cluster** (`CL-LAB-SC2N`) sin comprobar que ya existia
   uno. Como `cp_mgmt_simple_cluster` identifica por `name`, lanzarlo asi **no
   habria gestionado el cluster existente: habria creado un duplicado** con la
   misma VIP. Corregido a `CP-CLUSTER-HA`.

2. **Atribui el estado rojo de SmartConsole al desajuste de bonds.** Era
   **licencias**. La correlacion (el miembro rojo era justo el del bond raro) era
   real pero engañosa.

3. **Asumi que los bonds se unificarian a `bond1`.** La topologia real del cluster
   usa `bond2.x`. Corregido.

Leccion aplicable: no dar por buena la configuracion documentada sin contrastarla
con el equipo; en este lab el documento de partida fallaba en 4 puntos.

---

## 7. Estructura creada

```
checkpoint-platform/
├── ansible.cfg                         # timeouts httpapi + stdout_callback corregido
├── requirements.yml                    # check_point.mgmt / check_point.gaia
├── .gitignore                          # .vault_pass, vault.yml, discovery/
├── README.md
├── CONTEXTO-SESION.md                  # este fichero
├── inventories/lab/
│   ├── hosts.ini                       # [management] [gateways] [checkpoint:children]
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── main.yml                # conexion httpapi + redes del lab
│   │   │   ├── cluster.yml             # ClusterXL: VIPs, interfaces, blades
│   │   │   ├── objects.yml             # objetos de prueba + estado present/absent
│   │   │   └── vault.yml.example       # plantilla (vault.yml lo crea el usuario)
│   │   ├── management.yml              # credenciales Management API (grupo [management])
│   │   └── gateways.yml                # credenciales Gaia
│   └── host_vars/{sms,cp-gw-a,cp-gw-b}.yml
└── playbooks/
    ├── 00-discovery.yml                # READ-ONLY
    ├── 01-objects.yml                  # objetos + publish (reversible)
    └── 02-cluster.yml                  # ClusterXL — LANZAR SIEMPRE CON --check
```

**Nota (actualizada 22/07/2026):** el YAML **ya esta validado de verdad**. En la
VM Ansible (ansible-core 2.21.0, `check_point.mgmt` 6.9.0 y `check_point.gaia`
7.0.0) el `--syntax-check` **pasa en los dos playbooks**, lo que confirma ademas
que los modulos `check_point.mgmt.*` usados existen y estan bien escritos.

**Todos los comandos se lanzan desde `checkpoint-platform/`**, que es donde vive
`ansible.cfg` (ya define `inventory`, por eso no hace falta `-i`).

---

## 8. Orden de trabajo acordado

1. **`00-discovery.yml`** (read-only) — valida el camino Ansible → Management API
   y devuelve la topologia que el SMS tiene registrada.
2. **Objetos** (`cp_mgmt_host`, `cp_mgmt_network`) — primera escritura real, pero
   inocua: crear objetos nuevos no toca nada existente y se revierten con
   `state: absent`. Prueba el ciclo completo incluido el `publish`.
3. **Cluster** — solo cuando el paso 1 confirme que `cluster.yml` calca lo que ya
   hay en el SMS.

**Por que NO empezar por el cluster:** `cp_mgmt_simple_cluster` es declarativo. Si
la definicion no coincide exactamente con lo que hay en el SMS, **reescribe la
topologia de un cluster que ahora mismo funciona**. Es la peor pieza para estrenar
el camino de escritura.

---

## 9. Estado de la puesta en marcha (cerrado el 22/07/2026)

Todo lo que estaba pendiente en este punto **ya esta hecho**:

1. ~~Commit + push en GitLab~~ → **Cambio de plan:** el codigo vive ahora en
   **GitHub, `RodriGN99/CheckPoint-Ansible`** (repo personal). GitLab se dejara
   para cuando el proyecto entre en uso real. La duda sobre el acceso de la VM
   al GitLab queda sin efecto: la VM clona de GitHub sin problema.
2. **Colecciones:** instaladas — `check_point.mgmt` 6.9.0, `check_point.gaia` 7.0.0.
3. **Vault:** `vault.yml` creado en la VM. **Decision tomada: se deja EN CLARO y
   fuera de git** (esta en `.gitignore`), en vez de cifrarlo. Por eso los
   comandos ya **no llevan `--ask-vault-pass`**; solo haria falta si algun dia se
   cifra.
4. **API del SMS:** habilitada. El sintoma cuando no lo esta es un **403 con HTML
   de Apache** (no JSON) — si respondiera JSON, el problema serian las
   credenciales, no el filtro por IP. Util para diagnosticar:
   ```bash
   curl -sk -X POST https://192.168.90.170/web_api/login \
     -H "Content-Type: application/json" -d '{"user":"admin","password":"..."}'
   ```
   La IP de la VM Ansible en la red 90 es **192.168.90.5**.
5. **Descubrimiento lanzado y analizado** → ver punto 11.

### Dos tropiezos que costaron tiempo (para no repetirlos)

- **`stdout_callback = yaml` rompia todo.** El callback `community.general.yaml`
  **fue eliminado** en community.general 12.0.0. Con ansible-core 2.21 el
  playbook abortaba antes de arrancar. Sustituto oficial (ansible-core >= 2.13):
  `stdout_callback = default` + `result_format = yaml`. Ya corregido.
- **La contrasena lleva un punto final: `123ABC.123abc.`** No es `123ABC.123abc`.
  El sintoma es `err_login_failed` con **HTTP 400 y JSON** (a diferencia del 403
  con HTML del punto anterior). El mismo usuario `admin` sirve para la Management
  API; no hacia falta una cuenta distinta de SmartConsole.

---

## 10. Fuera de alcance de Ansible (sigue vigente del doc anterior)

- **Licencias** — sin modulo idempotente; via `cplic put` con `cp_mgmt_run_script`.
- **Pre-SSH** — instalacion de ISO, particionado, IP de consola.
- **VSX** — sin modulos para crear Virtual Systems.
- **Blades sin API de politica** — Mobile Access, DLP, QoS, SmartEvent.
- **Ciclo de vida** — Jumbo/upgrades (CPUSE), backups Gaia, `cpconfig` interactivo.
- **Diseño** — sin plan/diff previo, sin estado (no borra lo que desaparece de las
  vars), orden de reglas posicional, rollback = discard de sesion no publicada.

---

## 11. Resultado del descubrimiento (22/07/2026)

`00-discovery.yml` ejecutado con exito contra el SMS: `ok=8, changed=2, failed=0`
(los 2 `changed` son locales — crear `discovery/` y volcar el JSON; en el SMS no
se toco nada). Salida completa en `discovery/discovery-*.json` (ignorado por git).

### Lo que el SMS confirma de `cluster.yml`

| Campo | Valor real en el SMS | ¿Coincide? |
|---|---|---|
| `name` | `CP-CLUSTER-HA` | ✅ |
| `ip_address` (VIP) | `192.168.90.173` | ✅ |
| `cluster_version` | `R81.20` | ✅ |
| `cluster_mode` | `cluster-xl-ha` | ✅ |
| Blades | solo `firewall: true` | ✅ |
| `eth0`, `bond2.10`, `bond2.99` | IP y tipo correctos | ✅ |

Ademas: **SIC `communicating` en los dos miembros**, no hay gateways standalone
(los dos son miembros del cluster) y el unico paquete de politica es `Standard`.
Se confirma que la topologia del cluster usa el naming de CP-GW-A (`bond2.x`)
mientras CP-GW-B tiene `bond1.x`.

### ⚠️ Donde `cluster.yml` NO calca la realidad

Estas dos diferencias hacen que **el playbook de cluster sea destructivo tal cual
esta**. `cp_mgmt_simple_cluster` es declarativo: borraria la IP de sync y añadiria
un `eth4` que no existe, reescribiendo un cluster que ahora mismo funciona.

1. **`eth3` (sync): falta la IP en `cluster.yml`.**
   El SMS tiene `10.255.255.3/29` a nivel de cluster; `cluster.yml` declara `eth3`
   como `sync` **sin IP**.

2. **`eth4`: esta en `cluster.yml` y en los equipos, pero NO en el SMS.**
   ```
   Interfaces del cluster en el SMS: ['bond2.10', 'bond2.99', 'eth0', 'eth3']
   CP-GW-A: ['bond2.10','bond2.99','eth0','eth3']   <- sin eth4
   CP-GW-B: ['bond1.10','bond1.99','eth0','eth3']   <- sin eth4
   ```
   En Gaia **si existe** (`192.168.92.12` / `192.168.92.13`, OOB), pero el objeto
   cluster del SMS no la incluye. Declararla en `cluster.yml` no es un no-op:
   **la añadiria** a la topologia del SMS.

### Divergencia SMS vs equipo real (importante)

El SMS **no es fuente de verdad** para datos de interfaz a nivel de SO. Caso
concreto detectado el 22/07/2026 comparando el Gaia Portal de cada miembro con
lo que devuelve `cp_mgmt_simple_cluster_facts`:

| Dato | Gaia (real) | SMS |
|---|---|---|
| `eth3` CP-GW-A | `10.255.255.1` **/29** | /29 |
| `eth3` CP-GW-B | `10.255.255.2` **/30** | **/29** ← no coincide |
| `eth4` | existe en ambos (OOB) | **no existe** |

Durante la sesion se llego a "corregir" el documento afirmando que ambos miembros
eran `/29` (porque es lo que dice el SMS). **Era un error**: el SMS normaliza ese
campo. La afirmacion original del documento (`/29` vs `/30`) era correcta y queda
restaurada.

**Metodo a seguir:** contrastar SIEMPRE contra Gaia (Portal o `clish`) antes de
dar por buena la topologia que reporta el SMS.

### Aviso de laboratorio: hay mas de un par de Check Points

En el lab conviven al menos dos parejas de gateways con **los mismos hostnames**
(`CP-GW-A` / `CP-GW-B`):

- **`192.168.90.171` / `.172`** → los miembros **reales** de `CP-CLUSTER-HA`.
  Tienen bonds (`bond2.x` / `bond1.x`) y `eth4` OOB.
- **`192.168.90.181` / `.182`** → **otra pareja distinta**, sin bonds, con `eth1`
  en `192.168.100.252/253`. MACs de otro rango (`50:01:xx` frente a `50:00:xx`).

Comprobacion rapida para no confundirlos: la VIP `.173` responde con **la misma
MAC que el miembro ACTIVE** (hoy `.172`, MAC `50:00:00:09:00:00`).

**No usar los `.181/.182` como referencia para `cluster.yml`.**

**Conclusion:** el orden "descubrimiento antes que cluster" (punto 8) ha hecho
exactamente su trabajo. **No lanzar el paso 3 hasta corregir `cluster.yml`.**
Correccion pendiente por parte del usuario, decidiendo antes que se quiere como
estado final (el SMS y los equipos no coinciden, asi que hay que elegir).

---

## 12. Flapeo del cluster: diagnostico corregido (22/07/2026)

**Esta seccion sustituye al diagnostico anterior, que era erroneo.** Las dos
hipotesis que manejaba el documento estan ahora DESCARTADAS por evidencia.

### Hipotesis 1 (DESCARTADA): "el CCP va en multicast"

El documento deducia esto de que `cphaprob state` reporta
*"High Availability (Active Up) with IGMP Membership"*, y proponia arreglarlo con
`cphaconf set_ccp unicast`. **Las tres partes de ese razonamiento son falsas:**

- **`set_ccp` no existe en R81.20.** Buscado en las 378 paginas de la guia oficial
  de ClusterXL R81.20: cero apariciones. Los unicos `cphaconf ccp_*` documentados
  son `ccp_encrypt` y `ccp_encrypt_key`, que son de **cifrado**, no de transporte.
  El comando `set_ccp {auto|unicast|multicast|broadcast}` es de R80.x.
- **En R81.x el CCP ya va siempre en unicast.** La guia de R81.10 lo dice literal:
  *"In R81.10, the CCP always runs in the unicast mode."* Ya no es configurable.
- **`"with IGMP Membership"` es la salida NORMAL.** La propia guia de R81.20 usa
  esa cadena exacta en su ejemplo de un cluster sano (pag. 252). No indica nada
  anomalo ni que el CCP vaya en multicast.

**No lanzar `cphaconf set_ccp`.** No existe y la causa que pretendia arreglar no
es real.

### Hipotesis 2 (DESCARTADA): "la mascara de sync /29 vs /30"

Al caerse la hipotesis del multicast, la mascara inconsistente parecia el
candidato natural. **`cphaprob show_failover` la descarta**: en los 20 failovers
registrados, **ni uno solo** menciona `eth3` (sync) ni `eth0`. Si el sync fuera
el problema, apareceria en los motivos.

Unificarla a `/29` fue correcto como higiene, pero **no explica el flapeo**.

### El patron real: solo los bonds

Salida de `cphaprob show_failover` en CP-GW-A (22/07/2026):

```
Failover counter: 193      (reset: Wed Jul 1 09:00:14 2026, reboot)
Ultimo evento:    Wed Jul 22 11:01:42 2026
Motivo:           Interface bond2.99 is down
                  (Cluster Control Protocol packets are not received)
```

Los 20 failovers del historial son **todos** en `bond2.10`, `bond2.99` o `bond2`,
siempre con el mismo motivo: no se reciben paquetes CCP. **Nunca en `eth3` ni en
`eth0`.**

La guia de ClusterXL R81.20 (pag. 179) describe exactamente este sintoma:

> *"If a Bond interface goes down on one Cluster Member... the peer Cluster
> Members **stop receiving the CCP packets on that Bond interface** and cannot
> probe the local network to determine that their Bond interface is really
> working."*

### La configuracion de bonds en Check Point esta BIEN

Verificado el 22/07/2026 con `cphaprob show_bond` y `show bonding groups` en
ambos miembros. Son identicos:

| | CP-GW-A | CP-GW-B |
|---|---|---|
| Modo | `8023AD` (LACP) | `8023AD` (LACP) |
| Subordinadas | `eth1`, `eth2` | `eth1`, `eth2` |
| xmit-hash-policy | `layer2` | `layer2` |
| lacp-rate | `slow` | `slow` |
| Timers | up/down-delay 200, mii 100 | idem |
| Estado | UP, 2/2 slaves link up | UP, 2/2 slaves link up |

La guia exige que las subordinadas se añadan **en el mismo orden** en todos los
miembros (pag. 166): **se cumple**. Solo difiere el ID del bond (`bond2` vs
`bond1`), que es cosmetico.

**Conclusion: el lado Check Point no es el problema.** Hay que mirar hacia fuera.

### Sospechoso actual: LACP sobre EVE-NG + ArubaCX

Queda como hipotesis principal, **aun sin verificar**, que el problema este en el
lado del switch / la virtualizacion:

- Los bonds usan **802.3ad (LACP)**, que exige que el otro extremo tenga un LAG
  configurado **y que negocie LACPDUs correctamente**. Los switches virtuales de
  EVE-NG no siempre lo hacen bien.
- **Riesgo concreto a comprobar en el ArubaCX:** que las interfaces de LOS DOS
  gateways esten metidas en el **mismo LAG**, en vez de en **dos LAGs separados**
  (uno por gateway). Ese error provoca exactamente este cuadro: MACs saltando
  entre puertos y CCP perdiendose.
- `lacp-rate slow` (LACPDU cada 30s, timeout 90s) es lento para detectar
  problemas; `fast` daria deteccion mas rapida, pero es sintoma, no causa.

### Dato temporal a confirmar

El ultimo failover fue a las **11:01:42** del 22/07. A las **21:24** del mismo dia
seguia sin haber ninguno: **mas de 10 horas limpias**, cuando el ritmo previo era
de 20 failovers en 4,5 horas (uno cada ~13 min). **Algo corto el flapeo sobre las
11:00.** No esta confirmado que fuera el cambio de mascara — se desconoce la hora
exacta en que se aplico — pero el cambio de regimen es real.

### Como medir a partir de ahora

El contador de 193 arrastra tres semanas. Para medir limpio:

```bash
cphaprob -reset -c show_failover     # resetear el contador
cphaprob show_failover               # historial CON el motivo de cada failover
cphaprob -l 20 show_failover         # los ultimos 20
cphaprob syncstat                    # estadisticas de Delta Sync
cphaprob igmp                        # estado real de la membresia IGMP
cphaprob show_bond <bond>            # estado del bond
cphaprob -a if                       # interfaces monitorizadas
```

**Linea base historica: 193 failovers** entre el 01/07 (reboot) y el 22/07/2026.
