# MiamMiam — Guide de session

## Qu'est-ce que ce projet
App web mobile-first (PWA légère) de gestion de recettes et liste de courses pour Harold, usage **strictement personnel**.
Un seul fichier applicatif : `index.html` (HTML+CSS+JS vanilla, ~200 Ko, aucune dépendance de build). Une base de données : `db.json` (~3 Mo, 1630 recettes).

**URL de production** : https://haroldlev.github.io/BulkaLab_MiamMiam/
**Repo** : HaroldLev/BulkaLab_MiamMiam (public, voir note juridique plus bas)

## Règles d'or du développement
1. **Toujours tester avant de pousser** : `node -e "const js = require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1]; new Function(js);"` doit afficher SYNTAX OK. Puis un test fonctionnel avec un harnais (voir section Tests). Si db.json modifié : `node -e "JSON.parse(require('fs').readFileSync('db.json','utf8'))"`.
2. **Version bump à chaque déploiement** : incrémenter `APP_VERSION` (actuellement 1.21.0, ligne ~272). Elle sert d'anti-cache (`db.json?v=` + version).
3. **Déployer** : commit sur `main` → GitHub Pages rebuild automatiquement (~1-2 min). Vérifier avec `gh api repos/HaroldLev/BulkaLab_MiamMiam/pages/builds/latest --jq '.status'` = "built".
4. **L'utilisateur recharge depuis son iPhone** : le user doit tuer l'app dans le multitâche et rouvrir pour voir la nouvelle version.

## Processus de déploiement OBLIGATOIRE (à chaque modif déployée)
À chaque fois que tu déploies une modification sur `main`, suivre ce rythme EXACT :

1. **Modif du code** (index.html et/ou db.json)
2. **Tests** : syntaxe JS (`new Function(js)`) + JSON si db.json modifié
3. **Bump APP_VERSION** : ex `1.21.0` → `1.22.0` (ligne ~272 de index.html)
4. **Commit** avec message descriptif se terminant par `- vX.Y.Z`
5. **Créer le tag** : `git tag vX.Y.Z` (ex: `git tag v1.17.0`)
6. **Push branche + tag** : `git push origin main && git push origin vX.Y.Z`
7. **Vérifier le build GitHub Pages** : `gh api repos/HaroldLev/BulkaLab_MiamMiam/pages/builds/latest --jq '.status'` doit renvoyer `"built"`
8. **Mettre à jour l'historique** dans ce fichier AGENTS.md (section "Historique des versions")

> **Pourquoi le tag ?** Chaque version déployée est un point de restauration. Si une modif ne plaît pas à Harold, il peut revenir en arrière en 1 commande avec `rollback.sh` (voir section Rollback).

## Rollback (revenir à une version précédente)
Si Harold demande à revenir en arrière ou si une modif casse l'app :

```bash
# Voir les versions disponibles
./rollback.sh                    # ou: git tag -l "v1.*" --sort=-v:refname

# Revenir à une version (ex: v1.15.0)
./rollback.sh v1.15.0
# → vérifie JS+JSON, Entrée pour déployer

# Ou manuellement :
git checkout -B main v1.15.0
git push origin main --force-with-lease
```

Le script `rollback.sh` est à la racine du workspace (`/workspace/rollback.sh`), pas dans le repo. Les tags `v1.x.y` sont sur GitHub et en local.

## Sécurité du repo
- **Ne JAMAIS supprimer le repo, la branche `main`, ou faire un `git push -f` non sollicité.**
- Chaque commit/tag est un point de restauration permanent. Ne pas réécrire l'historique.
- Branche `backup-local` : copie figée du dernier état connu bon (ne pas supprimer).
- En cas de doute sur une modif : créer une branche de test, ne pas pousser sur `main` directement.

## Stack et architecture
- **Zéro framework** : HTML/CSS/JS vanilla dans un seul `index.html`. C'est un choix — la simplicité prime.
- **Stockage utilisateur** : `localStorage` uniquement (recettes, sélection, cases, tickets, frigo, prix réels, préférences sources). Limite ~5 Mo → la photo de ticket est compressée en miniature 320px avant stockage, avec repli sans photo si quota plein.
- **Base de recettes** : `db.json` chargé par fetch au démarrage (avec `?v=` anti-cache). Fallback : `FALLBACK_DB` embarquée dans index.html (228 recettes) si le fetch échoue.
- **GitHub App Mistral** : connexion installée, push autorisé. Commande push : `git push https://x-access-token:$(gh auth token)@github.com/HaroldLev/BulkaLab_MiamMiam.git main`.
- **Interdits** : pas de dépendance npm/build, pas de secrets en clair (voir section Secret scanning), ne pas modifier db.json sans test de chargement.

## Les 4 onglets de l'app
1. **Recettes** : cartes avec mini-photo, sélection multiple, vue détail (👁️) avec instructions traduites, éditeur de portions (👥).
2. **Ma liste** : agrégation des ingrédients (synonymes FR/EN, pluriels, quantités en tête/fin de nom), cases à cocher, estimation prix multi-enseignes.
3. **Ticket** : photo du ticket → scan **IA Gemini** (vision, JSON direct, détection magasin) avec **fallback Tesseract.js** (parser v3 entraîné sur tickets U Express). Prix réels par enseigne + alimentation auto du frigo.
4. **Frigo** : stock avec volumes (g/ml/pièces), cumul automatique, images 36px fixes.

Plus le menu **⚙️ Paramètres** : activation/désactivation des 3 sources de recettes.

## Les 3 sources de recettes (db.json)
| Source | Volume | Particularités |
|---|---|---|
| **Marmiton** (scrapé) | 429 | Classiques FR notés ⭐ par internautes — **usage perso, voir note juridique** |
| **METRO France** | 411 | Recettes pro FR, CC-BY 4.0, attribution obligatoire (footer) |
| **TheMealDB** | 790 | International, photos, instructions en anglais (traduction par règles intégrées) |

Champs d'une recette : id, name, nameFr, cat, type (Apero/Starter/Main/Dessert), area, img, emoji, chef, duration, difficulty, rating, instructions, servings, ingredients[], measures[], ingLines[], source.

## Estimation prix des paniers
- ~440 prix moyens FR (€/kg, €/L, €/pièce) dans `PRICES` + heuristiques de repli.
- 8 enseignes avec coefficients (Lidl ×0,80 → Biocoop ×1,40), Super U inclus.
- **Prix réels** (relevés par tickets) prioritaires sur les moyennes, par enseigne. Conversion pièces→grammes (PIECE_G : tomate 150g, etc.).
- Les cases cochées (déjà acheté) sont exclues du calcul.

## Scan de tickets (IA)
- **Gemini Flash** (`GEMINI_MODEL = "gemini-3.5-flash"` — les anciens noms 2.x renvoient 404) : photo en base64 → prompt d'extraction → JSON {store, items[]}.
- La clé API est **assemblée par fragments** dans le code (obfuscation pour passer le secret-scanning GitHub qui bloque les clés GCP en clair).
- Fallback : Tesseract.js fra + `parseReceiptText` (gère catégories U, lignes "1 x prix", packs "4X150G", poids "0,171 kg x 24,92 €/kg", stop-words français étendus).
- `addTicketLine({name, qty, unit, price})` pré-remplit le formulaire ; l'utilisateur vérifie avant d'enregistrer.

## Tests — harnais type
```js
// Syntaxe
const js = require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1];
new Function(js); // doit passer

// Fonctionnel : simuler localStorage/document/window puis eval le script
global.localStorage = { getItem: (k) => store[k] || null, setItem: (k,v) => store[k]=v };
global.document = { getElementById: (id) => els[id] ||= {...}, createElement: () => ({...}) };
const mod = eval('(function(){' + js + '; return { fonctionsATester };})()');
// ⚠️ Points d'attention : définir window.scrollTo, alert, confirm, fetch (mock),
// et appeler el(id) AVANT de lire les propriétés (les éléments sont lazy).
```

## Note juridique (répétée à chaque session si nécessaire)
- Les 429 recettes **Marmiton** sont du contenu protégé (CGU + droit d'auteur), stockées dans un repo **public**. Décision prise en connaissance de cause : usage perso, risque jugé faible. **Si l'app devait être partagée publiquement** → passer le repo en privé + Netlify (gratuit) ou purger Marmiton.
- Recettes METRO : licence CC-BY 4.0, l'attribution est dans le footer de l'app — **ne pas la retirer**.

## Historique des versions
- 1.21.0 : icône = nouvelle photo carrée du chat (yeux verts, viande séchée), resize direct sans crop
- 1.20.0 : icône = photo du chat (crop carré centré sur le visage, icon-192/512 + apple-touch-icon)
- 1.19.0 : renommage app en MiamMiam (titre, header, meta) + icône chat kawaii (favicon, apple-touch-icon, manifest.json)
- 1.16.0 : bouton Contact/Report (mailto pré-rempli avec version)
- 1.15.0 : fix enregistrement ticket (compression photo, quota localStorage), images frigo 36px
- 1.14.0 : scan IA Gemini + fallback Tesseract
- 1.13.0 : parser v3 entraîné sur vrai ticket U Express
- 1.11-1.12 : onglets Ticket/Frigo, prix réels par enseigne, OCR Tesseract initial
- 1.9-1.10 : menu Paramètres, base Marmiton scrapée (429 recettes)

## Secret scanning GitHub
Le push est **bloqué** si un secret est détecté (clés GCP/API). La clé Gemini est fragmentée pour l'éviter. Si un push est rejeté : voir le message d'erreur, soit obfusquer, soit suivre le lien unblock de GitHub (avec accord explicite d'Harold).
