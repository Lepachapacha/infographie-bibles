# Analyse comparative — Bibles Techniques
> Document de travail — 13 mars 2026
> Préparation sprint commun Nicolas × Vianney

---

## Ce qui a été réussi (commun aux deux)

Les deux projets ont résolu le problème difficile : capturer de la connaissance technique multimodale (vidéo, photo, audio, PDF) et la rendre interrogeable en langage naturel. C'est le cœur du problème, et il est résolu dans les deux cas.

---

## Bible Technique Nicolas — Torréfaction BIBAL Av12

**Statut :** En production — v1.0 stable depuis le 10/03/2026
**Métriques :** 48 fiches générées · 196 médias traités · 15+ sprints · 25+ bugs documentés

### Architecture

Pipeline déterministe avec 2 modèles IA :
- **Gemini 2.5 Flash** — perception multimodale (vidéo, photo, audio, texte)
- **Claude Sonnet 4** — raisonnement et structuration (2 passes anti-troncature)

Workflows n8n avec appels HTTP directs (pas de nœuds LangChain Agent) — paradigme "pipeline IA" plutôt qu'"agents autonomes".

### Forces

- Pipeline ingestion multimodale éprouvé (Gemini Files API + polling + retry HEVC)
- Architecture 2 passes Claude avec `staticData` — anti-troncature sur dossiers multi-médias
- `onError: continueErrorOutput` sur tous les nœuds HTTP critiques
- PROD/DEV séparés + scripts de promotion automatisés + procédures de rollback
- Documentation exhaustive : CLAUDE.md, STACK_TECHNIQUE.md, DECISIONS.md, ERREURS_ET_LECONS.md, RUNBOOK.md, SUIVI_DEV.md, GLOSSAIRE.md
- Analytics de lacunes automatiques (`a_documenter`, `machine_detectee`)
- Mémoire conversationnelle Telegram (sessions stockées Airtable)
- Méthode de développement formalisée et réutilisable (METHODE_TRAVAIL_CLAUDE.md)

### Faiblesses

- **RAG par mots-clés** : GET ALL fiches → filtre → Claude. Limite réelle ~70 fiches avant dégradation
- **Zéro chunking** : 1 fiche = 1 bloc entier envoyé au LLM
- Aucune mesure de qualité des réponses
- Clés API en dur dans les nœuds (sprint 3a planifié mais non fait)
- Paradigme pipeline — pas d'agents IA natifs (évolution planifiée)

---

## Bible Technique Vianney — Distributeurs Bibal

**Statut :** Architecture posée, non déployée en production
**Périmètre :** Distributeurs café, snack, fontaine · 6 workflows n8n · 3 agents IA

### Architecture

Multi-agents hiérarchiques avec nœuds LangChain n8n :
- **Agent Superviseur** — orchestration, score de confiance 0–100, JSON strict
- **Agent Analyseur** (agentTool) — 3 passes (cartographie → transcription exhaustive → synthèse), 4 outils
- **Agent Organisationnel** (agentTool) — gestion Drive, arborescence 3 niveaux, TOUT-EN-MAJUSCULES

Pattern `agentTool` / `toolWorkflow` / `$fromAI()` — les agents décident eux-mêmes des outils à appeler.

### Forces

- Architecture multi-agents propre et bien structurée
- Score de confiance automatique 0–100 par fiche
- Recherche vectorielle BigQuery VECTOR_SEARCH avec Gemini text-embedding-004 — **infrastructure déjà configurée**
- Organisation Drive auto-gérée par l'agent Organisationnel
- Prompts agents très détaillés avec séquence obligatoire définie
- Ingestion binaire directe (PDF, image, vidéo, audio)

### Risques critiques

- **Aucun error handler** — toute erreur arrête le pipeline silencieusement
- **Pipeline d'embedding non connecté** — BigQuery est prêt, le branchement `n8n → Gemini embedding → BigQuery` manque
- **Credential placeholder** `REPLACE_WITH_CREDENTIAL_ID` dans le tool `rechercher_bible`
- Aucun environnement DEV — workflow unique, chaque modification risque la production
- Zéro documentation — dépendance totale à la mémoire humaine

---

## Comparatif technique

| Dimension | Nicolas | Vianney |
|---|---|---|
| Statut | Production v1.0 stable | Architecture posée |
| Paradigme IA | Pipeline 2 modèles (Gemini + Claude) | Multi-agents hiérarchiques (LangChain n8n) |
| Ingestion médias | Gemini Files API + polling + retry HEVC | Gemini binary direct — PDF, image, vidéo, audio |
| Retrieval | Mots-clés · limite ~70 fiches | Vectoriel BigQuery · pipeline non connecté |
| Chunking | Absent — 1 fiche = 1 bloc | Absent — 1 fiche = 1 vecteur |
| Gestion d'erreur | Handlers sur tous les nœuds critiques | Absente |
| Environnement DEV | PROD / DEV séparés + scripts promotion | Workflow unique |
| Documentation | 7 fichiers MD + méthode de travail | Absente |
| Mémoire bot | Session conversationnelle Airtable | Non implémentée |
| Support vocal | Telegram + Gemini Speech | Prévu, non intégré |
| Analytics lacunes | Automatiques (a_documenter) | Absentes |
| Score confiance | Absent | 0–100 par fiche (Superviseur) |
| Organisation fichiers | Drive — nommage slug + record_id | Drive auto-géré par agent Organisationnel |
| Évaluation qualité | Inexistante | Inexistante |
| Feedback opérateur | Planifié (sprint Bot-Feedback) | Non prévu |

---

## Ce qui manque aux deux projets

Ces lacunes sont communes — elles définissent l'agenda du RAG v2.

### 1. Chunking sémantique (levier n°1)
Actuellement : 1 fiche entière = 1 vecteur ou 1 bloc. C'est la première cause de mauvaise réponse.
Cible : 1 fiche → N chunks par section (titre / procédure / symptômes / causes / matériel).
**Impact estimé : +30–50% de précision au retrieval.**

### 2. Évaluation de la qualité
Aucun système ne mesure si ses réponses sont bonnes. Sans mesure, chaque sprint est une amélioration dans le vide.
Minimum viable : **golden dataset de 20–30 questions réelles** avec réponses de référence, testé à chaque sprint.

### 3. Hybrid Search
- Vectoriel seul rate les termes exacts : "erreur E47", "Av12", "roulement 6205"
- BM25 seul rate la sémantique : "machine qui claque" ≠ "bruit anormal moteur"
- Solution : BM25 + vectoriel + Reciprocal Rank Fusion (RRF)

### 4. Feedback loop
Les 👎 Telegram sont la donnée la plus précieuse — actuellement perdus.
Signal explicite : bouton 👍/👎 après chaque réponse → identifier fiches mal indexées, chunks mal découpés.

### 5. Détection d'incertitude
Si les chunks récupérés ont un score faible, le LLM hallucine au lieu de dire "je ne sais pas".
Solution : seuil de confiance sur le retrieval (`score < 0.72 → "Je n'ai pas de fiche sur ce sujet."`)

### 6. Prompt versioning
Prompts dans les nœuds n8n, non versionnés. Toute régression est invisible.
Solution : un fichier MD par prompt, versionné git, avec numéro de version dans le nœud n8n.

---

## Opportunité externe — Gemini Embedding 2 (sorti 10/03/2026)

Premier modèle d'embedding nativement multimodal — texte, image, vidéo, audio, PDF dans **un seul espace vectoriel**.

**Spécifications :**
- Model ID : `gemini-embedding-2-preview`
- Dimensions : 128–3072 (Matryoshka MRL)
- Input texte : 8 192 tokens
- Multilingue : 100+ langues

**Limites actuelles (Preview) :**
- Vidéo : 128 secondes max
- Audio : 80 secondes max
- PDF : 6 pages max
- Incompatible avec text-embedding-004 → re-embedding requis

**Impact :** Au lieu de transcrire la vidéo puis l'embedder, on embedde directement le média — information visuelle préservée, cross-modal natif.

**Décision :** Ne pas utiliser aujourd'hui. Anticiper la migration dans le schéma de données (`embedding_model_version`).
