---
marp: true
theme: formation
paginate: true
lang: fr
title: Ansible — automatiser un parc, sans agent
description: Formation Ansible (1h30) — des principes au diagnostic CVE
header: "Ansible — automatiser un parc, sans agent"
footer: "Formation Ansible · 1h30"
---

<!-- _class: lead -->
<!-- _paginate: false -->
<!-- _header: "" -->
<!-- _footer: "" -->

# Ansible

## Automatiser un parc, sans agent

Des principes au **diagnostic CVE** sur tout un parc — en 1h30

<!--
Accueil. Public mixte : on part des bases, mais rythme soutenu.
Tout ce qui est montré est rejouable : `task --list` dans le dépôt.
-->

---

# Au programme

| # | Partie | ⏱ |
|---|---|---|
| 1 | Grands principes, push vs pull | 7 min |
| 2 | La CLI `ansible` : ad hoc, `ping`, `setup` | 6 min |
| 3 | L'inventaire | 10 min |
| 4 | Playbooks, langage (boucles, conditions, filtres, tests…) et rôles | 30 min |
| 5 | Idempotence, `ansible-lint`, Molecule | 10 min |
| 6 | Déployer sur N serveurs | 6 min |
| 7 | *Cas pratique : cartographier les versions — pour approfondir, hors séance* | — |
| 8 | **Cas pratique** : diagnostic CVE PHP / Node.js | 9 min |
| 9 | Bonnes pratiques, questions | 10 min |

<!--
Les démos tournent sur un lab Docker de 6 serveurs (3 PHP, 3 Node.js)
volontairement hétérogènes : à jour, en retard de patch, en fin de vie.
-->

---

# Le lab utilisé pendant la formation

<div class="cols">
<div>

```text
            contrôleur (votre poste)
            ansible / ansible-playbook
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
  web1  PHP 8.1   web2  PHP 8.3   web3  PHP 8.4
  api1  Node 20   api2  Node 22   api3  Node 24
  └─ staging ─┘   └─────────── prod ──────────┘
```

</div>
<div>

- 6 conteneurs Debian + Python
- connexion `docker exec` au lieu de SSH
  *(même principe, zéro clé à gérer)*
- tout se rejoue :

```bash
task lab:up      # démarre le parc
task demo:ping   # première démo
task --list      # toutes les démos
```

</div>
</div>

<!--
En production, rien ne change côté playbooks : on retire juste
ansible_connection et Ansible repasse en SSH.
-->

---

<!-- _class: lead -->

# 01 · Les grands principes

---

<!-- _class: tiny -->

# Ansible, de 2012 à aujourd'hui

<div class="cols">
<div>

### Les origines
- **2012** — **Michael DeHaan** (auteur de Cobbler) publie le 1ᵉʳ commit, le 23 février
- **2013** — société **AnsibleWorks** (puis Ansible Inc.) et offre commerciale **Ansible Tower**
- **Oct. 2015** — rachat par **Red Hat**. Montant non publié : **~150 M\$** selon TechCrunch (« plus de 100 M\$ » selon VentureBeat), pour 6 M\$ levés
- **2017** — Tower devient open source : projet **AWX**
- **2019** — **IBM** rachète Red Hat (34 Md\$) · lancement d'**Ansible Automation Platform** (AAP)

> Le nom vient de l'*ansible* d'Ursula K. Le Guin (*Rocannon's World*, 1966) : communication **instantanée**, quelle que soit la distance.

</div>
<div>

### Aujourd'hui : l'essor de **Semaphore UI**
- **AWX** : aucune *release* depuis la 24.6.1 (02/07/2024), installation **Kubernetes** obligatoire (*operator*)
- **AAP** : complet, mais sous **souscription** et lourd à opérer
- **Semaphore UI** (ex-*Ansible Semaphore*, 2014) : interface web + API, licence **MIT**, écrit en **Go**, ~14 k ⭐, v2.19.18 le 02/11/2026
  - **Installation ouverte et légère** : un binaire unique ou une image Docker, base SQLite, MySQL ou PostgreSQL, **sans Kubernetes**
  - **Intégration Terraform** : Ansible, **Terraform / OpenTofu / Terragrunt** (plan puis apply), PowerShell, Bash dans le **même outil**
  - Planification, coffre de secrets, LDAP / OIDC, rôles par projet

</div>
</div>

<!--
Sources : TechCrunch et VentureBeat (16/10/2015) pour le montant — Red Hat n'a jamais
communiqué de chiffre.
AWX / Semaphore : GitHub (releases ansible/awx et semaphoreui/semaphore, vérifié le 07/11/2026).
Message : Terraform crée l'infra, Ansible la configure → un seul outil pour lancer et
tracer les deux. Pour un parc qui n'a pas de cluster Kubernetes, Semaphore évite
d'en monter un juste pour héberger AWX.
-->

---

<!-- _class: small -->

# Ansible en une slide

<div class="cols">
<div>

### Ce que c'est
- Outil d'**automatisation** : configuration, déploiement, orchestration
- Open source (GPLv3 <img src="img/stallman.png" class="avatar" alt="Richard Stallman">), créé en 2012, porté par **Red Hat**
- Écrit en Python, piloté en **YAML**

### Ce qui le distingue
- **Sans agent** : SSH (Linux) / WinRM (Windows) / API (réseau, cloud)
- **Push** : le contrôleur pousse, quand **vous** le décidez
- **Déclaratif** et **idempotent** : on décrit un état cible

</div>
<div>

### Les briques
- **Inventaire** : *sur quoi* agir
- **Modules** : *quoi* faire (`copy`, `apt`, `user`…)
- **Playbooks** : *dans quel ordre*
- **Rôles / collections** : *réutiliser et partager*

> Prérequis sur la cible : un accès (SSH) et **Python**.
> C'est tout.

</div>
</div>


<span class="credit">Photo : Richard Stallman par Boro, <a href="https://commons.wikimedia.org/wiki/File:Richard_stallman_portrait.png">Wikimedia Commons</a>, CC BY-SA 3.0 (recadrée)</span>

---

# Comment une tâche s'exécute

```text
 contrôleur                                        cible
 ──────────                                        ─────
 1. lit l'inventaire + le playbook
 2. génère le module Python + ses arguments
 3. ───────────── SSH (ou docker exec, WinRM…) ─────────────▶
                                        4. exécute le module avec python3
                                        5. compare état actuel / état voulu
                                        6. agit seulement si nécessaire
 7. ◀──────────────── résultat JSON {changed, failed, …} ───
 8. affiche ok / changed / failed, passe à la tâche suivante
```

- Rien ne reste installé sur la cible : le module est copié, exécuté, supprimé
- `forks` (défaut 5) : nombre de cibles traitées **en parallèle**
- `pipelining = True` : moins d'aller-retours SSH → **beaucoup** plus rapide

<!--
Pour les expérimentés : pipelining nécessite que requiretty soit désactivé
dans sudoers. C'est le premier réglage de perf à activer.
-->

---

<!-- _class: small -->

# Push vs Pull

<div class="cols">
<div class="card">

### Push — *Ansible, Salt (aussi)*
Le **contrôleur se connecte** aux cibles et applique.

- ✅ Rien à installer ni à maintenir sur les cibles
- ✅ Exécution à la demande, ordre maîtrisé : idéal pour **orchestrer** (rolling update, migrations)
- ✅ Retour immédiat dans le terminal ou la CI
- ⚠️ Le contrôleur doit **joindre** chaque cible
- ⚠️ Pas de correction de dérive en continu… sauf si on planifie (cron, AWX)

</div>
<div class="card">

### Pull — *Puppet, Chef, (Salt)*
Un **agent sur chaque cible** interroge un serveur et converge.

- ✅ Convergence **continue** (toutes les 30 min…) : dérive corrigée seule
- ✅ Passe bien à très grande échelle, cibles derrière NAT
- ⚠️ Agent à installer, mettre à jour, superviser
- ⚠️ Serveur central + PKI à opérer
- ⚠️ Ordonnancement multi-serveurs plus difficile

</div>
</div>

> Ansible sait aussi faire du pull : `ansible-pull -U <repo git>` en cron sur chaque machine.

---

<!-- _class: small -->

# Comparatif rapide

| | **Ansible** | **Puppet** | **Chef** | **Salt** | **Terraform / OpenTofu** |
|---|---|---|---|---|---|
| Modèle | Push (pull possible) | Pull | Pull | Push + pull | Appels d'API |
| Agent | **Aucun** | puppet-agent | chef-client | salt-minion (ou salt-ssh) | Aucun |
| Langage | YAML + Jinja2 | DSL Puppet | Ruby DSL | YAML + Jinja2 | HCL |
| Serveur central | Optionnel (AWX / AAP) | Puppet Server | Chef Server | salt-master | *state* distant |
| Style | Procédural ordonné + modules déclaratifs | Déclaratif, graphe de ressources | Procédural | Déclaratif, événementiel | Déclaratif |
| Point fort | Simplicité, orchestration | Conformité continue | Flexibilité dev | Vitesse, temps réel | **Provisionner** l'infra |

> Terraform **crée** les machines, Ansible les **configure** : ce sont des outils complémentaires, pas concurrents.

<!--
Message clé : le choix dépend du besoin.
- Conformité continue sur 10 000 nœuds → pull.
- Orchestration, déploiement applicatif, actions ponctuelles (diag, patch) → push.
Le cas pratique CVE de la fin est typiquement un usage "push" : je veux une réponse maintenant.
-->

---

<!-- _class: lead -->

# 02 · La CLI `ansible`

## Commandes *ad hoc* : une action, tout le parc, une ligne

---

# Anatomie d'une commande ad hoc

```bash
ansible  <pattern>  -m <module>  -a "<arguments>"  [options]
ansible  web        -m copy      -a "src=motd dest=/etc/motd"  -b
```

| Option | Rôle |
|---|---|
| `-i inventaire` | fichier/dossier d'inventaire (sinon `ansible.cfg`) |
| `-m module` | module à exécuter (défaut : `command`) |
| `-a "args"` | arguments du module |
| `-b` / `-K` | *become* (sudo) / demander le mot de passe sudo |
| `-u user` | utilisateur distant |
| `-l limite` | restreindre le pattern |
| `-f N` | parallélisme (*forks*) |
| `-C` / `-D` | *check mode* (simulation) / afficher le *diff* |
| `-o` | une ligne par hôte *(déprécié en ansible-core 2.21, retrait prévu en 2.23)* |

---

# <span class="demo"></span> Module `ping`

```bash
$ ansible all -m ping -o
```

```text
web1 | SUCCESS => {"changed": false,"ping": "pong"}
web2 | SUCCESS => {"changed": false,"ping": "pong"}
web3 | SUCCESS => {"changed": false,"ping": "pong"}
api1 | SUCCESS => {"changed": false,"ping": "pong"}
api2 | SUCCESS => {"changed": false,"ping": "pong"}
api3 | SUCCESS => {"changed": false,"ping": "pong"}
```

- ⚠️ **Pas un ping ICMP** : il valide toute la chaîne
  *connexion → authentification → élévation (`-b`) → Python sur la cible*
- Le premier réflexe face à un nouveau parc ou un nouvel inventaire

<!--
Démo : task demo:ping
Montrer aussi ce qui se passe avec un hôte injoignable : UNREACHABLE.
-->

---

<!-- _class: small -->

# <span class="demo"></span> Module `setup` : les *facts*

```bash
$ ansible web1 -m setup -a 'filter=ansible_distribution*'
```

```text
web1 | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "Debian",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/os-release",
        "ansible_distribution_file_variety": "Debian",
        "ansible_distribution_major_version": "13",
        "ansible_distribution_minor_version": "2",
        "ansible_distribution_release": "trixie",
        "ansible_distribution_version": "13.2"
    },
    "changed": false
}
```

- Inventaire **automatique** de chaque machine : OS, CPU, RAM, disques, IP, …
- Collecté au début de chaque playbook (`gather_facts`), réutilisable : `{{ ansible_facts['distribution'] }}`
- Astuces : `filter=ansible_*_mb`, `gather_subset=network`, cache de facts (`fact_caching = jsonfile`)

---

# Quelques ad hoc utiles au quotidien

```bash
# Le module par défaut est "command" : pas de shell, pas de pipe
ansible web -a "uptime" -o

# Besoin d'un pipe / d'une redirection → module shell
ansible all -m shell -a "df -h / | tail -1" -o

# Installer un paquet, en simulation d'abord (-C), avec élévation (-b)
ansible web -b -m apt -a "name=htop state=present update_cache=yes" -C

# Copier un fichier, voir le diff
ansible prod -b -m copy -a "src=./ntp.conf dest=/etc/ntp.conf" -D

# Redémarrer un service sur 2 machines à la fois
ansible web -b -m service -a "name=nginx state=restarted" -f 2
```

- `ansible-doc -l` · `ansible-doc -s copy` : la doc des modules en local
- `ansible-console` : un shell interactif sur tout le parc

<!--
Pour les expérimentés : ansible-console est méconnu et très pratique en
diag ("cd web", puis on tape des modules directement).
-->

---

<!-- _class: lead -->

# 03 · L'inventaire

## *Sur quoi* agit-on ?

---

# Inventaire statique : INI ou YAML

<div class="cols">
<div>

```ini
# inventory/hosts.ini
[web]
web1 ansible_host=10.0.0.11
web[2:3].example.com

[api]
api[1:3].example.com

[prod]
web[2:3].example.com
api[2:3].example.com

[prod:vars]
ansible_user=deploy
```

</div>
<div>

```yaml
# inventory/hosts.yml
all:
  children:
    web:
      hosts:
        web1: { ansible_host: 10.0.0.11 }
        web[2:3].example.com:
    api:
      hosts:
        api[1:3].example.com:
    prod:
      hosts:
        web[2:3].example.com:
        api[2:3].example.com:
      vars:
        ansible_user: deploy
```

</div>
</div>

- Un hôte peut appartenir à **plusieurs groupes** : fonction (`web`) × environnement (`prod`)
- Groupes implicites : `all` et `ungrouped`

---

# Variables d'inventaire : `group_vars/` et `host_vars/`

<div class="cols">
<div>

```text
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml      # tout le monde
│   ├── web.yml      # groupe web
│   └── api.yml
└── host_vars/
    └── web2.yml     # un hôte précis
```

```yaml
# group_vars/web.yml
app_runtime: php
app_version_cmd: php -r 'echo PHP_VERSION;'
```

</div>
<div>

### Priorité (simplifiée, du plus faible au plus fort)

1. `defaults/` d'un rôle
2. `group_vars/all`
3. `group_vars/<groupe>`
4. `host_vars/<hôte>`
5. *facts*, `vars:` du play, `vars/` d'un rôle
6. **`-e` (extra vars) : gagne toujours**

> Règle d'or : une variable = **un seul endroit** où elle est définie.

</div>
</div>

<!--
La vraie liste fait 22 niveaux (docs "Variable precedence").
Pour les expérimentés : group_vars à côté de l'inventaire ET à côté du
playbook sont tous deux chargés — source classique de surprises.
-->

---

<!-- _class: small -->

# Dictionnaires : `hash_behaviour = merge`

<div class="cols">
<div>

```yaml
# group_vars/all.yml
php_ini:
  memory_limit: 128M
  max_execution_time: 30
  session: { upload_progress: "On", cookie_secure: true }
```

```yaml
# group_vars/web.yml  (plus prioritaire)
php_ini:
  memory_limit: 512M
  session: { upload_progress: "Off" }
```

```ini
# ansible.cfg  (ou ANSIBLE_HASH_BEHAVIOUR=merge)
[defaults]
hash_behaviour = merge      # défaut : replace
```

</div>
<div>

**`replace`** (défaut) : le dictionnaire le plus prioritaire **écrase tout**

```yaml
php_ini:                    # vu par web1
  memory_limit: 512M
  session: { upload_progress: "Off" }
  # max_execution_time, cookie_secure : perdus !
```

**`merge`** : fusion **récursive** des dictionnaires

```yaml
php_ini:                    # vu par web1
  memory_limit: 512M
  max_execution_time: 30
  session: { upload_progress: "Off", cookie_secure: true }
```

</div>
</div>

> ⚠️ Réglage **global** : il touche toutes les sources de variables, y compris les rôles et collections tierces ; les **listes** restent remplacées. La doc officielle le **déconseille**. Préférer une fusion **explicite** :
> `php_ini: "{{ php_ini_defaults | combine(php_ini_overrides, recursive=true) }}"`

<!--
Démo express (sans conteneur) : comparer
  ansible-inventory --host web1 --yaml
  ANSIBLE_HASH_BEHAVIOUR=merge ansible-inventory --host web1 --yaml
Toujours présent et non déprécié en ansible-core 2.21, mais la doc prévient :
"fragile, rend le contenu non portable". Les rôles Galaxy supposent tous replace.
Alternative avancée : lookups vars + varnames pour fusionner toutes les
variables suivant un motif (ex. ^php_ini__), ou un vars plugin dédié.
-->

---

<!-- _class: small -->

# Même avec `merge`, les **listes** sont remplacées

<div class="cols">
<div>

```yaml
# group_vars/all.yml
php_extensions: [opcache, intl, mbstring]
php_conf:
  extensions: [opcache, intl, mbstring]
```

```yaml
# group_vars/web.yml  (plus prioritaire)
php_extensions: [pdo_pgsql]
php_conf:
  extensions: [pdo_pgsql]
```

```bash
$ ANSIBLE_HASH_BEHAVIOUR=merge \
    ansible-inventory --host web1 --yaml
```

</div>
<div>

**Résultat avec `merge`** : aucune fusion de liste

```yaml
php_extensions:          # liste de 1er niveau
- pdo_pgsql              # remplacée
php_conf:
  extensions:            # liste DANS un dict fusionné
  - pdo_pgsql            # remplacée aussi !
```

**Fusion explicite** : `combine(…, list_merge=…)`

```yaml
php_conf: "{{ php_conf_defaults | combine(php_conf_overrides,
              recursive=true, list_merge='append') }}"
# → extensions: [opcache, intl, mbstring, pdo_pgsql]
```

</div>
</div>

> `list_merge` : `replace` (défaut) · `keep` · `append` · `prepend` · `append_rp` / `prepend_rp` (sans doublons).
> Pour une liste simple : `{{ php_extensions_all + php_extensions_web }}` ou le filtre `union`.

<!--
Testé avec ansible-core 2.21 : en replace comme en merge, php_extensions et
php_conf.extensions valent [pdo_pgsql] pour web1.
hash_behaviour ne concerne que les dictionnaires ; une liste est une valeur
"scalaire" du point de vue de la fusion.
append_rp : "append, remove present" — retire de la 1re liste les éléments
présents dans la 2e avant d'ajouter → pas de doublons.
-->

---

<!-- _class: small -->

# <span class="demo"></span> Patterns et visualisation

<div class="cols">
<div>

```bash
$ ansible-inventory --graph
```

```text
@all:
  |--@web:
  |  |--web1
  |  |--web2
  |  |--web3
  |--@api:
  |  |--api1
  |  |--api2
  |  |--api3
  |--@staging:
  |  |--web1
  |  |--api1
  |--@prod:
  |  |--web2
  |  |--web3
  |  |--api2
  |  |--api3
```

</div>
<div>

| Pattern | Cible |
|---|---|
| `all` ou `*` | tout |
| `web` | un groupe |
| `web:api` | union |
| `prod:&api` | intersection |
| `all:!staging` | exclusion |
| `web[0]` | 1er hôte du groupe |
| `~api[12]` | regex |

```bash
ansible 'prod:&api' -m ping
ansible-playbook site.yml -l web2
ansible-inventory --host web2
```

</div>
</div>

---

# Inventaires dynamiques

Le parc change en permanence ? **L'inventaire se calcule**, il ne s'écrit pas.

<div class="cols">
<div>

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions: [eu-west-3]
filters:
  tag:env: prod
keyed_groups:
  - key: tags.role       # → groupes role_web, role_api
    prefix: role
compose:
  ansible_host: private_ip_address
```

</div>
<div>

- Plugins : AWS, Azure, GCP, VMware, Proxmox, OpenStack, NetBox, Docker, Kubernetes…
- `keyed_groups` : des **groupes créés à partir des tags**
- `compose` : des variables calculées
- On peut mixer : `-i statique.yml -i aws_ec2.yml`
- Plugin `constructed` : groupes calculés à partir des *facts* ou de variables

</div>
</div>

---

<!-- _class: small -->

# `keyed_groups` : des groupes **calculés**

Pour chaque hôte, Ansible évalue `key` (une expression Jinja2 sur ses variables) et le range dans le groupe **`<prefix>_<valeur>`**.

<div class="cols" style="grid-template-columns: 0.75fr 1.25fr">
<div>

| Option | Effet |
|---|---|
| `key` | expression évaluée par hôte |
| `prefix` | préfixe du nom de groupe |
| `separator` | défaut `_` ; `""` pour coller |
| `parent_group` | groupe parent des groupes créés |
| `default_value` | nom utilisé si la valeur est **vide** |

</div>
<div>

| `key` | Valeur | Groupe créé |
|---|---|---|
| `ansible_distribution_release` | `bookworm` | `os_bookworm` |
| `app_runtime` *(group_vars)* | `php` | `runtime_php` |
| `ansible_distribution_version` | `12.10` | `ver_12_10` |
| `tags.role` *(aws_ec2)* | `web` | `role_web` |

`separator: ""` + `parent_group: os` → `os` › `debian12`, `debian13`

</div>
</div>

- Dans **tous** les plugins d'inventaire (`aws_ec2`, `azure_rm`, `vmware`, `docker_containers`…) et dans **`ansible.builtin.constructed`**, qui travaille sur les hôtes **déjà connus** (inventaire statique + facts en cache)
- Caractères invalides (`.` `-` espace…) → `_` · compléments : `groups:` (groupe **conditionnel**), `compose:` (variable **calculée**)

<!--
Tous les exemples du tableau ont été vérifiés sur le lab (ansible-core 2.21).
Attention : default_value ne s'applique que si la valeur est vide (""),
pas si la variable est indéfinie — dans ce cas, avec strict: false, l'hôte
n'est simplement pas groupé.
-->

---

<!-- _class: small -->

# <span class="demo"></span> Grouper par version d'OS à partir des *facts*

<div class="cols">
<div>

```yaml
# inventory/zz-constructed.yml  (lu APRÈS hosts.yml)
plugin: ansible.builtin.constructed
strict: false            # pas de facts → hôte ignoré
use_vars_plugins: true   # rend group_vars/ visibles

keyed_groups:
  - key: ansible_distribution_release
    prefix: os           # → os_bookworm, os_trixie
  - key: app_runtime
    prefix: runtime      # → runtime_php, runtime_nodejs

groups:                  # groupe conditionnel
  debian_oldstable: >-
    ansible_distribution_major_version is defined
    and ansible_distribution_major_version | int < 13
```

```ini
# ansible.cfg : les facts doivent être EN CACHE
inventory               = inventory/
fact_caching            = ansible.builtin.jsonfile
fact_caching_connection = .facts_cache
```

</div>
<div>

```text
$ ansible os_bookworm --list-hosts
[WARNING]: Could not match supplied host pattern,
           ignoring: os_bookworm
  hosts (0):                       ← pas encore de facts

$ ansible all -m setup -a 'gather_subset=min' > /dev/null

$ ansible os_bookworm --list-hosts
  hosts (4):
    web2
    api1
    api2
    api3

$ ansible os_bookworm -a "openssl version" -o
api2 | CHANGED | rc=0 | (stdout) OpenSSL 3.0.22 25 Aug 2026
web2 | CHANGED | rc=0 | (stdout) OpenSSL 3.0.22 25 Aug 2026
...

$ ansible 'runtime_php:&os_bookworm' --list-hosts
  hosts (1):
    web2
```

</div>
</div>

<!--
Démo : task demo:keyed-groups
Cas d'usage concret : un bulletin DSA Debian ne concerne que bookworm →
on cible os_bookworm sans maintenir de liste à la main.
Le cache expire (fact_caching_timeout, ici 24 h) : un cron "ansible all -m setup"
ou une exécution AWX garde les groupes à jour.
-->

---

<!-- _class: small -->

# Alternative à l'exécution : `group_by`

<div class="cols" style="grid-template-columns: 1.1fr 0.9fr">
<div>

```yaml
# playbooks/04-group-by.yml
- name: Classer les serveurs selon leurs facts
  hosts: all
  tasks:
    - name: Créer un groupe par version de Debian
      ansible.builtin.group_by:
        key: "release_{{ ansible_facts['distribution_release'] }}"

- name: Agir uniquement sur les Debian 12
  hosts: release_bookworm
  gather_facts: false
  tasks:
    - ansible.builtin.debug:
        msg: "{{ inventory_hostname }} : Debian
              {{ ansible_facts['distribution_version'] }}"
```

```text
ok: [web2] => msg: 'web2 : Debian 12.10'
ok: [api1] => msg: 'api1 : Debian 12.13'
...
```

</div>
<div>

| | `constructed` | `group_by` |
|---|---|---|
| Quand ? | chargement de l'inventaire | pendant le playbook |
| Prérequis | facts **en cache** | facts du play |
| `ansible-inventory` | ✅ visible | ❌ |
| ad hoc / `--limit` | ✅ | ❌ |
| Durée de vie | validité du cache | un `ansible-playbook` |

> **Règle pratique** : groupes réutilisés partout (ad hoc, AWX, `--limit`) → `constructed`.
> Besoin ponctuel dans un playbook → `group_by`.

</div>
</div>

<!--
Démo : task demo:group-by
-->

---

<!-- _class: lead -->

# 04 · Playbooks et rôles

## *Quoi* faire, dans *quel ordre*

---

# Vocabulaire

```text
playbook.yml                 ← un fichier YAML
└── play  (hosts: web)       ← "sur ces hôtes…"
    ├── vars                 ← données
    ├── pre_tasks / roles / tasks / post_tasks
    │   └── task             ← "…appliquer ce module avec ces arguments"
    │       └── module       ← ansible.builtin.copy, apt, user, template…
    └── handlers             ← tâches déclenchées par "notify", si changement
```

- Un playbook = **une liste de plays**, exécutés dans l'ordre
- Un play = des **tâches** exécutées dans l'ordre, sur tous les hôtes en parallèle
- Chaque tâche renvoie : `ok` · `changed` · `skipped` · `failed` · `unreachable`

---

<!-- _class: small -->

# <span class="demo"></span> Premier playbook

```yaml
# playbooks/01-premier-playbook.yml
- name: Socle commun sur tous les serveurs
  hosts: all
  become: true

  vars:
    admin_user: deploy

  tasks:
    - name: Créer l'utilisateur d'exploitation
      ansible.builtin.user:
        name: "{{ admin_user }}"
        shell: /bin/bash

    - name: Créer le répertoire applicatif
      ansible.builtin.file:
        path: /opt/app
        state: directory
        owner: "{{ admin_user }}"
        mode: "0755"

    - name: Déployer le motd via un rôle
      ansible.builtin.include_role:
        name: motd
```

---

<!-- _class: small -->

# <span class="demo"></span> Exécution

```bash
$ ansible-playbook playbooks/01-premier-playbook.yml
```

```text
PLAY [Socle commun sur tous les serveurs] ********************************

TASK [Gathering Facts] ***************************************************
ok: [web1]  ok: [web2]  ok: [api1] ...

TASK [Créer l'utilisateur d'exploitation] ********************************
changed: [web1] changed: [web2] changed: [api1] ...

TASK [motd : Déployer le message d'accueil] ******************************
changed: [web1] changed: [web2] changed: [api1] ...

RUNNING HANDLER [motd : Journaliser le changement de motd] ***************
changed: [web1] changed: [web2] changed: [api1] ...

PLAY RECAP ***************************************************************
api1 : ok=6  changed=4  unreachable=0  failed=0  skipped=0
web1 : ok=6  changed=4  unreachable=0  failed=0  skipped=0
...
```

*Sortie condensée — 2ᵉ exécution : `changed=0` partout.*

- Options du quotidien : `--check --diff` · `-l web1` · `--tags` · `--start-at-task` · `--step` · `-v…-vvvv`

---

<!-- _class: small -->

# `loop` : répéter une tâche

<div class="cols">
<div>

```yaml
- name: Créer les comptes applicatifs
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    append: true
  loop:
    - { name: alice, groups: [www-data, adm] }
    - { name: bob,   groups: [www-data] }
  loop_control:
    label: "{{ item.name }}"   # sortie lisible
```

```text
TASK [Créer les comptes applicatifs] ***********
changed: [web1] => (item=alice)
changed: [web1] => (item=bob)
```

</div>
<div>

- `item` = l'élément courant (liste de valeurs ou de dictionnaires)
- `loop_control` :
  - `label` : n'affiche que l'essentiel de `item`
  - `index_var: idx` : numéro de l'itération
  - `loop_var` : renommer `item` (boucles imbriquées via `include_tasks`)
- `register` sur une boucle → une liste `résultat.results`, un élément par itération
- Autre forme de boucle : `until` / `retries` / `delay` pour **réessayer** jusqu'à succès

> ⚠️ Pas de boucle si le module accepte une liste :
> `apt: { name: [curl, jq, htop] }` → **1** transaction apt au lieu de 3

</div>
</div>

<!--
loop est la forme recommandée depuis Ansible 2.5.
Le label évite d'afficher un dictionnaire entier à chaque itération (et des secrets).
L'exemple apt : piège classique, et anti-pattern relevé par les relecteurs expérimentés.
-->

---

<!-- _class: small -->

# Les anciennes boucles `with_*` : migrer vers `loop`

| Avant | Après |
|---|---|
| `with_items: "{{ l }}"` | `loop: "{{ l \| flatten(levels=1) }}"` — ⚠️ `with_items` aplatissait d'un niveau |
| `with_list` | `loop` (strictement identique) |
| `with_dict: "{{ d }}"` | `loop: "{{ d \| dict2items }}"` → `item.key` / `item.value` |
| `with_nested: [a, b]` | `loop: "{{ a \| product(b) \| list }}"` → `item.0` / `item.1` |
| `with_together: [a, b]` | `loop: "{{ a \| zip(b) \| list }}"` |
| `with_subelements: [users, groups]` | `loop: "{{ users \| subelements('groups') }}"` |
| `with_indexed_items` | `loop` + `loop_control: { index_var: idx }` |
| `with_sequence: start=1 end=3` | `loop: "{{ range(1, 4) \| list }}"` |
| `with_fileglob: "files/*.conf"` | `loop: "{{ query('fileglob', 'files/*.conf') }}"` |

- Règle générale : `with_X` **est** le lookup `X` → `loop: "{{ query('X', …) }}"`
  (`query` renvoie toujours une liste ; `lookup` renvoie une chaîne séparée par des virgules)
- `with_*` n'est **pas déprécié** (aucun avertissement en ansible-core 2.21) : migrer au fil de l'eau, pas d'urgence

<!--
Tous ces équivalents ont été vérifiés en 2.21 (même sortie avant / après).
Le seul vrai piège : with_items aplatit les listes de listes, loop non.
Migration mécanique : ansible-lint ne force pas loop par défaut, c'est un choix d'équipe.
-->

---

<!-- _class: small -->

# `when` : exécuter sous condition

<div class="cols">
<div>

```yaml
- name: Seulement sur Debian, en prod
  ansible.builtin.debug:
    msg: "Debian en prod"
  when:                       # liste = ET
    - ansible_facts['os_family'] == 'Debian'
    - app_env == 'prod'

- name: Si la variable est définie
  ansible.builtin.file:
    path: "{{ backup_dir }}"
    state: directory
  when: backup_dir is defined

- name: Selon une tâche précédente
  ansible.builtin.debug:
    msg: "Le compte deploy existe"
  when: deploy_user.rc == 0   # ou : result is changed
```

</div>
<div>

- Expression Jinja2 **brute**, sans `{{ }}`
  `when: "{{ x }}"` est **déprécié** (suppression en 2.23)
- Évaluée **par hôte**, et **par élément** avec `loop`
- `and` · `or` · `not` · `in` · tests `is defined`, `is changed`…
- Sur un `block` ou un `import_*` : recopiée sur **chaque** tâche
  Sur un `include_*` : évaluée **une seule fois**

> ⚠️ **Depuis 2.19, le résultat doit être un booléen**
> `when: ma_liste` → *Conditionals must have a boolean result.*
> → `when: ma_liste | length > 0`
> → `when: flag | bool` pour un `"yes"` venu d'une variable

</div>
</div>

<!--
Point important pour un public qui connaît Ansible "d'avant" :
les conditions non booléennes (liste, chaîne) fonctionnaient implicitement,
elles sont en erreur depuis ansible-core 2.19 (vérifié en 2.21).
Contournement temporaire : ALLOW_BROKEN_CONDITIONALS (à éviter).
-->

---

<!-- _class: small -->

# `register` : capturer et exploiter un résultat

<div class="cols">
<div>

```yaml
- name: Le compte deploy existe-t-il ?
  ansible.builtin.command: grep -c '^deploy:' /etc/passwd
  register: deploy_user
  changed_when: false     # une lecture ne change rien
  failed_when: deploy_user.rc not in [0, 1]
                          # rc=1 : "absent", pas une erreur

- name: Afficher
  ansible.builtin.debug:
    var: deploy_user.stdout
```

```yaml
# Contenu de deploy_user
rc: 0
stdout: "1"
stdout_lines: ["1"]
stderr: ""
changed: false
failed: false
# + results: [...] si la tâche boucle
```

</div>
<div>

- `register` : variable **par hôte**, réutilisable dans la suite du play, et par les autres hôtes via `hostvars` (partie 07)
- `changed_when` : `command` et `shell` annoncent **toujours** `changed` → sans ça, le `PLAY RECAP` ment (cf. idempotence)
- `failed_when` : définir **soi-même** ce qu'est un échec
  *Dans le lab : `failed_when: deployed.stdout != app_release` (rolling update)*
- Explorer un résultat : `debug: var=…` ou `-v`

</div>
</div>

<!--
Démo possible en direct : ansible-playbook … -v pour voir le résultat brut.
Le cas pratique 1 repose entièrement sur register + hostvars.
-->

---

<!-- _class: small -->

# `template` : des fichiers de configuration Jinja2

<div class="cols">
<div>

```jinja
{# templates/nginx.conf.j2 #}
{{ ansible_managed | comment }}
server {
    listen {{ nginx_port | default(80) }};
    server_name {{ vhosts | join(' ') }};
{% if app_env == 'prod' %}
    access_log off;
{% endif %}
}
```

```yaml
- name: Configurer nginx
  ansible.builtin.template:
    src: nginx.conf.j2       # cherché dans templates/
    dest: /etc/nginx/nginx.conf
    mode: "0644"
    backup: true             # garde l'ancienne version
    validate: nginx -t -c %s # %s = fichier temporaire
  notify: Recharger nginx
```

</div>
<div>

- Rendu **sur le contrôleur**, avec les variables et facts de l'hôte, puis copié : rien à installer sur la cible
- `{{ … }}` valeur · `{% … %}` logique (`if`, `for`) · `{# … #}` commentaire
- `validate` : la commande échoue → tâche en échec, **fichier d'origine intact**. On ne pousse jamais une conf cassée sur 50 serveurs
- `ansible_managed` : bandeau « ne pas modifier à la main » (cf. rôle `motd`)
- `--check --diff` : voir le diff **avant** d'appliquer
- `copy` pour un fichier statique, `template` dès qu'il y a une variable

</div>
</div>

<!--
validate vérifié : avec une commande en échec, "failed to validate" et dest inchangé.
ansible_managed : variable injectée par le module template ; à définir comme variable
ordinaire (le réglage ansible.cfg est déprécié pour 2.23).
-->

---

<!-- _class: small -->

# Handlers : réagir à un changement

<div class="cols">
<div>

```yaml
tasks:
  - name: Configurer nginx
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Recharger nginx

  - name: Déployer le vhost
    ansible.builtin.template:
      src: vhost.conf.j2
      dest: /etc/nginx/conf.d/app.conf
    notify: Recharger nginx

  - name: Déployer le certificat
    ansible.builtin.copy:
      src: app.pem
      dest: /etc/nginx/app.pem
    notify: config web modifiée

handlers:
  - name: Recharger nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
    listen: config web modifiée
```

</div>
<div>

- Déclenché **seulement** si la tâche est `changed` (`ok` → rien)
- Notifié 3 fois → exécuté **une seule fois**
- Exécuté à la **fin de chaque section** (`pre_tasks`, `roles`, `tasks`, `post_tasks`), dans l'ordre de **déclaration** des handlers
- `meta: flush_handlers` : les exécuter **maintenant** (ex. avant un test de santé)
- `listen` : un « sujet » auquel plusieurs handlers s'abonnent
- ⚠️ Si l'hôte échoue avant la fin, les handlers notifiés **ne tournent pas** → `--force-handlers`

</div>
</div>

<!--
Vérifié : 3 notifications (dont une via listen) → un seul RUNNING HANDLER.
Une tâche "ok" qui porte un notify ne déclenche rien.
Dans le lab : handler "Journaliser le changement de motd" du rôle motd.
-->

---

<!-- _class: small -->

# `block` / `rescue` / `always` : gérer les erreurs

<div class="cols">
<div>

```yaml
- name: Déployer avec un filet de sécurité
  block:
    - name: Basculer sur la nouvelle version
      ansible.builtin.file:
        src: "/opt/app/releases/{{ app_release }}"
        dest: /opt/app/current
        state: link
    - name: Test de santé
      ansible.builtin.uri:
        url: http://localhost:8080/health
  rescue:
    - name: Revenir à la version précédente
      ansible.builtin.file:
        src: "/opt/app/releases/{{ previous_release }}"
        dest: /opt/app/current
        state: link
    - name: Tracer l'échec
      ansible.builtin.debug:
        msg: "Échec de « {{ ansible_failed_task.name }} »"
  always:
    - name: Nettoyer
      ansible.builtin.file:
        path: /tmp/app.tgz
        state: absent
```

</div>
<div>

- `block` = *try* · `rescue` = *catch* · `always` = *finally*
- Si `rescue` réussit, l'hôte **n'est pas en échec** : le play continue
  `PLAY RECAP … failed=0 … rescued=1`
- Dans `rescue` : `ansible_failed_task`, `ansible_failed_result`
- Un `block` sert aussi à **factoriser** : `when`, `become`, `tags` posés dessus s'appliquent à chaque tâche
- `ignore_errors: true` masque tout : préférer un `failed_when` précis

</div>
</div>

<!--
Vérifié : tâche en échec → rescue → always → le play continue, recap rescued=1.
Lien avec la partie 06 : combiné à serial et max_fail_percentage pour un rolling update sûr.
-->

---

<!-- _class: small -->

# Filtres : le principe

<div class="cols">
<div>

```yaml
# Syntaxe :  valeur | filtre(arguments)
"{{ app_runtime | upper }}"                  # PHP
"{{ app_port | default(8080) }}"             # 8080 si absente

# Enchaînés de gauche à droite
"{{ 'web1' | upper | replace('1', '-01') }}" # WEB-01
"{{ servers | map(attribute='name') | join(', ') }}"
```

```bash
# Essayer en une ligne, sur les variables d'un hôte
ansible web1 -m debug -a "msg={{ app_runtime | upper }}"

# Trouver et lire la doc
ansible-doc -t filter -l          # ~260 filtres
ansible-doc -t filter combine
```

</div>
<div>

- Un filtre **transforme** une valeur et en renvoie une **nouvelle** : `x | f(a)` revient à `f(x, a)`
- Évalué **sur le contrôleur**, avant l'envoi du module : la cible ne voit que le résultat
- Trois sources :
  - **Jinja2** : `default`, `join`, `map`, `sort`…
  - **`ansible.builtin`** : `combine`, `regex_replace`…
  - **collections** : `community.general.json_query`…

> ⚠️ **Priorité** : un filtre s'applique **avant** `==`
> `env == 'prod' | ternary('warn', 'info')` → `false`
> `(env == 'prod') | ternary('warn', 'info')` → `warn`

</div>
</div>

<!--
Compteur : ansible-core 2.21 + collections installées dans le lab.
La priorité des filtres est le piège n°1 : 'prod' | ternary(...) est calculé en premier.
-->

---

<!-- _class: small -->

# Filtres : valeurs par défaut et types

<div class="cols">
<div>

```yaml
"{{ app_port | default(8080) }}"          # 8080 si non définie
"{{ app_label | default('n/a') }}"        # '' si app_label: ''
"{{ app_label | default('n/a', true) }}"  # 'n/a' : aussi si vide
"{{ db_password | mandatory('à définir dans le vault') }}"
                            # erreur explicite si absente

"{{ 'yes' | bool }}"                      # true
"{{ '42' | int + 1 }}"                    # 43
"{{ 8080 | string }}"                     # '8080'
"{{ (app_env == 'prod') | ternary('warn', 'debug') }}"
```

```yaml
- name: Paramètre facultatif
  ansible.builtin.user:
    name: deploy
    uid: "{{ deploy_uid | default(omit) }}"
```

</div>
<div>

- `default` ne joue que si la variable est **indéfinie** ; avec `true` en 2ᵉ argument, aussi si elle est **vide ou fausse**
- `default(omit)` : le paramètre **n'est pas envoyé** au module, qui applique sa propre valeur par défaut (≠ chaîne vide)
- `mandatory` : un message clair plutôt qu'un *undefined* au milieu d'un template
- `bool` : indispensable pour `-e flag=yes`, qui arrive toujours en **chaîne**
- `int` / `float` / `string` : comparer des nombres, pas du texte
- `ternary` : le `if / else` sur une ligne

</div>
</div>

<!--
Tous les résultats vérifiés en 2.21.
default(omit) : idiome très courant dans les rôles, pour exposer un paramètre facultatif.
-->

---

<!-- _class: small -->

# Filtres : listes

<div class="cols" style="grid-template-columns: 1.25fr 1fr">
<div>

```yaml
servers:
  - { name: web1, runtime: php,    version: 8.1.34,  env: staging }
  - { name: web2, runtime: php,    version: 8.3.20,  env: prod }
  - { name: api1, runtime: nodejs, version: 20.20.2, env: staging }
  - { name: api2, runtime: nodejs, version: 22.12.0, env: prod }
```

```jinja
servers | map(attribute='name')             → [web1, web2, api1, api2]
servers | selectattr('env', '==', 'prod')
        | map(attribute='name')             → [web2, api2]
servers | rejectattr('runtime', '==', 'php')
        | map(attribute='name')             → [api1, api2]
servers | map(attribute='runtime')
        | unique | sort                     → [nodejs, php]
servers | map(attribute='name') | join(', ')
                                    → "web1, web2, api1, api2"
servers | length                            → 4
[[1, 2], [3]] | flatten                     → [1, 2, 3]
[3, 1, 2] | max · sum · sort | first        → 3 · 6 · 1
groups['web'] | intersect(groups['prod'])   → [web2, web3]
groups['web'] | difference(groups['prod'])  → [web1]
```

</div>
<div>

- `map(attribute=…)` : extraire un champ
  `map('upper')` : appliquer un filtre à chaque élément
- `selectattr` / `rejectattr` : garder / écarter selon un champ et un **test** (`==`, `version`, `defined`…)
- `select` / `reject` : idem sur les éléments eux-mêmes
- Ensembles : `union` · `intersect` · `difference`, parfaits sur `groups[…]`
- `| list` en fin de chaîne : **plus nécessaire** en 2.21, encore très présent dans les playbooks existants

</div>
</div>

<!--
Les groupes viennent de l'inventaire du lab (web = web1-3, prod = web2, web3, api2, api3).
L'ordre de intersect n'est pas garanti.
-->

---

<!-- _class: small -->

# Filtres : dictionnaires

<div class="cols">
<div>

```yaml
ports:    { http: 80, https: 443 }
base:     { nginx: { port: 80, modules: [gzip] } }
override: { nginx: { modules: [brotli] } }
```

```jinja
ports | dict2items
  → [{key: http, value: 80}, {key: https, value: 443}]
ports | dict2items | items2dict     → {http: 80, https: 443}
servers | items2dict(key_name='name', value_name='version')
  → {web1: 8.1.34, web2: 8.3.20, api1: 20.20.2, …}
dict(['web1', 'web2'] | zip(['8.1', '8.3']))
  → {web1: '8.1', web2: '8.3'}

base | combine(override)
  → {nginx: {modules: [brotli]}}             # port perdu
base | combine(override, recursive=true)
  → {nginx: {port: 80, modules: [brotli]}}
base | combine(override, recursive=true, list_merge='append')
  → {nginx: {port: 80, modules: [gzip, brotli]}}

groups['web'] | map('extract', hostvars, 'app_runtime')
  → [php, php, php]
```

</div>
<div>

- `dict2items` : **boucler** sur un dictionnaire (`loop`)
- `items2dict` : construire un **index** (nom → version)
- `combine` : surcharger des valeurs par défaut
  - sans `recursive` : remplacement au **1ᵉʳ niveau**
  - `list_merge` : `replace` (défaut), `append`, `prepend`, `keep`…
  - cf. slide `hash_behaviour` : même logique, mais **explicite**
- `extract` : lire une variable **de chaque hôte** → consolider un rapport (partie 07)

</div>
</div>

<!--
combine est la façon recommandée de fusionner, plutôt que hash_behaviour = merge (global).
extract est utilisé dans 10-versions.yml et 20-cve-diag.yml.
-->

---

<!-- _class: small -->

# Filtres : chaînes et expressions régulières

<div class="cols">
<div>

```jinja
'PHP 8.3.20 (cli)' | regex_search('\d+\.\d+\.\d+')  → 8.3.20
'PHP 8.3.20 (cli)' | regex_findall('\d+')     → ['8', '3', '20']
'8.3.20' | regex_replace('^(\d+\.\d+).*', '\1')     → 8.3
'v22.12.0' | regex_replace('^v', '')               → 22.12.0

'8.3.20' | split('.')                    → ['8', '3', '20']
'web1.example.com' | split('.') | first  → web1
'  web1  ' | trim | upper                → WEB1
'/etc/nginx/nginx.conf' | basename       → nginx.conf
'/etc/nginx/nginx.conf' | dirname        → /etc/nginx

'my file.txt' | quote                    → 'my file.txt'
'secret' | b64encode                     → c2VjcmV0
'S3cr3t' | password_hash('sha512')       → $6$rounds=656000$…
```

</div>
<div>

- `regex_search` : **extraire** · `regex_findall` : **toutes** les occurrences · `regex_replace` : **réécrire** (`\1` = 1ᵉʳ groupe capturé)
- `quote` : **toujours** sur une variable passée à `shell` (espaces, injection)
- `password_hash` : le module `user` attend un **hash**, pas un mot de passe

> ⚠️ Dans une chaîne YAML entre `"…"`, chaque `\` doit être **doublé** :
> `"{{ v | regex_search('\\d+') }}"`
> Sinon : *found unknown escape character*

</div>
</div>

<!--
Les expressions de gauche sont du Jinja2 pur (ce qu'on met entre {{ }}).
Écrites dans un fichier YAML entre guillemets doubles, les antislashs se doublent.
-->

---

<!-- _class: small -->

# Filtres : formats de données

<div class="cols">
<div>

```yaml
- name: Versions embarquées par Node.js
  ansible.builtin.command: >-
    node -p "JSON.stringify(process.versions)"
  register: node_versions
  changed_when: false

- name: Extraire la version d'OpenSSL
  ansible.builtin.debug:
    msg: "{{ (node_versions.stdout | from_json).openssl }}"
```

```text
ok: [api2] => { "msg": "3.0.15+quic" }
```

```yaml
- name: Générer une conf YAML depuis une variable
  ansible.builtin.copy:
    dest: /etc/app/config.yml
    content: "{{ app_config | to_nice_yaml(indent=2) }}"
```

</div>
<div>

- `from_json` / `from_yaml` : **texte → structure**
  sortie de commande, fichier lu avec `slurp` ou `lookup('file')`
- `to_json` · `to_nice_json` · `to_nice_yaml` : **structure → texte**
  fichiers de conf, corps d'appel d'API
- `slurp` renvoie du base64 : `(fichier.content | b64decode | from_json)`
- `uri` : la réponse JSON est déjà décodée dans `.json`, pas besoin de `from_json`
- `community.general.json_query` (JMESPath) : puissant, mais dépendance Python `jmespath` ; `selectattr` / `map` suffisent le plus souvent

</div>
</div>

<!--
Exemple exécuté sur api2 (Node 22.12.0).
Beaucoup d'outils savent sortir du JSON (--output json, -o json) : c'est plus robuste qu'un grep.
-->

---

<!-- _class: small -->

# Tests : le principe

<div class="cols">
<div>

```yaml
# Syntaxe :  valeur is test(arguments)   → true / false
app_version is version('8.3.36', '<')
backup_dir is defined
backup_dir is not defined          # négation : is not

# Où les utiliser ?
when: app_version is version('8.3.36', '<')   # condition
until: health is succeeded                    # réessai
"{{ servers | selectattr('version', 'version', '9', '<') }}"
                     # critère de sélection, nom du test sans "is"
```

```bash
# Essayer en une ligne
ansible localhost -m debug \
  -a "msg={{ '8.3.20' is version('8.3.36', '<') }}"

# Trouver et lire la doc
ansible-doc -t test -l             # ~110 tests
ansible-doc -t test version
```

</div>
<div>

- Un test **pose une question** et renvoie **toujours un booléen**
  → exactement ce qu'exigent `when`, `until`, `failed_when`, `changed_when` depuis 2.19
- Dans `select` / `reject` / `selectattr` / `rejectattr`, on passe le **nom** du test : `'defined'`, `'version'`, `'=='`…
- Deux sources :
  - **Jinja2** : `defined`, `none`, `string`, `number`, `in`, `==`…
  - **`ansible.builtin`** : `version`, `match`, `search`, `succeeded`, `changed`, `subset`…

</div>
</div>

<!--
Les tests rendent les conditions lisibles et toujours booléennes :
c'est la bonne réponse à l'erreur "Conditionals must have a boolean result" vue avec when.
-->

---

<!-- _class: small -->

# Tests : valeurs et comparaisons

<div class="cols">
<div>

```jinja
backup_dir is defined         → false si absente
backup_dir is not defined     → true   (négation après is)
null_var is defined           → true   ⚠️ null est « défini »
null_var is none              → true

app_version is string         (aussi : number, mapping, sequence)
'web2' is in groups['prod']   (ou : 'web2' in groups['prod'])
groups['prod'] is contains('web2')
['web2'] is subset(groups['prod'])

app_version is match('8\.3')        ancré au début
php_v.stdout is search('cli')       n'importe où

app_version is version('8.3.36', '<')   → true
'1.10.0' is version('1.9.0', '>')       → true
'1.10.0' > '1.9.0'                      → false ⚠️
```

</div>
<div>

- Variable facultative : `x is defined and x is not none`
- ⚠️ Une chaîne est aussi `iterable` et `sequence` : tester `is string` **d'abord**
- `match` est ancré au début, `search` cherche partout : `'PHP 8.3' is match('8\.3')` → `false`
- ⚠️ Comparer des versions avec `>` compare du **texte** (ordre alphabétique) → toujours `is version(…)`
  - opérateurs : `<`, `<=`, `==`, `>=`, `>`, `!=`
  - `version_type='semver'` pour les versions sémantiques strictes

</div>
</div>

<!--
Tous les résultats vérifiés en 2.21.
'1.10.0' > '1.9.0' est faux car '1' < '9' au 3e caractère : bug classique dans les scripts de conformité.
C'est exactement ce qu'utilise le rôle cve_check du lab.
-->

---

<!-- _class: small -->

# Tests : résultats de tâches et chemins

<div class="cols">
<div>

```yaml
- name: Attendre que l'application réponde
  ansible.builtin.uri:
    url: http://localhost:8080/health
  register: health
  until: health is succeeded
  retries: 10
  delay: 3

- name: Redémarrer seulement si la conf a changé
  ansible.builtin.service:
    name: app
    state: restarted
  when: app_conf is changed
```

```jinja
out.results | select('changed') | map(attribute='item')
                              → éléments de la boucle modifiés
out.results | selectattr('rc', '!=', 0) | length
                              → nombre d'itérations en erreur
[check_a, check_b] is all     (ou : is any)
```

</div>
<div>

- `succeeded` · `failed` · `changed` · `skipped` · `unreachable` : sur le résultat d'un `register`
- Préférer `result is failed` à `result.rc != 0` : fonctionne avec **tous** les modules
- `truthy` / `falsy` : `'no' is truthy` → `true` ⚠️ pour un « oui / non », utiliser `| bool`

> ⚠️ `is file` · `is directory` · `is exists` · `is link` sont évalués **sur le contrôleur** :
> sur web1, `'/opt/app' is directory` → `false`, alors que le répertoire existe
> → sur la cible, utiliser `stat` : `app_dir.stat.isdir` → `true`

</div>
</div>

<!--
Vérifié sur le lab : /opt/app existe sur web1 (créé par 01-premier-playbook) mais pas sur le contrôleur.
Le test de chemin répond false, le module stat répond true.
-->

---

# Rôles : rendre réutilisable

<div class="cols">
<div>

```text
roles/motd/
├── defaults/main.yml   # variables "par défaut"
├── vars/main.yml       # variables internes
├── tasks/main.yml      # point d'entrée
├── handlers/main.yml
├── templates/motd.j2
├── files/
└── meta/main.yml       # dépendances, galaxy
```

```bash
ansible-galaxy role init roles/motd
```

</div>
<div>

```yaml
# Utilisation statique
- hosts: web
  roles:
    - motd
    - role: nginx
      vars: { nginx_port: 8080 }

# Ou dynamique, au fil des tâches
  tasks:
    - ansible.builtin.include_role:
        name: motd
      when: app_env == 'prod'
```

- Un rôle = **une responsabilité**, paramétrable par `defaults/`

</div>
</div>

---

# Collections & Galaxy

- Depuis Ansible 2.10, les modules sont livrés en **collections** : `namespace.collection.module`
  - `ansible.builtin.copy` · `community.docker.docker_container` · `amazon.aws.ec2_instance`
- Toujours utiliser le **FQCN** (nom complet) : lisible, sans ambiguïté
- `ansible-core` = moteur + `ansible.builtin` · paquet `ansible` = core + ~90 collections

```yaml
# requirements.yml
collections:
  - name: community.docker
    version: ">=4.0.0"
  - name: ansible.posix
roles:
  - name: geerlingguy.nginx
```

```bash
ansible-galaxy install -r requirements.yml
```

---

<!-- _class: lead -->

# 05 · Idempotence

## Lancer 1 fois ou 10 fois : même résultat

---

# Décrire un **état**, pas une **action**

<div class="cols">
<div class="card warn">

### Impératif
> « Ajoute la ligne `max_connections=200` au fichier »

Run 1 : 1 ligne · Run 2 : 2 lignes · Run 3 : 3 lignes…

</div>
<div class="card">

### Déclaratif
> « Le fichier **contient** la ligne `max_connections=200` »

Run 1 : `changed` · Run 2 : `ok` · Run 3 : `ok`

</div>
</div>

- Chaque module **vérifie avant d'agir** → `changed` seulement s'il a dû modifier
- Conséquences :
  - on peut **rejouer sans risque** (reprise après erreur, cron, CI)
  - `changed=0` au 2ᵉ passage = la preuve que le système est **conforme**
  - `--check --diff` devient un **détecteur de dérive**

---

# <span class="demo"></span> Le bon et le mauvais

```yaml
- name: "NON idempotent : shell + echo >> (ajoute une ligne à chaque run)"
  ansible.builtin.shell: echo "max_connections=200" >> /opt/app/bad.conf

- name: "Idempotent : lineinfile garantit un ÉTAT"
  ansible.builtin.lineinfile:
    path: /opt/app/good.conf
    line: max_connections=200
    create: true

- name: "Idempotent : command + creates (sautée si le fichier existe)"
  ansible.builtin.command: dd if=/dev/urandom of=/opt/app/secret.key bs=32 count=1
  args:
    creates: /opt/app/secret.key

- name: "Lecture seule : changed_when false"
  ansible.builtin.command: wc -l /opt/app/bad.conf
  register: bad_lines
  changed_when: false
```

---

<!-- _class: small -->

# <span class="demo"></span> Run 1 vs Run 2

<div class="cols">
<div>

**1er passage**

```text
TASK [NON idempotent : shell + echo >>]
changed: [web1]
TASK [Idempotent : lineinfile]
changed: [web1]
TASK [Idempotent : command + creates]
changed: [web1]
TASK [Constat]
ok: [web1] => bad.conf contient 1 ligne(s)

web1 : ok=6  changed=3
```

</div>
<div>

**2e passage**

```text
TASK [NON idempotent : shell + echo >>]
changed: [web1]                    ← ⚠️
TASK [Idempotent : lineinfile]
ok: [web1]
TASK [Idempotent : command + creates]
ok: [web1]
TASK [Constat]
ok: [web1] => bad.conf contient 2 ligne(s)

web1 : ok=6  changed=1
```

</div>
</div>

> Objectif : un 2ᵉ passage à **`changed=0`**. Tout `changed` résiduel est un bug… ou un mensonge.

---

# Les pièges classiques

| Piège | Remède |
|---|---|
| `command` / `shell` sont **toujours** `changed` | un module dédié ; sinon `creates:` / `removes:` |
| Commande de lecture qui pollue le récap | `changed_when: false` |
| Code retour non-zéro « normal » | `failed_when: result.rc not in [0, 1]` |
| `state: latest` | change dès qu'une MAJ sort → préférer `present` + version |
| Template avec `{{ now() }}` ou un ordre aléatoire | sortie déterministe (`sort`, pas d'horodatage) |
| Handlers non déclenchés après un échec | `--force-handlers`, `meta: flush_handlers` |

→ Détails : handlers perdus, puis deux outils pour attraper ces pièges : `ansible-lint` et **Molecule**

---

<!-- _class: tiny -->

# <span class="demo"></span> Handlers perdus après un échec

<div class="cols">
<div>

```yaml
# playbooks/05-handlers-perdus.yml (hosts: web1)
tasks:
  - name: Configurer l'application
    ansible.builtin.copy:
      dest: /opt/app/app.conf
      content: "port={{ app_port }}\n"
    notify: Redémarrer l'application

  - name: Migrer la base de données
    ansible.builtin.command: /bin/false   # échec simulé
    changed_when: false
    when: simulate_failure | bool
```

```text
# Run 1
changed: [web1]               ← notifié…
fatal: [web1]: FAILED!        ← … et perdu
# Run 2 : -e simulate_failure=false
ok: [web1]                    ← rien à notifier
web1 : changed=0  failed=0    ← tout est vert…
restarts.log : aucun redémarrage  ← ancienne conf !
```

</div>
<div>

- Une notification n'est traitée qu'**en fin de section** : si l'hôte échoue avant, elle est **perdue**
- Le run suivant ne la refait pas : le changement a déjà été « consommé » → **dérive invisible**

### Remède 1 — `--force-handlers`
Les handlers notifiés tournent **même si l'hôte a échoué**
```text
fatal: [web1]: FAILED!
RUNNING HANDLER [Redémarrer l'application]
changed: [web1]
```
À demeure : `force_handlers: true` (play) ou `force_handlers = True` (`ansible.cfg`). Sans effet sur un hôte **injoignable**.

### Remède 2 — `meta: flush_handlers`
Traiter les notifications **avant** l'étape risquée
```yaml
  - name: Appliquer les redémarrages en attente
    ansible.builtin.meta: flush_handlers
```

</div>
</div>

<!--
Démo : task demo:handlers:reset puis
  task demo:handlers                                   (échec, pas de redémarrage)
  task demo:handlers -- -e simulate_failure=false      (changed=0, toujours pas de redémarrage)
  task demo:handlers -- -e app_port=8082 --force-handlers  (le handler tourne malgré l'échec)
Variante flush_handlers : décommenter la tâche dans le playbook.
Molecule passe --force-handlers par défaut dans ses scénarios générés.
-->

---

<!-- _class: small -->

# <span class="demo"></span> `ansible-lint` : détecter les pièges sans rien exécuter

<div class="cols" style="grid-template-columns: 0.8fr 1.2fr">
<div>

```yaml
# exemples/a-corriger.yml (extrait)
- name: Lire la version de PHP
  ansible.builtin.command: php -v
  register: php_v

- name: Déployer la conf
  ansible.builtin.template: { ... }
  register: conf

- name: Redémarrer si la conf a changé
  ansible.builtin.service: { ... }
  when: conf.changed

- name: Installer jq
  apt:
    name: jq
    state: latest
```

</div>
<div>

```text
$ ansible-lint exemples/a-corriger.yml
no-changed-when: Commands should not change things
                 if nothing needs doing.
exemples/a-corriger.yml:7   Lire la version de PHP
no-handler: Tasks that run when changed should
            likely be handlers.
exemples/a-corriger.yml:22  Redémarrer si la conf a changé
package-latest: Package installs should not use latest.
exemples/a-corriger.yml:24  Installer jq
fqcn[action-core]: Use FQCN for builtin module actions (apt).
exemples/a-corriger.yml:25  Use `ansible.builtin.apt` …

Failed: 4 failure(s), 0 warning(s) … Rating: 2/5 star
```

- **Analyse statique** : lit le code sans l'exécuter → en **CI** et en hook **pre-commit**
- Règles d'idempotence : `no-changed-when`, `package-latest`, `no-handler`
- Exception assumée : `# noqa: <règle>` ; réglages d'équipe dans `.ansible-lint`

</div>
</div>

<!--
Démo : task demo:lint (4 erreurs attendues), puis task lab:lint (le lab passe).
Sortie réelle d'ansible-lint 26.11, légèrement condensée (colonnes, liens).
-->

---

<!-- _class: small -->

# <span class="demo"></span> Molecule : tester l'idempotence d'un rôle

<div class="cols">
<div>

```text
roles/motd/molecule/default/
├── molecule.yml    # instances, driver, séquence
├── converge.yml    # applique le rôle
└── verify.yml      # vérifie le résultat
```

```yaml
# molecule.yml
driver:
  name: docker
platforms:
  - name: motd-debian
    image: ansible-lab-web1:latest
    pre_build_image: true
scenario:
  test_sequence: [destroy, syntax, create, converge,
                  idempotence, verify, destroy]
```

</div>
<div>

```text
$ cd roles/motd && molecule test
default ➜ create:      Executed: Successful
default ➜ converge:    Executed: Successful   changed=2
default ➜ idempotence: Executed: Successful   changed=0
default ➜ verify:      Executed: Successful
default ➜ destroy:     Executed: Successful
```

```text
# Avec une tâche non idempotente ajoutée au rôle
CRITICAL Idempotence test failed because of the following tasks:
*  => motd : Horodater le déploiement
                                             (code retour 1)
```

- **create** une instance jetable → **converge** → **idempotence** : rejoue le rôle, **échoue** si une tâche est `changed` → **verify** (`assert`) → **destroy**
- Complète `ansible-lint` : trouve ce qui n'apparaît **qu'à l'exécution** (template non déterministe, `shell` oublié…)

</div>
</div>

<!--
Démo : task demo:molecule (≈ 1 min, nécessite l'image construite par task lab:up).
Molecule 26.11 + molecule-plugins[docker]. Le récapitulatif final de Molecule
affiche failed=0 même quand l'idempotence échoue : c'est le code retour (1) qui compte en CI.
Molecule ne détecte pas les handlers perdus : ses deux passages réussissent.
-->

---

<!-- _class: lead -->

# 06 · Déployer sur N serveurs

## Parallélisme, lots, tolérance aux pannes

---

# Les leviers

| Mot-clé | Effet |
|---|---|
| `forks` (ansible.cfg / `-f`) | nb d'hôtes traités **en parallèle** (défaut 5) |
| `serial: [1, "50%", "100%"]` | **lots successifs** : canari, puis moitié, puis le reste |
| `max_fail_percentage: 0` | stoppe les lots suivants dès qu'un hôte échoue |
| `any_errors_fatal: true` | arrête **tout** au premier échec |
| `strategy: free` | chaque hôte avance à son rythme (défaut : `linear`) |
| `throttle: 2` | limite une **tâche** précise (ex. appel API, migration) |
| `run_once: true` | une seule exécution pour tout le groupe |
| `delegate_to: lb01` | exécute ailleurs, pour le compte de l'hôte |
| `async` / `poll` | tâches longues en arrière-plan |

---

<!-- _class: tiny -->

# <span class="demo"></span> Rolling update

```yaml
- name: Déploiement progressif de l'application
  hosts: prod
  serial: [1, "50%", "100%"]       # canari → 50 % → reste
  max_fail_percentage: 0

  pre_tasks:
    - name: Retirer le serveur du load balancer
      ansible.builtin.debug: { msg: "LB: disable {{ inventory_hostname }}" }
      delegate_to: localhost

  tasks:
    - name: Déployer la release
      ansible.builtin.copy:
        dest: /opt/app/RELEASE
        content: "{{ app_release }}\n"
      notify: Redémarrer l'application

    - name: Healthcheck
      ansible.builtin.command: cat /opt/app/RELEASE
      register: deployed
      changed_when: false
      failed_when: deployed.stdout != app_release

  post_tasks:
    - name: Remettre le serveur dans le load balancer
      ansible.builtin.debug: { msg: "LB: enable {{ inventory_hostname }}" }
      delegate_to: localhost
```

---

<!-- _class: small -->

# <span class="demo"></span> Le déroulé

```bash
$ ansible-playbook playbooks/03-rolling-update.yml -e app_release=1.4.2
```

```text
PLAY [Déploiement progressif de l'application] ***************  ← lot 1 : canari
TASK [Retirer le serveur du load balancer]    ok: [api2 -> localhost]
TASK [Déployer la release]                    changed: [api2]
TASK [Healthcheck]                            ok: [api2]
RUNNING HANDLER [Redémarrer l'application]    ok: [api2]
TASK [Remettre le serveur dans le load balancer] ok: [api2 -> localhost]

PLAY [Déploiement progressif de l'application] ***************  ← lot 2 : 50 %
TASK [Déployer la release]                    changed: [api3]  changed: [web2]
...
PLAY [Déploiement progressif de l'application] ***************  ← lot 3 : reste
TASK [Déployer la release]                    changed: [web3]
...
```

- Si le canari échoue au healthcheck → **les autres lots ne partent pas**
- Le handler (restart) s'exécute **par lot**, avant la remise dans le LB

---

<!-- _class: lead -->

# 07 · Cas pratique

## Cartographier les versions applicatives du parc

<span class="tag">Pour approfondir</span> *non présenté en séance — rejouable avec* `task demo:versions`

<!--
Partie fournie dans le support pour approfondissement : on passe directement à la partie 08.
-->

---

# Le besoin

> « Quelles versions de PHP et de Node.js tournent en prod ? Sur quel OS ? Avec quel OpenSSL ? »
> — *le RSSI, le chef de projet, l'audit…*

### Étape 1 — ad hoc : la réponse brute, en 2 lignes

```bash
$ ansible web -a "php -r 'echo PHP_VERSION;'" -o
web2 | CHANGED | rc=0 | (stdout) 8.3.20
web3 | CHANGED | rc=0 | (stdout) 8.4.27
web1 | CHANGED | rc=0 | (stdout) 8.1.34

$ ansible api -a "node -v" -o
api1 | CHANGED | rc=0 | (stdout) v20.20.2
api3 | CHANGED | rc=0 | (stdout) v24.22.2
api2 | CHANGED | rc=0 | (stdout) v22.12.0
```

### Étape 2 — playbook : collecter, **consolider**, **présenter**

---

# Collecter : un rôle générique, piloté par l'inventaire

<div class="cols">
<div>

```yaml
# inventory/group_vars/web.yml
app_runtime: php
app_version_cmd: php -r 'echo PHP_VERSION;'
app_branch_depth: 2      # 8.3.20 → "8.3"

# inventory/group_vars/api.yml
app_runtime: nodejs
app_version_cmd: node -p process.versions.node
app_branch_depth: 1      # 22.12.0 → "22"
```

> Ajouter Java, Python, nginx… = **un fichier de variables**, zéro ligne de code.

</div>
<div>

```yaml
# roles/app_versions/tasks/main.yml
- name: Lire la version du runtime
  ansible.builtin.command: "{{ app_version_cmd }}"
  register: _version
  changed_when: false

- name: Exposer la version en fact
  ansible.builtin.set_fact:
    app_version: "{{ _version.stdout | trim }}"
    app_branch: >-
      {{ (_version.stdout | trim).split('.')
         [:app_branch_depth | int] | join('.') }}
```

</div>
</div>

---

# Consolider : un play sur `localhost` qui lit `hostvars`

```yaml
- name: Collecter les versions
  hosts: web:api
  gather_subset: [min]
  roles: [app_versions]

- name: Consolider et présenter
  hosts: localhost
  vars:
    report_hosts: "{{ groups['web'] + groups['api'] }}"
  tasks:
    - name: Générer les rapports (Markdown + CSV)
      ansible.builtin.template:
        src: "{{ item }}.j2"     # playbooks/templates/
        dest: "reports/{{ item }}"
      loop: [versions.md, versions.csv]
```

```jinja
{# playbooks/templates/versions.md.j2 #}
| Serveur | Env | OS | Runtime | Version | OpenSSL |
{% for h in report_hosts | sort %}{% set hv = hostvars[h] %}
| {{ h }} | {{ hv.app_env }} | {{ hv.ansible_facts.distribution }} | {{ hv.app_runtime }} | {{ hv.app_version }} | {{ hv.openssl_version }} |
{% endfor %}
```

---

# <span class="demo"></span> Résultat : `reports/versions.md`

| Serveur | Env | OS | Runtime | Version | OpenSSL (système) |
|---|---|---|---|---|---|
| api1 | staging | Debian 12.13 | nodejs | **20.20.2** | 3.0.22 |
| api2 | prod | Debian 12.8 | nodejs | **22.12.0** | 3.0.22 |
| api3 | prod | Debian 12.15 | nodejs | **24.22.2** | 3.0.22 |
| web1 | staging | Debian 13.2 | php | **8.1.34** | 3.5.7 |
| web2 | prod | Debian 12.10 | php | **8.3.20** | 3.0.22 |
| web3 | prod | Debian 13.7 | php | **8.4.27** | 3.5.7 |

- Le rapport est… **du Marp** : `task demo:reports` → slide HTML prête à présenter
- Également en CSV (`reports/versions.csv`) pour un tableur ou une CMDB

<!--
Le tableau ci-dessus est la sortie réelle du lab au moment de la préparation :
la relancer en direct montre que c'est bien généré.
-->

---

<!-- _class: lead -->

# 08 · Cas pratique

## Diagnostic CVE : « suis-je exposé ? » en 30 secondes

---

<!-- _class: small -->

# Le contexte : les bulletins d'octobre 2026

<div class="cols">
<div class="card warn">

### PHP — 21/10/2026 « PHP2Shell »
- **CVE-2026-109618** : RCE via corruption mémoire sur OOM
- **CVE-2026-109619** : RCE via état global résiduel entre requêtes
- CVSS v4 **9.2 — critique**, SAPI web : FPM, mod_php, CGI, FrankenPHP…
- Corrigé en **8.2.35 · 8.3.36 · 8.4.27 · 8.5.12**
- Mitigation *partielle* : `session.upload_progress.enabled = 0`
- 8.1 et avant : **plus de correctif officiel**

</div>
<div class="card warn">

### Node.js — 21/10/2026
- 10 CVE dont **4 High**, notamment :
  - **CVE-2026-111497** : écriture hors limites dans le décodeur Brotli (RCE possible)
  - **CVE-2026-105483 / 106069** : contournement de vérification du nom d'hôte TLS
- Corrigé en **22.23.4 · 24.22.1 · 26.12.1**
- Node.js 20 : **fin de vie depuis le 30/04/2026** → vulnérable par définition

</div>
</div>

Sources : php.net/archive/2026.php · nodejs.org/en/blog/vulnerability/october-2026-security-releases

<!--
Données vérifiées au 31/10/2026. À mettre à jour avant chaque session :
seul lab/vars/security_baseline.yml change, le playbook reste identique.
-->

---

# Étape 1 — Diagnostic ad hoc : exposé ? mitigé ?

```bash
# Quelles versions ?
$ ansible web -a "php -r 'echo PHP_VERSION;'" -o
$ ansible api -a "node -v" -o

# La mitigation PHP2Shell est-elle active ?
$ ansible web -m shell -a "php -i | grep session.upload_progress.enabled" -o
web2 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => On => On
web3 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => On => On
web1 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => On => On
```

> Réponse en quelques secondes sur 6 serveurs… ou sur 600 (`-f 50`).

---

# Étape 2 — Mitigation d'urgence, en ad hoc

```bash
$ ansible 'web:!web3' -b -m copy \
    -a "dest=/usr/local/etc/php/conf.d/zz-cve-2026-109619.ini
        content='session.upload_progress.enabled = 0\n'" -o
web2 | CHANGED => {"changed": true, "checksum": "1d0449…", "dest": "/usr/local/etc/php/conf.d/zz-cve-…", …}
web1 | CHANGED => {"changed": true, "checksum": "1d0449…", "dest": "/usr/local/etc/php/conf.d/zz-cve-…", …}

$ ansible web -m shell -a "php -i | grep session.upload_progress.enabled" -o
web2 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => Off => Off
web3 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => On => On
web1 | CHANGED | rc=0 | (stdout) session.upload_progress.enabled => Off => Off
```

- Rejouer la 1ʳᵉ commande → `SUCCESS`, `"changed": false` : **idempotent**
- ⚠️ Mitigation **partielle** : la seule vraie correction reste la **mise à jour**
- En vrai : `--check --diff` d'abord, puis `notify` d'un reload php-fpm

---

# Étape 3 — Industrialiser : un référentiel + un rôle

<div class="cols">
<div>

```yaml
# vars/security_baseline.yml
security_baseline:
  php:
    advisory: "PHP2Shell : CVE-2026-109618/109619"
    fixed_in:
      "8.2": "8.2.35"
      "8.3": "8.3.36"
      "8.4": "8.4.27"
      "8.5": "8.5.12"
  nodejs:
    advisory: "Oct. 2026 : 10 CVE dont 4 High"
    fixed_in:
      "22": "22.23.4"
      "24": "24.22.1"
      "26": "26.12.1"
# branche absente = plus supportée = EOL
```

</div>
<div>

```yaml
# roles/cve_check/tasks/main.yml (extrait)
- name: Évaluer l'exposition
  vars:
    _fixed_in: "{{ security_baseline[app_runtime]
                  .fixed_in[app_branch] | default('') }}"
  ansible.builtin.set_fact:
    cve_status: >-
      {%- if not _fixed_in -%} EOL
      {%- elif app_version is version(_fixed_in, '<')
      -%} VULNERABLE
      {%- else -%} OK
      {%- endif -%}
```

+ un play `localhost` qui interroge **endoflife.date** (module `uri`, une seule fois) pour la dernière version et la date de fin de vie de chaque branche

</div>
</div>

---

<!-- _class: small -->

# <span class="demo"></span> Résultat : `reports/cve.md`

```bash
$ ansible-playbook playbooks/20-cve-diag.yml
```

| Serveur | Runtime | Version | Corrigé en | Dernière | Fin de vie | Mitigation | Statut |
|---|---|---|---|---|---|---|---|
| api1 | nodejs | 20.20.2 | aucun | 20.20.2 | 2026-04-30 | n/a | ⛔ EOL |
| api2 | nodejs | 22.12.0 | 22.23.4 | 22.23.4 | 2027-04-30 | n/a | ⚠️ VULNÉRABLE |
| api3 | nodejs | 24.22.2 | 24.22.1 | 24.22.2 | 2028-04-30 | n/a | ✅ OK |
| web1 | php | 8.1.34 | aucun | 8.1.34 | 2025-12-31 | active | ⛔ EOL |
| web2 | php | 8.3.20 | 8.3.36 | 8.3.36 | 2027-12-31 | active | ⚠️ VULNÉRABLE |
| web3 | php | 8.4.27 | 8.4.27 | 8.4.27 | 2028-12-31 | absente | ✅ OK |

```text
TASK [Synthèse]
ok: [localhost] => msg: 4/6 serveur(s) à traiter : web1, web2, api1, api2
```

---

# Aller plus loin

- **Contrôle officiel Node.js** : `npx is-my-node-vulnerable` sur chaque cible
  → `task demo:cve:deep` : api1 *EOL* · api2 **45 CVE connues** · api3 OK
- **Paquets de la distribution** :
  `apt list --upgradable` · `debsecan` · `dnf updateinfo list --security`
- **Dépendances applicatives** : `osv-scanner`, `trivy fs`, `grype` lancés par Ansible, résultats rapatriés avec `fetch`
- **Planifier** le diagnostic (AWX / AAP, Semaphore, CI) et **échouer** si `VULNERABLE` → alerte
- **Remédier** avec le même outil : playbook de mise à jour en `serial`, healthcheck, rollback

> Diagnostic, mitigation, remédiation, preuve de conformité : **le même inventaire, le même outil.**

---

<!-- _class: lead -->

# 09 · Pour finir

---

# Bonnes pratiques

<div class="cols">
<div>

### Code
- **FQCN** partout : `ansible.builtin.copy`
- Un module dédié plutôt que `shell`
- Nommer **toutes** les tâches
- Des rôles petits, paramétrés par `defaults/`
- Tout dans **git**, revu en merge request

</div>
<div>

### Exploitation
- `--check --diff` avant d'appliquer
- **Ansible Vault** (ou un coffre externe) pour les secrets
- `ansible-lint` + **Molecule** en CI
- *Execution Environments* + `ansible-navigator` : un runtime reproductible
- AWX / AAP / Semaphore : RBAC, planification, journal d'audit

</div>
</div>

---

# À retenir

1. **Sans agent, en push** : SSH + Python, et c'est parti
2. **Inventaire** = *sur quoi* · **playbook** = *quoi et dans quel ordre* · **rôle** = *réutiliser*
3. **Idempotence** : on décrit un état ; un 2ᵉ passage doit donner `changed=0`
4. **N serveurs** : `forks`, `serial`, `max_fail_percentage`, `delegate_to`
5. Au-delà de la configuration : **audit et diagnostic** du parc en quelques secondes

### Ressources
- docs.ansible.com · galaxy.ansible.com · `ansible-doc <module>`
- Ce dépôt : `task lab:up` puis `task --list`

---

<!-- _class: lead -->
<!-- _paginate: false -->

# Questions ?

## Merci !

`task lab:down` pour ranger le lab 🧹
