# Plan de sprints communs — RAG v2
> Nicolas × Vianney — À partir de mars 2026
> Méthode : METHODE_TRAVAIL_CLAUDE.md (backup → DEV → test → PROD)

---

## Règles non-négociables (issues de la méthode Nicolas)

1. **Backup avant chaque sprint** — format : `[workflow] — Sprint[Xx] backup.json`
2. **Développer sur DEV, jamais sur PROD**
3. **GET live avant tout PUT** — jamais patcher depuis un fichier local
4. **Test nominal + test de casse** avant toute promotion PROD
5. **Mettre à jour les MD** en fin de sprint (sans qu'on le demande)

---

## Phase 0 — Dette technique (à solder avant tout sprint commun)

> Ces points bloquent la suite. Aucun sprint commun ne peut démarrer proprement sans eux.

| Sprint | Titre | Responsable | Effort | Priorité |
|---|---|---|---|---|
| V-0a | Brancher `REPLACE_WITH_CREDENTIAL_ID` dans rechercher_bible + test RAG end-to-end | Vianney | 2h | 🔴 Bloquant |
| V-0b | Error handlers `onError: continueErrorOutput` sur tous les nœuds HTTP Vianney | Vianney | 1 jour | 🔴 Bloquant |
| V-0c | Créer environnement DEV Vianney (dupliquer workflow + séparer PROD/DEV) | Vianney | 2h | 🔴 Bloquant |
| V-0d | Créer documentation minimale Vianney (CLAUDE.md + STACK_TECHNIQUE.md) | Vianney | 2h | 🔴 Bloquant |
| N-0a | Sprint 3a Nicolas : clés API → variables d'environnement n8n | Nicolas | 2h | 🔴 Bloquant |
| N-0b | Valider sprint Text-Only en production (test dossier sans médias) | Nicolas | 1h | 🟡 P2 |

---

## Phase 1 — Fondations RAG v2 (sprints communs)

### RAG-1 · BigQuery — branchement pipeline embedding
**Durée estimée :** 1 jour · **Priorité :** P1

**Contexte :** BigQuery VECTOR_SEARCH est déjà configuré (Vianney). Le pipeline d'alimentation `n8n → Gemini embedding → BigQuery INSERT` manque côté Nicolas.

**Livrable :**
- Table `chunks` créée dans BigQuery (schéma commun)
- Workflow n8n : à chaque nouvelle fiche → Gemini embedding-001 → INSERT chunks
- Test : requête VECTOR_SEARCH retourne des résultats

---

### RAG-2 · Chunking sémantique
**Durée estimée :** 1 jour · **Priorité :** P1

**Contexte :** 1 fiche = 1 vecteur actuellement. La découpe par section multiplie la précision.

**Livrable :**
- Code node n8n : 1 fiche → N chunks selon sections (titre / procédure / symptômes / causes / matériel / notes)
- Métadonnées au chunk : `machine`, `type_intervention`, `section`, `score_confiance`, `embedding_model_version`
- Re-embedding des 48 fiches Nicolas existantes

---

### RAG-3 · Remplacement RAG mots-clés Nicolas
**Durée estimée :** 1 jour · **Priorité :** P1

**Contexte :** Le bot Nicolas fait GET ALL fiches → filtre mots-clés → Claude. Saturation à ~70 fiches.

**Livrable :**
- Nœud retrieval remplacé : question → embedding → BigQuery VECTOR_SEARCH → top 5 chunks → Claude
- Test sur golden dataset (RAG-4)
- Vérification : la qualité est ≥ à l'ancienne méthode sur les 20 questions

---

### RAG-4 · Golden dataset — évaluation initiale
**Durée estimée :** 2h (travail humain) · **Priorité :** P1

**Contexte :** Sans mesure de base, on ne sait pas si on améliore ou on dégrade.

**Livrable :**
- Fichier `golden_dataset.json` : 20–30 questions réelles (10 Nicolas + 10 Vianney + 10 communes)
- Format : `{ question, reponse_reference, fiche_source, section_attendue }`
- Score de base mesuré : taux de retrieval correct + qualité réponse LLM-as-judge

---

### RAG-5 · Hybrid search BM25 + vectoriel + RRF
**Durée estimée :** 1 jour · **Priorité :** P2

**Livrable :**
- Requête BigQuery combinant `VECTOR_SEARCH` (cosine) + fulltext search
- Fusion RRF : `score_final = 0.6 × score_vecteur + 0.4 × score_BM25`
- Benchmark sur golden dataset : hybrid vs vectoriel seul

---

### RAG-6 · Seuil de confiance + réponse "je ne sais pas"
**Durée estimée :** 2h · **Priorité :** P2

**Livrable :**
- Code node : si `max_similarity_score < 0.72` → message de redirection + suggestion de créer un dossier
- Test avec questions hors-sujet

---

### RAG-7 · Intégration architecture agents Vianney dans pipeline Nicolas
**Durée estimée :** 2 jours · **Priorité :** P2

**Contexte :** Nicolas utilise des appels HTTP directs. Migrer vers le pattern `agentTool` de Vianney.

**Livrable :**
- Agent Superviseur dans workflow Nicolas
- Agent Analyseur avec les 4 outils (vidéo, photo, PDF, audio) en agentTool
- Score de confiance 0–100 écrit dans Airtable
- Benchmark qualité fiches : avant/après sur 5 dossiers tests

---

## Phase 2 — Qualité et feedback

| Sprint | Titre | Livrable | Durée |
|---|---|---|---|
| Q-1 | Feedback 👍/👎 Telegram | Bouton post-réponse + table Airtable Feedbacks + dashboard lacunes | 1 jour |
| Q-2 | Monitoring pipeline | Alerte Telegram si pipeline échoue 2× consécutifs (sprint Bot-Mon Nicolas) | 1 jour |
| Q-3 | Reranking Cohere Rerank v3 | Appel Cohere API après top-20 BigQuery → top-5 reranké | 1 jour |
| Q-4 | Score confiance commun | Champ score 0–100 dans fiches Nicolas ET Vianney, visible Airtable | 2h |
| GE-1 | Migration Gemini Embedding 2 | Table `chunks_v2` BigQuery + re-embedding cross-modal (dès GA) | TBD |

---

## Suivi

| Sprint | Statut | Date | Notes |
|---|---|---|---|
| V-0a | ⏳ À faire | — | |
| V-0b | ⏳ À faire | — | |
| V-0c | ⏳ À faire | — | |
| V-0d | ⏳ À faire | — | |
| N-0a | ⏳ À faire | — | Sprint 3a déjà planifié |
| N-0b | ⏳ À faire | — | Sprint Text-Only déjà déployé sur DEV |
| RAG-1 | ⏳ À faire | — | |
| RAG-2 | ⏳ À faire | — | |
| RAG-3 | ⏳ À faire | — | |
| RAG-4 | ⏳ À faire | — | 2h humain — à faire ensemble |
| RAG-5 | ⏳ À faire | — | |
| RAG-6 | ⏳ À faire | — | |
| RAG-7 | ⏳ À faire | — | |
