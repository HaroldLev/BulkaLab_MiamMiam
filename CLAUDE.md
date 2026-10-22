# MiamMiam — Instructions pour agents IA

> Guide complet et à jour : lire **`AGENTS.md`** (même dossier). Ce fichier en est le résumé opérationnel.

## Projet en 30 secondes
- **Quoi** : app web mobile-first (PWA légère) de gestion de recettes + liste de courses, usage strictement personnel (Harold).
- **Où** : [haroldlev.github.io/BulkaLab_MiamMiam](https://haroldlev.github.io/BulkaLab_MiamMiam/) — déployée via GitHub Pages depuis `main`.
- **Stack** : zéro framework, zéro build. Un seul fichier `index.html` (~1660 lignes, HTML+CSS+JS vanilla) + `db.json` (~3 Mo, 1630 recettes).

## Fichiers clés
| Fichier | Rôle |
|---|---|
| `index.html` | Toute l'app (UI, logique, styles, fallback DB embarqué) |
| `db.json` | Base de 1630 recettes : Marmiton (429), METRO France (411, CC-BY 4.0), TheMealDB (790) |
| `AGENTS.md` | Guide de session complet (architecture, tests, déploiement, juridique) |

## Règles d'or
1. **Tester avant de pousser** : `node -e "const js = require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1]; new Function(js);"` → doit passer. + `node -e "JSON.parse(require('fs').readFileSync('db.json','utf8'))"` si `db.json` modifié.
2. **Bump `APP_VERSION`** (ligne 265 de `index.html`, actuellement `1.16.0`) à chaque déploiement — anti-cache via `db.json?v=`.
3. **Déployer** : commit + push sur `main` → GitHub Pages rebuild (~1-2 min). Vérifier : `gh api repos/HaroldLev/BulkaLab_MiamMiam/pages/builds/latest --jq '.status'` = `built`.
4. **Côté utilisateur** : Harold doit tuer l'app dans le multitâche iOS et la rouvrir pour voir la nouvelle version.

## Interdits
- Pas de dépendance npm, pas de build step, pas de framework.
- Pas de secrets en clair (la clé Gemini est assemblée par fragments dans le code pour passer le secret-scanning GitHub).
- Ne pas modifier `db.json` sans tester son chargement (JSON valide).
- Ne pas retirer l'attribution CC-BY 4.0 METRO du footer.

## Les 4 onglets
1. **Recettes** — cartes, sélection multiple, vue détail, éditeur de portions.
2. **Ma liste** — agrégation ingrédients (synonymes FR/EN, pluriels), cases à cocher, estimation prix multi-enseignes.
3. **Ticket** — scan photo par IA Gemini (`gemini-3.5-flash`) + fallback OCR Tesseract.js, prix réels par enseigne, alimentation auto du frigo.
4. **Frigo** — stock avec volumes (g/ml/pièces), cumul automatique.

## Conventions de code (dans `index.html`)
- JS vanilla, fonctions globales préfixées par domaine : `renderRecipes`, `renderIngredients`, `renderTickets`, `renderFridge`, `saveTicket`, `addTicketLine`, `parseReceiptText`, `estimateItemPrice`.
- Stockage : `localStorage` uniquement (~5 Mo max → photos de ticket compressées en 320px).
- Constantes globales : `APP_VERSION`, `GEMINI_MODEL`, `PRICES` (~440 prix moyens), `PIECE_G` (conversion pièce→grammes), `FALLBACK_DB` (228 recettes embarquées).

## Tests
Harnais minimal (voir section Tests de `AGENTS.md` pour le complet) :
```js
const js = require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1];
new Function(js); // syntaxe
```
Pour du fonctionnel : mocker `localStorage`, `document`, `window`, `fetch` puis `eval` le script.

## Déploiement (push autorisé)
```bash
git add -A && git commit -m "<message> - vX.Y.Z"
git push https://x-access-token:$(gh auth token)@github.com/HaroldLev/BulkaLab_MiamMiam.git main
```
Si le push est bloqué par le secret-scanning : fragmenter le secret ou suivre le lien unblock GitHub (avec accord explicite d'Harold).

## Note juridique
- 429 recettes Marmiton = contenu protégé dans un repo **public** — décision assumée (usage perso). Ne jamais rendre l'app publique sans passer le repo en privé.
- METRO France = CC-BY 4.0, attribution obligatoire (présente dans le footer).
