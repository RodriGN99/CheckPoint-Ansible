# Plataformado Check Point con Ansible — Contexto de sesion

**Fecha:** 2026-07-21
**Repo:** `sc2n` (git.gmv.es) · **Rama:** `dev-checkpoint` (la rama `dev` no se toca)
**Ruta del proyecto:** `Ansible/checkpoint-platform/`

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
| Miembro B | `cp-gw-b` | 192.168.90.172 | 10.255.255.2 **/30** | 192.168.92.13 | **CP Bound 1** |
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
- **`eth4` no aparece en `cphaprob -a if`** porque esta como `private`, y
  ClusterXL no monitoriza esas interfaces. Es lo esperado, no un fallo.
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

Desajustes menores entre miembros, sin efecto demostrado: nombres de bond
distintos (`bond2` vs `bond1`) y mascara de sync distinta (`/29` vs `/30`).

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
│   │   │   └── vault.yml.example       # plantilla (vault.yml lo crea el usuario)
│   │   ├── sms.yml                     # credenciales Management API
│   │   └── gateways.yml                # credenciales Gaia
│   └── host_vars/{sms,cp-gw-a,cp-gw-b}.yml
└── playbooks/
    └── 00-discovery.yml                # READ-ONLY
```

**Nota:** el YAML **no esta validado de verdad**. En el equipo Windows no hay
Python (solo el stub de Microsoft Store), asi que solo se comprobo estructura
basica (sin tabs, sin BOM). La primera validacion real sera el `--syntax-check`
en la VM Ansible.

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

## 9. Pendiente del usuario

1. **Commit + push** de `Ansible/checkpoint-platform/` en `dev-checkpoint`, y
   llevar el codigo a la VM Ansible (`git pull` alli).
   ❓ *Sin confirmar: si la VM Ansible tiene acceso al GitLab. Si no, habra que
   copiar por scp.*
2. **Colecciones:** `ansible-galaxy collection install -r requirements.yml`
3. **Vault:** copiar `vault.yml.example` → `vault.yml`, rellenar y
   `ansible-vault encrypt`. Para el descubrimiento bastan
   `vault_cp_mgmt_api_user` y `vault_cp_mgmt_api_password`.
4. **API del SMS:** comprobar con `api status` que da
   `API readiness test SUCCESSFUL` y que acepta llamadas desde la IP de la VM
   Ansible. Si no:
   ```bash
   mgmt_cli set api-settings accepted-api-calls-from \
     "all ip addresses that can be used for gui clients" --domain "System Data" -r true
   api restart && api status
   ```
5. **Lanzar y compartir la salida:**
   ```bash
   ansible-playbook playbooks/00-discovery.yml --syntax-check
   ansible-playbook playbooks/00-discovery.yml --ask-vault-pass
   ```

---

## 10. Fuera de alcance de Ansible (sigue vigente del doc anterior)

- **Licencias** — sin modulo idempotente; via `cplic put` con `cp_mgmt_run_script`.
- **Pre-SSH** — instalacion de ISO, particionado, IP de consola.
- **VSX** — sin modulos para crear Virtual Systems.
- **Blades sin API de politica** — Mobile Access, DLP, QoS, SmartEvent.
- **Ciclo de vida** — Jumbo/upgrades (CPUSE), backups Gaia, `cpconfig` interactivo.
- **Diseño** — sin plan/diff previo, sin estado (no borra lo que desaparece de las
  vars), orden de reglas posicional, rollback = discard de sesion no publicada.
