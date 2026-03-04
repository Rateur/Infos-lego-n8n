<div align="center">

  <h1 style="margin-bottom: 10px;">Automatisations n8n : Veille RSS & IA sur Discord</h1>
  
  <a href="https://discord.gg/tVcBVzBGc3">
    <img src="https://img.shields.io/badge/Rejoindre_le_Serveur-Discord-5865F2?style=for-the-badge&logo=discord&logoColor=white" alt="Rejoindre le Discord">
  </a>

  <p style="margin-top: 20px; margin-bottom: 20px;">
    <img src="https://img.shields.io/badge/n8n-FF6D5A?style=flat-square&logo=n8n&logoColor=white" alt="n8n">
    <img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white" alt="Docker">
    <img src="https://img.shields.io/badge/Serper-4285F4?style=flat-square&logo=google&logoColor=white" alt="Serper">
    <img src="https://img.shields.io/badge/OpenRouter-000000?style=flat-square&logo=openai&logoColor=white" alt="OpenRouter">
  </p>

  <p style="font-size: 14px;">
    <a href="#présentation">Présentation</a> •
    <a href="#structure-du-dépôt">Structure</a> •
    <a href="#installation">Installation</a> •
    <a href="#aperçu-des-résultats">Aperçus</a>
  </p>

</div>

---

## Présentation

Ce dépôt contient les fichiers de configuration nécessaires au déploiement d'un pipeline d'information automatisé (veille technologique). Le système agrège des données depuis des flux RSS, les filtre temporellement et peut les enrichir via un modèle d'Intelligence Artificielle (LLM) avant de les publier sur un serveur Discord.

L'objectif est de démontrer l'utilisation de **n8n** pour l'orchestration, **OpenRouter** pour le traitement du langage naturel et **Discord** pour la restitution.

## Structure du dépôt

```text
.
├── docker-compose.yml       # Configuration pour le déploiement local de n8n
├── LEGO avec IA.json        # Workflow avec LLM
├── LEGO sans IA.json        # Workflow simple
└── README.md                # Documentation du projet
```

<details><summary>Détails des Workflows</summary>

### 1. Workflow Standard (LEGO sans IA)

Ce workflow assure une veille simple et rapide.

* **Agrégation :** Lecture parallèle de 5 flux RSS de blogs technologiques.
* **Filtrage :** Fusion des données et conservation uniquement des articles publiés il y a moins de 24 heures.
* **Publication :** Formatage d'un "Embed Discord" contenant le titre, la date, la source et le résumé.
* **Sécurité :** Intégration de délais (Wait) pour respecter les limites de débit (Rate Limiting) de l'API Discord.

### 2. Workflow Augmenté (LEGO avec IA)

Ce workflow intègre une couche d'analyse intelligente via les nœuds LangChain de n8n.

* **Recherche et Analyse :** Utilisation de l'API **Serper.dev** pour la recherche contextuelle et **OpenRouter** pour le traitement par LLM.
* **Synthèse :** Le modèle analyse les contenus récupérés et génère une synthèse structurée.
* **Architecture :** Le choix de Serper et OpenRouter permet de bénéficier de modèles performants avec des coûts maîtrisés (ou gratuits selon les paliers).
* **Formatage :** Parsing du résultat via un script JavaScript pour un affichage propre sur Discord.

</details>



## Installation

Le projet est conçu pour être auto-hébergé via Docker.

### Prérequis

* Docker et Docker Compose installés sur la machine.
* Clés API pour : OpenRouter, Serper.dev.
* URL du Webhook Discord.

### Démarrage

Lancez la commande suivante à la racine du dossier pour démarrer le conteneur n8n :

```bash
docker compose up -d
```

L'interface de n8n sera accessible via : [http://localhost:5678](https://www.google.com/search?q=http://localhost:5678).
Les données sont persistantes grâce aux volumes configurés dans le fichier `docker-compose.yml`.

## Aperçu des résultats

### Phase 1 : Veille Standard
*Agrégation RSS simple et filtrage.*

**Vue du Workflow n8n :**
![Workflow n8n Standard](https://i.imgur.com/w05HuE9.png)

**Rendu sur Discord :**
![Rendu Discord Standard](https://i.imgur.com/eJKvL3g.png)



**[Code source du workflow n8n sans IA](https://github.com/Rateur/Infos-lego-n8n/blob/main/LEGO%20sans%20IA.json)**

---

### Phase 2 : Veille Augmentée (IA)
*Analyse contextuelle et résumé par LLM.*

**Vue du Workflow n8n :**
![Workflow n8n IA](https://i.imgur.com/ucpF9wN.png)

**Rendu sur Discord :**
![Rendu Discord IA](https://i.imgur.com/6ZMCGRU.png)

[Code source du workflow n8n avec IA](https://github.com/Rateur/Infos-lego-n8n/blob/main/LEGO%20avec%20IA.json)
