# Ansible — Exercices avancés (Débutant → Expert)

> 22 exercices progressifs en français pour pratiquer Ansible.
> Chaque exercice comporte : **contexte**, **objectifs**, **prérequis**, **indices**, **solution** (repliée).

---

## Niveau Débutant

### Exercice 1 — Premier ping

**Contexte** : vérifier la connectivité Ansible vers 2 machines virtuelles.

**Objectifs** :
- Créer un inventaire INI avec un groupe `lab`.
- Pinger les hôtes.

**Prérequis** : 2 VM accessibles en SSH (ou conteneurs Docker avec sshd).

**Indices** : `ansible -i inventory <pattern> -m ping`.

<details>
<summary>Solution</summary>

`inventory.ini` :
```ini
[lab]
vm1 ansible_host=192.168.56.10 ansible_user=vagrant
vm2 ansible_host=192.168.56.11 ansible_user=vagrant
```

```bash
ansible -i inventory.ini lab -m ping
```
</details>

---

### Exercice 2 — Installer un paquet

**Objectif** : installer `htop` sur tout le groupe `lab` en mode ad-hoc.

<details>
<summary>Solution</summary>

```bash
ansible -i inventory.ini lab -m apt -a "name=htop state=present update_cache=yes" -b
```
</details>

---

### Exercice 3 — Premier playbook

**Objectif** : écrire `web.yml` qui installe et démarre Nginx, et vérifier qu'il répond en HTTP.

<details>
<summary>Solution</summary>

```yaml
---
- name: Setup Nginx
  hosts: lab
  become: true
  tasks:
    - name: Installer Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Démarrer Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

    - name: Vérifier la page d'accueil
      ansible.builtin.uri:
        url: http://localhost
        status_code: 200
```

```bash
ansible-playbook -i inventory.ini web.yml
```
</details>

---

### Exercice 4 — Variables et debug

**Objectif** : afficher la distribution et la version du noyau de chaque hôte.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  tasks:
    - debug:
        msg: "{{ inventory_hostname }} → {{ ansible_facts.distribution }} {{ ansible_facts.distribution_version }} / kernel {{ ansible_facts.kernel }}"
```
</details>

---

### Exercice 5 — Boucle d'utilisateurs

**Objectif** : créer 3 utilisateurs (`alice`, `bob`, `carol`) avec des groupes spécifiques, à partir d'une liste de dictionnaires.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  vars:
    users:
      - { name: alice, groups: 'sudo' }
      - { name: bob,   groups: 'docker' }
      - { name: carol, groups: 'developers' }
  tasks:
    - name: Créer les utilisateurs
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        append: yes
        state: present
        shell: /bin/bash
      loop: "{{ users }}"
      loop_control:
        label: "{{ item.name }}"
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — Template Jinja2

**Objectif** : générer `/etc/motd` avec le hostname, l'IP et l'environnement (`vars: env: prod`).

<details>
<summary>Solution</summary>

`templates/motd.j2` :
```
==============================================
  Hôte    : {{ inventory_hostname }}
  IP      : {{ ansible_default_ipv4.address }}
  Env     : {{ env }}
  Géré par: Ansible — {{ ansible_date_time.iso8601 }}
==============================================
```

```yaml
- hosts: lab
  become: true
  vars: { env: prod }
  tasks:
    - template:
        src: motd.j2
        dest: /etc/motd
        mode: '0644'
```
</details>

---

### Exercice 7 — Handler

**Objectif** : déposer un `nginx.conf` custom et **reloader** Nginx uniquement si le fichier a changé.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  tasks:
    - template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        validate: 'nginx -t -c %s'
      notify: Reload Nginx
  handlers:
    - name: Reload Nginx
      service: { name: nginx, state: reloaded }
```
</details>

---

### Exercice 8 — Condition `when` multi-OS

**Objectif** : installer `nginx` via `apt` sur Debian/Ubuntu et via `dnf` sur RHEL/CentOS.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  tasks:
    - apt:
        name: nginx
        state: present
        update_cache: yes
      when: ansible_facts.os_family == 'Debian'

    - dnf:
        name: nginx
        state: present
      when: ansible_facts.os_family == 'RedHat'
```
</details>

---

### Exercice 9 — Convertir un playbook en rôle

**Objectif** : refactoriser le playbook Nginx en rôle `nginx`.

<details>
<summary>Solution</summary>

```bash
ansible-galaxy role init roles/nginx
```

`roles/nginx/tasks/main.yml` :
```yaml
- ansible.builtin.apt: { name: nginx, state: present, update_cache: yes }
- ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Reload Nginx
- ansible.builtin.service: { name: nginx, state: started, enabled: yes }
```

`roles/nginx/handlers/main.yml` :
```yaml
- name: Reload Nginx
  service: { name: nginx, state: reloaded }
```

`roles/nginx/defaults/main.yml` :
```yaml
nginx_port: 80
nginx_worker_connections: 1024
```

`site.yml` :
```yaml
- hosts: lab
  become: true
  roles: [nginx]
```
</details>

---

### Exercice 10 — Ansible Vault

**Objectif** : créer `vars/secrets.yml` chiffré contenant `db_password` et l'utiliser dans un playbook.

<details>
<summary>Solution</summary>

```bash
ansible-vault create vars/secrets.yml
# contenu : db_password: "S3cret!"
```

```yaml
- hosts: lab
  become: true
  vars_files: [vars/secrets.yml]
  tasks:
    - debug:
        msg: "Mot de passe DB chargé (longueur : {{ db_password | length }})"
```

```bash
ansible-playbook site.yml --ask-vault-pass
```
</details>

---

### Exercice 11 — Inventaire dynamique AWS

**Objectif** : lister vos EC2 `Role=web` en `eu-west-1` avec un inventaire dynamique.

<details>
<summary>Solution</summary>

`inventory.aws_ec2.yml` :
```yaml
plugin: amazon.aws.aws_ec2
regions: [eu-west-1]
filters:
  tag:Role: web
  instance-state-name: running
keyed_groups:
  - key: tags.Env
    prefix: env
hostnames:
  - tag:Name
```

```bash
ansible-inventory -i inventory.aws_ec2.yml --graph
ansible -i inventory.aws_ec2.yml all -m ping
```
</details>

---

### Exercice 12 — Loops imbriquées

**Objectif** : créer N répertoires, et dans chacun, créer 2 sous-répertoires `logs` et `data`.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  vars:
    apps: [app1, app2, app3]
    subs: [logs, data]
  tasks:
    - file:
        path: "/opt/{{ item.0 }}/{{ item.1 }}"
        state: directory
        mode: '0755'
      loop: "{{ apps | product(subs) | list }}"
```
</details>

---

### Exercice 13 — `register` + boucle conditionnelle

**Objectif** : lancer `systemctl is-active` sur une liste de services et afficher seulement ceux **inactifs**.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  vars:
    svcs: [nginx, ssh, cron, fail2ban]
  tasks:
    - command: "systemctl is-active {{ item }}"
      register: status
      changed_when: false
      failed_when: false
      loop: "{{ svcs }}"

    - debug:
        msg: "{{ item.item }} → {{ item.stdout }}"
      loop: "{{ status.results }}"
      when: item.stdout != 'active'
```
</details>

---

## Niveau Avancé

### Exercice 14 — Block / rescue / always

**Objectif** : tenter un déploiement ; en cas d'échec, rollback ; toujours nettoyer le verrou.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  become: true
  tasks:
    - file: { path: /var/run/deploy.lock, state: touch }

    - block:
        - command: /opt/deploy/release.sh
      rescue:
        - debug: { msg: "Rollback en cours…" }
        - command: /opt/deploy/rollback.sh
      always:
        - file: { path: /var/run/deploy.lock, state: absent }
```
</details>

---

### Exercice 15 — Rolling update avec health-check

**Objectif** : déployer une nouvelle version d'application sur 4 nœuds, **2 à la fois**, en sortant chaque nœud du load-balancer avant.

<details>
<summary>Solution</summary>

Voir le guide complet section 17.1 — l'exercice consiste à reproduire ce playbook et à le faire fonctionner avec HAProxy local.
</details>

---

### Exercice 16 — Tester un rôle avec Molecule

**Objectif** : initialiser un scénario Molecule Docker pour le rôle `nginx` et passer `molecule test`.

<details>
<summary>Solution</summary>

```bash
pip install molecule molecule-plugins[docker] ansible-lint
cd roles/nginx
molecule init scenario default -d docker
# Éditer molecule/default/molecule.yml pour utiliser une image ubuntu:22.04
molecule test
```

`molecule/default/converge.yml` :
```yaml
- hosts: all
  become: true
  roles: [nginx]
```

`molecule/default/verify.yml` :
```yaml
- hosts: all
  tasks:
    - uri: { url: http://localhost, status_code: 200 }
```
</details>

---

### Exercice 17 — Custom filter Jinja2

**Objectif** : écrire un filtre Python custom `reverse_dns` et l'utiliser dans un template.

<details>
<summary>Solution</summary>

`filter_plugins/dns_filters.py` :
```python
import socket

class FilterModule:
    def filters(self):
        return {'reverse_dns': self.reverse_dns}

    def reverse_dns(self, ip):
        try:
            return socket.gethostbyaddr(ip)[0]
        except Exception:
            return ip
```

Usage :
```jinja2
{{ ansible_default_ipv4.address | reverse_dns }}
```
</details>

---

### Exercice 18 — `delegate_to` pour notifier Slack

**Objectif** : envoyer un message Slack depuis le **contrôleur** (localhost) à la fin d'un déploiement.

<details>
<summary>Solution</summary>

```yaml
- hosts: lab
  tasks:
    - name: Notifier Slack
      community.general.slack:
        token: "{{ slack_token }}"
        msg: "Déploiement terminé sur {{ ansible_play_hosts | length }} hôte(s)."
      delegate_to: localhost
      run_once: true
```
</details>

---

### Exercice 19 — Multi-environnement (prod / staging)

**Objectif** : organiser un repo Ansible pour gérer 2 environnements avec des `group_vars` distinctes.

<details>
<summary>Solution</summary>

```
infra/
├── inventories/
│   ├── prod/
│   │   ├── hosts.yml
│   │   └── group_vars/all.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/all.yml
├── roles/
└── site.yml
```

```bash
ansible-playbook -i inventories/prod    site.yml
ansible-playbook -i inventories/staging site.yml
```
</details>

---

### Exercice 20 — Pipeline CI (GitHub Actions)

**Objectif** : créer `.github/workflows/ansible.yml` qui exécute `ansible-lint` et `molecule test` sur PR.

<details>
<summary>Solution</summary>

```yaml
name: Ansible CI
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install ansible ansible-lint yamllint molecule molecule-plugins[docker]
      - run: ansible-lint
      - run: yamllint .
      - run: cd roles/nginx && molecule test
```
</details>

---

## Niveau Expert

### Exercice 21 — Déploiement Kubernetes via Ansible

**Objectif** : utiliser la collection `kubernetes.core` pour créer un Namespace, un Deployment Nginx et un Service ClusterIP.

<details>
<summary>Solution</summary>

```yaml
- hosts: localhost
  collections: [kubernetes.core]
  tasks:
    - k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata: { name: demo }

    - k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata: { name: nginx, namespace: demo }
          spec:
            replicas: 3
            selector: { matchLabels: { app: nginx } }
            template:
              metadata: { labels: { app: nginx } }
              spec:
                containers:
                  - name: nginx
                    image: nginx:1.27
                    ports: [{ containerPort: 80 }]

    - k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata: { name: nginx, namespace: demo }
          spec:
            selector: { app: nginx }
            ports: [{ port: 80, targetPort: 80 }]
```
</details>

---

### Exercice 22 — Module Python custom

**Objectif** : écrire un module Ansible custom `hello` qui prend un paramètre `name` et renvoie `Bonjour <name>`.

<details>
<summary>Solution</summary>

`library/hello.py` :
```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(name=dict(type='str', required=True)),
        supports_check_mode=True,
    )
    name = module.params['name']
    module.exit_json(changed=False, message=f"Bonjour {name}")

if __name__ == '__main__':
    main()
```

Usage :
```yaml
- hosts: localhost
  tasks:
    - hello: { name: "Ansible" }
      register: r
    - debug: { var: r.message }
```
</details>

---

## Pour aller plus loin

- Refactorez un de vos rôles existants en suivant les bonnes pratiques du guide.
- Ajoutez du **fact caching** (Redis ou jsonfile) pour accélérer vos runs.
- Implémentez un **callback plugin** custom pour logger en JSON dans Loki.
- Mettez en place un **playbook de durcissement CIS** (regardez `dev-sec.hardening`).

