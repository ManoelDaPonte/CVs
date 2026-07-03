# Préparation aux entretiens — Manoel Da Ponte

> Document de travail (markdown, convertible en PDF plus tard). Trois parties :
> **1.** Inventaire de ce qui a été fait (la matière brute), **2.** Glossaire technique
> avec définitions et exemples tirés de mes propres projets, **3.** Questions
> d'entretien probables avec éléments de réponse, **4.** Limites assumées,
> **5.** Antisèche des chiffres clés.

---

## 1. Inventaire — ce que j'ai construit (2024-2026, WiseTwin)

### 1.1 Le SaaS WiseTwin (`app.wisetwin.eu`)

Plateforme SaaS **multitenant** de gestion et distribution de formations à la
sécurité industrielle, regroupant trois produits : **WiseTrainer** (modules
immersifs Unity WebGL), **WisePaper** (formations documentaires générées par IA)
et **SafetyTour** (visites de prévention sur scènes 3D).

- **Échelle** : ~80 000 lignes de TypeScript, 124 routes API, ~40 modèles Prisma, 15 migrations.
- **Multitenancy** : isolation par organisation jusqu'au stockage — un container
  Azure Blob **dédié par organisation**. Rôles OWNER/ADMIN/MEMBER + super-admin
  plateforme, wrappers d'auth maison (`withAuth` / `withOrgAuth`).
- **Sécurité & conformité** : SSO d'entreprise (WorkOS), MFA TOTP avec backup
  codes chiffrés, audit logs, consentement horodaté avec preuve IP/User-Agent.
- **Conformité formation** : exports **xAPI 1.0.3 / cmi5**, rapports **Qualiopi**,
  vérification de certificats (génération PDF via Puppeteer).
- **FinOps IA** : chaque requête LLM tracée (tokens input/output/cache, coût EUR),
  **budget IA mensuel par organisation** selon le tier (FREE 1€ / PRO 10€ /
  BUSINESS 30€), erreurs 429/402 remontées au front avec CTA d'upgrade.
- **Stack** : Next.js 15 (App Router) + React 19, Prisma 7, PostgreSQL (Neon),
  NextAuth v4, Zustand + TanStack Query, Tailwind 4 + shadcn/ui, déployé sur
  Vercel (région `fra1`), crons Vercel.

### 1.2 Le backend IA — copilote sécurité HSE (`wisetwin-ai`)

Backend Python qui croise l'accidentologie publique française (base **ARIA**,
62 726 incidents industriels) avec les incidents internes d'une organisation,
son catalogue de formations et son équipe — via un **agent LLM conversationnel**.

- **Agent** : boucle agentique **codée à la main, sans framework** (~50 lignes de
  cœur) sur l'API tool-use d'Anthropic. **12 tools** (10 lecture, 2 écriture),
  max 10 étapes par tour, streaming **SSE** token par token avec annonce des
  étapes d'outil. Les mutations (créer un plan de formation, assigner un membre)
  exigent une **confirmation conversationnelle** de l'utilisateur (human-in-the-loop).
- **RAG** : embeddings **Voyage AI voyage-3 (1024 dimensions)**, stockés dans
  **pgvector** ; recherche par distance cosine **exécutée côté SQL** (les 62k
  vecteurs ne quittent jamais Postgres), filtres métier combinés (date, secteur
  NAF), scoping `org_id` systématique. Retrieval "par l'exemple" : recherche
  ancrée sur le **centroïde des embeddings** d'une sélection manuelle d'incidents.
- **Sécurité de l'agent** : `org_id` injecté depuis le JWT côté serveur, **jamais
  exposé dans les schémas des tools** → le LLM ne peut ni le deviner ni le
  manipuler. **Tests d'isolation multi-tenant en CI**.
- **Multimodal** : "chasse aux risques" — analyse d'une photo de chantier par
  Claude → JSON structuré (risques, sévérité, confiance) enrichi de formations
  recommandées et d'accidents ARIA similaires. Extraction d'incidents depuis PDF
  avec **anonymisation PII** avant toute persistance.
- **Génération de formations** (module WisePaper industrialisé) : PPTX/PDF/DOCX →
  plan de cours proposé par le LLM → **validation humaine** → rédaction
  multimodale par chapitre (le modèle voit les slides en image, concurrence
  bornée à 4) → **vérification des QCM par un modèle moins cher (Haiku)** qui
  supprime les questions non démontrées par le support (pattern LLM-as-judge).
- **Data engineering** : ETL ARIA idempotent (IDs déterministes `uuid5`,
  `ON CONFLICT DO NOTHING`, batches de 500, embeddings par batch de 64,
  ingestion complète ≈ 20 min), sync du catalogue de formations, migrations
  Alembic async.
- **FinOps** : module de pricing (grille USD/MTok → EUR), consommation
  enregistrée **par modèle et par feature** (CHAT / RISK_HUNT / REPORT /
  WISEPAPER_IMPORT), rate-limit par utilisateur + budget mensuel par org.
- **Stack & déploiement** : FastAPI + Uvicorn, Python 3.12, SQLAlchemy 2 async,
  Pydantic v2, mypy strict, ruff, pytest ; Claude Sonnet 4.6 (principal) +
  Haiku 4.5 (vérification), Voyage AI ; **Docker → Azure Container Registry →
  Azure Container Apps** (scale-to-zero) via **GitHub Actions**.

### 1.3 WiseAtlas (`wiseatlas.wisetwin.eu`)

Éditeur SaaS multitenant de **cartes de territoire interactives** : points
d'intérêt, réseaux, zones, timeline narrative et mode présentation.

- **Cartographie 3D photoréaliste** : Google Maps Platform, API `maps3d` (beta) —
  tuiles 3D photoréalistes, overlays impératifs (markers, polylignes, polygones)
  et **modèles 3D GLB géolocalisés** (lat/lng/altitude, heading/tilt/roll).
  Double moteur 2D/3D interchangeable par projet.
- **Contenus** : blocs type Notion (texte, image, vidéo, **graphiques Recharts**
  multi-séries à double axe Y), import Excel.
- **Traduction IA multilingue en production** : batching par longueur de texte,
  contrôle de concurrence, back-off sur 429/529, comptage de tokens et
  **estimation de coût en USD par modèle** (Claude Haiku).
- **Sécurité** : NextAuth v5, MFA TOTP, autorisation à **deux axes indépendants**
  (plan de l'organisation × rôle de l'utilisateur), versioning avec restauration,
  publication publique par slug protégeable par mot de passe.
- **Stack** : Next.js 16, React 19, Zustand + zundo (undo/redo), Prisma 6,
  PostgreSQL, Azure Blob (un container par org).

### 1.4 Splat Editor — SafetyTour

Éditeur web de **Gaussian Splats 3D** pour créer des visites de prévention
interactives sur des numérisations de sites industriels réels.

- **Moteur 3D : PlayCanvas (WebGL2)**, initialisé impérativement, **totalement
  découplé de React** (dossier `engine/` sans aucun import React, communication
  par event bus + stores Zustand).
- **Édition de splats** : état par splat en bit-flags (selected/hidden/deleted),
  **shaders GLSL custom d'intersection exécutés sur GPU** pour la sélection
  (rectangle, pinceau, polygone), picking GPU, undo/redo en **Command pattern**.
- **Formats** : `.ply`, `.splat`, `.ksplat`, `.spz`, `.sog` + `.glb`.
- **Pédagogie** : hotspots hiérarchiques avec blocs de contenu (texte, image,
  vidéo, **quiz**), visites guidées scénarisées (storyboard, transitions caméra
  animées, mode Player pour l'apprenant), **export HTML autonome** (viewer +
  contenu embarqués dans un seul fichier).
- **Intégration SaaS** : JWT 8h émis par le SaaS, stockage Azure Blob via SAS URLs,
  splats en `compressed-ply` (~4× plus légers), cache IndexedDB, **analytics
  apprenant** (progression, réponses aux quiz, complétion) remontées au SaaS.

### 1.5 Landing page (`wisetwin.eu`)

Site vitrine : Next.js 15, **i18n FR/EN** (next-intl, routes localisées),
**blog MDX** avec front matter et CMS Git-based (Outstatic), SEO, formulaire de
contact avec protection **CSRF**, Vercel Analytics / Speed Insights.

### 1.6 Infrastructure cloud transversale

- **Vercel** : tous les frontends Next.js (SaaS, WiseAtlas, splat editor,
  landing), déploiement par push Git, crons.
- **Azure** : Blob Storage (stockage commun, isolation par container par tenant,
  SAS URLs), Container Registry + **Container Apps** (backend IA, scale-to-zero).
- **Neon** : PostgreSQL managé serverless (branches copy-on-write pour dev/prod).
- **CI/CD** : GitHub Actions pour le backend IA (`az acr build` →
  `az containerapp update`, image taguée par SHA).
- **Auth inter-services** : JWT signés partagés (SaaS ↔ backend IA, SaaS ↔ Unity,
  SaaS ↔ splat editor), CORS restreints par middleware.

### 1.7 Avant WiseTwin (rappel rapide)

- **TotalEnergies (2022-2024, Danemark)** — Data Engineer : data warehouse IoT
  offshore avec détection d'anomalies et monitoring Prefect, prévention
  d'accidents par NLP sur rapports + fine-tuning de LLM, contrôle de factures
  par computer vision, extraction massive SAP, déploiements Azure/on-premise.
- **CGI (2020-2022, Toulouse)** — Data Scientist : chatbot d'accès à la
  documentation technique avec **recherche sémantique** (Elasticsearch) — du RAG
  avant l'heure —, migration de données AWS, analytics comportementale.

### 1.8 Pitchs — vendre chaque projet en 30 secondes

> Format : problème → solution → mon rôle → preuve. À dérouler tel quel quand on
> demande « parlez-moi de ce projet », puis laisser l'interlocuteur creuser.

**Pitch personnel (l'elevator pitch)**
« Je construis des produits IA de bout en bout. Ces deux dernières années, j'ai
cofondé WiseTwin et développé seul l'essentiel de la plateforme : un SaaS
multitenant de formation à la sécurité industrielle, un copilote IA qui croise
62 000 accidents industriels avec les incidents internes des clients, et des
éditeurs 3D web pour créer les contenus. Avant ça, j'étais data engineer chez
TotalEnergies au Danemark et data scientist chez CGI. Mon profil, c'est ça :
l'ingénierie IA et data appliquée, avec la capacité de livrer le produit complet
autour — backend, front, cloud. »

**WiseTwin (l'entreprise, vue d'ensemble)**
*Problème* : la formation sécurité des industriels repose sur des PowerPoints en
salle — peu engageant, peu mesurable, déconnecté du terrain. *Solution* :
WiseTwin fait s'entraîner les techniciens sur des reproductions 3D de leurs
propres installations et utilise l'IA pour personnaliser la prévention à partir
de leurs incidents réels. *Mon rôle* : cofondateur, responsable de toute la
technique — du SaaS au moteur 3D en passant par la couche IA. *Preuve* : en
production sur app.wisetwin.eu avec des clients industriels.

**Le SaaS WiseTwin**
*Problème* : distribuer, tracer et prouver la formation dans des organisations
industrielles aux fortes exigences de conformité. *Solution* : plateforme
multitenant regroupant trois produits de formation (3D immersive, documentaire,
visites de prévention), avec SSO d'entreprise, MFA, audit logs et exports
réglementaires xAPI/cmi5 et Qualiopi. *Mon rôle* : architecture et développement
de bout en bout. *Preuve* : ~80 000 lignes de TypeScript, 124 routes API,
isolation par tenant jusqu'au stockage, en production.

**Le copilote IA HSE**
*Problème* : les responsables sécurité disposent de bases d'accidents énormes
(62 726 cas publics ARIA) et de leurs incidents internes, mais aucun outil ne
les croise — l'analyse prend des heures et les signaux faibles passent à la
trappe. *Solution* : un agent conversationnel qui interroge ces corpus par
recherche sémantique, cite ses sources, recommande des formations du catalogue
et va jusqu'à créer les plans de formation — après confirmation humaine.
*Mon rôle* : conçu et développé intégralement — ETL, RAG pgvector, boucle
agentique sans framework, streaming SSE, vision multimodale, FinOps. *Preuve* :
12 outils, ~0,03 $ par conversation, isolation multi-tenant prouvée par tests
en CI.

**La génération de formations (WisePaper)**
*Problème* : les industriels ont des années de supports PowerPoint mais les
digitaliser en e-learning coûte des semaines par module. *Solution* : un
pipeline qui transforme un PPTX en formation structurée avec quiz en quelques
minutes — extraction, transcription des audios, plan proposé par l'IA et validé
par l'humain, rédaction multimodale, QCM vérifiés par un second modèle.
*Mon rôle* : du prototype R&D à la version industrialisée. *Preuve* : porte
humaine sur le plan, LLM-as-judge sur les QCM — de la qualité contrôlée, pas du
« generate and pray ».

**WiseAtlas**
*Problème* : présenter un territoire ou un site industriel (réseaux, zones,
projets d'aménagement) exige soit un SIG complexe, soit des captures d'écran
figées. *Solution* : un éditeur SaaS de cartes 3D photoréalistes avec
storytelling — timeline, mode présentation, blocs de contenu riches, graphiques,
modèles 3D posés sur la carte. *Mon rôle* : conception et développement complets.
*Preuve* : en production sur wiseatlas.wisetwin.eu, traduction multilingue par
LLM intégrée avec suivi des coûts.

**Le Splat Editor (SafetyTour)**
*Problème* : créer du contenu de formation 3D fidèle au terrain coûte cher —
modéliser une usine à la main prend des mois. *Solution* : on numérise le site
réel (gaussian splatting, à partir de simples vidéos), puis mon éditeur web
permet de nettoyer la scène, d'y ancrer des points pédagogiques avec quiz et de
publier des visites de prévention interactives. *Mon rôle* : l'éditeur complet,
du moteur 3D (PlayCanvas, shaders GPU) à l'intégration SaaS. *Preuve* : tout
tourne dans le navigateur, export HTML autonome, analytics de complétion des
apprenants.

**Bootstrap-Now**
*Problème* : les auto-entrepreneurs se lancent sans étude de marché ni business
plan, faute d'outils abordables. *Solution* : plateforme SaaS qui agrège des
opportunités business et guide la création d'un business plan assisté par IA.
*Mon rôle* : cofondateur, développement complet. *Preuve* : en ligne sur
bootstrap-now.com. (Pitch court — projet à présenter comme secondaire face à
WiseTwin.)

---

## 2. Glossaire technique — définitions + exemples tirés de mes projets

> Format : définition courte → « **Chez moi :** » l'exemple concret à citer en
> entretien. Réviser en priorité les sections 2.1 et 2.2.

### 2.1 IA générative & agents

**LLM (Large Language Model)** — Modèle de langage entraîné sur de vastes corpus,
utilisé via API (Claude, GPT) ou auto-hébergé. On le pilote par le *prompt*
(system + messages) et des paramètres (température, max tokens).
**Chez moi :** Claude Sonnet 4.6 pour l'agent HSE et la génération de cours,
GPT-4 + Whisper dans le prototype WisePaper, Claude Haiku pour les tâches
« cheap » (traduction WiseAtlas, vérification de QCM).

**Prompt système (system prompt)** — Instructions permanentes qui cadrent le
comportement du modèle pour toute la conversation.
**Chez moi :** le system prompt de l'agent HSE impose les citations sourcées, le
protocole de confirmation avant mutation, le filtrage sectoriel ARIA selon
l'industrie du client, et la langue de réponse (fr/en).

**Tool use / function calling** — Mécanisme par lequel le LLM ne répond pas en
texte mais demande l'exécution d'une fonction déclarée (nom + schéma JSON des
paramètres). Le code exécute la fonction et renvoie le résultat au modèle.
**Chez moi :** 12 tools déclarés à l'API Anthropic (`search_public_incidents`,
`recommend_formations`, `create_training_plan`…). Le modèle en enchaîne
typiquement 2 à 5 par tour.

**Agent (agentic loop)** — Boucle : appel LLM → si le modèle demande des tools,
les exécuter et renvoyer les résultats → nouvel appel → … jusqu'à une réponse
finale (`end_turn`). C'est ce qui transforme un chatbot en système capable
d'*agir*.
**Chez moi :** boucle maison d'environ 50 lignes, garde-fou `MAX_AGENT_STEPS=10`,
deux variantes (bloquante et streamée). Choix assumé de **ne pas** utiliser
LangChain/LangGraph (voir question 3.1).

**Human-in-the-loop** — Un humain valide les actions sensibles proposées par
l'IA avant leur exécution.
**Chez moi :** deux niveaux — (1) l'agent doit annoncer une mutation et attendre
un « oui » explicite avant d'appeler `create_training_plan` ; (2) dans la
génération de cours, le plan proposé par le LLM est validé/édité par
l'utilisateur avant la rédaction (porte humaine entre les 2 phases).

**LLM-as-judge** — Utiliser un second modèle pour évaluer/filtrer la production
du premier, souvent un modèle moins cher.
**Chez moi :** Haiku vérifie chaque QCM généré par Sonnet et supprime les
questions dont la bonne réponse n'est pas démontrée par le support de cours
(politique *fail-open* : si le juge échoue, on garde la question plutôt que de
bloquer le pipeline).

**Sorties structurées (JSON mode / forced tool choice)** — Forcer le modèle à
répondre dans un schéma JSON validable plutôt qu'en texte libre.
**Chez moi :** `tool_choice` forcé sur un tool « émetteur » (`emit_risk_hunt`,
`emit_incident`…) côté Anthropic ; `response_format={"type": "json_object"}`
côté OpenAI dans le prototype. Validation Pydantic/Zod derrière — le schéma est
le contrat entre l'IA et le code.

**Multimodalité / vision** — Envoyer autre chose que du texte au modèle (images,
PDF, audio).
**Chez moi :** photos de chantier analysées pour la chasse aux risques
(redimensionnées à 1568 px, l'optimum de Claude), PDF d'incidents envoyés
nativement en bloc `document`, slides de cours vues en image pendant la
rédaction des chapitres, audio transcrit par Whisper.

**Fine-tuning vs RAG** — Fine-tuning : ré-entraîner partiellement un modèle sur
ses données (coûteux, fige la connaissance, utile pour le *style* ou un domaine
très spécifique). RAG : injecter la connaissance à la volée dans le contexte
(à jour, traçable, moins cher).
**Chez moi :** j'ai fait les deux — fine-tuning de modèles chez TotalEnergies
(classification de rapports d'incidents), RAG chez WiseTwin. Règle que
j'applique : RAG d'abord ; le fine-tuning seulement si le format/style de sortie
ne s'obtient pas par prompt.

**Prompt caching** — Les fournisseurs facturent moins cher les préfixes de
prompt déjà vus (system prompt, gros contextes répétés).
**Chez moi :** le suivi de consommation trace déjà `cacheReadTokens` /
`cacheCreationTokens` ; l'activation effective du caching est dans la roadmap —
je sais expliquer le mécanisme et son ROI (system prompt long × requêtes
fréquentes = candidat idéal).

**FinOps IA** — Suivi et gouvernance des coûts d'inférence.
**Chez moi :** chaque appel enregistre tokens et coût en EUR (Decimal), **par
modèle et par feature** ; budget mensuel par organisation selon le tier, quota
par utilisateur en fenêtre glissante, erreurs structurées 429 (rate-limité) /
402 (budget dépassé) avec CTA d'upgrade dans l'UI. Ordre de grandeur : ~0,03 $
par tour de chat.

### 2.2 RAG & données

**RAG (Retrieval-Augmented Generation)** — Avant de répondre, on *récupère* les
documents pertinents (recherche sémantique) et on les met dans le contexte du
LLM, avec consigne de citer ses sources.
**Chez moi :** le copilote HSE croise trois corpus embarqués — 62 726 accidents
ARIA, incidents internes de l'org, catalogue de formations — via des tools de
recherche vectorielle appelés par l'agent (RAG *agentique* : c'est le modèle qui
décide quand et quoi chercher, plutôt qu'un retrieval systématique).

**Embedding** — Représentation d'un texte en vecteur numérique dense ; deux
textes proches sémantiquement ont des vecteurs proches.
**Chez moi :** Voyage AI `voyage-3`, 1024 dimensions, en distinguant
`input_type="document"` (indexation) et `input_type="query"` (recherche) —
bonne pratique qui améliore la pertinence. Batches de 64 à l'ingestion.

**Base vectorielle / pgvector** — Stockage + recherche de vecteurs par
similarité. pgvector est l'extension Postgres qui l'apporte au SQL.
**Chez moi :** pgvector plutôt qu'un service dédié (Pinecone, Weaviate) : mes
données relationnelles sont déjà dans Postgres, la recherche vectorielle se
combine en une seule requête avec les filtres métier (`WHERE org_id = … AND
date > …`), zéro infra en plus (voir question 3.3).

**Similarité cosine** — Mesure d'angle entre deux vecteurs (1 = identiques).
En SQL : `ORDER BY embedding <=> query LIMIT k` ; je renvoie
`similarity = 1 - distance` à l'agent.
**Chez moi :** la recherche s'exécute **dans** Postgres — seules les k lignes
gagnantes remontent, jamais les 62k vecteurs.

**Chunking** — Découper les documents en morceaux avant embedding (un vecteur ne
résume bien qu'un texte court).
**Chez moi :** granularité *document* : 1 incident = 1 embedding (résumés courts
~1500 caractères), pas de découpage en passages. À dire en entretien : le
chunking classique (500-1000 tokens avec chevauchement) s'impose pour des
documents longs, mes unités étaient naturellement courtes.

**Reranking** — Second passage plus précis (cross-encoder) sur les top-k
résultats du retrieval vectoriel pour réordonner.
**Chez moi :** pas implémenté — assumé comme amélioration future : top-k faible
et granularité document rendaient le gain marginal face au coût/latence au stade
POC.

**Index ANN (HNSW / IVFFlat)** — Index de recherche vectorielle *approximative*
pour passer de O(n) au sous-linéaire quand le corpus grossit.
**Chez moi :** recherche en scan exact (~100-300 ms sur 62k lignes), acceptable
à cette échelle ; HNSW est le premier levier de scalabilité identifié
(compromis : construction plus lente, RAM, léger risque de rappel < 100 %).

**Retrieval par centroïde** — Chercher à partir de la *moyenne* des embeddings
d'un ensemble d'exemples plutôt que d'un libellé textuel.
**Chez moi :** quand l'utilisateur sélectionne manuellement des incidents pour
un rapport, la recherche d'accidents ARIA similaires s'ancre sur le centroïde de
sa sélection — « cherche-moi des cas comme *ceux-là* ».

**ETL / pipeline de données** — Extract, Transform, Load : ingestion de données
sources vers un stockage exploitable.
**Chez moi :** ingestion ARIA (CSV cp1252 → pandas → mapping vers schéma unifié
→ `raw_payload` JSONB pour ne rien perdre → Postgres), scripts d'embedding
séparés et rejouables, sync du catalogue de formations depuis la base SaaS.

**Idempotence** — Rejouer une opération produit le même état final (pas de
doublons).
**Chez moi :** IDs déterministes `uuid5(NAMESPACE_URL, "ARIA:<num>")` +
`ON CONFLICT DO NOTHING` → l'ingestion est relançable à volonté ; chez
TotalEnergies, même principe sur les pipelines IoT.

**Anonymisation / PII** — Retirer les données personnelles identifiantes avant
stockage ou envoi au LLM.
**Chez moi :** les PDF d'incidents clients sont transformés en fiches
dé-sensibilisées (résumé, tags, gravité — aucun nom) ; seule cette version est
persistée et vue par l'agent.

**Data warehouse** — Base analytique centralisée alimentée par plusieurs sources.
**Chez moi :** data warehouse IoT offshore chez TotalEnergies (capteurs →
détection d'anomalies → monitoring Prefect → alertes).

### 2.3 Architecture web & temps réel

**SSE (Server-Sent Events)** — Flux HTTP unidirectionnel serveur → client
(`Content-Type: text/event-stream`), idéal pour streamer une réponse LLM.
Plus simple que WebSocket (pas de bidirectionnel, passe les proxies HTTP,
reconnexion native côté navigateur).
**Chez moi :** le chat agentique streame des événements typés `text` / `step`
(annonce d'un appel d'outil) / `tool_done` / `done` / `error` ; l'import PPTX
streame sa progression en 8 étapes. Détails production : keepalives, padding
anti-buffering, timeouts. WebSocket n'aurait rien apporté ici : le client ne
fait qu'écouter (voir question 3.6).

**BFF (Backend For Frontend)** — Le backend Next.js sert d'API dédiée au front
et de proxy authentifié vers les services internes.
**Chez moi :** le SaaS est le *gatekeeper* : NextAuth détient la session, signe
un JWT court (1 h) et proxifie le flux SSE du backend IA — le backend Python ne
connaît jamais NextAuth.

**Multitenancy** — Une seule instance applicative sert plusieurs organisations
avec isolation stricte des données.
**Chez moi :** défense en profondeur sur 4 couches — (1) `org_id` extrait du JWT
uniquement, jamais du body ; (2) pattern Repository avec `org_id` obligatoire
dans les signatures ; (3) un container Azure Blob par organisation ; (4) tests
d'isolation cross-tenant en CI qui prouvent l'absence de fuite.

**RBAC (Role-Based Access Control)** — Autorisations par rôles.
**Chez moi :** OWNER/ADMIN/MEMBER + super-admin plateforme dans le SaaS ; dans
WiseAtlas, deux axes indépendants (plan de l'org × rôle de l'utilisateur,
`requireOrgRole(minRole, minPlan)`).

**JWT (JSON Web Token)** — Jeton signé porteur de claims, vérifiable sans état
partagé.
**Chez moi :** monnaie d'échange inter-services — JWT 1 h `{org_id, user_id}`
SaaS → backend IA ; JWT 24 h pour les builds Unity ; JWT 8 h pour le splat
editor. Le service receveur vérifie la signature et fait confiance aux claims,
pas au client.

**SSO / SAML / OIDC** — Authentification déléguée à l'annuaire de l'entreprise
cliente (indispensable pour vendre du B2B).
**Chez moi :** SSO via WorkOS dans le SaaS, avec workflow d'approbation.

**MFA / TOTP** — Second facteur par code à usage unique basé sur le temps
(RFC 6238 — Google Authenticator etc.).
**Chez moi :** implémenté deux fois (SaaS et WiseAtlas) : secrets chiffrés en
base, QR code d'enrôlement, backup codes, MFA rendu obligatoire par org.

**ORM / Prisma / migrations** — Couche d'accès aux données typée + évolution
versionnée du schéma.
**Chez moi :** Prisma (~40 modèles, 15 migrations, `migrate deploy` au build)
côté TypeScript ; SQLAlchemy 2 async + Alembic côté Python.

**Validation de schéma (Zod / Pydantic)** — Valider les entrées/sorties à la
frontière du système, types garantis à l'exécution.
**Chez moi :** Zod centralisé dans `validators/` côté Next.js, Pydantic v2 avec
discriminated unions côté FastAPI ; c'est aussi ce qui valide les sorties JSON
des LLM.

**Command pattern (undo/redo)** — Chaque modification est un objet
« commande » avec `do()`/`undo()`, empilé dans un historique.
**Chez moi :** l'éditeur de splats (EditHistory/EditOps) ; WiseAtlas utilise
zundo au-dessus de Zustand.

**Event bus / découplage moteur-UI** — Communication par événements pour éviter
le couplage direct entre modules.
**Chez moi :** le moteur 3D du splat editor est *framework-agnostic* (zéro
import React) ; React et le moteur PlayCanvas communiquent via stores Zustand et
un event bus. Testabilité et portabilité du moteur.

**xAPI / cmi5** — Standards e-learning de traçage d'activité (« qui a fait quoi »)
consommés par les LMS d'entreprise.
**Chez moi :** export xAPI 1.0.3/cmi5 des analytics de formation + rapport
Qualiopi (certification qualité des organismes de formation français).

### 2.4 Cloud & DevOps

**Conteneurisation (Docker)** — Empaqueter l'app et ses dépendances système en
image immuable.
**Chez moi :** Dockerfile du backend IA sur `python:3.12-slim` incluant
WeasyPrint et LibreOffice headless (conversion PPTX → PDF → images) — exemple de
dépendances système impossibles à gérer sans conteneur.

**Azure Container Apps (ACA)** — Conteneurs serverless managés (scale-to-zero,
HTTP ingress) sans gérer de cluster Kubernetes.
**Chez moi :** le backend IA tourne sur ACA avec scale-to-zero (un POC ne paie
pas quand il dort) ; ACA Jobs visés pour les tâches longues.

**CI/CD** — Pipeline automatisé build → test → déploiement.
**Chez moi :** GitHub Actions : `az acr build` (build dans le cloud, image
taguée par SHA de commit) puis `az containerapp update`. Les fronts sont en
déploiement continu Vercel par push Git. Chez TotalEnergies : pipelines CI/CD
Azure DevOps.

**SAS URLs (Shared Access Signatures)** — URLs Azure Blob signées à durée et
permissions limitées : le client accède au fichier sans exposer les credentials
du compte de stockage.
**Chez moi :** partout — le splat editor upload/download via SAS, le backend IA
émet des SAS courtes (15 min) pour les images.

**Serverless / Neon / scale-to-zero** — Facturation à l'usage, la ressource
s'éteint sans trafic. Neon apporte à Postgres les *branches copy-on-write*
(brancher la base comme du code).
**Chez moi :** Neon pour les Postgres, branches séparées dev/prod ; Vercel pour
les fronts ; ACA pour le backend.

**Observabilité** — Logs, métriques, alertes pour comprendre la prod.
**Chez moi :** audit logs applicatifs, suivi de consommation IA par tenant,
Vercel Analytics ; chez TotalEnergies : monitoring Prefect + alertes SMTP sur
les pipelines IoT.

### 2.5 3D temps réel

**Gaussian Splatting** — Représentation 3D issue de la photogrammétrie : des
millions de « gaussiennes » (position, covariance, couleur, opacité) rendues par
rasterisation — photoréalisme à partir de simples photos/vidéos d'un site.
**Chez moi :** SafetyTour numérise des sites industriels réels en splats ; mon
éditeur les charge (5 formats), les nettoie (sélection/suppression), y ancre des
hotspots pédagogiques et publie des visites de prévention.

**WebGL2 / shaders GLSL** — Programmes exécutés sur le GPU dans le navigateur.
**Chez moi :** shaders custom d'intersection pour la sélection de splats
(rectangle/pinceau/polygone calculés côté GPU sur des millions de points),
picking GPU, grille infinie en shader.

**PlayCanvas vs Three.js vs Unity** — Trois approches du 3D web : moteur complet
orienté jeu (Unity, lourd, export WebGL), bibliothèque de rendu (Three.js),
moteur web natif avec ECS (PlayCanvas).
**Chez moi :** les trois en production — Unity WebGL pour WiseTrainer (contenus
immersifs riches), PlayCanvas pour l'édition de splats (support GSplat natif,
léger), Google Maps 3D pour WiseAtlas (tuiles photoréalistes planétaires sans
gérer le rendu).

**glTF / GLB** — Format standard d'échange de modèles 3D (« le JPEG de la 3D »).
**Chez moi :** modèles GLB géolocalisés sur la carte 3D de WiseAtlas, GLB
gzippés dans le splat editor.

---

## 3. Questions d'entretien probables & éléments de réponse

### Choix d'architecture IA

**3.1 « Pourquoi ne pas avoir utilisé LangChain / LangGraph ? »**
Réponse : la boucle agentique dont j'avais besoin fait ~50 lignes au-dessus de
l'API tool-use d'Anthropic. Un framework aurait ajouté une couche d'abstraction
sur un protocole déjà simple, compliqué le débogage (stack traces opaques), et
créé une dépendance à un écosystème qui bougeait plus vite que mes besoins. En
codant la boucle, je contrôle finement le streaming SSE, l'injection sécurisée
de l'`org_id`, le gating des mutations et le comptage des tokens. Nuance à
donner : je ne suis pas dogmatique — sur un système multi-agents complexe avec
checkpointing, LangGraph ou un équivalent redevient pertinent ; c'est un
arbitrage build vs buy que je sais réévaluer.

**3.2 « Comment sécurise-t-on un agent LLM qui peut écrire en base ? »**
Réponse en couches : (1) séparation lecture/écriture — 10 tools read, 2 write ;
(2) le tenant (`org_id`) vient exclusivement du JWT vérifié côté serveur et
n'apparaît **pas** dans les schémas exposés au modèle — le LLM ne peut pas le
manipuler, c'est le code qui l'injecte ; (3) human-in-the-loop : le system
prompt impose l'annonce de la mutation et l'attente d'une confirmation
explicite ; (4) idempotence des écritures (`ON CONFLICT DO NOTHING`) ;
(5) traçabilité (`acting_user_id` injecté sur les writes) ; (6) garde-fou
`MAX_AGENT_STEPS` contre les boucles infinies ; (7) tests d'isolation
cross-tenant en CI. Conclusion : ne jamais faire confiance à la sortie du
modèle, la traiter comme une entrée utilisateur.

**3.3 « Pourquoi pgvector plutôt que Pinecone/Weaviate/Qdrant ? »**
Réponse : mes données relationnelles (organisations, incidents, formations)
vivaient déjà dans Postgres. pgvector me donne la recherche vectorielle **dans
la même requête SQL** que les filtres métier (org, date, secteur), une seule
technologie à opérer, des transactions ACID, et zéro coût/latence réseau
supplémentaires. À 62k vecteurs, le scan exact tient en 100-300 ms — largement
en dessous de la latence LLM. Un service vectoriel dédié se justifie à partir de
dizaines de millions de vecteurs ou pour des besoins de sharding spécifiques.

**3.4 « Comment évaluez-vous la qualité de votre RAG / de votre agent ? »**
Réponse honnête : au stade POC, l'évaluation était manuelle (jeux de questions
métier avec les experts HSE) + garde-fous structurels (citations obligatoires,
similarité renvoyée à l'agent, LLM-as-judge sur les QCM). Un harness d'éval
formel (golden set de questions/réponses, métriques de retrieval type
recall@k, LLM-as-judge sur la fidélité des réponses, éval en CI) est la
prochaine étape — je sais précisément ce qu'il faut construire et je peux le
détailler. Ne pas prétendre avoir un harness qui n'existe pas : c'est un
excellent sujet pour montrer la lucidité d'ingénierie.

**3.5 « RAG ou fine-tuning ? Quand choisir quoi ? »**
Réponse : RAG quand la connaissance évolue, doit être traçable (citations) et
scopée par client — mon cas chez WiseTwin. Fine-tuning quand c'est le
*comportement* qu'on veut changer (style, format, classification spécialisée) —
fait chez TotalEnergies pour la classification de rapports d'incidents.
Les deux se combinent. Et souvent, un bon prompt + des sorties structurées
suffisent avant d'envisager l'un ou l'autre.

**3.6 « Pourquoi SSE et pas WebSocket ? »**
Réponse : le besoin est unidirectionnel (le serveur pousse des tokens et des
événements de progression ; le client, lui, fait des POST classiques). SSE =
HTTP simple, compatible proxies/CDN, reconnexion native, pas de serveur stateful.
J'ai géré les subtilités réelles : keepalives, padding anti-buffering des
navigateurs, timeouts, négociation de contenu pour garder la rétro-compatibilité
JSON. WebSocket se justifierait pour du bidirectionnel temps réel (collaboration
simultanée, par exemple).

**3.7 « Comment maîtrisez-vous les coûts LLM en production ? »**
Réponse : (1) mesurer — chaque appel enregistre tokens in/out/cache et coût EUR,
par modèle et par feature ; (2) plafonner — budget mensuel par organisation lié
au pricing (tiers), rate-limit par utilisateur, erreurs 429/402 propres dans
l'UX ; (3) optimiser — le bon modèle pour la bonne tâche (Sonnet pour l'agent,
Haiku pour la traduction et la vérification de QCM : ~5× moins cher), images
redimensionnées à l'optimum du modèle, résumés courts plutôt que documents
bruts dans le contexte, prompt caching identifié comme prochain levier. Ordre de
grandeur à citer : ~0,03 $ par tour de chat agentique.

**3.8 « Racontez-moi le pipeline de génération de formations. »**
Réponse structurée : PPTX/PDF → extraction (texte, notes, médias ; LibreOffice →
PDF → PNG par slide) → transcription des audios (Whisper) → **phase A** : un
appel LLM sur toute la matière produit un *plan* (chapitres, objectifs, concepts
clés → budget de QCM proportionnel) → **porte humaine** : l'utilisateur valide
ou édite le plan → **phase B** : rédaction par chapitre en parallèle borné
(concurrence 4), le modèle voyant les slides en image → vérification des QCM par
Haiku (suppression des questions non démontrées par le source) → blocs
WisePaper + médias uploadés sur Azure avec SAS. Pourquoi deux phases : qualité
(le plan global évite les redites), contrôle utilisateur, et coût (on ne rédige
pas un cours entier pour le jeter).

### Data engineering

**3.9 « Comment rendez-vous un pipeline d'ingestion fiable ? »**
Réponse : idempotence par conception (IDs déterministes uuid5 + upsert), rien
n'est jeté (`raw_payload` JSONB conserve la source brute), batching (500
lignes / 64 embeddings), scripts rejouables avec `--force`, encodages et
formats sources gérés explicitement (CSV cp1252, séparateur `;`, skiprows).
Anecdote TotalEnergies disponible : pipelines IoT offshore avec monitoring
Prefect et alertes — la fiabilité passe par l'observabilité autant que par le
code.

**3.10 « Comment gérez-vous des données sensibles avec des LLM ? »**
Réponse : anonymisation avant tout — les PDF d'incidents sont réduits à des
fiches dé-sensibilisées (aucun nom) avant persistance et avant tout passage dans
le contexte de l'agent ; scoping strict par tenant à chaque couche ; SAS URLs
courtes pour les fichiers ; et contractuellement, API Anthropic en mode
zéro-rétention des données (ZDR).

### Web, cloud, architecture

**3.11 « Monolithe Next.js + un backend Python : pourquoi cette découpe ? »**
Réponse : le SaaS Next.js est un monolithe BFF — productivité maximale pour un
produit piloté par une toute petite équipe, une seule base de code front+API,
déploiement Vercel trivial. Le backend IA est séparé parce que ses contraintes
divergent : Python (écosystème IA), dépendances système lourdes (LibreOffice,
WeasyPrint → Docker), scaling indépendant (scale-to-zero), et un cycle de vie
R&D plus rapide. La frontière est un contrat JWT simple. C'est du pragmatisme :
séparer quand les *contraintes* divergent, pas pour suivre une mode
microservices.

**3.12 « Comment implémenteriez-vous la multitenancy ? » (question système design)**
Dérouler mon existant : tenant résolu côté serveur (JWT), jamais côté client ;
`org_id` obligatoire dans les signatures de la couche data (le type system
force le scoping) ; isolation du stockage fichiers par container ; budgets et
quotas par tenant ; audit logs ; tests automatisés d'isolation. Mentionner les
alternatives et leurs arbitrages : schéma-par-tenant, base-par-tenant, RLS
Postgres (Row-Level Security) — je suis en « shared schema + scoping applicatif
testé », le bon compromis à ma taille ; RLS est l'étape d'après.

**3.13 « Vercel + Azure + Neon : pourquoi trois fournisseurs ? »**
Réponse : chaque workload sur la plateforme qui le sert le mieux — DX Vercel
pour les fronts Next.js (preview deployments, CDN), Azure pour le stockage
d'entreprise et les conteneurs (crédibilité B2B industriel, ACR/ACA), Neon pour
le Postgres serverless (branches dev/prod). Le coût de coordination est faible
(des URLs et des secrets). Contrepartie assumée : pas d'IaC unifié à ce stade —
voir limites.

**3.14 « Une décision technique que vous regrettez / referiez autrement ? »**
Candidats honnêtes : (1) dans le prototype WisePaper, les appels OpenAI étaient
synchrones dans un service async — ils bloquaient l'event loop ; corrigé dans la
version industrialisée (SDK async, concurrence bornée) — belle histoire
d'apprentissage asyncio ; (2) SAS URLs de 10 ans dans le prototype vs 15 min
dans la V2 — le durcissement sécurité fait partie de l'industrialisation ;
(3) avoir écrit deux fois le pipeline PPTX (prototype puis réécriture propre) —
mais c'est le cycle POC → prod assumé d'une startup.

### Questions comportementales (préparer en format STAR)

**3.15 « Cofondateur qui redevient salarié : pourquoi ? »** — Préparer une
réponse positive et honnête : ce que l'aventure WiseTwin m'a appris (ownership
de bout en bout, arbitrages coût/qualité/délai, vente technique), pourquoi le
prochain chapitre est en Suisse, et ce que j'apporte précisément à l'employeur :
un profil qui a *déjà* porté un produit IA en production seul. Anticiper la
sous-question « allez-vous repartir créer une boîte ? » et la question de
l'avenir de WiseTwin (rester factuel : ce qui continue, ce qui s'arrête, mon
niveau d'implication restant).

**3.16 « Votre plus grosse difficulté technique ? »** — Bons candidats :
sélection GPU de millions de splats (shaders d'intersection custom), isolation
multi-tenant d'un agent LLM (l'org_id hors de portée du modèle), streaming SSE
robuste à travers proxies (padding, keepalives), pipeline multimodal en deux
phases avec porte humaine.

**3.17 « Comment travaillez-vous avec les LLM au quotidien ? »** — Assumer un
workflow moderne : développement assisté par IA (Claude Code), mais avec revue
systématique, tests (isolation en CI), typage strict (mypy strict, TS strict)
comme filets. Les entreprises suisses y sont de plus en plus sensibles — c'est
un différenciateur si c'est présenté avec rigueur.

**3.18 Questions à poser en fin d'entretien** — Préparer 3-4 questions :
maturité IA de l'équipe (éval ? gouvernance des coûts ?), politique de données
avec les fournisseurs LLM, organisation data/ML vs engineering, critères de
succès du poste à 6 mois.

---

## 4. Limites & axes d'amélioration (à assumer si on creuse)

| Sujet | État actuel | Ce que je réponds |
|---|---|---|
| Éval RAG/agent | Pas de harness formel (éval manuelle + garde-fous) | Roadmap claire : golden set, recall@k, LLM-as-judge en CI |
| Index vectoriel | Scan exact (pas de HNSW) | Suffisant à 62k vecteurs (100-300 ms) ; HNSW = premier levier de scale |
| Reranking | Absent | Gain marginal au stade POC ; je sais où l'insérer |
| Prompt caching | Tracé mais pas activé | Prochain levier FinOps, ROI évident sur le system prompt long |
| IaC | Aucun (az CLI + scripts) | Assumé au stade startup ; Bicep/Terraform dès que l'infra se stabilise |
| Démo ARIA | Sous-ensemble BTP (~640) chargé en démo | Pipeline conçu et testé pour les 62 726 ; formulation honnête |
| Tests | Bonne couverture ciblée (isolation, santé) mais pas exhaustive | Priorité aux tests qui protègent les invariants critiques (tenant isolation) |

---

## 5. Antisèche — les chiffres à retenir

| Chiffre | Quoi |
|---|---|
| **62 726** | Accidents industriels ARIA ingérés (ETL complet ≈ 20 min) |
| **12** | Tools de l'agent (10 read / 2 write), 2-5 appelés par tour |
| **1024** | Dimensions des embeddings (Voyage voyage-3) |
| **~80 000** | Lignes de TypeScript du SaaS |
| **124** | Routes API du SaaS |
| **~40 / 15** | Modèles Prisma / migrations |
| **100-300 ms** | Latence recherche pgvector (scan exact, 62k lignes) |
| **~0,03 $** | Coût d'un tour de chat agentique |
| **10** | Étapes max de la boucle agent |
| **4** | Chapitres rédigés en parallèle (génération de cours) |
| **~4×** | Compression des splats (compressed-ply) |
| **5** | Formats de splats supportés (.ply/.splat/.ksplat/.spz/.sog) |
| **3** | Produits du SaaS (WiseTrainer, WisePaper, SafetyTour) + WiseAtlas |
| **1€/10€/30€** | Budgets IA mensuels par tier (FREE/PRO/BUSINESS) |
| **4** | Couches de défense d'isolation multi-tenant |

---

*Généré le 2026-07-03 à partir de l'analyse des repos. À convertir en PDF
(pandoc ou LaTeX) une fois le contenu stabilisé.*
