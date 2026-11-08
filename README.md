# Formation Ansible (1h30)

Support de cours [Marp](https://marp.app) + lab Docker pour rejouer toutes les démos.

## Prérequis

| Outil | Usage |
|---|---|
| [Task](https://taskfile.dev) | lanceur de commandes (`task --list`) |
| Node.js (`npx`) **ou** Docker | génération du PDF (Marp CLI) |
| Docker + Compose | lab de démo (6 conteneurs) |
| Ansible (paquet `ansible` ou `ansible-core`) | démos — testé avec ansible-core 2.21 |
| `ansible-lint`, `molecule`, `molecule-plugins[docker]` | démos qualité (`lab:lint`, `demo:lint`, `demo:molecule`) — ex. `pipx inject --include-apps ansible ansible-lint molecule "molecule-plugins[docker]"` |

## Générer le support

```bash
task pdf                      # -> dist/ansible.pdf
task pdf MARP_ENGINE=docker   # sans Node/Chromium local, via l'image marpteam/marp-cli
task html                     # -> dist/ansible.html
task pdf:notes                # PDF + notes du présentateur (timings, messages clés)
task watch                    # prévisualisation live sur http://localhost:8080
```

## Lab

```bash
task lab:deps     # collections community.docker + ansible.posix (dans lab/collections)
task lab:up       # web1-3 (PHP 8.1 / 8.3.20 / 8.4), api1-3 (Node 20 / 22.12.0 / 24)
task --list       # toutes les démos, dans l'ordre du cours
task lab:reset    # repartir d'un parc vierge entre deux sessions
task lab:down
```

Le lab se connecte aux conteneurs par `docker exec` (`community.docker.docker`) au lieu de SSH :
les playbooks sont identiques à ceux d'un vrai parc.

| Démo | Slide | Commande |
|---|---|---|
| ping / setup / ad hoc | 02 · CLI | `task demo:ping` · `demo:setup` · `demo:adhoc` |
| Inventaire | 03 | `task demo:inventory` |
| Groupes calculés | 03 | `task demo:keyed-groups` · `demo:group-by` |
| Premier playbook + rôle | 04 | `task demo:playbook` (2 fois → `changed=0`) |
| Idempotence | 05 | `task demo:idempotence` (2 fois) |
| Handlers perdus | 05 | `task demo:handlers` → `-- -e simulate_failure=false` → `-- -e app_port=8082 --force-handlers` (`demo:handlers:reset`) |
| ansible-lint | 05 | `task demo:lint` (4 erreurs attendues) · `task lab:lint` (lab propre) |
| Molecule | 05 | `task demo:molecule` (après `task lab:up`, qui construit l'image) |
| Rolling update | 06 | `task demo:rolling RELEASE=1.4.2` |
| Versions applicatives | 07 | `task demo:versions` → `lab/reports/versions.{md,csv}` |
| Diagnostic CVE | 08 | `task demo:diag` → `demo:mitigate` → `demo:cve` (`demo:cve:deep`) |
| Rapports en slides | 07-08 | `task demo:reports` → `dist/report-*.html` |

## Avant chaque session

Les bulletins de sécurité évoluent : mettre à jour
[lab/vars/security_baseline.yml](lab/vars/security_baseline.yml) (versions corrigées par branche)
et la slide « Le contexte » du cas pratique 2. Les données de fin de vie sont récupérées en direct
sur [endoflife.date](https://endoflife.date). État au 31/10/2026 :

- PHP : [bulletin du 21/10/2026](https://www.php.net/archive/2026.php) — CVE-2026-109618 / 109619, corrigé en 8.2.35, 8.3.36, 8.4.27, 8.5.12
- Node.js : [security releases du 21/10/2026](https://nodejs.org/en/blog/vulnerability/october-2026-security-releases) — corrigé en 22.23.4, 24.22.1, 26.12.1
