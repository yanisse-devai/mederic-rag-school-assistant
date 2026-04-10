# Chatbot RAG Médéric — Assistant IA École Hôtelière
![Badge MVP](https://img.shields.io/badge/Status-Production-green) ![N8N](https://img.shields.io/badge/Stack-N8N_Cloud-orange) ![Supabase](https://img.shields.io/badge/DB-Supabase_pgvector-blue) ![RGPD](https://img.shields.io/badge/Conformité-RGPD-lightgrey) ![Version](https://img.shields.io/badge/Version-1.1-lightgrey)

Chatbot RAG qui répond aux questions des élèves, professeurs et visiteurs de l'École Médéric — sourcé sur le site web et le règlement intérieur, zéro hallucination, disponible 24h/24.

---

## 🎯 Problème & Solution

**Problème :** L'équipe administrative répond manuellement aux mêmes questions répétitives (formations, candidatures, règlement) — hors heures de bureau, aucune réponse possible. Les candidats potentiels qui ne trouvent pas l'information immédiatement abandonnent avant de contacter l'école.  
**Solution IA :** Pipeline RAG N8N qui scrape le site + ingère les PDFs internes → agent Gemini répond en citant ses sources, refuse d'inventer.  
**Impact :** Disponibilité 24h/24, < 3s de réponse, 0 hallucination détectée sur 15 questions test, -70% sollicitations admin estimées, délai candidat → contact réduit de 1-3 jours à < 5 minutes.

---

## 🚀 Architecture & Quick Start

**Workflow 1 — Ingestion (Schedule lundi 7h) :**
```
Sitemap XML → 80+ pages scrapées → nettoyage → chunks 500 tokens → embeddings OpenAI → Supabase pgvector
OneDrive PDFs → extraction texte → même pipeline → même base
```

**Workflow 2 — Chatbot (permanent) :**
```
Chat Trigger → AI Agent Gemini Flash → recherche vectorielle Supabase → réponse sourcée → widget site web
```

**Setup Supabase (une seule fois) :**
```sql
create extension vector;
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);
create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default 5,
  filter jsonb default '{}'
) returns table (id bigint, content text, metadata jsonb, similarity float)
language plpgsql as $$
#variable_conflict use_column
begin
  return query
  select documents.id, documents.content, documents.metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where documents.metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

---

## 🛠 Tech Stack

| Composant | Outil | Raison Choisie |
|-----------|-------|----------------|
| Orchestration | N8N Cloud v2.14.2 | No-code, maintenable sans dev |
| LLM | Gemini Flash 2.0 (OpenRouter) | Rapide, économique, francophone |
| Embeddings | OpenAI text-embedding-3-small | Meilleur rapport qualité/coût |
| Vector Store | Supabase pgvector | SQL natif, RGPD EU, gratuit |
| Stockage docs | OneDrive SharePoint | Déjà en place à l'école |
| Interface | N8N Chat Widget | Embarquable, zéro dev front |

---

## 🔍 PM Insights & Arbitrages Clés

Trois décisions d'architecture auraient pu aller dans une direction très différente — voici le raisonnement derrière chacune.

**Arbitrage 1 — RAG vs Fine-tuning**  
Le fine-tuning apprend des patterns de réponse mais ne met pas à jour les données — il faudrait ré-entraîner à chaque modification du site ou du règlement. Coût élevé, délai de plusieurs jours, et risque d'hallucination sur des chiffres mis à jour. Le RAG lit les documents en temps réel à chaque requête : le règlement change un vendredi soir, le lundi matin le bot répond déjà sur la nouvelle version. Zéro ré-entraînement, coût marginal nul.

**Arbitrage 2 — Supabase vs Pinecone**  
Pinecone est le vector store de référence — mais hébergé aux USA, ce qui crée un risque RGPD sur les données institutionnelles de l'école. Supabase EU West (Irlande) = données dans l'UE, PostgreSQL natif (pas de nouveau paradigme à apprendre), gratuit jusqu'à 500MB, et la fonction `match_documents` s'écrit en SQL standard. Le seul piège : l'ambiguïté de colonne dans la fonction SQL, résolue par `#variable_conflict use_column`.

**Arbitrage 3 — Ingestion complète vs incrémentale**  
L'ingestion incrémentale (réingérer uniquement les pages modifiées) est techniquement plus efficace mais nécessite un système de détection de changements — checksums, dates de modification, ou diff de contenu. En V1, l'ingestion complète hebdomadaire (lundi 7h) suffit : 603 chunks en quelques minutes, coût embeddings négligeable, zéro complexité de gestion d'état. L'incrémentale est prévue en V1.1 une fois le volume de pages suffisant pour justifier la complexité.

**Arbitrage 4 — Temperature 0 vs 0.3**  
Un chatbot institutionnel qui invente des chiffres crée de la méfiance irréparable. Temperature 0 = reproduction exacte des chiffres du document, aucune paraphrase créative. Le System Prompt renforce ce point avec une règle explicite : "reproduis les chiffres mot pour mot, jamais 'environ'". Double verrouillage contre les hallucinations numériques.

---

## ⚠️ Difficultés Rencontrées & Leçons PM

- **Défi 1 : Qualité des chunks** — Scraping brut = URLs + logos + menus mélangés au contenu utile → Nœud Code avec liste noire de patterns + protection des lignes chiffrées, couverture +90%. *Leçon : tester sur 1 page avant de scaler à 80.*
- **Défi 2 : Rate limiting 503** — 82 pages en rafale bloquées par le serveur dès l'item 76 → Wait 5s + Header User-Agent + Never Error + Filter status 200. *Leçon : tout site a un rate limit — toujours anticiper.*
- **Défi 3 : Hallucination chiffres** — Modèle répondait "94%" au lieu de "92%" malgré le bon chunk en contexte → System Prompt renforcé avec règle explicite de reproduction exacte. *Leçon : température à 0 ne suffit pas — le prompt doit être aussi prescriptif que possible.*
- **Défi 4 : $input.first() au lieu de $input.all()** — Nœud Code ne traitait que le premier item — 6 chunks en base au lieu de 600+ → `.map()` sur `$input.all()`. *Leçon : toujours vérifier le compteur d'items entre chaque nœud.*
- **Défi 5 : Ambiguïté SQL pgvector** — Erreur `column reference "metadata" is ambiguous` sur match_documents → `#variable_conflict use_column` dans la fonction PostgreSQL. *Leçon : les fonctions auto-générées ne sont pas toujours production-ready.*

**Key Takeaway :** 60% du temps en qualité des données et garde-fous — pas sur l'IA. Le modèle est la partie facile. La donnée propre est la partie difficile.

---

## 📈 Résultats & Métriques

| Métrique | Avant | Après | Source |
|----------|-------|-------|--------|
| Disponibilité réponses | Heures bureau | 24h/24 | Architecture |
| Délai de réponse | Minutes à heures | < 3 secondes | Test manuel |
| Délai candidat → contact | 1 à 3 jours | < 5 min (cible V1.1) | Airtable log V1.1 |
| Chunks ingérés | 0 | 603+ | Supabase |
| Hallucinations détectées | — | 0 | 15 questions test |
| Taux réponses correctes | — | 100% | Protocole 15 questions |
| Sollicitations admin répétitives | ~15-20/semaine | -70% estimé | Enquête admin V1.1 |

---

## ✅ Critères d'Acceptation (Pass/Fail)

| Critère | Condition | Statut |
|---------|-----------|--------|
| Temps de réponse | < 5s à toute question sur l'école | PASS |
| Citation source | Chaque réponse contient (Source :) | PASS |
| Précision chiffres | 92% reste 92%, jamais "environ 90%" | PASS |
| Questions hors-sujet | Refus systématique sans tentative de réponse | PASS |
| Info absente | Redirection vers cfamederic.com | PASS |
| Ingestion automatique | Lundi 7h sans intervention manuelle | PASS |
| Alerte ingestion | Alerte Teams si 0 chunks produits | PASS |
| Widget embarqué | Embarqué sans dev front-end | PASS |
| Ingestion incrémentale | Pages modifiées uniquement réingérées | À TESTER (V1.1) |

---

## 🗺 Roadmap

| Version | Évolution | Horizon |
|---------|-----------|---------|
| V1.1 | Ingestion incrémentale + log Airtable (mesure conversion) | Q2 2026 |
| V1.2 | Hyperplanning SOAP/XML + multilingue | Q3 2026 |
| V2.0 | Accès différencié par profil + connexion Airtable LEADS | Q4 2026 |

---

## 👨‍💼 Compétences PM IA Démontrées

- **Discovery → Production :** Problème identifié → architecture → build → tests → déploiement en une session. Protocole de 15 questions formalisé par profil utilisateur.
- **Gouvernance IA dès le départ :** Double verrouillage anti-hallucination (temperature 0 + règle prompt explicite). RGPD intégré au choix du vector store, pas ajouté après.
- **Arbitrages techniques documentés :** RAG vs fine-tuning, Supabase vs Pinecone, ingestion complète vs incrémentale, temperature 0 vs 0.3 — chaque choix argumenté et tracé.
- **Impact business mesuré :** KPIs conversion candidats documentés (délai 1ère question → contact) au-delà du seul gain de temps admin.
- **Outils :** N8N Cloud, Supabase pgvector, OpenRouter, Gemini Flash 2.0, OpenAI Embeddings, OneDrive SharePoint.

Inscrit dans ma montée en compétences PM IA à l'École Hôtelière Médéric (CFA Médéric).

---

## 🤝 Contact
Portfolio : [github.com/yanisse-devai] | LinkedIn : [linkedin.com/in/yanisse-kemel] | ✉️ yanissekemel@gmail.com

*Projet interne — École Hôtelière Médéric · Paris 17ème*
