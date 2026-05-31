# Ansible — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Ansible, de l'installation à la production.

---

## Table des matières

1. [Introduction à Ansible](#1-introduction-à-ansible)
2. [Installation et configuration](#2-installation-et-configuration)
3. [Inventaires](#3-inventaires)
4. [Modules et commandes ad-hoc](#4-modules-et-commandes-ad-hoc)
5. [Playbooks](#5-playbooks)
6. [Variables et Facts](#6-variables-et-facts)
7. [Templates Jinja2](#7-templates-jinja2)
8. [Handlers et notifications](#8-handlers-et-notifications)
9. [Rôles Ansible](#9-rôles-ansible)
10. [Ansible Vault](#10-ansible-vault)
11. [Collections et Galaxy](#11-collections-et-galaxy)
12. [Stratégies d'exécution](#12-stratégies-dexécution)
13. [Gestion des erreurs](#13-gestion-des-erreurs)
14. [Tests et debugging](#14-tests-et-debugging)
15. [Bonnes pratiques de production](#15-bonnes-pratiques-de-production)
16. [Ansible Tower / AWX](#16-ansible-tower--awx)
17. [Cas pratiques avancés](#17-cas-pratiques-avancés)

---

## 1. Introduction à Ansible

### 1.1 Qu'est-ce qu'Ansible ?

**Ansible** est un outil open-source d'**automatisation IT** qui couvre :

- Le **provisioning** (création de ressources)
- La **gestion de configuration** (config management)
- Le **déploiement d'applications**
- L'**orchestration** (séquencement multi-machines)
- La **sécurité et conformité** (audits, durcissement)

Créé par **Michael DeHaan** en **2012**, racheté par **Red Hat** en **2015**, puis intégré au portefeuille **IBM** depuis 2019.

### 1.2 Philosophie

Ansible repose sur quatre principes :

| Principe | Description |
|---|---|
| **Agentless** | Aucun agent à installer sur les machines cibles. Seul SSH (Linux) ou WinRM (Windows) suffit. |
| **Déclaratif** | On décrit l'**état souhaité**, pas la procédure. |
| **Idempotent** | Exécuter un playbook plusieurs fois aboutit au même état final, sans effets de bord. |
| **Lisible** | Le YAML est compréhensible par des non-développeurs (ops, SRE, sécurité). |

### 1.3 Push vs Pull

- **Push (mode par défaut)** : le contrôleur Ansible se connecte aux nœuds et y exécute les tâches.
- **Pull** (`ansible-pull`) : chaque nœud télécharge son playbook depuis un dépôt Git et l'exécute localement (utile pour des flottes massives).

```
┌──────────────────┐  SSH/WinRM   ┌────────┐
│ Contrôleur       │ ───────────► │ Node A │
│ (laptop / CI)    │ ───────────► │ Node B │
│ playbooks/roles  │ ───────────► │ Node C │
└──────────────────┘              └────────┘
```

### 1.4 Cas d'usage typiques

- Configurer 200 serveurs Nginx avec la même version et le même `nginx.conf`.
- Déployer un cluster Kubernetes via `kubespray` (qui est… un playbook Ansible).
- Patcher OS et appliquer un baseline CIS.
- Orchestrer un déploiement **blue/green** sur un cluster d'app.
- Provisionner des VMs sur AWS/Azure/GCP/VMware et les configurer dans la foulée.

### 1.5 Comparaison rapide

| Outil | Agent | Langage | Modèle | Forces |
|---|---|---|---|---|
| **Ansible** | Non | YAML + Jinja2 | Push (par défaut) | Simplicité, agentless |
| **Puppet** | Oui | DSL Ruby-like | Pull | Énormes flottes, modèle ressource mature |
| **Chef** | Oui | Ruby DSL | Pull | Flexibilité programmatique |
| **SaltStack** | Oui (minions) | YAML + Jinja | Push (event-driven) | Performance, scalabilité |
| **Terraform** | Non | HCL | Déclaratif | Provisioning cloud (≠ config) |

> Règle pragmatique : **Terraform provisionne**, **Ansible configure**. Les deux sont complémentaires.

---

## 2. Installation et configuration

### 2.1 Installer Ansible

Sur **Linux (Debian/Ubuntu)** :

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

Sur **RHEL/CentOS/Fedora** :

```bash
sudo dnf install -y ansible-core
```

Via **pip** (recommandé pour les versions récentes ou environnement virtuel) :

```bash
python3 -m venv ~/.venvs/ansible
source ~/.venvs/ansible/bin/activate
pip install --upgrade pip
pip install ansible           # méta-paquet (core + collections communes)
# ou minimal :
pip install ansible-core
```

Sur **macOS** :

```bash
brew install ansible
```

Sur **Windows** : Ansible **ne tourne pas nativement** comme contrôleur sous Windows. Utilisez **WSL2** (Ubuntu) puis installez via pip.

### 2.2 Premier test

```bash
ansible --version
ansible localhost -m ping
```

Sortie attendue :

```json
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 2.3 Fichier `ansible.cfg`

Ansible lit la configuration dans cet ordre (1er trouvé gagne) :

1. `ANSIBLE_CONFIG` (variable d'environnement)
2. `./ansible.cfg` (dossier courant)
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

Exemple de `ansible.cfg` minimaliste :

```ini
[defaults]
inventory       = ./inventory.ini
remote_user     = ubuntu
host_key_checking = False
retry_files_enabled = False
forks           = 20
gathering       = smart
fact_caching    = jsonfile
fact_caching_connection = /tmp/ansible_facts
stdout_callback = yaml
deprecation_warnings = False

[ssh_connection]
pipelining = True
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s
```

> **Astuce performance** : `pipelining = True` peut diviser par 2 le temps d'exécution. Désactivez `requiretty` dans `/etc/sudoers` côté cible.

### 2.4 Clés SSH

```bash
ssh-keygen -t ed25519 -C "ansible-controller"
ssh-copy-id ubuntu@host1.example.com
ansible all -i "host1.example.com," -u ubuntu -m ping
```

---

## 3. Inventaires

L'inventaire décrit **quels hôtes** Ansible peut gérer et **comment** s'y connecter.

### 3.1 Format INI

```ini
[web]
web1.example.com
web2.example.com ansible_port=2222

[db]
db1.example.com ansible_user=postgres

[prod:children]
web
db

[prod:vars]
environment=production
ntp_server=ntp.example.com
```

### 3.2 Format YAML (recommandé)

```yaml
all:
  children:
    web:
      hosts:
        web1.example.com:
        web2.example.com:
          ansible_port: 2222
    db:
      hosts:
        db1.example.com:
          ansible_user: postgres
    prod:
      children:
        web:
        db:
      vars:
        environment: production
```

### 3.3 Inventaires dynamiques

Pour AWS, on utilise le plugin `aws_ec2` :

`inventory.aws_ec2.yml` :

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
filters:
  instance-state-name: running
hostnames:
  - tag:Name
  - dns-name
```

```bash
ansible-inventory -i inventory.aws_ec2.yml --graph
```

Autres plugins natifs : `azure_rm`, `gcp_compute`, `vmware_vm_inventory`, `kubernetes.core.k8s`, `docker_containers`.

### 3.4 Variables par hôte / par groupe

Structure recommandée :

```
inventory/
├── production
├── staging
├── group_vars/
│   ├── all.yml
│   ├── web.yml
│   └── db.yml
└── host_vars/
    ├── web1.example.com.yml
    └── db1.example.com.yml
```

Ansible chargera automatiquement ces fichiers.

### 3.5 Patterns de sélection

```bash
ansible all -m ping                       # tout
ansible web -m ping                       # un groupe
ansible 'web:db' -m ping                  # union
ansible 'web:!web2.example.com' -m ping   # exclusion
ansible 'web:&prod' -m ping               # intersection
ansible '~web[0-9]+' -m ping              # regex
```

---

## 4. Modules et commandes ad-hoc

### 4.1 Anatomie d'une commande ad-hoc

```bash
ansible <pattern> -m <module> -a "<arguments>" [-b] [-K]
```

- `-b` : become (sudo)
- `-K` : demande le mot de passe sudo
- `-i` : inventaire
- `-u` : user SSH

### 4.2 Modules essentiels

```bash
# Ping
ansible all -m ping

# Exécuter une commande shell
ansible web -m command -a "uptime"
ansible web -m shell  -a "ps aux | grep nginx"

# Copier un fichier
ansible web -m copy -a "src=./nginx.conf dest=/etc/nginx/nginx.conf owner=root mode=0644" -b

# Gérer un service
ansible web -m service -a "name=nginx state=restarted enabled=yes" -b

# Installer un paquet
ansible web -m apt -a "name=nginx state=present update_cache=yes" -b
ansible web -m yum -a "name=httpd state=latest" -b

# Créer un utilisateur
ansible all -m user -a "name=deploy shell=/bin/bash groups=sudo append=yes" -b

# Gérer un fichier / répertoire
ansible all -m file -a "path=/opt/app state=directory mode=0755 owner=deploy" -b

# Récupérer les facts
ansible web -m setup
ansible web -m setup -a "filter=ansible_distribution*"
```

### 4.3 `command` vs `shell` vs `raw`

| Module | Shell | Variables/pipes | Idempotent | Quand l'utiliser |
|---|---|---|---|---|
| `command` | non | non | non par défaut | commande simple sans pipe |
| `shell` | oui | oui | non par défaut | besoin de `|`, `&&`, redirection |
| `raw` | oui | oui | non | SSH sans Python sur la cible |

> Préférez **toujours** un module métier (`apt`, `service`, `file`) à `command`/`shell` pour bénéficier de l'idempotence.

---

## 5. Playbooks

### 5.1 Structure de base

```yaml
---
- name: Configurer les serveurs web
  hosts: web
  become: true
  gather_facts: true
  vars:
    nginx_port: 80
  tasks:
    - name: Installer Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Déployer la configuration
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Reload Nginx

    - name: Démarrer Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Exécution :

```bash
ansible-playbook -i inventory.yml site.yml
ansible-playbook -i inventory.yml site.yml --check --diff
ansible-playbook -i inventory.yml site.yml --tags nginx
ansible-playbook -i inventory.yml site.yml --limit web1.example.com
```

### 5.2 Mots-clés essentiels au niveau Play

| Mot-clé | Rôle |
|---|---|
| `hosts` | Pattern d'hôtes |
| `become` | Élève en sudo |
| `become_user` | User cible du sudo |
| `gather_facts` | Récupère les facts (true par défaut) |
| `serial` | N hôtes en parallèle (rolling) |
| `strategy` | `linear` (défaut), `free`, `host_pinned` |
| `vars` / `vars_files` | Variables |
| `vars_prompt` | Demander à l'utilisateur |
| `pre_tasks` / `tasks` / `post_tasks` | Phases |
| `handlers` | Tâches déclenchées par `notify` |
| `roles` | Liste de rôles à exécuter |
| `tags` | Étiquettes pour exécution sélective |
| `any_errors_fatal` | Stop tout si une erreur |

### 5.3 Conditions `when`

```yaml
- name: Installer firewalld sur RHEL
  ansible.builtin.dnf:
    name: firewalld
    state: present
  when:
    - ansible_facts['os_family'] == 'RedHat'
    - ansible_facts['distribution_major_version'] | int >= 8
```

### 5.4 Boucles

```yaml
- name: Créer plusieurs utilisateurs
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: alice, groups: 'sudo,docker' }
    - { name: bob,   groups: 'docker' }
    - { name: carol, groups: 'developers' }
  loop_control:
    label: "{{ item.name }}"
```

Boucles avancées :

```yaml
- name: Installer plusieurs paquets
  ansible.builtin.apt:
    name: "{{ packages }}"          # batch (plus rapide)
    state: present
  vars:
    packages: [git, vim, htop, jq]

- name: Boucler sur un dict
  ansible.builtin.debug:
    msg: "{{ item.key }} = {{ item.value }}"
  loop: "{{ my_dict | dict2items }}"
```

### 5.5 Blocks

```yaml
- name: Bloc avec rescue
  block:
    - name: Tentative risquée
      ansible.builtin.command: /opt/risky-script.sh
  rescue:
    - name: Notifier l'échec
      ansible.builtin.debug:
        msg: "L'échec a été géré."
  always:
    - name: Nettoyer
      ansible.builtin.file:
        path: /tmp/state.lock
        state: absent
```

### 5.6 Tags

```yaml
- name: Installer Nginx
  ansible.builtin.apt: { name: nginx, state: present }
  tags: [install, nginx]
```

```bash
ansible-playbook site.yml --tags "install"
ansible-playbook site.yml --skip-tags "nginx"
ansible-playbook site.yml --list-tags
```

---

## 6. Variables et Facts

### 6.1 Sources de variables

Par ordre **croissant** de priorité (les dernières gagnent) :

1. Defaults d'un rôle (`roles/x/defaults/main.yml`)
2. `group_vars/all`
3. `group_vars/<group>`
4. `host_vars/<host>`
5. Facts (collectés)
6. `vars:` du play
7. `vars_files:`
8. Variables passées par `set_fact`
9. `--extra-vars` en ligne de commande (priorité max)

### 6.2 Définir et utiliser

```yaml
vars:
  app_name: monapp
  app_versions:
    web: 1.2.3
    api: 4.5.6

tasks:
  - debug:
      msg: "Déploiement de {{ app_name }} version {{ app_versions.web }}"
```

### 6.3 `set_fact` et `register`

```yaml
- name: Lancer une commande et stocker la sortie
  ansible.builtin.command: hostname -f
  register: hostname_result

- name: Définir un fact dérivé
  ansible.builtin.set_fact:
    fqdn: "{{ hostname_result.stdout }}"

- debug:
    var: fqdn
```

### 6.4 Facts

```bash
ansible web1 -m setup | less
```

Quelques facts utiles :

```yaml
ansible_facts['distribution']            # Ubuntu, CentOS, ...
ansible_facts['distribution_version']    # 22.04
ansible_facts['architecture']            # x86_64
ansible_facts['memtotal_mb']
ansible_facts['processor_vcpus']
ansible_facts['default_ipv4']['address']
ansible_facts['interfaces']
```

### 6.5 `vars_prompt`

```yaml
vars_prompt:
  - name: db_password
    prompt: "Mot de passe DB ?"
    private: yes
    encrypt: sha512_crypt
```

---

## 7. Templates Jinja2

### 7.1 Bases

`templates/nginx.conf.j2` :

```nginx
worker_processes {{ ansible_processor_vcpus }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    upstream backend {
        {% for host in groups['app'] %}
        server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:8080;
        {% endfor %}
    }

    server {
        listen {{ nginx_port }};
        server_name {{ server_name }};

        {% if ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate     /etc/ssl/{{ inventory_hostname }}.crt;
        ssl_certificate_key /etc/ssl/{{ inventory_hostname }}.key;
        {% endif %}

        location / {
            proxy_pass http://backend;
        }
    }
}
```

```yaml
- name: Générer nginx.conf
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
    validate: 'nginx -t -c %s'
  notify: Reload Nginx
```

### 7.2 Filtres utiles

```jinja2
{{ name | upper }}                       # MAJUSCULES
{{ list | join(',') }}                   # joindre
{{ data | to_nice_yaml }}                # YAML pretty
{{ data | to_json }}                     # JSON
{{ password | password_hash('sha512') }} # hash crypt
{{ ip | ipaddr('network') }}             # plugins réseau
{{ items | selectattr('active') | list }}# filtrage
{{ path | basename }}
{{ value | default('fallback') }}
{{ value | mandatory }}                  # erreur si absent
```

---

## 8. Handlers et notifications

```yaml
tasks:
  - name: Copier la conf
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify:
      - Restart app
      - Send alert

handlers:
  - name: Restart app
    ansible.builtin.service:
      name: app
      state: restarted

  - name: Send alert
    ansible.builtin.uri:
      url: https://alerts.example.com/hook
      method: POST
```

Caractéristiques :

- Un handler **ne s'exécute qu'une fois** par play, même si notifié N fois.
- Il s'exécute à la **fin du play** (ou via `meta: flush_handlers`).
- Si la tâche notifiante n'est pas en `changed`, le handler **n'est pas déclenché**.

```yaml
- name: Forcer les handlers immédiatement
  ansible.builtin.meta: flush_handlers
```

---

## 9. Rôles Ansible

### 9.1 Structure d'un rôle

```
roles/
└── nginx/
    ├── defaults/main.yml      # variables par défaut (faible priorité)
    ├── vars/main.yml          # variables figées (haute priorité)
    ├── tasks/main.yml         # tâches principales
    ├── handlers/main.yml      # handlers
    ├── templates/             # .j2
    ├── files/                 # fichiers à copier
    ├── meta/main.yml          # dépendances, métadonnées Galaxy
    ├── README.md
    └── tests/
```

Créer avec :

```bash
ansible-galaxy role init nginx
```

### 9.2 Utiliser un rôle

```yaml
- hosts: web
  become: true
  roles:
    - role: common
    - role: nginx
      vars:
        nginx_port: 8080
```

Forme avancée avec `import_role` / `include_role` :

```yaml
tasks:
  - import_role:               # statique (au parsing)
      name: nginx
    when: deploy_web | bool

  - include_role:              # dynamique (à l'exécution)
      name: app
      tasks_from: deploy.yml
```

### 9.3 Dépendances

`roles/app/meta/main.yml` :

```yaml
dependencies:
  - role: common
  - role: nginx
    vars:
      nginx_port: 8080
```

### 9.4 Galaxy

```bash
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy role install -r requirements.yml
```

`requirements.yml` :

```yaml
roles:
  - name: geerlingguy.docker
    version: 7.0.0
collections:
  - name: community.general
  - name: amazon.aws
    version: 7.2.0
```

---

## 10. Ansible Vault

### 10.1 Chiffrer un fichier entier

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt vars/prod.yml
ansible-vault decrypt vars/prod.yml
ansible-vault rekey secrets.yml          # changer le mdp
```

### 10.2 Chiffrer une seule valeur

```bash
ansible-vault encrypt_string 'MonSuperMot2Passe' --name 'db_password'
```

Donne :

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3433333061363236623238343...
```

### 10.3 Fournir le mot de passe à l'exécution

```bash
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
```

`ansible.cfg` :

```ini
[defaults]
vault_password_file = ~/.vault_pass.txt
```

### 10.4 Vault IDs (multi-environnements)

```bash
ansible-vault encrypt --vault-id prod@prompt secrets-prod.yml
ansible-vault encrypt --vault-id dev@dev_pass.txt secrets-dev.yml
ansible-playbook site.yml --vault-id prod@prompt --vault-id dev@dev_pass.txt
```

---

## 11. Collections et Galaxy

Depuis Ansible 2.10, les modules sont packagés en **collections** (`<namespace>.<collection>`).

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws:==7.2.0
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-galaxy collection list
```

Utilisation explicite :

```yaml
- name: Lister les buckets S3
  amazon.aws.s3_bucket_info:
  register: buckets
```

---

## 12. Stratégies d'exécution

### 12.1 `serial` et rolling updates

```yaml
- hosts: web
  serial: 2          # 2 hôtes à la fois
  # serial: "20%"    # ou en pourcentage
  # serial: [1, 5, 10]  # batchs croissants
  tasks:
    - ...
```

### 12.2 `strategy`

| Strategy | Comportement |
|---|---|
| `linear` (défaut) | Toutes les machines attendent la fin d'une tâche avant la suivante |
| `free` | Chaque machine avance à son rythme |
| `host_pinned` | Variante de `free`, avec affinité |

### 12.3 `delegate_to` et `run_once`

```yaml
- name: Ajouter un enregistrement DNS (une seule fois, sur le contrôleur)
  community.general.nsupdate:
    server: dns.example.com
    record: "{{ inventory_hostname }}"
    value: "{{ ansible_default_ipv4.address }}"
  delegate_to: localhost
  run_once: true
```

### 12.4 Forks

```ini
[defaults]
forks = 50          # parallélisme (5 par défaut)
```

---

## 13. Gestion des erreurs

```yaml
- name: Tentative pouvant échouer
  ansible.builtin.command: /opt/may-fail
  register: result
  ignore_errors: true
  failed_when:
    - result.rc != 0
    - "'expected_warning' not in result.stderr"
  changed_when: "'modified' in result.stdout"
  retries: 3
  delay: 5
  until: result.rc == 0
```

`block/rescue/always` (voir section 5.5).

`any_errors_fatal: true` au niveau du play pour stopper **tous** les hôtes si l'un échoue.

---

## 14. Tests et debugging

### 14.1 Outils intégrés

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check          # dry-run
ansible-playbook site.yml --check --diff   # voir les diffs
ansible-playbook site.yml -vvv             # verbose (jusqu'à -vvvv)
ansible-playbook site.yml --step           # confirmer chaque tâche
ansible-playbook site.yml --start-at-task "Installer Nginx"
```

### 14.2 Debug et assertion

```yaml
- ansible.builtin.debug:
    var: ansible_facts.distribution

- ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['Debian', 'RedHat']
      - app_port | int > 1024
    fail_msg: "Conditions non remplies."
```

### 14.3 `ansible-lint`

```bash
pip install ansible-lint
ansible-lint playbooks/
ansible-lint --fix playbooks/
```

### 14.4 Molecule (tests de rôles)

```bash
pip install molecule molecule-plugins[docker]
cd roles/nginx
molecule init scenario default -d docker
molecule test            # create -> converge -> idempotence -> verify -> destroy
molecule converge        # juste appliquer
molecule login           # se connecter au conteneur de test
```

---

## 15. Bonnes pratiques de production

### 15.1 Structure de projet recommandée

```
infra/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   ├── prod/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   └── host_vars/
│   └── staging/
├── roles/
│   ├── common/
│   ├── nginx/
│   └── app/
├── collections/        # collections vendored
├── playbooks/
│   ├── site.yml
│   ├── web.yml
│   └── db.yml
└── group_vars/         # variables globales
```

### 15.2 Règles d'or

1. **Idempotence d'abord** : préférer modules métier à `shell`.
2. **Nommer chaque tâche** (sans exception).
3. **Versionner** les collections et rôles externes.
4. **Chiffrer tout secret** avec Vault — jamais de mots de passe en clair dans Git.
5. **Tester en `--check --diff`** avant `apply` en prod.
6. **Limiter le scope** avec `--limit` lors des hotfix.
7. **Pas de logique métier dans les playbooks** : la mettre dans des **rôles**.
8. **Tagger** install / config / deploy pour exécutions ciblées.
9. **Linter** systématiquement (`ansible-lint`, `yamllint`).
10. **CI/CD** : faire tourner `ansible-playbook --check` en pipeline sur chaque PR.

### 15.3 Performance

| Réglage | Effet |
|---|---|
| `forks = 50` | Plus de parallélisme |
| `pipelining = True` | Moins de SSH round-trips |
| `gather_facts: smart` + `fact_caching` | Évite de rescanner |
| `strategy: free` | Hôtes lents ne bloquent pas les rapides |
| `serial:` | Évite l'effet "thundering herd" |
| Batch des paquets | `apt: name: [a,b,c]` au lieu d'une boucle |

---

## 16. Ansible Tower / AWX

**Tower** (commercial Red Hat, devenu **Ansible Automation Platform**) et **AWX** (open-source) sont des UI/API au-dessus d'Ansible.

Apports :

- Interface web pour lancer/planifier des jobs
- RBAC fin (utilisateurs, équipes, organisations)
- Stockage centralisé des credentials
- Workflows (chaînage de jobs)
- Logs centralisés, notifications (Slack, email, webhook)
- API REST + collection `awx.awx`
- Inventaires synchronisés (cloud, satellite…)

Installation AWX (Kubernetes) :

```bash
helm repo add awx-operator https://ansible.github.io/awx-operator/
helm install awx-operator awx-operator/awx-operator -n awx --create-namespace
kubectl apply -f awx-cr.yml
```

---

## 17. Cas pratiques avancés

### 17.1 Déploiement rolling avec health-check

```yaml
- hosts: web
  serial: "25%"
  max_fail_percentage: 0
  pre_tasks:
    - name: Sortir du LB
      community.general.haproxy:
        backend: web
        host: "{{ inventory_hostname }}"
        state: disabled
      delegate_to: lb1.example.com

  tasks:
    - name: Déployer la nouvelle version
      ansible.builtin.unarchive:
        src: "https://artifacts/app-{{ app_version }}.tar.gz"
        dest: /opt/app
        remote_src: yes
      notify: Restart app

    - meta: flush_handlers

    - name: Vérifier que l'app répond
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
      retries: 12
      delay: 5
      register: hc
      until: hc.status == 200

  post_tasks:
    - name: Réintégrer au LB
      community.general.haproxy:
        backend: web
        host: "{{ inventory_hostname }}"
        state: enabled
      delegate_to: lb1.example.com
```

### 17.2 Provisioning AWS + configuration

```yaml
- hosts: localhost
  gather_facts: false
  vars:
    region: eu-west-1
  tasks:
    - amazon.aws.ec2_instance:
        name: "web-{{ item }}"
        key_name: ops
        instance_type: t3.small
        image_id: ami-0abcd1234
        security_groups: [web-sg]
        region: "{{ region }}"
        tags: { Role: web, Env: prod }
        wait: yes
      loop: [1, 2, 3]
      register: created

- hosts: tag_Role_web
  become: true
  roles:
    - common
    - nginx
```

### 17.3 Intégration Kubernetes

```yaml
- hosts: localhost
  collections: [kubernetes.core]
  tasks:
    - k8s:
        state: present
        definition: "{{ lookup('file', 'manifests/deployment.yml') | from_yaml }}"
    - k8s_info:
        kind: Pod
        namespace: default
      register: pods
```

### 17.4 Modèle GitOps

1. Repo Git unique pour `inventories/`, `roles/`, `playbooks/`.
2. CI : `ansible-lint` + `molecule test` à chaque PR.
3. CD : `ansible-playbook site.yml -i inventories/prod` déclenché par merge sur `main`.
4. Notifications dans Slack via callback plugin.
5. Audit trail dans AWX/Tower.

---

## Conclusion

Vous disposez maintenant d'une base solide pour utiliser Ansible **en production**. Les étapes recommandées :

1. Pratiquez avec les [exercices avancés](./EXERCICES_AVANCES.md).
2. Préparez votre entretien avec le [guide d'interview](./INTERVIEW.md).
3. Lisez la doc officielle : <https://docs.ansible.com>
4. Contribuez à des rôles publics sur Galaxy.

> **Souvenez-vous** : automatisation ≠ complexité. Le meilleur playbook est celui qu'un collègue peut lire et comprendre en 5 minutes.

