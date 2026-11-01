# Assistant « Quel test choisir ? » — livraison

**Date** : 31 octobre 2026 · **Module** : arbre de décision statistique
**Fichiers** : `statflow/decision.py`, `static/decision.js`,
`tests/test_decision.py`, `tests/smoke_decision.py`,
`validation/decision_*.png`, `API.md`, `README.md`,
`validation/VALIDATION.md`, `static/aide.js`, `static/app.js`,
`static/index.html`, `static/styles.css`.

---

## 1. Le problème visé

Un utilisateur non statisticien se trouve devant trois familles de décisions qu'il
ne maîtrise pas :

1. **Quel test ?** — t de Student, Wilcoxon, ANOVA, Kruskal–Wallis, khi², Fisher,
   McNemar, kappa, ICC, log-rank… le nom du test ne dit pas quand l'utiliser.
2. **Sous quelles conditions ?** — normalité, homogénéité des variances,
   effectifs attendus : trois vérifications qui décident souvent du test.
3. **Et ensuite ?** — une fois le test choisi, il faut retrouver le module qui le
   calcule et régler ses paramètres.

L'assistant répond aux trois en un seul écran : il **mesure** les conditions sur
les données du projet, applique un arbre explicite, explique le résultat attendu,
puis **ouvre le module compétent avec la méthode présélectionnée**.

## 2. Principe de conception

| Choix | Raison |
|---|---|
| Les conditions sont **calculées**, jamais déclarées | Un utilisateur qui ne connaît pas la normalité ne peut pas la déclarer. Shapiro–Wilk, Levene et les effectifs attendus sont mesurés sur la population analysée (filtres appliqués). |
| Une condition inconnue n'est **jamais** « respectée » | Variable absente, effectif < 3, valeurs constantes → statut `indeterminee`. L'arbre retient alors la solution la plus prudente et la confiance affichée baisse. |
| L'arbre est **explicite et tracé** | La réponse contient `trace`, la suite des étapes réellement parcourues, affichée à l'écran et versée au rapport si on le souhaite. Un statisticien peut refaire le raisonnement. |
| L'assistant **ne calcule aucun test** | Aucune duplication de la logique statistique : chaque test reste dans son module, avec ses tests unitaires et sa recette déjà validés. |
| Le module d'accueil est **présélectionné** | `bivariate` (`student`, `welch`, `mannwhitney`, `anova`, `kruskal`, `chi2`, `fisher`, `montecarlo`, `pearson`, `spearman`), `paired`, `reliability`, `survival`, `diagnostic`, `descriptive`. |
| Aucune dépendance, aucun appel externe | `numpy`, `pandas`, `scipy.stats` uniquement, déjà requis. |

## 3. Ce que le module mesure

| Condition | Méthode | Règle appliquée | Statuts |
|---|---|---|---|
| Normalité | Shapiro–Wilk, par groupe (ou sur les **différences** si mesures appariées) | une seule p < alpha suffit à retenir une solution non paramétrique | `respectee`, `non_respectee`, `indeterminee` |
| Homogénéité des variances | Levene centré sur la médiane (Brown–Forsythe) | p ≥ alpha ⇒ variances comparables | `respectee`, `non_respectee`, `indeterminee` |
| Effectifs | effectifs attendus du khi², ou comptage par groupe | attendu minimal ≥ 5 ; entre 1 et 5 ⇒ Monte-Carlo ; < 1 ⇒ exact/regroupement | `suffisants`, `a_surveiller`, `insuffisants` |
| Effectif par groupe (moyennes) | comptage | ≥ 30 par groupe ⇒ approximation normale solide | idem |

Détails techniques : Shapiro–Wilk est limité à 5 000 valeurs (échantillonnage
déterministe au-delà, graine fixe), les groupes de moins de 2 valeurs sont
signalés dans `groupes_ignores`, une variable constante renvoie
`indeterminee` avec une note explicite.

## 4. L'arbre de décision

Sept objectifs : **décrire**, **comparer des groupes**, **étudier un lien**,
**mesures répétées**, **mesurer un accord**, **évaluer un test diagnostique**,
**analyser un délai**. Quatre natures : quantitative, qualitative nominale,
ordinale, durée censurée. Deux à trois groupes, appariement déclaré (imposé pour
les mesures répétées).

| Objectif | Nature | Appariement | Conditions | Test proposé | Module / méthode |
|---|---|---|---|---|---|
| comparer | quantitative | indépendant | normalité + homogénéité | t de Student | `bivariate` / `student` |
| comparer | quantitative | indépendant | normalité, variances inégales | t de Welch | `bivariate` / `welch` |
| comparer | quantitative | indépendant | normalité non tenue | Mann–Whitney | `bivariate` / `mannwhitney` |
| comparer | quantitative | indépendant | 3 groupes et +, conditions tenues | ANOVA à un facteur | `bivariate` / `anova` |
| comparer | quantitative | indépendant | 3 groupes et +, conditions non tenues | Kruskal–Wallis | `bivariate` / `kruskal` |
| comparer | qualitative | indépendant | effectifs suffisants | khi² de Pearson | `bivariate` / `chi2` |
| comparer | qualitative | indépendant | effectifs faibles | Fisher exact | `bivariate` / `fisher` |
| repetees | quantitative | apparié | différences normales | t apparié | `paired` / `deux_quantitatives` |
| repetees | quantitative | apparié | différences non normales | Wilcoxon signé-rang | `paired` / `deux_quantitatives` |
| repetees | qualitative | apparié | deux mesures binaires | McNemar | `paired` / `deux_binaires` |
| repetees | quantitative | apparié | 3 mesures et + | ANOVA à mesures répétées / Friedman | `paired` / `format_long` |
| relation | quantitative | indépendant | normalité | Pearson | `bivariate` / `pearson` |
| relation | quantitative | indépendant | normalité non tenue | Spearman | `bivariate` / `spearman` |
| relation | qualitative | indépendant | effectifs | khi² / Fisher | `bivariate` / `chi2`, `fisher` |
| relation | ordinale | indépendant | — | Spearman | `bivariate` / `spearman` |
| accord | qualitative | apparié | 2 juges | kappa de Cohen | `reliability` / `kappa` |
| accord | ordinale | apparié | 2 juges | kappa pondéré | `reliability` / `kappa` |
| accord | qualitative | apparié | 3 juges et + | kappa de Fleiss | `reliability` / `fleiss` |
| accord | quantitative | apparié | 2 mesures continues | ICC(2,1) | `reliability` / `icc` |
| accord | quantitative | apparié | plusieurs items | alpha de Cronbach | `reliability` / `alpha` |
| diagnostic | qualitative | — | — | Se, Sp, VPP/VPN, ROC | `diagnostic` |
| survie | durée | — | — | Kaplan–Meier + log-rank | `survival` / `km` |
| relation | durée | — | covariables | modèle de Cox | `survival` / `cox` |
| decrire | quantitative | — | — | aucun test | `descriptive` |

Chaque recommandation renvoie : `test`, `famille`, `page`, `methode`, `mode`,
`parametres`, `trace`, `pourquoi`, `interpretation`, `tailles_effet`,
`rapporter`, `alternatives`, `reserves`, `confiance`, `confusion` (ce que la
p-value ne dit pas) et `libelles_conditions`.

## 5. Interface

Page **Quel test choisir ?** (`#decision`), entrée de navigation placée **avant
les modules d'analyse** avec l'étiquette `ASSISTANT`.

* **Colonne gauche** — questionnaire : sept cartes d'objectif, nature de la
  variable, nombre de groupes, appariement (désactivé et imposé pour les mesures
  répétées), sélection des variables réelles du projet, risque alpha.
* **Colonne droite** — recommandation : famille, test, badge de confiance,
  bouton **« Lancer l'analyse dans … »**, trace du chemin, conditions avec
  pastilles de statut et règle appliquée sous chaque ligne, tableau détaillé de
  normalité par groupe (n, W, p, asymétrie), « Pourquoi ce test », « Comment lire
  le résultat », taille d'effet, points de vigilance, « À rapporter dans le
  document », tableau « Si les conditions changent ».
* La recommandation est **conservée dans le projet** (`results.decision`) : elle
  réapparaît à l'ouverture du module et suit la révision des données.
* Deux boutons « Ajouter au rapport » : résumé de décision (calcul) et tableau
  des conditions. Le tableau de normalité détaillé reste hors du rapport.
* Sans projet importé, la page explique quoi faire au lieu de rester vide.

### Lancement de l'analyse

Le bouton présélectionne, quand le module le permet :

| Page | Ce qui est préparé |
|---|---|
| Analyse bivariée | `Y` = variable quantitative, `X` = variable de groupe (ou seconde variable), `méthode` = test proposé |
| Tests appariés | `mode` (`deux_quantitatives` / `deux_binaires` / `format_long`), mesure 1 et mesure 2 |
| Reproductibilité | `mode` (`kappa`, `fleiss`, `icc`, `alpha`), juges ou mesures |
| Survie, exactitude diagnostique, descriptive | navigation directe, choix guidé par la carte de recommandation |

Le préremplissage passe par `window.statflowPairedSet` et
`window.statflowReliabilitySet`, deux points d'entrée ajoutés aux modules
existants sans modifier leur logique.

## 6. Preuves

### 6.1 Tests unitaires et d'interface — 19 tests

```
python3 -m pytest tests/test_decision.py -q
19 passed
```

| Bloc | Ce qui est vérifié |
|---|---|
| Conditions | normalité et homogénéité calculées, asymétrie et hétérogénéité détectées, différences appariées, effet d’un effectif minuscule (`indeterminee`), effectifs attendus du khi², statuts du tableau de synthèse |
| Arbre | t/Student, Welch, Mann–Whitney, ANOVA, Kruskal–Wallis, khi², Fisher, t apparié, Wilcoxon, McNemar, mesures répétées, Pearson, Spearman, kappa (2 juges, pondéré, Fleiss), ICC, alpha, diagnostic, Kaplan–Meier, Cox, description, confiance et trace |
| Refus | objectif, nature, nombre de groupes, appariement et alpha invalides (`422`) |
| HTTP | conditions réellement mesurées sur un CSV importé, persistance dans `results.decision`, journal, erreurs `422` explicites, questionnaire `GET /api/decision/arbre` |
| Cohérence | `static/decision.js`, entrée de navigation, script chargé, routeur, chapitre d'aide (Shapiro–Wilk cité), `API.md` |

### 6.2 Recette navigateur — 12 étapes

```
python3 tests/smoke_decision.py
Assistant « Quel test choisir ? » : 12 étapes validées (navigation, questionnaire,
conditions mesurées sur les données, recommandation paramétrique puis non
paramétrique, lancement depuis l’assistant vers Analyse bivariée et Tests appariés
avec méthode présélectionnée, accord entre juges, rapport et journal, mobile
390 px), aucune erreur JavaScript.
```

Valeurs relevées sur le jeu de recette (150 patients, 8 colonnes) :

| Situation | Conditions mesurées | Recommandation |
|---|---|---|
| `score_qualite` par groupe (3 groupes) | Shapiro p = 0,6957 / 0,8433 / 0,6940 · Levene p = 0,6579 · effectifs suffisants | **ANOVA à un facteur**, confiance élevée |
| `delai_attente` par groupe | Shapiro p < 0,0001 dans les trois groupes · Levene p = 0,1363 | **Kruskal–Wallis**, confiance élevée |
| `avant` / `apres` appariés | normalité des **différences** respectée | **t de Student apparié** |
| `juge1` / `juge2` | effectifs attendus suffisants | **kappa de Cohen** |

Contrôles complémentaires : `#biv-method` vaut `kruskal` et `#biv-y` vaut
`delai_attente` après le clic « Lancer l'analyse » ; le module apparié reçoit
`avant` / `apres` ; `results.decision` et le journal « Quel test choisir »
contiennent la recommandation ; mobile 390 px sans débordement
(`scrollWidth = 390`) ; aucune ressource externe.

Captures : `validation/decision_conditions.png` (1440 px, recommandation
paramétrique, conditions détaillées), `validation/decision_non_parametrique.png`
(1440 px, bascule non paramétrique et normalité non respectée),
`validation/decision_mobile.png` (390 px).

### 6.3 Suite complète

```
python3 -m pytest -q
621 passed, 13 warnings in 123.62s
```

## 7. Aide, documentation et journal

* `static/aide.js` : chapitre **« Quel test choisir ? »** (30 chapitres au
  total) — objectif, gestes, lecture de la trace, solutions de repli, limites.
* `API.md` : contrats `GET /api/decision/arbre` et
  `POST /api/projects/{id}/decision`, tableau complet de l'arbre, statuts,
  erreurs `422`.
* `README.md`, `validation/VALIDATION.md` (§17), `LISEZ_MOI_FINAL.txt`.
* Journal du projet : événement `Quel test choisir` avec le test, la famille, la
  confiance et les variables utilisées.

## 8. Limites assumées

- L'arbre couvre les méthodes **présentes dans StatFlow** : pas de régression
  logistique proposée comme alternative d'un khi², pas de modèle mixte, pas de
  tests de non-infériorité. Une situation hors cadre renvoie une erreur explicite
  plutôt qu'une recommandation inventée.
- La vérification porte sur la **population analysée** (filtres appliqués) ; elle
  ne remplace pas un examen graphique (histogrammes, boîtes à moustaches), que
  les modules descriptifs fournissent.
- Shapiro–Wilk est très sensible à l'écart à la normalité sur les grands
  effectifs ; l'assistant affiche la p-value et l'asymétrie, et rappelle dans ses
  réserves que l'ANOVA reste robuste au-delà de 30 par groupe.
- Un choix de test ne remplace jamais un **plan d'analyse écrit** : l'assistant
  documente le raisonnement, il ne légitime pas une exploration opportuniste.
- Le module ne produit aucune p-value : il n'ajoute donc aucun risque de
  multiplication des tests.
