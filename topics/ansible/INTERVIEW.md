# Ansible — Préparation à l'entretien

> ~50 questions/réponses en français pour préparer un entretien DevOps/SRE avec un focus Ansible.
> Organisées par niveau : **Junior**, **Intermédiaire**, **Senior/Architecte**.

---

## Junior

<details>
<summary>1. Qu'est-ce qu'Ansible et à quoi sert-il ?</summary>

Ansible est un outil open-source d'**automatisation IT** permettant la **gestion de configuration**, le **déploiement d'applications**, le **provisioning** et l'**orchestration** multi-machines. Il fonctionne **sans agent**, en utilisant SSH (Linux) ou WinRM (Windows), et décrit l'état souhaité dans des fichiers **YAML** appelés *playbooks*.
</details>

<details>
<summary>2. Pourquoi Ansible est-il dit "agentless" ?</summary>

Aucun démon Ansible n'a besoin d'être installé sur les machines cibles. Le contrôleur Ansible se connecte en SSH/WinRM, pousse les modules nécessaires (en Python en général), les exécute, puis se déconnecte. Cela simplifie le déploiement et réduit la surface d'attaque.
</details>

<details>
<summary>3. Citez les composants principaux d'Ansible.</summary>

- **Inventory** : liste des hôtes et groupes.
- **Module** : unité de code exécutée (ex. `apt`, `service`).
- **Task** : appel à un module avec des paramètres.
- **Play** : ensemble de tâches appliquées à un groupe d'hôtes.
- **Playbook** : fichier YAML contenant un ou plusieurs plays.
- **Role** : structure réutilisable regroupant tâches, variables, templates, handlers.
- **Handler** : tâche déclenchée par notification (`notify`).
- **Facts** : informations collectées sur l'hôte cible.
- **Variables** : valeurs paramétrables.
- **Collection** : package distribuable de modules, rôles, plugins.
</details>

<details>
<summary>4. Qu'est-ce que l'idempotence en Ansible ?</summary>

Une opération est **idempotente** si l'exécuter une ou plusieurs fois aboutit toujours au même état final. La plupart des modules Ansible sont idempotents : ils vérifient l'état actuel avant d'agir et ne font des modifications que si nécessaire. Cela permet de relancer un playbook sans craindre des effets de bord.
</details>

<details>
<summary>5. Différence entre `command` et `shell` ?</summary>

- `command` exécute la commande **sans shell** : pas de pipes, redirections, variables d'environnement, ni globbing.
- `shell` passe la commande à `/bin/sh` (ou autre) : tout cela fonctionne.

`shell` est plus puissant mais moins sûr. On préfère un module métier (`apt`, `copy`, `service`…) qui sera idempotent, plutôt que `command`/`shell`.
</details>

<details>
<summary>6. Qu'est-ce qu'un inventaire ? Donnez un exemple.</summary>

L'inventaire liste les hôtes gérés. Il peut être **statique** (INI/YAML) ou **dynamique** (plugin AWS, Azure, K8s…).

```ini
[web]
web1.example.com
web2.example.com

[db]
db1.example.com
```
</details>

<details>
<summary>7. Comment exécuter un playbook ?</summary>

```bash
ansible-playbook -i inventory.yml site.yml
```

Options utiles : `--check` (dry-run), `--diff` (affichage des diffs), `--limit host1` (cibler), `--tags install`, `-vvv` (verbose).
</details>

<details>
<summary>8. À quoi sert `gather_facts` ?</summary>

Au début d'un play, Ansible exécute le module `setup` pour collecter des **facts** sur la machine cible (OS, IP, mémoire, CPU, interfaces…). Ces facts sont utilisables comme variables (`ansible_facts.distribution`).

Désactiver `gather_facts: false` accélère les exécutions si on n'en a pas besoin.
</details>

<details>
<summary>9. Comment passer une variable en ligne de commande ?</summary>

```bash
ansible-playbook site.yml -e "app_version=1.2.3 env=prod"
ansible-playbook site.yml -e "@vars/prod.yml"
```

`--extra-vars` a la **priorité maximale**.
</details>

<details>
<summary>10. Qu'est-ce qu'un handler ?</summary>

Un handler est une tâche **déclenchée par notification** (`notify`) et exécutée à la fin du play. Il s'exécute **une seule fois** même s'il est notifié plusieurs fois. Cas typique : recharger un service après modification de sa configuration.
</details>

---

## Intermédiaire

<details>
<summary>11. Expliquez l'ordre de précédence des variables.</summary>

Du plus faible au plus fort (simplifié) :
1. defaults d'un rôle
2. `group_vars/all`
3. `group_vars/<group>`
4. `host_vars/<host>`
5. facts
6. vars du play
7. `vars_files`
8. `set_fact`
9. **`--extra-vars`** (plus fort)

Voir la doc officielle pour la liste exhaustive (22+ niveaux).
</details>

<details>
<summary>12. Quelle est la structure d'un rôle Ansible ?</summary>

```
roles/<name>/
├── defaults/main.yml
├── vars/main.yml
├── tasks/main.yml
├── handlers/main.yml
├── templates/
├── files/
├── meta/main.yml
└── tests/
```

Créer avec `ansible-galaxy role init <name>`.
</details>

<details>
<summary>13. `import_role` vs `include_role` ?</summary>

- `import_role` : **statique**, résolu au parsing. Les conditions `when` s'appliquent à chaque tâche du rôle.
- `include_role` : **dynamique**, résolu à l'exécution. Les conditions s'appliquent à l'inclusion entière. Permet de générer des inclusions en boucle.
</details>

<details>
<summary>14. Qu'est-ce qu'Ansible Vault ?</summary>

Un mécanisme natif de **chiffrement** (AES-256) pour stocker des secrets dans Git. On peut chiffrer un fichier entier (`ansible-vault encrypt`) ou une seule valeur (`ansible-vault encrypt_string`). Le mot de passe est fourni à l'exécution via `--ask-vault-pass` ou `--vault-password-file`.
</details>

<details>
<summary>15. Comment faire un rolling update ?</summary>

Avec `serial:` au niveau du play :

```yaml
- hosts: web
  serial: 2          # ou "25%"
  max_fail_percentage: 0
```

On combine avec `pre_tasks` (sortir du LB) et `post_tasks` (réintégrer), et un health-check via `uri` + `retries/delay/until`.
</details>

<details>
<summary>16. Différence entre `register` et `set_fact` ?</summary>

- `register` : capture le résultat d'**une tâche** dans une variable (scope : host).
- `set_fact` : définit explicitement une variable (peut être basée sur un calcul, un filtre, etc.), persistante pour la suite du play.
</details>

<details>
<summary>17. Qu'est-ce qu'un block ? Quand l'utiliser ?</summary>

Un `block` regroupe plusieurs tâches. Combiné avec `rescue` et `always`, il permet une gestion d'erreurs similaire à `try/except/finally` :

```yaml
- block: [...]
  rescue: [...]
  always: [...]
```

Utile pour rollback ou nettoyage de ressources temporaires.
</details>

<details>
<summary>18. Comment optimiser la performance d'Ansible ?</summary>

- `forks = 50` (ou plus) dans `ansible.cfg`
- `pipelining = True` (réduit les SSH round-trips)
- `gather_facts: smart` + `fact_caching` (jsonfile/redis)
- `strategy: free` (les hôtes ne se bloquent pas mutuellement)
- Batch des paquets : `apt: name: [a,b,c]` au lieu d'une boucle
- Désactiver `gather_facts` si non nécessaire
- Utiliser `async`/`poll: 0` pour les tâches longues
- ControlMaster SSH (mux des connexions)
</details>

<details>
<summary>19. Que sont les collections ?</summary>

Depuis Ansible 2.10, les modules ne sont plus livrés monolithiquement. Une **collection** est un package versionné (`<namespace>.<collection>`) contenant modules, rôles, plugins. Installation : `ansible-galaxy collection install community.general`.
</details>

<details>
<summary>20. Comment tester un rôle ?</summary>

- `ansible-playbook --syntax-check` et `--check --diff`
- `ansible-lint` pour le style/erreurs courantes
- `yamllint` pour le YAML
- **Molecule** : framework de test (create → converge → idempotence → verify → destroy) sur Docker, Podman, Vagrant, EC2…
</details>

<details>
<summary>21. Comment garantir qu'un playbook est idempotent ?</summary>

- Utiliser des modules **métier** (pas `shell` quand on peut éviter).
- Si `shell` est inévitable, ajouter `creates:` ou `removes:` ou définir `changed_when:`.
- Tester avec Molecule : la phase `idempotence` re-exécute le playbook et vérifie qu'aucune tâche n'est `changed`.
</details>

<details>
<summary>22. Que fait `delegate_to: localhost` ?</summary>

La tâche s'exécute sur le **contrôleur** (localhost), mais dans le contexte de l'hôte itéré (variables accessibles). Cas d'usage : appels API externes, notifications, mises à jour DNS centralisées.
</details>

<details>
<summary>23. Comment cibler dynamiquement les hôtes (par tag AWS) ?</summary>

Avec le plugin d'inventaire `amazon.aws.aws_ec2`, on génère automatiquement des groupes via `keyed_groups`. On peut alors écrire `hosts: tag_Role_web`.
</details>

<details>
<summary>24. Différence entre `vars`, `defaults` et `vars` (de rôle) ?</summary>

- `defaults/main.yml` : valeurs **par défaut**, faciles à surcharger (plus faible priorité).
- `vars/main.yml` : valeurs **figées** du rôle (haute priorité).
- `vars:` du play : surcharge encore.
- `--extra-vars` : surcharge tout.
</details>

<details>
<summary>25. À quoi servent les tags ?</summary>

À exécuter **sélectivement** un sous-ensemble de tâches : `--tags install`, `--skip-tags db`, ou `--list-tags`. Pratique pour des hotfix ou re-exécuter une seule partie d'un playbook.
</details>

<details>
<summary>26. Comment debugger un playbook qui échoue ?</summary>

- `-vvv` ou `-vvvv` pour voir les détails SSH/modules.
- `--start-at-task "nom"` pour reprendre après une tâche.
- `--step` pour confirmer chaque tâche.
- `debug:` pour afficher des variables.
- Activer `ANSIBLE_DEBUG=1` pour les traces internes.
- Lancer en mode `--check --diff` pour comprendre l'écart d'état.
</details>

<details>
<summary>27. Comment gérer plusieurs environnements (prod/staging/dev) ?</summary>

Un inventaire par environnement, sous `inventories/<env>/hosts.yml`, avec `group_vars/` et `host_vars/` propres à chaque env. Les rôles restent identiques ; seules les variables changent. Vault par environnement (`--vault-id prod@...`).
</details>

<details>
<summary>28. Que sont les "strategy plugins" ?</summary>

Ils contrôlent l'ordre d'exécution :
- `linear` : tâche par tâche, attendre tous les hôtes.
- `free` : chaque hôte avance à son rythme.
- `host_pinned` : agrégation par lots.
</details>

---

## Senior / Architecte

<details>
<summary>29. Comment scaler Ansible à 10 000 hôtes ?</summary>

- Utiliser **AWX/Ansible Automation Platform** avec un pool d'**execution nodes**.
- `ansible-pull` : chaque nœud télécharge et exécute son playbook depuis Git (modèle pull, scalable horizontalement).
- Augmenter `forks` mais surveiller la RAM.
- **Fact caching** distribué (Redis).
- Stratégie `free`, `serial:` en pourcentage.
- Découpage par région/AZ avec inventaires dynamiques.
- Pour des cas extrêmes : SaltStack ou solutions event-driven.
</details>

<details>
<summary>30. Comment intégrer Ansible dans une chaîne GitOps ?</summary>

- Repo Git unique pour playbooks/rôles/inventaires (signés).
- Pipeline CI : `ansible-lint`, `yamllint`, `molecule test`, `ansible-playbook --check` sur PR.
- Pipeline CD : sur merge `main`, exécuter `ansible-playbook` (via runner GitHub Actions, GitLab CI, Jenkins ou AWX webhook).
- Inventaires dynamiques (cloud) pour garantir la cohérence.
- Audit trail (AWX) + notifications Slack.
- **Drift detection** : exécuter régulièrement en `--check` et alerter si des changements seraient appliqués.
</details>

<details>
<summary>31. Comment sécuriser un déploiement Ansible ?</summary>

- **Vault** pour tous les secrets.
- Clés SSH dédiées, sans passphrase pour CI mais protégées dans le secret store (Vault HashiCorp, AWS SM, etc.).
- `become_user`, `become_method: sudo` avec sudoers limité.
- Pas de mots de passe SSH (clés uniquement).
- Signature des collections (`ansible-galaxy collection verify`).
- Sandbox Tower/AWX (execution environments isolés en conteneurs).
- Auditer les playbooks via `ansible-lint` + revue de code.
- Principe du moindre privilège : un compte Ansible par usage.
</details>

<details>
<summary>32. Comment écrire un module Ansible custom ?</summary>

Créer un fichier Python dans `library/` ou dans une collection. Utiliser `AnsibleModule` de `ansible.module_utils.basic`. Le module doit :
- Définir un `argument_spec`.
- Supporter `check_mode` si possible.
- Renvoyer un dict via `module.exit_json(changed=..., ...)` ou `module.fail_json()`.
- Être idempotent.

Voir l'exercice 22.
</details>

<details>
<summary>33. Comment éviter le piège du "snowflake server" avec Ansible ?</summary>

- **Re-jouer** régulièrement le playbook complet (idempotence garantit l'état).
- **Drift detection** automatisée (cron + `--check`).
- Pas de modifications manuelles : tout passe par Git/playbook.
- Reconstruction périodique (immutable infra) plutôt que patch sur place.
- Tests Molecule pour valider chaque changement.
</details>

<details>
<summary>34. `async` / `poll` : à quoi ça sert ?</summary>

Pour exécuter des tâches **longues** sans bloquer la connexion SSH :

```yaml
- name: Build lourd
  command: /opt/build.sh
  async: 3600    # timeout
  poll: 0        # fire and forget
  register: job

- name: Attendre la fin
  async_status: { jid: "{{ job.ansible_job_id }}" }
  register: r
  until: r.finished
  retries: 60
  delay: 60
```
</details>

<details>
<summary>35. Comment gérer les dépendances entre rôles ?</summary>

Dans `meta/main.yml` :
```yaml
dependencies:
  - role: common
  - role: postgres
    vars: { pg_version: 15 }
```

Les dépendances s'exécutent **avant** le rôle. Alternative : composition explicite dans le playbook (`roles: [common, postgres, app]`) — souvent plus claire.
</details>

<details>
<summary>36. Quels callbacks plugins connaissez-vous ?</summary>

- `yaml`, `default`, `dense` : format de sortie console.
- `json`, `minimal` : pour parsing.
- `profile_tasks` : temps par tâche.
- `mail`, `slack`, `splunk`, `logentries` : notifications.
- Custom : on peut écrire un callback Python pour, par exemple, pousser vers Loki/Elasticsearch.
</details>

<details>
<summary>37. Ansible vs Terraform : comment les positionner ?</summary>

- **Terraform** : provisioning d'infrastructure immuable (déclaratif, state, plan/apply).
- **Ansible** : configuration de systèmes vivants, orchestration, déploiements applicatifs.
- Ils sont **complémentaires** : Terraform crée les VMs/K8s, Ansible configure l'OS et déploie l'app.
- On peut aussi faire l'inverse (Ansible provisionne via `amazon.aws.ec2_instance`), mais Terraform est généralement supérieur pour la gestion d'état cloud.
</details>

<details>
<summary>38. Comment fonctionne le "fact caching" et quand l'activer ?</summary>

`gather_facts` peut être coûteux. Avec `fact_caching`, les facts collectés sont stockés (jsonfile, redis, memcached) avec un TTL. Les exécutions suivantes les lisent depuis le cache.

```ini
[defaults]
gathering = smart
fact_caching = redis
fact_caching_connection = localhost:6379
fact_caching_timeout = 86400
```
</details>

<details>
<summary>39. Comment factoriser plusieurs playbooks proches ?</summary>

- Extraire les tâches communes en **rôles**.
- Utiliser **`import_playbook`** pour composer (`- import_playbook: web.yml`).
- Centraliser les variables dans `group_vars`.
- Conditionner avec `when:` selon `inventory_hostname_short` / facts.
</details>

<details>
<summary>40. Comment gérer la rotation de secrets avec Vault ?</summary>

- Stocker les secrets dans HashiCorp Vault (pas Ansible Vault) avec la collection `community.hashi_vault`.
- Au runtime, le playbook lit le secret via `hashi_vault` lookup ; aucun secret n'est commité.
- Rotation : un cron change le secret dans Vault ; le playbook le récupère systématiquement frais.
- Pour Ansible Vault, utiliser `ansible-vault rekey` lors d'une rotation du mot de passe maître.
</details>

<details>
<summary>41. Quelle est la différence entre un "execution environment" et un environnement Python virtuel ?</summary>

- **venv** : isolation au niveau OS.
- **Execution Environment (EE)** : image OCI/Docker contenant Ansible, collections, Python deps et bibliothèques système. Construite avec **ansible-builder**, utilisée par AWX/Navigator. C'est l'unité standard pour exécuter Ansible en prod de façon reproductible.
</details>

<details>
<summary>42. Comment éviter qu'un playbook "casse la prod" ?</summary>

- Phase **`--check --diff`** systématique.
- `serial:` faible (1 ou 2 hôtes).
- `max_fail_percentage: 0` pour stopper au premier échec.
- `any_errors_fatal: true`.
- Health-checks et rollback automatiques (`block/rescue`).
- Sortir/réintégrer du LB.
- Notifications Slack avant/après.
- Revue de code obligatoire sur les PRs touchant la prod.
</details>

<details>
<summary>43. Comment gérer plusieurs versions de collections simultanément ?</summary>

Chaque projet peut avoir son `requirements.yml` et installer les collections en local (`-p ./collections`). Ansible respectera `ANSIBLE_COLLECTIONS_PATH`. Coupler avec un **execution environment** dédié par projet pour isolation totale.
</details>

<details>
<summary>44. Comment écrire un plugin de filtre Jinja2 ?</summary>

Voir exercice 17. Un fichier dans `filter_plugins/`, classe `FilterModule` avec méthode `filters()` retournant un dict `nom → fonction`.
</details>

<details>
<summary>45. Différence entre Ansible et SaltStack ?</summary>

| Critère | Ansible | SaltStack |
|---|---|---|
| Agent | Non (SSH) | Oui (minion) |
| Performance | Moyen (SSH coûteux) | Très bonne (ZeroMQ) |
| Modèle | Push principal | Push + event-driven |
| Langage | YAML + Jinja2 | YAML + Jinja2 (similaire) |
| Courbe | Facile | Plus raide |
| Écosystème | Énorme | Plus restreint |
| Cas d'usage | Petites/moyennes flottes, ad-hoc | Flottes massives, event-driven |
</details>

<details>
<summary>46. Comment gérer Windows avec Ansible ?</summary>

- Connexion via **WinRM** (HTTP/HTTPS) ou SSH (Windows 2019+).
- Variables `ansible_connection: winrm`, `ansible_winrm_transport: ntlm/kerberos/credssp`.
- Modules dédiés `ansible.windows.*` (win_package, win_service, win_user, win_chocolatey…).
- Idempotence garantie comme sur Linux.
</details>

<details>
<summary>47. Comment utiliser Ansible pour la conformité (CIS) ?</summary>

Utiliser des rôles publics éprouvés (`devsec.hardening`, `cis-ansible`). Combiner avec :
- **Compliance scan** (OpenSCAP, Lynis) en pre-tasks.
- Rapport HTML/JSON publié par `community.general.archive` ou stocké en S3.
- Tickets automatiques en cas d'échec (Jira plugin).
</details>

<details>
<summary>48. Scénario : un playbook échoue uniquement sur 1 hôte sur 50. Comment investiguez-vous ?</summary>

1. Re-jouer avec `--limit hostX -vvv`.
2. Vérifier la connectivité SSH manuellement.
3. Vérifier la version Python sur la cible.
4. Comparer les facts (`ansible hostX -m setup > a.json`, idem sur un hôte OK, diff).
5. Vérifier les permissions sudo et `/etc/sudoers`.
6. Si réseau : MTU, proxy, DNS.
7. Activer `ANSIBLE_KEEP_REMOTE_FILES=1` pour conserver les modules sur la cible et les rejouer en debug.
</details>

<details>
<summary>49. Comment écririez-vous un playbook qui auto-déclare le contrôleur dans son propre inventaire dynamique ?</summary>

Combiner `add_host` (en mémoire) avec un appel API au début du play, puis itérer sur le groupe créé dans un play suivant :

```yaml
- hosts: localhost
  tasks:
    - uri: { url: "https://cmdb/api/hosts" }
      register: cmdb
    - add_host: { name: "{{ item.name }}", groups: dyn, ansible_host: "{{ item.ip }}" }
      loop: "{{ cmdb.json }}"

- hosts: dyn
  tasks: [...]
```
</details>

<details>
<summary>50. Quel est le futur d'Ansible selon vous ?</summary>

- **Event-Driven Ansible (EDA)** : déclenchement de playbooks par événements (webhooks, Kafka, alertes Prometheus).
- Adoption accrue des **Execution Environments** (containerisés).
- Convergence avec **GitOps** (ArgoCD pour K8s + Ansible pour le reste).
- Renforcement de la sécurité (signatures collections, sandbox).
- IA générative pour suggérer/lint des playbooks (Ansible Lightspeed).
- Compétition avec Pulumi/Crossplane sur le terrain "infra unifiée".
</details>

---

## Conseils généraux pour l'entretien

1. **Connaissez votre projet** : ayez un exemple concret de playbook/rôle/inventaire issu de votre expérience.
2. **Idempotence, idempotence, idempotence** : c'est *le* concept central qu'on vous demandera.
3. **Sécurité** : sachez expliquer Vault, sudo, agentless.
4. **Comparaisons** : Ansible vs Puppet/Chef/Salt/Terraform.
5. **Scénarios de panne** : préparez 2-3 anecdotes "ça a foiré et voilà comment j'ai géré".
6. **Soft skills** : Ansible est aussi un outil de **collaboration ops/dev**. Insistez sur la lisibilité et la documentation.

