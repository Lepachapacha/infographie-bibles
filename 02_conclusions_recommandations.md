# Conclusions & Recommandations — RAG v2
> Document de travail — 13 mars 2026

---

## Recommandation principale : Scénario B — Convergence

**Ne pas repartir de zéro. Ne pas continuer en parallèle.**

Fusionner les deux projets en une infrastructure unifiée :
- **Base opérationnelle** : pipeline Nicolas (DevOps, error handling, documentation, méthode)
- **Architecture** : multi-agents Vianney (Superviseur → Analyseur + Organisationnel)
- **Retrieval** : BigQuery VECTOR_SEARCH — déjà configuré, zero nouvelle infrastructure
- **Ajouts** : chunking sémantique + hybrid search + golden dataset + feedback loop

---

## Trois scénarios évalués

### Scénario A — Évolution incrémentale de Nicolas
**Durée : 4–6 semaines · Risque faible · Récompense modérée**

Greffer BigQuery VECTOR_SEARCH sur la base Nicolas sans changer l'architecture.
- ✓ Résout la saturation RAG (~70 fiches)
- ✗ Pas d'agents IA, pas d'évaluation, pas de feedback loop
- À choisir si : urgence uniquement sur la scalabilité, projets restent séparés

### Scénario B — Convergence ⭐ Recommandé
**Durée : 8–12 semaines · Risque modéré · Récompense élevée**

Fusionner les deux systèmes + ajouter les fondations manquantes aux deux.
- ✓ Capitalise sur 15+ sprints Nicolas sans les jeter
- ✓ Intègre l'architecture avancée de Vianney
- ✓ Pose les vraies fondations RAG (chunking, évaluation, hybrid search)
- ✓ Prépare la migration Gemini Embedding 2

### Scénario C — Greenfield microservice RAG
**Durée : 16–20 semaines · Risque élevé · Récompense maximale**

n8n pour les déclencheurs uniquement, tout le RAG dans un service Python (FastAPI + LlamaIndex).
- ✓ Testable, observable, évaluable avec RAGAS
- ✗ Compétences Python requises
- ✗ 4+ mois sans amélioration prod
- À considérer dans 12–18 mois si volume > 500 fiches

---

## Stack technique validée

| Couche | Choix | Justification |
|---|---|---|
| Orchestration | n8n self-hosted | Déjà en place, maîtrisé |
| Agents | n8n LangChain agentTool | Pattern Vianney éprouvé |
| Perception multimodale | Gemini 2.5 Flash | Déjà intégré |
| Raisonnement / Q&R | Claude Sonnet 4 | Structuration complexe + réponses nuancées |
| Embedding Phase 1 | Gemini text-embedding-001 | Stable, production-ready |
| Embedding Phase 2 | Gemini Embedding 2 | Dès GA — cross-modal vidéo+photo+texte |
| Vector DB | BigQuery VECTOR_SEARCH | **Déjà configuré** — zéro nouvelle infrastructure |
| Retrieval | BigQuery fulltext + VECTOR_SEARCH + RRF | Hybrid search natif BigQuery |
| Reranking | Cohere Rerank v3 | API simple, ~$1/1M tokens |
| Données structurées | Airtable | Interface technicien existante |
| Fichiers | Google Drive auto-géré | Pattern Vianney |
| Bot | Telegram (texte + vocal) | Avec mémoire session |
| Sécurité | Variables d'env n8n | Sprint 3a Nicolas — appliquer aux deux |

---

## Architecture cible

### Pipeline d'ingestion
```
Formulaire Airtable
  → Webhook n8n
    → Agent Superviseur (n8n LangChain)
        ├── Agent Analyseur (agentTool)
        │     → Gemini 2.5 Flash (multimodal)
        │       → Transcription brute
        │         → Chunking sémantique (Code node)
        │           → Gemini embedding-001
        │             → BigQuery VECTOR_SEARCH (table chunks)
        │
        ├── Agent Organisationnel (agentTool)
        │     → Google Drive (arborescence auto)
        │
        └── Claude Sonnet 4
              → Fiche structurée Airtable
              → Score confiance 0–100
```

### Pipeline de retrieval (bot Telegram)
```
Question opérateur (texte ou vocal)
  → Gemini Speech (si vocal)
    → Gemini embedding-001 (vecteur question)
      → BigQuery : VECTOR_SEARCH cosine + fulltext BM25
        → Fusion RRF (top 20)
          → Cohere Rerank v3 (top 5)
            → Seuil confiance (score < 0.72 → "Je ne sais pas")
              → Claude Sonnet 4 (génération réponse)
                → Réponse Telegram + bouton 👍/👎
```

### Schéma chunk BigQuery
```sql
CREATE TABLE chunks (
  chunk_id STRING,
  fiche_id STRING,
  projet STRING,            -- "nicolas" | "vianney"
  section STRING,           -- "titre" | "procedure" | "symptomes" | "causes" | "materiel" | "notes"
  machine STRING,
  type_intervention STRING, -- "Panne" | "Procédure" | "Info"
  contenu STRING,
  vecteur ARRAY<FLOAT64>,   -- 768 ou 3072 dims (Matryoshka)
  embedding_model_version STRING,  -- anticipe migration Embedding 2
  score_confiance INT64,    -- 0-100, hérité de l'agent Superviseur
  created_at TIMESTAMP
)
```

---

## Migration vers Gemini Embedding 2 (Phase 2)

Stratégie sans big bang — deux tables BigQuery coexistantes :
- `chunks` → text-embedding-001 (Phase 1, opérationnel)
- `chunks_v2` → gemini-embedding-2 (Phase 2, dès GA)

Migration progressive par lot — jamais de coupure prod.

---

## Ce qu'on n'utilise pas — et pourquoi

| Rejeté | Raison |
|---|---|
| RAG mots-clés Nicolas | Remplacé par BigQuery VECTOR_SEARCH (déjà configuré) |
| Délimiteurs textuels `---TITRE---` | Remplacés par JSON strict (plus robuste) |
| GET ALL fiches → contexte entier | Remplacé par retrieval ciblé (scalable) |
| LangGraph / CrewAI | Complexité sans bénéfice à cette échelle |
| Fine-tuning LLM | Inutile avant RAG fonctionnel |
| Pinecone / Weaviate managed | BigQuery couvre largement le volume actuel |

---

## Ordre des leviers de qualité (par impact décroissant)

1. **Chunking sémantique par section** — +30–50% précision retrieval
2. **Métadonnées au chunk** — filtrage pré-retrieval réduit le bruit
3. **Hybrid search BM25 + vectoriel** — couvre termes exacts ET sémantique
4. **Évaluation systématique** — golden dataset 20–30 questions
5. **Feedback loop 👍/👎** — signal terrain directement exploitable

---

## Décisions à confirmer lors du sprint commun

1. Valider le Scénario B comme direction commune
2. Confirmer BigQuery comme vector DB unique (Nicolas + Vianney)
3. Définir le schéma de chunking commun aux deux projets
4. Adopter METHODE_TRAVAIL_CLAUDE.md pour les deux
5. Construire le golden dataset ensemble (20–30 questions réelles)
