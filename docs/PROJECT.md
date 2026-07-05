# Moteur de backtesting event-driven en C++ moderne

## Contexte

Ce projet vise à apprendre le C++ en construisant un outil réel plutôt qu'en
enchaînant des exercices isolés. Le domaine choisi est le backtesting de
stratégies de trading systématiques — un problème familier professionnellement,
ce qui permet de concentrer l'effort sur le langage plutôt que sur le métier.

## Ce qu'est un backtester

Un backtester rejoue une stratégie de trading sur des données de marché
historiques pour estimer comment elle se serait comportée en réalité.
En entrée : une série de prix (par exemple les barres OHLCV journalières
d'une action sur dix ans) et une logique de décision (par exemple *acheter
quand la moyenne mobile 20 jours croise au-dessus de la 50 jours*). En sortie :
la courbe de PnL, les métriques de risque (Sharpe, drawdown maximum, ratio de
gain), et la liste détaillée des trades exécutés.

## Vectorisé ou event-driven

Il existe deux familles de backtesters.

Les backtesters **vectorisés** appliquent la stratégie à toute la série
temporelle d'un coup, via des opérations numpy ou pandas. C'est rapide à
écrire et à exécuter, mais fragile : il est facile d'introduire du *lookahead
bias* (utiliser une information du futur), l'exécution des ordres est très
simplifiée (pas de vraie notion de fill, de slippage, de cash disponible), et
le code n'est pas portable vers du live trading.

Les backtesters **event-driven** simulent le temps qui passe. À chaque instant,
un événement de marché arrive, la stratégie décide, le broker exécute, le
portfolio se met à jour. C'est l'architecture des systèmes de production
(QuantConnect, Zipline, systèmes propriétaires en HFT ou en fonds
systématiques). Plus lent à coder, mais fidèle à la réalité, et le même code
peut basculer d'un backtest vers du paper trading ou du live en ne remplaçant
que deux composants (la source de données et le broker).

Ce projet implémente un backtester event-driven, précisément parce que c'est
l'architecture qui a une vraie valeur d'ingénierie et une portabilité vers du
réel.

## Architecture

Le moteur repose sur quatre composants qui communiquent via un bus
d'événements typés.

- **DataHandler** — lit les données historiques et publie un événement de
  marché à chaque nouvelle barre.
- **Strategy** — reçoit les événements de marché, applique sa logique de
  décision, publie un signal (par exemple *passer long AAPL avec conviction 1.0*).
- **Portfolio** — traduit le signal en ordre concret (par exemple *acheter 100
  actions AAPL au marché*), en tenant compte du cash disponible et de la
  position déjà détenue.
- **Broker** — simule l'exécution de l'ordre en appliquant slippage et
  commissions, puis publie un fill (par exemple *100 actions AAPL achetées à
  152.34, commission 0.50*). Le portfolio reçoit le fill et met à jour ses
  positions et son cash.

Une boucle principale itère sur les données, dispatche les événements publiés
à chaque tour, et produit à la fin le rapport de performance. Cette séparation
stricte a deux mérites : elle imite la structure d'un vrai système de trading,
et chaque composant peut être développé et testé isolément.

## Roadmap

**Phase 1 — Lecture des données et types de base.**
Structures représentant une barre et une série temporelle, lecture de fichiers
CSV, itération. Petit programme de démonstration qui affiche les premières
barres et calcule un rendement cumulé buy-and-hold. À ce stade, pas encore de
stratégie ni de bus d'événements — l'objectif est de poser la chaîne d'outils,
les types fondamentaux et la structure du projet.

**Phase 2 — Première stratégie et métriques.**
Ajout d'une classe abstraite `Strategy` et d'une implémentation concrète
(croisement de moyennes mobiles). Le moteur applique la stratégie sur les
données, enregistre les trades, calcule PnL, drawdown et Sharpe. Toujours dans
un mode simplifié, sans bus d'événements, pour se concentrer sur la
modélisation d'une stratégie et le calcul propre des métriques.

**Phase 3 — Bascule vers l'architecture event-driven.**
Introduction du bus d'événements (`MarketEvent`, `SignalEvent`, `OrderEvent`,
`FillEvent`), du portfolio et du broker simulé. Refactoring de la Phase 2 dans
cette nouvelle architecture. C'est la phase la plus structurante : le moteur
devient une vraie simulation temps par temps, avec gestion propre du cash, des
positions et des ordres en cours.

**Phase 4 — Généricité, instrumentation et benchmark.**
Le moteur devient paramétrable : différents types de données (barres
journalières, minute, tick), différents modèles d'exécution, différentes
précisions numériques. Ajout de logs structurés et d'un profiling systématique.
Benchmark face à une implémentation équivalente en Python (pandas ou vectorbt)
pour mesurer le gain de performance et vérifier l'équivalence des résultats.

**Phase 5 — Extensions.**
Selon le temps et l'intérêt, une ou plusieurs directions : parallélisation
d'un *grid search* sur les paramètres d'une stratégie ; carnet d'ordres à
limit levels pour simuler des ordres non-marché ; exposition du moteur à
Python via pybind11 pour l'appeler depuis un notebook ; modèle de coûts de
transaction plus riche (spread, market impact linéaire).

## Périmètre

Le projet cible des données OHLCV (open, high, low, close, volume) sur actions
ou instruments liquides, avec un modèle d'exécution simplifié : ordres au
marché fillés à l'ouverture ou à la clôture suivante, slippage paramétrable,
commissions linéaires. Les cas complexes — market impact non-linéaire, carnet
d'ordres complet, données tick-by-tick, dark pools — sont explicitement hors
périmètre initial, mais l'architecture reste compatible avec leur ajout
ultérieur.

## Stack

C++17 minimum, C++20 privilégié. Build via CMake, tests via GoogleTest ou
Catch2, formatage clang-format, compilation stricte
(`-Wall -Wextra -Wpedantic`). Dépendances externes minimales : `fmt` pour le
formatage, `pybind11` uniquement en Phase 5.

## Livrables

- Bibliothèque C++ réutilisable et exécutable de démonstration.
- Suite de tests unitaires couvrant les composants critiques.
- Documentation de l'architecture et de l'API.
- Scripts d'exemple illustrant l'usage sur des stratégies simples (SMA
  crossover, mean reversion).
- Rapport de benchmark comparant le moteur C++ à une implémentation Python
  de référence.
