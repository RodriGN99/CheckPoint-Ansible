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
| Miembro B | `cp-gw-b` | 192.168.90.172 | 10.255.255.2 **/29** | 192.168.92.13 | **CP Bound 1** |
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
- **`eth4` no aparece en `cphaprob -a if`**. Al principio se atribuyo a que
  estaba como `private` (ClusterXL no monitoriza esas). El descubrimiento del
  22/07/2026 muestra que la razon es mas simple: **el SMS no conoce `eth4` en
  absoluto**, ni en el objeto cluster ni en ningun miembro. Ver punto 11.
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

2. **Flapeo: 135 failovers** entre el 1 y el 21 de julio de 2026. Motivo del
   ultimo: `Interface bond2.10 is down (Cluster Control Protocol packets are
   not received)`.
   **Causa probable: el CCP va en multicast**, no unicast — `cphaprob state`
   reporta *"High Availability (Active Up) with **IGMP Membership**"*. Multicast
   sobre los switches virtuales de EVE-NG + ArubaCX es fragil (sin IGMP querier,
   o con snooping podando), y encaja con que se pierda justo en `bond2.10`, el
   camino con mas conmutacion por medio.
   Se cambiaria con `cphaconf set_ccp unicast` en cada miembro (verificar la
   sintaxis exacta en la version antes de aplicar).

Desajuste menor entre miembros, sin efecto demostrado: nombres de bond distintos
(`bond2` vs `bond1`).

> **CORREGIDO (22/07/2026):** este documento afirmaba tambien que la mascara de
> sync era distinta (`/29` vs `/30`). **Es falso.** El descubrimiento contra el
> SMS devuelve `/29` en los dos miembros: `CP-GW-A 10.255.255.1/29` y
> `CP-GW-B 10.255.255.2/29`. Ese desajuste no existe.

---

## 4. Correcciones al documento de contexto anterior

Verificado contra la documentacion oficial de `check_point.mgmt` 6.9.0 y contra
el lab. **El documento anterior tenia cuatro cosas mal:**

| Documento anterior | Realidad |
|---|---|
| `cp_mgmt_api_call` como escape genérico | **No existe** (404 en la doc oficial) |
| `cluster-xl-ls-unicast` | `cluster-ls-unicast` |
| `version:` en `simple_cluster` | `cluster_version:` |
| *"En R81.20 el CCP es unicast por defecto"* | **Este cluster va en multicast** (IGMP Membership) |

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
- **`ansible-vault` desde el dia uno.** El `vault.yml` cifrado si se commitea; la
  contrasena del vault (`.vault_pass`) nunca — esta en `.gitignore`.
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
Ansible/checkpoint-platform/
├── ansible.cfg                         # timeouts httpapi ampliados
├── requirements.yml                    # check_point.mgmt / check_point.gaia
├── .gitignore                          # .vault_pass, discovery/
├── README.md
├── CONTEXTO-SESION.md                  # este fichero
├── inventories/lab/
│   ├── hosts.ini                       # [sms] [gateways] [checkpoint:children]
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
    └── 01-objects.yml                  # objetos + publish (reversible)
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

2. **`eth4`: esta en `cluster.yml` pero el SMS no lo conoce.**
   ```
   Interfaces del cluster en el SMS: ['bond2.10', 'bond2.99', 'eth0', 'eth3']
   CP-GW-A: ['bond2.10','bond2.99','eth0','eth3']   <- sin eth4
   CP-GW-B: ['bond1.10','bond1.99','eth0','eth3']   <- sin eth4
   ```
   Puede existir en Gaia a nivel de SO, pero **el objeto cluster del SMS no lo
   tiene**. Esta es la razon real de que no salga en `cphaprob -a if` (ver punto 2).

**Conclusion:** el orden "descubrimiento antes que cluster" (punto 8) ha hecho
exactamente su trabajo. **No lanzar el paso 3 hasta corregir `cluster.yml`.**
Correccion pendiente por parte del usuario: añadir `10.255.255.3/29` a `eth3` y
quitar `eth4`.
