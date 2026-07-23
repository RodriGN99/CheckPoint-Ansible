# CheckPoint-Ansible — instrucciones de proyecto

Automatizacion de un lab Check Point (**SMS + ClusterXL R81.20**) con Ansible.
El historial completo, con las hipotesis descartadas y por que, esta en
`checkpoint-platform/CONTEXTO-SESION.md`. **Leelo antes de tocar el cluster.**

Fuera de alcance a proposito: FortiGate, Cisco C9K, ArubaCX, clientes Windows/Docker.

---

## Reglas duras

### 1. Todos los comandos se lanzan desde `checkpoint-platform/`

Ahi vive `ansible.cfg`, que ya define `inventory = inventories/lab/hosts.ini`.
Fuera de ese directorio Ansible no lo lee y haria falta pasar `-i` a mano.

```bash
cd ~/CheckPoint-Ansible/checkpoint-platform
ansible-playbook playbooks/00-discovery.yml
```

### 2. Orden de trabajo: descubrimiento -> objetos -> cluster

Nunca al reves. `00-discovery.yml` es read-only y valida que la definicion local
calca lo que el SMS tiene registrado.

### 3. El cluster SIEMPRE con `--check` primero

`cp_mgmt_simple_cluster` es **declarativo** sobre un cluster en produccion. Si la
definicion no calca el SMS, **reescribe la topologia de un cluster que funciona**.

```bash
ansible-playbook playbooks/00-discovery.yml          # 1. estado real
ansible-playbook playbooks/02-cluster.yml --check    # 2. debe dar changed=0
ansible-playbook playbooks/02-cluster.yml            # 3. solo si el check limpio
```

`02-cluster.yml` **no envia `members` a proposito**: el helper `api_call` compara
con una llamada `equals` que solo evalua los campos enviados, asi que omitirlos
los deja intactos. Enviarlos exigiria la clave SIC y reescribiria la pertenencia
al cluster.

### 4. El SMS NO es fuente de verdad para interfaces

Aprendido por las malas: el SMS **normaliza** datos de interfaz. Reportaba `/29`
en la mascara de sync de los dos miembros cuando CP-GW-B tenia `/30` en Gaia.

**Para cualquier dato a nivel de SO, contrastar contra el Gaia Portal o `clish`,
no contra `cp_mgmt_*_facts`.**

### 5. Secretos

`vault_cp_sic_key` es la **one-time password del alta**, no una credencial de
gestion. Con el SIC ya `communicating` no hay forma de validarla: solo se usaria
al dar de alta un gateway nuevo o tras un reset de SIC. **No "probarla" mandando
`members` en `cp_mgmt_simple_cluster`** — eso no comprueba nada, rompe el trust.

`inventories/lab/group_vars/all/vault.yml` va **en claro y en `.gitignore`**. Se
gestiona a mano en la VM Ansible y **nunca se commitea**. Por eso los comandos no
llevan `--ask-vault-pass`. Si algun dia se cifra con `ansible-vault`, hay que
volver a añadir ese flag.

**No escribir credenciales en ficheros versionados** (este incluido).

---

## Trampas conocidas

| Trampa | Realidad |
|---|---|
| `color: green` | **No es valido** en Check Point. Si lo son `dark green`, `forest green`, `light green`, `sea green` |
| `stdout_callback = yaml` | Eliminado en community.general 12. Usar `default` + `result_format = yaml` |
| `version:` en simple_cluster | Es **`cluster_version:`** |
| `cluster-xl-ls-unicast` | Es **`cluster-ls-unicast`** |
| `cp_mgmt_api_call` | **No existe** |
| `cphaconf set_ccp unicast` | **No existe en R81.20.** En R81.x el CCP ya va siempre unicast. No proponerlo |
| `"with IGMP Membership"` en `cphaprob state` | Es la salida **NORMAL** de un cluster sano, no un sintoma |
| Grupo `[sms]` en el inventario | Se llama **`[management]`** (colisionaba con el host `sms`) |
| `cp_mgmt_access_rulebase_facts` | **No existe.** Es `cp_mgmt_access_rule_facts` (y no acepta `limit`) |
| `cp_mgmt_export_access_rulebase` | El modulo existe pero la API devuelve **404**: no esta en esta version |
| `package_facts` -> `.objects` | Es **`ansible_facts.packages.packages`**. Los demas `*_facts` si usan `.objects` |
| Variables en el `name:` de un play | Se evalua **antes** de cargar `group_vars` -> `undefined`. Idem en el `name:` de una tarea con `loop` (`item`): usar `loop_control.label` |
| `position: top` / `bottom` | La doc avisa de que **puede** no ser idempotente. Verificado que aqui SI lo es. Para reglas de verdad, usar `relative_position` |

### Hay DOS parejas de gateways con los mismos hostnames

- **`192.168.90.171` / `.172`** -> miembros **reales** de `CP-CLUSTER-HA`. Con
  bonds (`bond2.x` / `bond1.x`) y `eth4` OOB.
- **`192.168.90.181` / `.182`** -> **otra pareja distinta**. Sin bonds, MACs
  `50:01:xx` en vez de `50:00:xx`.

Para distinguirlas: la VIP `.173` responde con **la misma MAC que el miembro
ACTIVE**. No usar los `.181/.182` como referencia para nada.

### Naming de bonds

La topologia del cluster en el SMS usa el naming de **CP-GW-A (`bond2.x`)**,
aunque CP-GW-B tenga fisicamente `bond1.x`. Es correcto, no "arreglarlo".

---

## Playbooks

| Playbook | Que hace | Riesgo |
|---|---|---|
| `00-discovery.yml` | Lista gateways, clusters y paquetes. Vuelca a `discovery/`. Verifica el SIC | Ninguno (read-only), pero **falla si algun SIC no esta `communicating`** |
| `01-objects.yml` | Crea objetos + `publish` | Bajo. Reversible: `-e cp_objects_state=absent` |
| `02-cluster.yml` | ClusterXL declarativo | **Alto.** Siempre `--check` primero |
| `03-policy.yml` | Reglas de acceso + `publish` | Bajo mientras no se instale. Reversible: `-e cp_policy_state=absent` |

Nada instala politica todavia: `publish` solo consolida en la base de datos del
SMS. **Los gateways no reciben nada hasta un `install-policy`**, que aun no
existe en el proyecto.

`cp_access_rules` esta **vacia a proposito**: `03-policy.yml` no crea nada por si
solo. El ciclo se probo con una regla de humo (`accept any/any/any`) el
23/07/2026 y se borro; quedo verificado crear -> idempotente -> borrar ->
idempotente. **No volver a dejar un `accept any/any` declarado en el repo**: el
dia que exista un `install-policy`, abre el firewall entero. El playbook avisa
si detecta uno, pero solo avisa.

Al anadir reglas reales, usar **`relative_position`** (`above`/`below` respecto a
una regla con nombre), no `position: top|bottom`.

---

## Convenciones

- **No commitear a `main` directamente.** Rama -> PR -> merge.
- `discovery/` esta en `.gitignore` (puede contener topologia).
- Si se corrige un dato de `CONTEXTO-SESION.md`, dejar constancia de **que era
  falso y con que evidencia se desmiente**. Ese documento ya ha inducido a error
  varias veces por afirmar causas sin verificar.
