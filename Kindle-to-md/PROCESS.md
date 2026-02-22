# Kindle-to-MD — Process Documentation

## Overview

Pipeline automatisé qui convertit des livres (PDF ou Markdown) en Markdown structuré, distille chaque chapitre à travers 3 lenses analytiques, et synthétise des insights thématiques — le tout propulsé par Claude.

**Repo** : [yasser-ensembl3/Kindle-to-pdf](https://github.com/yasser-ensembl3/Kindle-to-pdf)

---

## Stack technique

| Composant | Technologie |
|-----------|-------------|
| Langage | Python 3.10+ |
| CLI | Typer + Rich |
| Extraction PDF | pdfplumber |
| Modèles de données | Pydantic v2 |
| Appels LLM | Claude Code CLI (`claude -p`) |
| Google Drive | google-api-python-client + OAuth2 |
| Parallélisme | concurrent.futures.ThreadPoolExecutor (10 workers) |

---

## Architecture

```
Kindle-to-md/
├── src/
│   ├── cli/commands.py            # Point d'entrée CLI — 5 commandes
│   ├── extractors/
│   │   ├── pdf.py                 # Extraction PDF (pdfplumber)
│   │   ├── markdown_parser.py     # Parser de .md (3 formats supportés)
│   │   └── chapter_detector.py    # Détection chapitres/parties (regex)
│   ├── converters/
│   │   └── book_markdown.py       # Book → Markdown Obsidian
│   ├── models/
│   │   └── book.py                # Modèles Pydantic (Book, Chapter, Part)
│   ├── prompts/
│   │   └── generator.py           # Génération de prompts depuis templates
│   ├── drive/
│   │   └── client.py              # Client Google Drive API v3
│   └── config.py                  # Chemins et config par défaut
├── templates/
│   ├── distillation_chapter.md    # Prompt 3-lenses par chapitre
│   ├── distillation_assembly.md   # Prompt d'assemblage
│   └── insights_synthesis.md      # Prompt de synthèse thématique
├── watch.sh                       # Watcher launchd pour inbox/
├── com.kindle2md.watcher.plist    # Config macOS launchd
├── pyproject.toml
├── requirements.txt
└── INSTALL.md
```

---

## Pipeline — Comment ça marche

### Étape 1 : Extraction

**Commande** : `kindle2md extract <fichier>`

- **PDF** : pdfplumber extrait le texte page par page → détecte et supprime les headers/footers récurrents (>30% de fréquence) → corrige les mots coupés par trait d'union en fin de ligne → détecte les chapitres/parties via regex
- **Markdown** : Le parser gère 3 formats :
  - Epub-converted : `C[HAPTER]{.small}`
  - Standard : `## Chapter N - Title`
  - Numéroté : `## 1. Title`
- Auto-détecte les parties, supprime le backmatter (notes, appendice, etc.)

**Output** : `book.md` (Markdown structuré avec frontmatter YAML + TOC Obsidian) + fichiers chapitres individuels dans `chapters/`

### Étape 2 : Distillation

**Commande** : `kindle2md distill <fichier>`

Chaque chapitre est envoyé à Claude avec un prompt structuré (template `distillation_chapter.md`). Analyse à travers 3 lenses :

1. **Phénoménologie** (4-6 bullets) — ce que ça *fait ressentir* de l'intérieur. Citations, métaphores, expériences vécues
2. **Deep Facts** (4-6 bullets) — insights de 3e à 5e niveau. Vérités contre-intuitives, mécanismes cachés
3. **Action Items** (3-5 checkboxes) — recommandations concrètes et actionnables

**Parallélisme** : 10 appels Claude simultanés. Les gros chapitres (>15 000 mots) sont découpés en chunks automatiquement.

**Assemblage** : L'assemblage final est fait localement (concaténation, pas d'appel LLM).

**Output** : `book_distillation.md`

### Étape 3 : Synthèse

**Commande** : `kindle2md synthesize <fichier>`

La distillation complète est envoyée à Claude pour réorganiser **par thèmes** (pas par chapitre). Produit 6-10 sections thématiques numérotées en chiffres romains, avec une synthèse finale de 4-6 phrases.

Si la distillation est trop grande (>15 000 mots), elle est découpée en chunks, synthétisée en parallèle, puis fusionnée.

**Output** : `book_insights.md`

### Pipeline complet

**Commande** : `kindle2md pipeline <fichier>` — exécute les 3 étapes d'affilée.

```
kindle2md pipeline book.pdf --model haiku
```

---

## Drive Sync

**Commande** : `kindle2md drive-sync <folder_url>`

Workflow :
1. Authentification OAuth2 Google (ouvre le navigateur au premier lancement, token caché ensuite)
2. Scan récursif du dossier Drive pour trouver les `.md`
3. Filtre les fichiers déjà générés (`_distillation.md`, `_insights.md`)
4. Pour chaque livre :
   - Télécharge le `.md`
   - Parse les chapitres (auto-détection du format)
   - Distille en parallèle (10 workers)
   - Synthétise les insights
   - Upload `_distillation.md` et `_insights.md` dans le même sous-dossier Drive

**Performance** : ~2.5 min pour un livre de ~60k mots / 20 chapitres.

---

## Installation & Setup

### Prérequis

- Python 3.10+
- Claude Code CLI installé et authentifié (`claude -p "test"` doit fonctionner)
- Google OAuth credentials (uniquement pour Drive sync)

### Installation

```bash
cd Kindle-to-md
python3 -m venv venv && source venv/bin/activate
pip install -e .
kindle2md --help
```

### Google Drive (optionnel)

1. Google Cloud Console → créer un projet
2. Activer Google Drive API
3. Créer des credentials OAuth 2.0 (Desktop ou Web)
4. Télécharger le JSON → sauvegarder comme `credentials.json` à la racine du projet
5. Si type Web : ajouter `http://localhost:8080/` dans les redirect URIs

### Watcher launchd (optionnel)

Surveille `inbox/` et lance le pipeline automatiquement sur les PDF déposés.

```bash
# Éditer les chemins dans le plist
nano com.kindle2md.watcher.plist

# Installer
cp com.kindle2md.watcher.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.kindle2md.watcher.plist
```

---

## Commandes de référence

| Commande | Description |
|----------|-------------|
| `kindle2md extract <pdf>` | Extraire PDF → book.md + chapitres |
| `kindle2md distill <pdf>` | Distiller chaque chapitre (3 lenses) |
| `kindle2md synthesize <pdf>` | Synthétiser par thèmes |
| `kindle2md pipeline <pdf>` | Les 3 étapes d'affilée |
| `kindle2md drive-sync <url>` | Sync depuis Google Drive |
| `kindle2md version` | Version de l'outil |

### Options communes

| Option | Description |
|--------|-------------|
| `--model, -m` | Modèle Claude : `haiku` (rapide), `sonnet` (équilibré), `opus` (meilleur) |
| `--title, -t` | Forcer le titre du livre |
| `--author, -a` | Forcer l'auteur |
| `--output-dir, -o` | Dossier de sortie custom |

---

## Modèles de données clés

### Book (Pydantic)

```python
Book
├── metadata: BookMetadata (title, author, published, tags, aliases, related)
├── introduction: str
└── parts: list[Part]
    └── chapters: list[Chapter]
        ├── number, title, text
        ├── part_number, part_title
        └── start_page, end_page
```

- Sérialisable en JSON (`book.to_json()` / `Book.from_json()`)
- Propriétés calculées : `all_chapters`, `total_chapters`, `total_words`

---

## Troubleshooting

| Problème | Solution |
|----------|----------|
| `claude -p` ne fonctionne pas | Vérifier que Claude Code CLI est installé et authentifié |
| Pas de chapitres détectés | Le format du livre n'est pas reconnu — vérifier les patterns dans `chapter_detector.py` |
| Drive auth échoue | Vérifier `credentials.json`, supprimer `token.json` et réessayer |
| Timeout sur gros livres | Augmenter le timeout dans `_call_claude()` (défaut : 300s) |
| Chapitre trop gros | Automatiquement chunké si >15 000 mots — pas d'action requise |
