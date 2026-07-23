# checkpoint-platform

Plataformado con Ansible de Check Point R81.20 (SMS + ClusterXL) sobre el lab de EVE-NG.

**Scope actual: solo Check Point.** FortiGate, Cisco C9K, ArubaCX y los clientes
Windows/Docker quedan deliberadamente fuera.

## Topologia

Datos tomados del Gaia Portal de cada equipo.

| Rol | Host | eth0 (red 90) | eth3 (sync) | eth4 (OOB) | Etiqueta EVE-NG |
|---|---|---|---|---|---|
| Management | `sms` | 192.168.90.170 | — | 192.168.92.11 *(en eth3)* | SMS |
| Miembro A | `cp-gw-a` | 192.168.90.171 | 10.255.255.1 | 192.168.92.12 | **CP Bound 2** |
| Miembro B | `cp-gw-b` | 192.168.90.172 | 10.255.255.2 | 192.168.92.13 | **CP Bound 1** |
| VIP cluster | — | **192.168.90.173** | — | — | — |

Interior (bond hacia ArubaCX):

| VLAN | Red | CP-GW-A | CP-GW-B | VIP |
|---|---|---|---|---|
| 10 | 192.168.10.0/24 | .3 (`bond2.10`) | .2 (`bond1.10`) | **192.168.10.1** |
| 99 | 192.168.99.0/24 | .3 (`bond2.99`) | .2 (`bond1.99`) | **192.168.99.1** |

VIPs confirmadas con `cphaprob -a if`. La topologia del cluster usa el naming de
**CP-GW-A** (`bond2.x`), aunque CP-GW-B tenga fisicamente `bond1.x`.
`eth4` (OOB) no aparece en `cphaprob -a if` porque esta como `private` y ClusterXL
no monitoriza esas interfaces — es lo esperado.

- **SIC y gestion:** por la **red 90** (Management Interface = `eth0`).
- **Red 90** (192.168.90.0/24, gw .1): plana, con salida a Internet; ahi vive tambien la VM Ansible.
- **OOB:** 192.168.92.0/24, VLAN 920.

> Ojo con el mapeo: `cp-gw-a` es **"CP Bound 2"** y `cp-gw-b` es **"CP Bound 1"**.
> El orden del diagrama va al reves de lo que sugiere el nombre.

## Estado actual del lab

El cluster **ya existe** en el SMS como **`CP-CLUSTER-HA`** (VIP 192.168.90.173), con
CP-GW-A y CP-GW-B como miembros.

En SmartConsole, CP-CLUSTER-HA y CP-GW-A aparecen en rojo: **es un problema de
licencias** en CP-GW-A, no de topologia. Hasta resolverlo, el cluster no llega a
formarse del todo.

> El nombre del objeto importa: `cp_mgmt_simple_cluster` identifica por `name`. Usar
> un nombre distinto de `CP-CLUSTER-HA` no gestionaria el cluster existente — crearia
> **un duplicado** con la misma VIP.

### Salud del cluster: funciona, pero mejorable

Verificado con `cphaprob state` / `cphaprob -a if` / `cphaprob list`:
CP-GW-B ACTIVE, CP-GW-A STANDBY, todas las interfaces UP, **sin pnotes**.
El cluster esta formado y operativo. Lo de abajo son mejoras, no fallos:

1. **Flapeo: 135 failovers** entre el 1 y el 21 de julio de 2026. El motivo del ultimo:
   `Interface bond2.10 is down (Cluster Control Protocol packets are not received)`.

2. **El CCP va en multicast**, no unicast: `cphaprob state` reporta
   *"High Availability (Active Up) with **IGMP Membership**"*. Multicast sobre los
   switches virtuales de EVE-NG + ArubaCX es fragil (sin IGMP querier, o con snooping
   podando), y encaja con que el CCP se pierda justo en `bond2.10`, el camino con mas
   conmutacion por medio. Candidato principal para el flapeo.
   Se cambia con `cphaconf set_ccp unicast` en cada miembro (verificar sintaxis exacta
   en la version antes de aplicar).

3. **Nombres de bond distintos** (`bond2` en A, `bond1` en B) y **mascara de sync
   distinta** (`/29` en A, `/30` en B). No impiden que el cluster funcione, pero son
   desajustes que conviene unificar.

## Estructura

```
checkpoint-platform/
├── ansible.cfg                         # define inventory -> no hace falta pasar '-i'
├── requirements.yml                    # colecciones check_point.mgmt / check_point.gaia
├── .gitignore                          # .vault_pass, vault.yml, discovery/
├── inventories/lab/
│   ├── hosts.ini                       # grupos [management] y [gateways]
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── main.yml                # conexion httpapi + redes del lab
│   │   │   ├── cluster.yml             # definicion logica del ClusterXL
│   │   │   ├── objects.yml             # objetos de prueba + estado present/absent
│   │   │   ├── policy.yml              # reglas de acceso (contiene un accept any/any de prueba)
│   │   │   ├── vault.yml               # credenciales (lo creas tu, NO esta en el repo)
│   │   │   └── vault.yml.example       # plantilla
│   │   ├── management.yml              # credenciales Management API (grupo [management])
│   │   └── gateways.yml                # credenciales Gaia
│   └── host_vars/{sms,cp-gw-a,cp-gw-b}.yml
└── playbooks/
    ├── 00-discovery.yml                # READ-ONLY
    ├── 01-objects.yml                  # primera escritura: objetos + publish (reversible)
    ├── 02-cluster.yml                  # ClusterXL declarativo — LANZAR SIEMPRE CON --check
    └── 03-policy.yml                   # reglas de acceso + publish (NO instala politica)
```

> El grupo se llama `[management]` (no `[sms]`) para no colisionar con el host
> `sms`. Los plays siguen apuntando al host con `hosts: sms`.

## Puesta en marcha (en la VM Ansible)

> **Todos los comandos se lanzan desde `CheckPoint-Ansible/checkpoint-platform/`.**
> Es donde vive `ansible.cfg`, que ya define `inventory = inventories/lab/hosts.ini`.
> Fuera de ese directorio Ansible no lee ese `ansible.cfg` y habria que pasar
> `-i inventories/lab/hosts.ini` a mano.

```bash
git clone https://github.com/RodriGN99/CheckPoint-Ansible.git
cd CheckPoint-Ansible/checkpoint-platform

# 1. Colecciones
ansible-galaxy collection install -r requirements.yml

# 2. Credenciales (vault.yml esta en .gitignore, no viene en el clon)
cp inventories/lab/group_vars/all/vault.yml.example \
   inventories/lab/group_vars/all/vault.yml
# editar con los valores reales.
# Opcional, si lo quieres cifrado (entonces anade --ask-vault-pass a los plays):
#   ansible-vault encrypt inventories/lab/group_vars/all/vault.yml

# 3. Validar sintaxis (no conecta con el lab)
ansible-playbook playbooks/00-discovery.yml --syntax-check
ansible-playbook playbooks/01-objects.yml  --syntax-check

# 4. Descubrimiento READ-ONLY (primero esto, siempre)
#    Ademas de listar, comprueba que el SIC sigue 'communicating' en todos los
#    equipos. Es lo unico que puede hacer fallar este playbook.
ansible-playbook playbooks/00-discovery.yml

# 5. Objetos: primera escritura real, reversible
ansible-playbook playbooks/01-objects.yml
#    rollback:
ansible-playbook playbooks/01-objects.yml -e cp_objects_state=absent

# 6. Cluster: SIEMPRE en simulacion primero. Solo ejecutar si da changed=0.
ansible-playbook playbooks/02-cluster.yml --check
ansible-playbook playbooks/02-cluster.yml

# 7. Politica: crea las reglas y publica. NO instala nada en los gateways.
ansible-playbook playbooks/03-policy.yml
#    rollback:
ansible-playbook playbooks/03-policy.yml -e cp_policy_state=absent
```

`00-discovery.yml` **no cambia nada**: valida el camino Ansible -> Management API
y lista lo que ya existe en el SMS (gateways, clusters, paquetes de politica).
Vuelca el resultado a `discovery/` (ignorado por git).

Al final comprueba el **SIC** de miembros de cluster y gateways, y **falla si
alguno no esta `communicating`**. No hace llamadas extra: `sic-state` ya viene en
los facts que se piden con `details_level: full`. El chequeo va despues del
volcado a JSON a proposito, para que el fichero de `discovery/` se genere aunque
el assert falle.

> Ese chequeo **no valida `vault_cp_sic_key`**. Esa clave es la one-time password
> del alta; con el SIC establecido se sustituye por certificados y ya no hay forma
> de comprobarla sin resetear el trust.

Si falla la conexion, lo mas probable es que la API no acepte llamadas desde la IP
de la VM Ansible. En el SMS:

```bash
mgmt_cli set api-settings accepted-api-calls-from \
  "all ip addresses that can be used for gui clients" --domain "System Data" -r true
api restart && api status   # esperar "API readiness test SUCCESSFUL"
```

## Politica de acceso

`03-policy.yml` crea reglas en la capa **`Network`** del paquete **`Standard`**
(nombres reales del SMS) y hace `publish`. **No instala politica.**

> **Publicar no es instalar.** `publish` consolida los cambios en la base de datos
> del SMS: la regla se ve en SmartConsole, pero los gateways **no la aplican**.
> El trafico no cambia hasta un `install-policy`, que este repo no hace.

Hoy solo hay una regla, `ans-test-rule-1`, que es un **`accept any/any/any`**
creado como prueba de humo. Verificado idempotente (segunda pasada `changed=0`).

> **Borrarla antes de anadir cualquier `install-policy`.** Instalada, abre el
> firewall entero.
> ```bash
> ansible-playbook playbooks/03-policy.yml -e cp_policy_state=absent
> ```

## Convenciones

- **`ansible_network_os` se fija a nivel de play, nunca en `group_vars`.** El SMS se
  ataca de dos formas (Management API y Gaia) y fijarlo globalmente obligaria a
  duplicarlo en el inventario.
- **Datos en vars, nunca en tareas.** Plataformar algo nuevo = tocar `host_vars` +
  inventario, no el playbook.
- **Secretos en `ansible-vault` desde el dia uno.** El `vault.yml` cifrado si se
  commitea; la contrasena del vault (`.vault_pass`) nunca — esta en `.gitignore`.

## Pendiente

1. **Licencias de CP-GW-A** — causa del estado rojo en SmartConsole.
2. **Borrar `ans-test-rule-1`** (`accept any/any`) antes de que exista cualquier
   `install-policy`. Ver la seccion "Politica de acceso".
3. **`install-policy`** — el paso que falta para que los gateways apliquen algo.
   Solo cuando haya reglas reales.

Resuelto: el blade **`vpn` SI existe** en `simple_cluster` (leido del objeto real
el 23/07/2026). Declarado a `false` en `cluster.yml`: no se usa VPN ni en lab ni
en produccion.

## Notas sobre la coleccion

Verificado contra la doc oficial de `check_point.mgmt` 6.9.0:

- `cp_mgmt_api_call` **no existe** (aparece en documentacion de terceros, pero da 404).
- El modo de cluster es **`cluster-ls-unicast`**, no `cluster-xl-ls-unicast`.
- El parametro de version en `cp_mgmt_simple_cluster` es **`cluster_version`**, no `version`.
- Confirmados: `cp_mgmt_simple_cluster`, `cp_mgmt_simple_gateway_facts`,
  `cp_mgmt_simple_cluster_facts`, `cp_mgmt_package_facts`, `cp_mgmt_set_api_settings`,
  `cp_mgmt_show_commands`.
