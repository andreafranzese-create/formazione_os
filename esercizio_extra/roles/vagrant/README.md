# Ruolo `vagrant`

Crea e avvia le VM con Vagrant **direttamente da Ansible**, senza lanciare `vagrant up` a mano. Poi genera l'inventory, così il resto del playbook può collegarsi alle VM.

Il ruolo gira sulla **macchina di controllo** (`hosts: localhost`, `gather_facts: false`).

---

## Task

| # | Task | Modulo | Cosa fa |
|---|---|---|---|
| 1 | **Directory di lavoro per Vagrant** | `ansible.builtin.file` | Crea la cartella `vagrant_workdir` (di default `vms/`), dove finiscono il `Vagrantfile`, la cartella `.vagrant/` con le chiavi SSH e i log `vagrant.out` / `vagrant.err`. |
| 2 | **Stato del Vagrantfile prima** | `ansible.builtin.stat` | Legge il checksum del `Vagrantfile` attuale (se esiste) e lo salva in `vagrantfile_before`. Serve per capire dopo se la configurazione è cambiata. |
| 3 | **Crea e avvia le VM** | `community.vagrant.vagrant` | Genera il `Vagrantfile` a partire dalla lista `vms` e fa l'equivalente di `vagrant up`. Se le VM esistono già e sono accese non fa nulla. La variabile d'ambiente `MOLECULE_EPHEMERAL_DIRECTORY` dice al modulo in quale cartella lavorare (il modulo nasce da Molecule e usa quel nome). |
| 4 | **Stato del Vagrantfile dopo** | `ansible.builtin.stat` | Rilegge il checksum del `Vagrantfile` e lo salva in `vagrantfile_after`. |
| 5 | **Riavvia le VM per applicare la nuova configurazione** | `ansible.builtin.command` | Esegue `vagrant reload` **solo se** il `Vagrantfile` esisteva già e il checksum è cambiato (es. hai modificato RAM, CPU o IP). Il modulo `vagrant` da solo non riapplica le modifiche a VM già accese. |
| 6 | **Directory per l'inventory** | `ansible.builtin.file` | Crea la cartella che conterrà il file di inventory (`inventory/`). |
| 7 | **Genera l'inventory delle VM** | `ansible.builtin.template` | Dal template `inventory.j2` scrive `inventory/vagrant.ini`: un gruppo per ogni VM, con IP, utente `vagrant` e path della chiave privata generata da Vagrant. |
| 8 | **Imposta l'inventory in ansible.cfg** | `community.general.ini_file` | Scrive in `ansible.cfg`, sezione `[defaults]`, `inventory = <vagrant_inventory_file>`. Così i comandi lanciati dopo (`ansible all -m ping`, ecc.) usano già l'inventory giusto. |
| 9 | **Aggiungi le VM all'inventory in memoria** | `ansible.builtin.add_host` | Aggiunge le VM all'inventory del run in corso, ognuna in un gruppo con il suo nome. Serve perché l'inventory viene letto all'avvio di `ansible-playbook`: senza questo passaggio, al primo lancio il secondo play (`hosts: all`) non troverebbe nessuna VM. |

---

## Template

### `templates/inventory.j2`
Genera l'inventory in formato INI:

- `[all:vars]` → utente `vagrant` e opzioni SSH che disattivano il controllo delle host key (le VM vengono ricreate spesso e cambierebbero chiave).
- Un gruppo per ogni VM della lista `vms`, con `ansible_host` (il primo IP in `interfaces`) e `ansible_ssh_private_key_file` (`<vagrant_workdir>/.vagrant/machines/<nome>/<provider>/private_key`).

---

## Variabili (`defaults/main.yml`)

| Variabile | Default | Significato |
|---|---|---|
| `vagrant_workdir` | `"{{ playbook_dir }}/vms"` | Cartella in cui Vagrant lavora: contiene `Vagrantfile`, `.vagrant/` e i log. |
| `vagrant_provider` | `virtualbox` | Provider di Vagrant. Si usa sia per creare le VM sia per costruire il path della chiave SSH. |
| `vagrant_box` | `generic/rocky9` | Box usata per tutte le VM (Rocky Linux 9). |
| `vagrant_inventory_file` | `"{{ playbook_dir }}/inventory/vagrant.ini"` | Path dell'inventory generato, scritto anche in `ansible.cfg`. |
| `vms` | vedi sotto | Lista delle VM da creare. |

### Struttura della lista `vms`

```yaml
vms:
  - name: elasticsearch-host      # nome della VM, hostname e nome del gruppo nell'inventory
    memory: 1024                  # RAM in MB
    cpus: 1                       # numero di vCPU
    interfaces:
      - network_name: private_network   # tipo di rete Vagrant (rete host-only)
        ip: 192.168.56.10               # IP statico della VM
```

| Campo | Significato |
|---|---|
| `name` | Nome della VM in Vagrant. Diventa anche l'hostname, il nome dell'host e il nome del gruppo nell'inventory. |
| `memory` | RAM assegnata in MB. |
| `cpus` | Numero di CPU virtuali. |
| `interfaces[].network_name` | Tipo di rete Vagrant; `private_network` crea una rete host-only raggiungibile dalla macchina di controllo. |
| `interfaces[].ip` | IP statico. Il **primo** IP della lista è quello usato da Ansible per collegarsi. |