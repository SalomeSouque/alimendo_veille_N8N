# Alimendo — veille technique et réglementaire (n8n)

Dispositif de veille du projet **Alimendo** (projet étudiant de fin de formation Développeur·se en IA, compétence **C6**). Alimendo est une application d'information destinée aux personnes atteintes d'endométriose : score d'inflammation des aliments et chatbot RAG sourcé. Ce n'est pas un dispositif médical et les synthèses ne constituent pas un conseil médical.

Le dépôt contient le workflow n8n qui collecte les sources, les fait trier par un modèle de langage, puis produit une synthèse HTML accessible. Chaque synthèse est relue avant d'être versionnée dans `veille_genere/`.

## Sommaire

1. [Thématiques de veille](#thématiques-de-veille)
2. [Fonctionnement](#fonctionnement)
3. [Sources surveillées](#sources-surveillées)
4. [Installation](#installation)
5. [Lancer une veille](#lancer-une-veille)
6. [Rythme de veille](#rythme-de-veille)
7. [Format et diffusion des synthèses](#format-et-diffusion-des-synthèses)
8. [Choix des outils](#choix-des-outils)
9. [Limites connues](#limites-connues)
10. [Historique des modifications](#historique-des-modifications)

## Thématiques de veille

Trois axes, classés par ordre de criticité pour le projet :

| Axe | Thème | Pourquoi c'est surveillé |
|-----|-------|--------------------------|
| **B — Cadre réglementaire** (pilier) | RGPD et données de santé (CNIL, CEPD), AI Act, allégations nutritionnelles, statut de dispositif médical (HAS), accessibilité numérique (RGAA) | Une évolution ici peut rendre l'application non conforme. |
| **C — État de l'art scientifique** | Lien alimentation / inflammation / endométriose | Repérer toute publication qui confirme, nuance ou contredit un score du référentiel, ou qui peut enrichir le corpus du chatbot. |
| **A — Outils d'IA de la stack** | Qwen3-VL (VLM), Mistral (LLM du chatbot), LangChain, ChromaDB, n8n | Nouvelles versions, dépréciations de modèles, changements de licence ou de CGU. |

## Fonctionnement

```
Lancer la veille (déclencheur manuel)
 ├─ Flux a surveiller ──► Lire les flux RSS ───────────────┐
 ├─ ANSES - page HTML ──► découper ──► normaliser ─────────┤
 └─ Mistral - changelog ─► normaliser ─────────────────────┤
                                                           ▼
                                                    Fusionner flux
                                                           ▼
                                       Filtrer la période + construire le corpus
                                                           ▼
                                     Synthèse IA (Gemini, sortie JSON imposée)
                                                           ▼
                                         Générer HTML accessible ──► Fichier HTML
```

1. **Collecte** : flux RSS/Atom, plus deux sources sans flux récupérées par scraping (ANSES, changelog Mistral).
2. **Filtrage** : seuls les articles publiés dans la fenêtre de collecte (`FENETRE_JOURS`, 7 jours par défaut) sont conservés, puis concaténés en un corpus compact. Un seul appel au modèle par veille.
3. **Tri par l'IA** : le prompt impose les 3 axes, une note de pertinence de 1 à 3, le type de source (primaire ou média), le niveau de preuve pour l'axe C et un format JSON strict. Le modèle trie et met en forme. La décision reste humaine.
4. **Mise en forme** : le JSON est transformé en page HTML structurée (titres hiérarchisés, `lang="fr"`, liens explicites, contrastes suffisants).
5. **Relecture puis publication** : la synthèse est relue, puis commitée dans `veille_genere/`.

## Sources surveillées

| Source | Axe | Type | Accès |
|--------|-----|------|-------|
| CNIL | B | Primaire | RSS |
| CEPD / EDPB | B | Primaire | RSS |
| HAS — Recommandations | B | Primaire | RSS |
| HAS — Dispositifs médicaux | B | Primaire | RSS |
| Numerique.gouv — RGAA | B | Primaire | RSS |
| Euractiv — Tech | B | Média | RSS |
| ANSES — actualités nutrition | B / C | Primaire | Scraping HTML |
| Santé publique France — Nutrition | C | Primaire | RSS |
| PubMed — `Endometriosis AND (diet OR nutrition OR inflammation)` | C | Primaire | RSS (recherche enregistrée) |
| ScienceDaily — Nutrition / Santé des femmes | C | Média | RSS |
| GitHub — Qwen3-VL (commits `main`) | A | Primaire | Atom |
| GitHub — SDK Python Mistral (releases) | A | Primaire | Atom |
| Mistral — changelog officiel de l'API | A | Primaire | Scraping HTML |
| GitHub — n8n, ChromaDB, LangChain (releases) | A | Primaire | Atom |
| Hugging Face — blog | A | Primaire | RSS |
| Siècle Digital — IA, MarkTechPost — open source | A | Média | RSS |

Les sources **primaires** (institutions, dépôts officiels) peuvent atteindre la pertinence 3. Les sources **média** servent à repérer une information et sont plafonnées à 2, sauf si elles rapportent une décision applicable. Dans ce cas, l'item est marqué « à vérifier ». La liste se modifie dans le nœud `Flux a surveiller`.

## Installation

Prérequis : Docker et Docker Compose, plus une clé API Google Gemini (offre gratuite suffisante pour un appel par semaine).

```bash
git clone https://github.com/SalomeSouque/alimendo_veille_N8N.git
cd alimendo_veille_N8N
docker compose up -d
```

n8n est alors accessible sur http://localhost:5678. Au premier lancement :

1. Créer le compte propriétaire local.
2. **Workflows → Import from File** → `alimendo_veille_n8n.json`.
3. **Credentials → New → Google Gemini (PaLM) API**, coller la clé, puis l'associer au nœud `Google Gemini`.

La clé n'est jamais versionnée : elle reste dans le volume Docker `n8n_data`.

Pour arrêter : `docker compose down`. Le volume, les identifiants et l'historique des exécutions sont conservés.

## Lancer une veille

1. `docker compose up -d`, puis ouvrir le workflow.
2. Si la dernière veille date de plus de 7 jours, ajuster `FENETRE_JOURS` dans le nœud `Filtrer 7 jours + corpus` (par exemple 14 après une semaine en entreprise).
3. **Execute workflow**.
4. Télécharger le fichier produit par le nœud `Fichier HTML` (`veille-AAAA-MM-JJ.html`).
5. Relire la synthèse : vérifier les items « à vérifier », corriger si besoin.
6. Déposer le fichier dans `veille_genere/` et le commiter :
   ```bash
   git add veille_genere/veille-AAAA-MM-JJ.html JOURNAL_DIFFUSION.md
   git commit -m "docs: ajout de la veille html pour la semaine NN"
   git push
   ```
7. Ajouter une ligne dans [`JOURNAL_DIFFUSION.md`](JOURNAL_DIFFUSION.md).

## Rythme de veille

Le projet est mené **en alternance** : les semaines en formation sont consacrées au projet, les semaines en entreprise à l'employeur.

- **Semaine en formation** : une session de veille d'environ 1 h, en début de semaine (exécution, lecture, relecture, publication).
- **Semaine en entreprise** : session si le temps le permet. Sinon, la veille suivante couvre les deux semaines (`FENETRE_JOURS = 14`), donc aucun article n'est perdu.

Le déclenchement est **manuel** et non planifié. n8n tourne en local dans Docker sur un ordinateur personnel, qui n'est pas allumé en permanence. Or un déclencheur planifié ne rattrape pas une exécution manquée pendant que la machine est éteinte. La régularité repose donc sur un créneau réservé dans l'agenda, et chaque session est tracée par un commit daté et une ligne dans le journal de diffusion.

## Format et diffusion des synthèses

- **Format** : une page HTML par veille, structurée par axe (résumé de la période, items avec source, date, pertinence, impact projet, lien vers la source, liste des articles écartés). La page suit les recommandations d'accessibilité courantes : langue déclarée, hiérarchie de titres, liens explicites, contrastes, pas d'information portée par la couleur seule.
- **Diffusion** : les synthèses sont publiées dans ce dépôt public (`veille_genere/`). Chaque publication est consignée dans [`JOURNAL_DIFFUSION.md`](JOURNAL_DIFFUSION.md), avec la date, le fichier, les destinataires et les décisions prises pour le projet.

## Choix des outils

| Besoin | Outil retenu | Raison |
|--------|--------------|--------|
| Agrégation des flux | **n8n** auto-hébergé (Docker) | Gratuit, gère RSS, scraping HTTP et appel LLM dans un même workflow versionnable en JSON. |
| Tri et synthèse | **Gemini** (offre gratuite) | Un seul appel par veille, coût nul, sortie JSON contrainte par le prompt. |
| Partage | **GitHub** (dépôt public) + HTML | Gratuit, historique daté, accessible sans compte. |

Budget : 0 €.

## Limites connues

- **Scraping fragile** (ANSES, Mistral) : si la structure HTML du site change, la branche renvoie 0 article sans bloquer la veille. À contrôler quand une source primaire reste silencieuse plusieurs semaines.
- **Flux GitHub** : le dépôt Qwen3-VL publie peu de releases. On suit donc ses commits, ce qui peut produire du bruit, filtré par le prompt.
- **Erreurs de flux** : le nœud RSS continue en cas d'erreur sur un flux (par exemple une URL morte). Une source absente plusieurs semaines de suite doit être vérifiée à la main.
- **Le modèle peut se tromper** : la relecture humaine est obligatoire avant publication.

## Historique des modifications

| Date | Modification |
|------|--------------|
| 2026-08-13 | Création du dépôt, premier workflow et docker-compose |
| 2026-08-21 | Ajout de sources (HAS, CEPD, RGAA, ScienceDaily, GitHub…) et refonte du prompt en 3 axes |
| 2026-10 | Remplacement de Qwen2.5-VL par Qwen3-VL, ajout de Mistral (SDK + changelog), fenêtre de collecte paramétrable, tolérance aux flux en erreur, README et journal de diffusion |
