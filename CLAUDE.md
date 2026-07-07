# Suivi implémentation IA (plateforme web Triactis)

Application web mono-page qui donne à l'équipe Triactis une vue partagée et temps réel
de l'avancement du chantier d'implémentation de l'IA (été 2026). Feuille de route :
tâches, jalons, événements, avancement par projet.

## Source de vérité et déploiement (à lire en premier)

Dépôt canonique (décision du 2026-07-07) : **`FiExplorer11020/triactis-suivi-implementation-ia`,
branche `main`**. C'est le remote `origin` de ce dossier. On développe et on pousse ici.
L'historique complet de l'app (12 commits) y a été migré depuis l'ancien dépôt, plus la
page « Aujourd'hui ».

Ancien dépôt `FiExplorer11020/triactis-suivi-ia` (branche `main`) : figé, à ne plus modifier.
C'est encore lui qui sert la page publique https://fiexplorer11020.github.io/triactis-suivi-ia/
**tant que GitHub Pages n'a pas été rebasculé** sur le nouveau dépôt (à faire dans
Settings > Pages du nouveau repo, source = branche `main`). Le lien public changera alors
pour https://fiexplorer11020.github.io/triactis-suivi-implementation-ia/ (à rediffuser à
l'équipe, avec le code d'accès inchangé).

## Architecture technique

- **Front** : un seul fichier HTML autonome, `index.html` (vanilla JS + SVG, aucun build,
  aucune dépendance npm). `triactis-roadmap.html` en est une **copie strictement identique**
  (ancien point d'entrée). Toute modification de `index.html` doit être recopiée à
  l'identique dans `triactis-roadmap.html` (`cp index.html triactis-roadmap.html`).
  `.nojekyll` désactive le traitement Jekyll de GitHub Pages.
- **Données partagées** : Firebase Firestore, projet `triactis-suivi-ia` (Europe eur3,
  forfait Spark). Document `roadmaps/roadmap-<hash(codeAccès)>`. Synchro temps réel
  (`onSnapshot`), sauvegarde debouncée via `persist()`. Sans config Firebase, l'app
  bascule en mode local (localStorage) et démarre sur le jeu de données SEED.
- **Accès** : écran de code partagé (`gate()`), code d'équipe hors dépôt. En dev, le code
  n'est pas nécessaire : ouvrir la console et appeler `start()` après avoir retiré
  `#gateScrim` suffit pour bypasser l'écran en local.

## Modèle de données (événement)

```
{ id, title, type: 'tache'|'jalon'|'evenement', start, end, progress (0..100),
  tags[], deps[] (titres des prérequis), note, parent (id), status, createdAt, updatedAt }
```

- `updatedAt` est horodaté à chaque changement de progression, édition ou import (mais
  **pas** sur le recalcul de progression d'un parent). C'est un « dernier touché », pas un
  historique complet. Les éléments SEED n'ont pas d'`updatedAt` tant qu'on ne les a pas
  modifiés.
- Un parent porte des sous-tâches (`parent`), sa progression est un roll-up
  (`effProgress`, `recomputeParents`).

## Structure du code (index.html)

- État global : `const state = {...}` (vue active, filtres, recherche, collapsed, mode…).
- Rendu : `renderAll()` -> `renderOverview()`, `renderFilterBar()`, `renderView()`.
  `renderView()` dispatche sur `state.view` vers `renderToday / renderFocus / renderProjets /
  renderKanban / renderGantt / renderBilan`. Chaque vue pose `#view.innerHTML` puis appelle
  `wireRows()` / `wireSections()`.
- Helpers réutilisables : `projectOf/projColor`, `statusKey/statusOf`, `isDone/depState/
  focusBucket`, `childrenOf/effProgress`, `rowHTML(e)`, `wireRows(root)`, `visibleEvents()`,
  `openEvent(id)`, `persist()`, `showToast()`, `esc()`, `fmtDate/parseD/toISO`, `hexA()`.
- Tokens CSS dans `:root` : `--p1/--p2/--p3` (couleurs projets), `--gold*`, `--ok/--warn/--bad`,
  `--radius`, etc. Réutiliser ces variables plutôt que des couleurs en dur.
- Hooks navigateur pour l'alimentation par les skills : `window.coworkSnapshot()` (lecture),
  `window.coworkApplyPatch(json)` (upsert + écriture Firestore).

## Vues

- **Aujourd'hui** (`renderToday`, vue d'entrée par défaut) : nudge à l'action. Une seule
  « prochaine action » mise en avant, momentum du jour (fait/avancé via `updatedAt`),
  « à enchaîner » priorisé sur les projets déjà touchés puis proches de 100 %, bloc
  « à surveiller » (bloqués + en retard). Bascule Aujourd'hui / 7 jours (`state.tdRange`).
- **Focus** : Now / Next / Later dépendance-aware (`focusBucket`, `depState`).
- **Projets** : projet -> tâche -> sous-tâche, sections repliables, roll-up.
- **Kanban** : colonnes par statut, swimlanes par projet.
- **Gantt** : au jour près, jalons en losange, zoom trimestre/mois/semaine/jour.
- **Bilan** : anneau de complétion par mois, répartition par projet.

## Conventions

- Français professionnel, typographie française (espace insécable avant `: ; ! ?`,
  guillemets « », virgule décimale). **Aucun tiret cadratin** dans le texte destiné à
  l'utilisateur : utiliser `:`, `,`, `(...)`.
- Espace avant `%` (`45 %`), cohérent avec l'existant.

## Lancer en local

Serveur statique (config dans `.claude/launch.json`) :

```
python -m http.server 8137 --directory .
# puis http://localhost:8137/index.html
```

Aucun test automatisé dans ce dépôt : la vérification se fait en pilotant la page dans le
navigateur (rendu, interactions, absence d'erreur console).
