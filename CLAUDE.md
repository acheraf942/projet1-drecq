# Contexte du projet — « Socle Web Automatisé » (Terraform + Ansible)

> Ce fichier résume le cadrage fait avant de commencer à coder. Les énoncés complets sont dans
> `ENONCE.md` (énoncé général), `ANSIBLE.md` et `TERRAFORM.md` : ils font foi en cas de doute.

## Qui nous sommes

- **Étudiants** (ESIEA), équipe de **5 personnes**, projet noté avec soutenance orale.
- **Soutenance : jeudi 5 novembre 2026.** Cadrage fait le jeudi 29 octobre.
- **Budget temps : environ 5 h par personne** (25 h pour l'équipe). C'est peu pour le périmètre
  complet (estimé à ~90 h), donc **on vise l'essentiel, simple et bien compris**.
- À l'oral, **n'importe quel membre peut être interrogé sur n'importe quelle partie**, même
  une partie qu'il n'a pas codée. Le code doit donc rester **simple, lisible et expliqué**.

## Ce que le projet demande (résumé)

Automatiser le déploiement d'une petite application web, « Fleet Info », sur des serveurs :

- **Fleet Info** : page web générée par Ansible à partir des *facts* de chaque serveur (hostname,
  IP, OS, noyau, vCPU, RAM, environnement local/cloud, version de l'appli, date du déploiement ;
  zone et type de machine en plus sur GCP, absents proprement en local).
  - `/health` : statut + version déployée (sert de contrôle après déploiement).
  - `/status` : `stub_status` nginx protégé par mot de passe (secret dans Ansible Vault).
- **Environnement local (Docker Desktop + Docker Compose)** : 2 conteneurs « nœuds » Debian 12
  (systemd, sshd, python3, sudo) gérés comme des VM + 1 conteneur « controller » avec Ansible.
  Ports HTTP publiés (8081, 8082). Clés SSH générées au démarrage, jamais committées.
- **Ansible** : 3 rôles (`common`, `nginx`, `fleetinfo`), `site.yml` court, FQCN partout,
  idempotence (2e passage → `changed=0`), handlers (`nginx -t` puis `reloaded`), templates avec
  en-tête `{{ ansible_managed }}`, Vault (variables `vault_*` référencées dans `vars.yml`),
  `no_log: true` sur les secrets, `ansible-lint` propre, README + `meta/main.yml` par rôle.
- **Rollback (rôle `fleetinfo`)** : releases dans `/opt/fleetinfo/releases/<version>/`, lien
  `/opt/fleetinfo/current`. `block` (mémoriser N-1, basculer, tester `/health` avec
  `ansible.builtin.uri`) / `rescue` (rebasculer sur N-1, revérifier, faire échouer le play avec un
  message clair) / `always` (journal `/var/log/fleetinfo/deploy.log`). Échec provocable via
  `-e fleetinfo_simulate_failure=true`.
- **Terraform (GCP)** : modules `modules/network/` et `modules/compute/`, VPC dédié, firewall par
  tag réseau (80 ouvert, 22 restreint), N VM Debian 12 (nombre en variable), labels `env`/`role`/
  `team`, clé SSH via métadonnées, `versions.tf` unique avec versions épinglées, variables typées
  et décrites, `terraform.tfvars.example`, outputs (IP, VPC, sous-réseau, URL), state local
  documenté.
- **Inventaire dynamique** `inventories/cloud/fleet.gcp.yml` (`google.cloud.gcp_compute`) : filtre
  sur labels, `compose` pour `ansible_host`, **mêmes noms de groupes** que l'inventaire local.
- **Un seul code Ansible pour les deux environnements** : seuls l'inventaire et les `group_vars`
  changent (pas de `when: env == 'local'` dans les rôles).
- **Documentation de transfert** : README pour un consultant qui reprend le projet (déploiement de
  zéro, runbooks, variables, décisions techniques, dépannage, limites connues).

## Contrainte importante : le cloud n'est pas disponible

Le prof a indiqué que Terraform ne fonctionne pas encore sur les labs Google Skills.
→ **On prépare d'abord l'environnement local.** La démo se fera en local si le cloud reste
indisponible. Le code Terraform est écrit et doit passer `terraform fmt -check` et
`terraform validate`, même s'il n'est pas testé en vrai.

## Ordre de priorité (si on manque de temps, on coupe par le bas)

1. Local fonctionnel : Docker + `site.yml` + page Fleet Info sur les 2 nœuds.
2. Rollback démontrable + `/status` protégé par Vault.
3. Idempotence (`changed=0`) + `ansible-lint`.
4. README et runbooks (version courte mais claire).
5. Terraform + inventaire dynamique (écrits, validés, non testés si pas de lab).
6. Bonus (`argument_specs.yml`, finitions).

## Répartition de l'équipe

| # | Rôle | Périmètre | Suppléant |
|---|---|---|---|
| P1 | Tech lead / intégrateur | Dépôt, `.gitignore`, `ansible.cfg`, `requirements.yml`, `site.yml`, inventaires, `group_vars`, Vault, `ansible-lint`, README principal | P4 |
| P2 | Environnement local | Dockerfiles nœud + controller, `docker-compose.yml`, script clés SSH, doc local | P5 |
| P3 | Rôles `common` + `nginx` | Socle serveur, conf nginx (template), `/status` + htpasswd, handlers | P1 |
| P4 | Rôle `fleetinfo` | Templates page + `/health`, releases + symlink, block/rescue/always, journal | P2 |
| P5 | Terraform + GCP + runbooks | Modules, variables, outputs, `fleet.gcp.yml`, runbooks de la doc | P3 |

Le suppléant relit et doit savoir expliquer la partie. Règle : on ne fusionne pas un code que
son suppléant ne sait pas expliquer.

## Conventions à respecter

- Nom du groupe d'hôtes commun local/cloud : **`web`**.
- Variable d'environnement dans les `group_vars` : `fleet_env: local` / `fleet_env: cloud`.
- Variables préfixées par le rôle (`fleetinfo_version`, `nginx_listen_port`…) ; la version de
  l'appli se change à **un seul endroit**.
- nginx sert `/opt/fleetinfo/current` (interface entre les rôles `nginx` et `fleetinfo`).
- Chaque fichier commence par un commentaire qui explique son rôle.
- Rôle nginx **maison** (pas de rôle Galaxy) : plus simple à écrire et à défendre à l'oral.
- **Réutiliser au maximum les images Docker et le compose des labs Ansible** (autorisé par
  l'énoncé) pour gagner du temps.
- Rien de sensible dans Git : state, `.terraform/`, `terraform.tfvars`, clés SSH, mot de passe
  Vault (`.vault_pass`), identifiants GCP.

## Ce qu'on attend de toi (Claude Code)

- Faire **simple** : c'est un projet étudiant avec peu de temps. Pas de sur-ingénierie.
- **Commenter le code en français** et expliquer les choix non évidents, pour que chacun puisse
  répondre aux questions du prof.
- Après chaque étape, **expliquer brièvement ce qui a été fait et pourquoi**, et donner la commande
  pour vérifier que ça marche.
- Commencer par la priorité 1 (environnement local), puis suivre l'ordre de priorité ci-dessus.
- Signaler les pièges connus : systemd dans Docker sous Docker Desktop (cgroups, `privileged`),
  tâches non idempotentes (date de déploiement, `command`/`shell` sans `changed_when`).
