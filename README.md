 TP2 — Déploiement d'un Portfolio Next.js avec Coolify et Forgejo

Déploiement continu d'un portfolio Next.js dans une VM locale grâce à Coolify (PaaS auto-hébergé) et Forgejo (forge Git auto-hébergée), avec redéploiement automatique à chaque git push via webhook.

Auteur : Narjess Dabbech — Master DevOps & Cloud — 2026

 Sommaire
Objectifs
Outils et rôles
Environnement
Architecture
Partie A — Installation de Coolify
Partie B — Forgejo et dépôt du portfolio
Partie C — Déploiement du portfolio
Partie D — Redéploiement automatique (webhook)
Rollback et supervision
Comparaison PaaS (Coolify) vs IaC (TP1)
Difficultés rencontrées
Structure du dépôt
1.  Objectifs
Installer Coolify et Forgejo dans une VM locale
Déployer une application Next.js depuis un dépôt Forgejo, accessible par un nom d'hôte local
Mettre en place un webhook Forgejo → Coolify : chaque git push redéploie l'application
Pratiquer le rollback et comparer l'approche PaaS à l'approche IaC du TP1
2.  Outils et rôles
Outil	Rôle dans le TP
Forgejo	Héberge le dépôt Git du portfolio ; émet le webhook à chaque push
Coolify	Clone, construit, déploie et expose l'application
Traefik (intégré à Coolify)	Routage HTTP par nom d'hôte
Docker / Docker Compose	Exécution des conteneurs (Forgejo, application)
Nixpacks	Build automatique de l'application Next.js
3.  Environnement
Élément	Valeur
VM	Ubuntu 24.04 — 2 vCPU — 4 Go RAM — 20 Go disque
IP de la VM	192.168.56.10
Nom d'hôte local	portfolio.coolify.local (fichier hosts du poste)
Domaine DNS / compte GitHub	Aucun (tout est local)

Ports utilisés dans la VM

Port	Service
8000	Dashboard Coolify
80	Traefik (proxy inverse)
3001	Forgejo (interface web)
2222	Forgejo (SSH)
3000	Application Next.js (interne)
4.  Architecture
HTTP :3001
Webhook POSTpush sur main
git clone
Build Nixpacks
Poste développeurgit push
Forgejodépôt portfolio
Coolify :8000
Conteneur Next.js :3000
Traefik :80
Navigateurportfolio.coolify.local

Chaîne de déploiement : push → Forgejo → webhook → Coolify (clone + build) → Traefik → navigateur.

5. 🔧 Partie A — Installation de Coolify
bash
ip -4 a                                   # relever l'IP de la VM
ssh <utilisateur>@192.168.56.10
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
docker --version && docker ps             # vérifier Docker et les conteneurs Coolify
Ouvrir http://192.168.56.10:8000
Créer immédiatement le compte administrateur
Suivre l'assistant (serveur localhost)

Vérifications : dashboard accessible, serveur localhost en état Reachable, proxy démarré.

Dashboard Coolify

6.  Partie B — Forgejo et dépôt du portfolio
B.1 Installation de Forgejo
bash
docker compose up -d      # avec forgejo/docker-compose.yml
docker ps                 # le conteneur « forgejo » doit être Up

Points importants du docker-compose.yml :

Port web 3001 (le 3000 est réservé à Next.js)
FORGEJO__webhook__ALLOWED_HOST_LIST=external,private : indispensable, sinon Forgejo refuse d'envoyer un webhook vers une adresse privée

Forgejo — conteneur Docker

B.2 Configuration initiale (http://192.168.56.10:3001)
Champ	Valeur
Base de données	SQLite3
Domaine du serveur	192.168.56.10
URL de base	http://192.168.56.10:3001/
Administrateur	Compte créé à l'installation

Forgejo — installation

B.3 Dépôt du portfolio

Dépôt portfolio créé en public (clone sans authentification par Coolify), puis :

bash
git remote add forgejo http://192.168.56.10:3001/Narjess/portfolio.git
git push -u forgejo main
7.  Partie C — Déploiement du portfolio
C.1 Nom d'hôte local

Ligne ajoutée au fichier hosts du poste (C:\Windows\System32\drivers\etc\hosts sous Windows) :

192.168.56.10  portfolio.coolify.local

Test : ping portfolio.coolify.local répond depuis l'IP de la VM.

C.2 Configuration dans Coolify
Paramètre	Valeur
Projet / environnement	portfolio / production
Source	Public Repository
Repository URL	http://192.168.56.10:3001/Narjess/portfolio.git (IP, pas de nom d'hôte)
Branche	main
Build Pack	Nixpacks
Ports Exposes	3000
Domains	http://portfolio.coolify.local (préfixe http:// obligatoire)

Configuration Coolify

C.3 Déploiement

Déploiement Coolify

Vérifications

bash
curl -I http://portfolio.coolify.local   # HTTP 200
docker ps                                # conteneur du portfolio visible
Résultat

Portfolio déployé

8.  Partie D — Redéploiement automatique (webhook)
D.1 Côté Coolify
Application → onglet Webhooks → Manual Git Webhooks
Section Gitea (compatible Forgejo) : saisir un secret d'au moins 16 caractères, enregistrer, copier l'URL
Advanced → Deployment → Auto deploy : Deploy on push (webhooks)
D.2 Côté Forgejo

Dépôt → Paramètres → Webhooks → Ajouter un webhook → Forgejo

Champ	Valeur
URL cible	URL copiée depuis Coolify
Type de contenu	application/json
Secret	Identique à celui de Coolify
Événement	Push
Filtre de branche	main

Clic sur Tester la livraison : réponse 2xx attendue.

Webhook Forgejo

D.3 Vérification
bash
git add . && git commit -m "Mise à jour du titre" && git push forgejo main

Un nouveau déploiement apparaît automatiquement dans l'onglet Deployments de Coolify, et le site affiche la modification.

Déploiement automatique

9. Rollback et supervision

Rollback : une erreur de build a été poussée volontairement ; le déploiement automatique a échoué, puis la version précédente a été restaurée depuis l'onglet des déploiements. Le dépôt a ensuite été corrigé.

Rollback Coolify

Supervision : logs de l'application, docker stats et utilisation des ressources dans le dashboard Coolify.

10.  Comparaison PaaS (Coolify) vs IaC (TP1)
Critère	TP1 — IaC (Vagrant / Ansible / Compose)	TP2 — PaaS (Coolify)
Mise en place	Plusieurs outils à écrire et maintenir	Installation en une commande
Configuration	Fichiers versionnés (code)	Interface web
Reproductibilité	Très forte (tout est dans Git)	Plus faible (état dans Coolify)
Déploiement continu	Pipeline à écrire (Jenkinsfile)	Webhook + auto deploy intégrés
Proxy / routage	À configurer (Nginx, ports)	Traefik géré automatiquement
Contrôle fin	Total	Limité aux options de la plateforme
Courbe d'apprentissage	Élevée	Faible
Rollback	À scripter	Intégré à l'interface

Conclusion : l'IaC offre contrôle et reproductibilité ; le PaaS offre rapidité et simplicité d'exploitation.

11.  Difficultés rencontrées
Webhook refusé par Forgejo vers une IP privée → ajout de ALLOWED_HOST_LIST=external,private
Coolify ne résout pas le fichier hosts du poste → URL du dépôt avec l'IP de la VM
Domaine sans préfixe http:// → Coolify tentait d'obtenir un certificat HTTPS
(Compléter avec vos propres difficultés)
12.  Structure du dépôt
.
├── forgejo/
│   └── docker-compose.yml
├── captures/
│   ├── coolify-dashboard.png
│   ├── forgejo-docker.png
│   ├── forgejo-install.png
│   ├── coolify-config.png
│   ├── coolify-deploy.png
│   ├── portfolio.png
│   ├── forgejo-webhook.png
│   ├── coolify-autodeploy.png
│   └── coolify-rollback.png
└── README.md
Contenu

Épreuve pratique — Master DevOps & Cloud Sujet : Mise en place d'une chaîne CI/CD Jenkins et déploiement d'une application Web avec Docker, Vagrant et Ansible Durée : 2 heures Travail demandé 1. Créer un dépôt GitHub public nommé garderie-devops contenant une application Web composée d'un

PASTED

Oui ! Pendant que les déploiements tournent, vous pouvez préparer le reste Voici ce que vous pouvez faire sans bloquer les builds en cours. 🎯 Priorité 1 : Préparer les Screenshots (obligatoire pour la note) Le sujet demande 6 captures d'écran (point 33). Vous pouvez déjà en faire 3 : ✅ 1. s

PASTED

TP2 : Déploiement d'un Portfolio Next.js avec Coolify et Forgejo Coolify : alternative open source et auto-hébergeable à Heroku, Netlify et Vercel. Forgejo : forge Git auto-hébergée (alternative à GitHub). Ce TP s’exécute entièrement en local, dans une machine virtuelle : ni domaine DNS, ni compte

PASTED

Mini-projet 1 : Application web à deux niveaux sur Kubernetes (K3s) Déploiement, exposition via Ingress, mise à jour sans coupure et diagnostic de panne Kubernetes K3s Traefik Nginx Status Table des matières Contexte et objectifs Architecture Structure du dépôt Prérequis Guide de déploie

PASTED

 TP2 — Déploiement d'un Portfolio Next.js avec Coolify et Forgejo Déploiement continu d'un portfolio Next.js dans une VM locale grâce à Coolify (PaaS auto-hébergé) et Forgejo (forge Git auto-hébergée), avec redéploiement automatique à chaque git push via webhook. Auteur : Narjess Dabbech — Maste

PASTED

forjego Public Go to file t T author DELL User TP2: deploiement Coolify + Forgejo 93b17b1 · 4 minutes ago Name captures TP2: deploiement Coolify + Forgejo 4 minutes ago README.md TP2: deploiement Coolify + Forgejo 4 minutes ago Repository files navigation README Mini-projet 1

PASTED

TP2 : Déploiement d'un Portfolio Next.js avec Coolify et Forgejo Coolify : alternative open source et auto-hébergeable à Heroku, Netlify et Vercel. Forgejo : forge Git auto-hébergée (alternative à GitHub). Ce TP s’exécute entièrement en local, dans une machine virtuelle : ni domaine DNS, ni compte

PASTED
