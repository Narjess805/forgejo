#  TP2 — Déploiement d'un Portfolio Next.js avec Coolify et Forgejo

Déploiement continu d'un portfolio Next.js dans une VM locale grâce à **Coolify** (PaaS auto-hébergé) et **Forgejo** (forge Git auto-hébergée), avec redéploiement automatique à chaque `git push` via webhook.

**Auteur :** Narjess Dabbech — Master DevOps & Cloud — 2026

---

## 📑 Sommaire
1. [Objectifs](#1-objectifs)
2. [Architecture](#2-architecture)
3. [Installation de Coolify (Partie A)](#3-installation-de-coolify-partie-a)
4. [Forgejo et Dépôt (Partie B)](#4-forgejo-et-dépôt-partie-b)
5. [Déploiement du Portfolio (Partie C)](#5-déploiement-du-portfolio-partie-c)
6. [Webhook et Redéploiement Automatique (Partie D)](#6-webhook-et-redéploiement-automatique-partie-d)
7. [Rollback et Supervision](#7-rollback-et-supervision)
8. [Comparaison PaaS vs IaC](#8-comparaison-paas-vs-iac)
9. [Difficultés Rencontrées](#9-difficultés-rencontrées)

---

## 1.  Objectifs

- Installer Coolify et Forgejo dans une VM locale.
- Déployer une application Next.js depuis un dépôt Forgejo, accessible par un nom d'hôte local.
- Mettre en place un webhook Forgejo → Coolify : chaque `git push` redéploie l'application.
- Pratiquer le rollback et comparer l'approche PaaS (Coolify) à l'approche IaC (TP1).

## 2.  Architecture

| Composant | Rôle |
| :--- | :--- |
| **Forgejo** | Héberge le dépôt Git du portfolio et émet le webhook. |
| **Coolify** | Clone, construit, déploie et expose l'application. |
| **Traefik** | Proxy inverse intégré à Coolify pour le routage HTTP. |

**Chaîne de déploiement :**
`Poste (git push) → Forgejo → Webhook → Coolify (Clone + Build) → Traefik → Navigateur`

## 3.  Installation de Coolify (Partie A)

Installation en une seule commande sur la VM Ubuntu :
```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
Accès au dashboard : http://192.168.56.20:8000

Serveur localhost configuré et en état "Reachable".

 Capture : captures/coolify-dashboard.png

4.  Forgejo et Dépôt (Partie B)
Déploiement de Forgejo via Docker Compose (port web 3001).

Configuration initiale (SQLite3) et création du compte administrateur.

Création d'un dépôt public portfolio et envoi du code source.

 Captures : captures/forgejo-docker.png, captures/forgejo-install.png, captures/forgejo-repo.png

5.  Déploiement du Portfolio (Partie C)
Ajout du nom d'hôte local dans le fichier hosts de Windows :
192.168.56.20 portfolio.coolify.local

Configuration dans Coolify :

Source : Public Repository

URL : http://192.168.56.20:3001/Narjess/portfolio.git

Build Pack : Nixpacks (ou Docker Image)

Port : 3000

Domaine : http://portfolio.coolify.local

 Captures : captures/coolify-config.png, captures/coolify-deploy.png, captures/portfolio.png

6.  Webhook et Redéploiement Automatique (Partie D)
Côté Coolify : Activation de l'auto-deploy via webhook (section Gitea).

Côté Forgejo : Ajout du webhook avec l'URL de Coolify, un secret, et l'événement "Push".

Test : "Tester la livraison" renvoie un code 200 (succès).

 Captures : captures/forgejo-webhook.png, captures/coolify-autodeploy.png

7.  Rollback et Supervision
Rollback : Accès à l'historique des déploiements pour restaurer une version antérieure en cas d'échec.

Supervision : Utilisation des logs en temps réel et de docker stats pour surveiller les ressources.

 Capture : captures/coolify-rollback.png

8.  Comparaison PaaS vs IaC
Critère	TP1 — IaC (Vagrant/Ansible/Jenkins)	TP2 — PaaS (Coolify)
Mise en place	Complexe (plusieurs outils à écrire)	Simple (une commande)
Configuration	Fichiers versionnés (code)	Interface web
Déploiement continu	Pipeline à écrire (Jenkinsfile)	Webhook + auto deploy intégrés
Proxy / Routage	À configurer manuellement (Nginx)	Traefik géré automatiquement
Contrôle	Total	Limité aux options de la plateforme
Rollback	À scripter	Intégré à l'interface (1 clic)
Conclusion : L'IaC offre contrôle et reproductibilité. Le PaaS offre rapidité et simplicité.

9.  Difficultés Rencontrées
Manque de mémoire vive (RAM) : La VM (2 Go) ne supportait pas le build Next.js (plantage silencieux). Solution : construction de l'image sur une machine plus puissante, push sur Docker Hub, et utilisation de l'option "Docker Image" dans Coolify.

Bug du proxy Traefik : Erreur 404 malgré un conteneur sain. Solution : contournement en exposant directement le port 3000 sur la VM pour valider l'application.

Webhook et IP privée : Forgejo refuse les requêtes vers une IP privée par défaut. Solution : configuration de ALLOWED_HOST_LIST.
