# 🤖 Agent Candidature Alternance

> Agent IA autonome de recherche d'alternance — génère des dossiers de candidature personnalisés (lettre de motivation PDF + brouillon Gmail) à partir d'un CV et d'une liste d'entreprises, en trouvant automatiquement les vrais contacts RH/IT.

---

## 👤 Contexte

Projet développé par **Jonathan Desreumaux**, étudiant en 2ème année de BTS CIEL (Cybersécurité, Informatique et Réseaux Électroniques), dans le cadre de sa recherche d'alternance pour un **Bachelor Administrateur Systèmes, Réseaux & Cybersécurité**.

Objectif : automatiser la partie chronophage de la recherche d'alternance (recherche d'entreprises, rédaction de lettres personnalisées, envoi d'emails) tout en gardant le contrôle sur l'envoi final.

---

## 🎯 Ce que fait l'agent

L'utilisateur fournit :
- Son **CV en PDF**
- Une **liste d'entreprises** à cibler

L'agent fait le reste **de manière autonome** :

1. **Lit et comprend le CV** → extrait compétences, formations, expériences
2. **Analyse chaque entreprise** → secteur, taille, stack tech, actualités via recherche web
3. **Trouve le vrai contact RH ou IT** → pas de `contact@` générique, mais `marie.dupont@entreprise.fr`
4. **Génère une lettre de motivation personnalisée** en PDF, adaptée au poste et à l'entreprise
5. **Crée un brouillon Gmail** avec destinataire, objet personnalisé et LM en pièce jointe
6. **Met à jour le fichier Excel de suivi** des candidatures
7. **Envoie un email de confirmation** à Jonathan → il n'a plus qu'à aller dans ses brouillons Gmail et cliquer "Envoyer"

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    SITE WEB (Dashboard)                  │
│  Upload CV · Liste entreprises · Statut en temps réel   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                     ORCHESTRATEUR                        │
│         Lit le CV · Planifie · Délègue · Suit           │
└──────┬──────────┬───────────────┬────────────┬──────────┘
       │          │               │            │
       ▼          ▼               ▼            ▼
  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │  Agent  │ │  Agent   │ │  Agent   │ │  Agent   │
  │   CV    │ │ Contact  │ │  Lettre  │ │  Gmail   │
  └─────────┘ └──────────┘ └──────────┘ └──────────┘
       │          │               │            │
       └──────────┴───────────────┴────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    RÉSULTATS SESSION                     │
│   Brouillons Gmail · Excel mis à jour · Email notif     │
└─────────────────────────────────────────────────────────┘
```

### Les 4 agents

| Agent | Rôle | Modèle IA |
|---|---|---|
| **Agent CV** | Lit le PDF, extrait le profil, suggère des postes ciblés | Claude Haiku |
| **Agent Contact** | Trouve les vrais emails RH/IT via Google, LinkedIn, Hunter.io | Claude Haiku |
| **Agent Lettre** | Génère une LM personnalisée et l'exporte en PDF | Claude Sonnet |
| **Agent Gmail** | Crée le brouillon Gmail prêt à envoyer | Claude Haiku |

---

## 🔍 Stratégie de recherche d'email (Agent Contact)

L'objectif est de trouver une **vraie adresse nominative**, pas un alias générique.

**Étape 1 — Recherche Google ciblée**
```
site:linkedin.com/in "[Entreprise]" "Responsable RH" OR "Chargée de recrutement"
site:linkedin.com/in "[Entreprise]" "DSI" OR "Responsable IT"
```
Google indexe les profils LinkedIn publics → récupère nom + titre sans scraper LinkedIn directement.

**Étape 2 — Hunter.io**
```python
hunter.domain_search(domain="entreprise.fr", department="hr")
hunter.domain_search(domain="entreprise.fr", department="it")
```
Retourne les emails connus + le pattern de l'entreprise (`prenom.nom@...`).

**Étape 3 — Pattern guessing + vérification SMTP**
Si rien trouvé → génère les variantes classiques et vérifie via SMTP sans envoyer de mail.

**Fallback** : si aucun email trouvé → brouillon créé sans destinataire, à compléter manuellement.

---

## 💰 Coût estimé (API Anthropic)

| Usage | Coût estimé |
|---|---|
| 1 session (20 entreprises) | ~0,29€ |
| 1 mois (2-3 sessions) | ~0,60 à 0,90€ |
| Campagne complète (80 candidatures) | ~1,50 à 3,00€ |

### Optimisations appliquées
- **Prompt caching** Anthropic → -30% sur les tokens répétés
- **Haiku partout sauf la lettre** → Sonnet uniquement pour la génération de texte qualitatif
- **CV analysé une seule fois par session** → résultat mis en cache JSON
- **DuckDuckGo pour la recherche web** → gratuit, zéro token consommé

---

## 🗂️ Structure du projet

```
agent-candidature/
│
├── frontend/
│   ├── index.html              # Dashboard principal
│   ├── static/
│   │   ├── style.css
│   │   └── app.js
│
├── backend/
│   ├── main.py                 # FastAPI — point d'entrée
│   ├── orchestrator.py         # Orchestrateur principal
│   │
│   ├── agents/
│   │   ├── base_agent.py       # Classe mère (boucle ReAct)
│   │   ├── cv_agent.py         # Lecture et analyse CV
│   │   ├── contact_agent.py    # Recherche email RH/IT
│   │   ├── letter_agent.py     # Génération LM + PDF
│   │   └── gmail_agent.py      # Création brouillons Gmail
│   │
│   ├── tools/
│   │   ├── web_search.py       # DuckDuckGo (gratuit)
│   │   ├── hunter.py           # Hunter.io API
│   │   ├── smtp_verify.py      # Vérification email sans envoi
│   │   ├── pdf_generator.py    # ReportLab → PDF lettre
│   │   ├── excel_tool.py       # openpyxl → suivi candidatures
│   │   └── gmail_tool.py       # Gmail API OAuth
│   │
│   └── memory/
│       ├── session.json        # Profil CV + état session en cours
│       └── Suivi_Candidatures.xlsx
│
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── .env.example                # Clés API à renseigner
```

---

## ⚙️ Stack technique

| Brique | Technologie | Pourquoi |
|---|---|---|
| Backend | **FastAPI** (Python) | Async natif, parfait pour agents en parallèle |
| IA | **Anthropic API** (Haiku + Sonnet) | Contrôle total, pas de framework opaque |
| Recherche email | **Hunter.io API** | 25 recherches/mois gratuites |
| Recherche web | **DuckDuckGo** | Gratuit, aucune clé API |
| Génération PDF | **ReportLab** | LM professionnelle en Python |
| Suivi | **openpyxl** | Mise à jour Excel automatique |
| Email | **Gmail API** (OAuth) | Brouillons sans envoi automatique |
| Notifications | **smtplib** | Email de fin de session à Jonathan |
| Conteneurisation | **Docker + Docker Compose** | Reproductible, tourne sous WSL |
| Planification | **Windows Task Scheduler** | Lance l'agent la nuit si besoin |

---

## 🔐 Variables d'environnement (`.env`)

```env
# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Hunter.io
HUNTER_API_KEY=...

# Gmail OAuth
GMAIL_CLIENT_ID=...
GMAIL_CLIENT_SECRET=...
GMAIL_REFRESH_TOKEN=...

# Notification
NOTIFICATION_EMAIL=jonathan@gmail.com
```

---

## 🚀 Lancement

```bash
# Cloner le projet
git clone https://github.com/jonathan/agent-candidature.git
cd agent-candidature

# Configurer les variables
cp .env.example .env
# → remplir les clés API

# Lancer avec Docker
docker-compose up --build

# Accéder au dashboard
http://localhost:8000
```

---

## 📋 Utilisation

1. Ouvrir le dashboard sur `http://localhost:8000`
2. Uploader ton CV PDF et le sélectionner pour la session
3. Entrer la liste des entreprises à cibler
4. Cliquer "Lancer la session"
5. Suivre l'avancement en temps réel sur le dashboard
6. Recevoir l'email de confirmation quand c'est terminé
7. Aller dans Gmail → Brouillons → Envoyer

---

## 📌 Statut du projet

| Sprint | Contenu | Statut |
|---|---|---|
| Sprint 1 | Site web + upload CV + dashboard | 🔲 À faire |
| Sprint 2 | Agent CV (lecture + extraction profil) | 🔲 À faire |
| Sprint 3 | Agent Contact (recherche email RH/IT) | 🔲 À faire |
| Sprint 4 | Agent Lettre (génération LM + PDF) | 🔲 À faire |
| Sprint 5 | Agent Gmail + email de notification | 🔲 À faire |
| Sprint 6 | Docker + planification automatique | 🔲 À faire |

---

## 🎓 Contexte académique

Ce projet est développé dans le cadre de la candidature de Jonathan à un **Bachelor Administrateur Systèmes, Réseaux & Cybersécurité** pour septembre 2026. Il illustre des compétences en :
- Architecture logicielle (multi-agents, orchestration)
- Intégration d'APIs (Anthropic, Gmail, Hunter.io)
- Automatisation de processus métier
- Développement Python backend (FastAPI)
- Conteneurisation Docker

---

*Développé avec l'API Anthropic (Claude Haiku & Sonnet) — Python 3.12 — Docker — WSL2*
