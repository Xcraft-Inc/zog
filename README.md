# 📘 zog

## Aperçu

Le module `zog` est le projet qui rend le **Zog Shell** (`zogsh`) utilisable : un **client en ligne de commande pour les serveurs Xcraft**. Avec l'option `--connect`, `zogsh` se connecte à un serveur local ou distant (via ses ports de commandes et d'événements). Il offre deux modes d'utilisation :

- **mode commande** : les commandes du bus Xcraft indiquées après le séparateur `--` sont exécutées directement, par exemple `activity.status` pour lister les activités en cours ;
- **mode interactif** : sans commande après `--` (ou sans `--` du tout), `zogsh` ouvre le shell Zog, une sorte de REPL avec un prompt, depuis lequel on peut aussi exécuter les commandes du bus.

Il permet ainsi d'administrer, d'inspecter ou de piloter un serveur en cours d'exécution.

Le module contient peu de code, car il s'appuie sur [xcraft-zog] pour le shell lui-même. Son rôle est de fournir le lanceur `zogsh` et l'environnement qui l'accompagne : définition de la racine Xcraft (en variable d'environnement et dans le fichier `etc/xcraft/config.json`), génération des configurations par défaut des modules installés et application de la surcharge locale `etc.js`. Il ne contient ni acteur Elf/Goblin, ni widget React.

## Sommaire

- [Aperçu](#aperçu)
- [Structure du module](#structure-du-module)
- [Fonctionnement global](#fonctionnement-global)
- [Exemples d'utilisation](#exemples-dutilisation)
- [Interactions avec d'autres modules](#interactions-avec-dautres-modules)
- [Configuration avancée](#configuration-avancée)
- [Détails des sources](#détails-des-sources)
- [Licence](#licence)

## Structure du module

```
zog/
├── bin/
│   └── zogsh        # Script exécutable (point d'entrée du shell)
├── etc/
│   └── xcraft/
│       └── config.json   # Généré au lancement (contient xcraftRoot)
├── etc.js           # Surcharge locale de la configuration Xcraft
└── package.json     # Métadonnées, dépendance à xcraft-zog, binaire zogsh
```

- **`bin/zogsh`** : point d'entrée du client en ligne de commande. Ce lanceur Node.js initialise l'environnement Xcraft puis démarre le Zog Shell.
- **`etc/xcraft/config.json`** : fichier écrit par `zogsh` à chaque lancement, qui contient la racine Xcraft (`xcraftRoot`).
- **`etc.js`** : fichier de surcharge de configuration, lu par [xcraft-core-etc] au démarrage.
- **`package.json`** : déclare le binaire `zogsh` et la dépendance vers [xcraft-zog], qui fournit le shell.

## Fonctionnement global

Au lancement de `zogsh`, les étapes suivantes sont exécutées dans l'ordre :

1. Calcul de la racine Xcraft (`xcraftRoot`), qui correspond au répertoire parent du dossier `bin/` (donc la racine du module `zog`), et définition de la variable d'environnement `XCRAFT_ROOT` avec cette valeur.
2. Écriture du fichier `etc/xcraft/config.json` contenant `{ "xcraftRoot": "<racine>" }`. Le fichier est écrasé à chaque lancement.
3. Instanciation de [xcraft-core-etc].
4. Création de toutes les configurations par défaut (`createAll`) pour les modules installés dans `./node_modules/` dont le nom correspond à l'expression `^(goblin-|(xcraft-(core|contrib)))`, c'est-à-dire les modules `goblin-*`, `xcraft-core-*` et `xcraft-contrib-*`. Le fichier `./etc.js` est transmis comme source de surcharge.
5. Ajout de l'argument `--` à `process.argv` s'il est absent.
6. Chargement de `xcraft-zog/bin/zog`, qui démarre le Zog Shell.

Tous les chemins utilisés par le lanceur sont résolus à partir de l'emplacement du script (`__dirname`) et non du répertoire courant : `zogsh` peut donc être lancé depuis n'importe quel répertoire.

Les arguments de la ligne de commande sont structurés en deux parties séparées par `--` :

- **avant `--`** : les options du shell, par exemple `--connect <hôte>:<port-commandes>:<port-événements>` pour se connecter à un serveur Xcraft ;
- **après `--`** : les commandes du bus Xcraft à exécuter sur le serveur auquel le shell est connecté, par exemple `activity.status`.

Si aucune commande ne suit le `--`, ou si le `--` est absent, le shell Zog s'ouvre en mode interactif (prompt de type REPL). Les commandes du bus peuvent alors y être saisies et exécutées de la même manière.

```text
zogsh
  │
  ├─ XCRAFT_ROOT = <racine du module zog>
  ├─ etc/xcraft/config.json ◄── { xcraftRoot }
  ├─ xcraft-core-etc ── createAll(node_modules, regex, ./etc.js)
  ├─ process.argv : ajout de « -- » si absent
  └─ require(xcraft-zog/bin/zog)  ──►  Zog Shell
```

## Exemples d'utilisation

### Installation et lancement

```bash
npm install
npx zogsh
```

Sans commande, cette dernière instruction ouvre le shell Zog en mode interactif (voir plus bas).

### Exécuter une commande sur un serveur Xcraft

```bash
node bin/zogsh --connect localhost:35400:35800 -- activity.status
```

Dans cet exemple :

- `--connect localhost:35400:35800` : connexion à un serveur Xcraft local, avec le port de commandes `35400` et le port d'événements `35800` ;
- `--` : séparateur entre les options du shell et les commandes du bus ;
- `activity.status` : commande du bus exécutée sur le serveur, qui liste les activités en cours.

Toute autre commande exposée sur le bus (via les `xcraftCommands` des modules chargés par le serveur) peut être appelée de la même manière après le `--`.

### Utiliser le shell interactif

```bash
node bin/zogsh --connect localhost:35400:35800
```

Aucune commande n'étant fournie après `--` (le séparateur est ajouté automatiquement par `zogsh` s'il est absent), le shell Zog s'ouvre avec son prompt. On peut alors y saisir les mêmes commandes que dans le mode commande, par exemple `activity.status`, et enchaîner plusieurs commandes au cours d'une même session.

Le binaire `zogsh` est également exposé via le champ `bin` du `package.json`, ce qui permet de l'appeler directement lorsque le module est installé globalement ou lié.

### Surcharger la configuration d'un module

Pour modifier le comportement d'un module Xcraft sans toucher à ses sources, il suffit d'ajouter une entrée dans `etc.js` à la racine du projet. Par exemple, pour désactiver la journalisation de [xcraft-core-log] :

```javascript
"use strict";

module.exports = {
  default: {
    ["xcraft-core-log"]: {
      journalize: false,
    },
  },
};
```

## Interactions avec d'autres modules

- [xcraft-zog] : fournit le Zog Shell réel (`bin/zog`) qui est chargé en fin de lancement et qui gère la connexion au serveur (`--connect`) ainsi que l'exécution des commandes.
- **Serveur Xcraft** : `zogsh` agit comme un client du bus Xcraft d'un serveur en cours d'exécution, via son port de commandes et son port d'événements.
- [xcraft-core-etc] : gère la création et la surcharge des configurations des modules, à partir des fichiers `config.js` de chacun d'eux et du fichier `etc.js` local. Le fichier `etc/xcraft/config.json` écrit par `zogsh` lui fournit la racine Xcraft.
- [xcraft-core-log] : son option `journalize` est activée par la surcharge fournie dans `etc.js`.
- Les modules `goblin-*`, `xcraft-core-*` et `xcraft-contrib-*` présents dans `node_modules` voient leur configuration par défaut générée au démarrage.

Le champ `allowScripts` du `package.json` autorise l'exécution des scripts d'installation de `xcraft-core-bus`, `koffi` et `better-sqlite3`, des dépendances transitives qui nécessitent une compilation ou un binaire natif.

## Configuration avancée

Le fichier `etc.js` à la racine du module est transmis à [xcraft-core-etc] comme surcharge des configurations par défaut. Sa structure est un objet dont la clé `default` contient une entrée par module à configurer.

| Option                               | Description                               | Type    | Valeur par défaut  |
| ------------------------------------ | ----------------------------------------- | ------- | ------------------ |
| `default.xcraft-core-log.journalize` | Active la journalisation du module de log | boolean | `true` (surcharge) |

### Variables d'environnement

| Variable      | Description                                                        | Exemple          | Valeur par défaut                   |
| ------------- | ------------------------------------------------------------------ | ---------------- | ----------------------------------- |
| `XCRAFT_ROOT` | Racine du projet Xcraft, définie par `zogsh` avant tout chargement | `/home/user/zog` | Répertoire parent du dossier `bin/` |

## Détails des sources

### `bin/zogsh`

Script Node.js exécutable (`#!/usr/bin/env node`). Il calcule la racine Xcraft (répertoire parent de `bin/`), la positionne dans `XCRAFT_ROOT` et l'écrit dans `etc/xcraft/config.json` sous la forme `{ "xcraftRoot": "..." }`. Ce fichier est écrit de manière synchrone (`fs.writeFileSync`) à chaque lancement.

Il charge ensuite [xcraft-core-etc] et génère les configurations de tous les modules Xcraft détectés dans `node_modules`, en tenant compte de la surcharge locale `etc.js`. Tous les chemins sont résolus à partir de l'emplacement du script et non du répertoire courant, le shell n'a donc pas besoin d'être lancé depuis la racine du module `zog`.

Il garantit enfin la présence de `--` dans les arguments avant de déléguer au Zog Shell de [xcraft-zog], qui interprète les options (comme `--connect`) puis les commandes du bus qui suivent le séparateur. Lorsqu'il n'y a aucune commande après le séparateur, [xcraft-zog] ouvre le shell interactif.

### `etc.js`

Fichier de surcharge de configuration (voir [Configuration avancée](#configuration-avancée)). Il active `journalize` pour [xcraft-core-log].

### `package.json`

Déclare le module `zog` en version `1.0.0` sous licence MIT, le binaire `zogsh` (`bin/zogsh`) et la dépendance `xcraft-zog` (`^3.2.0`). Le champ `allowScripts` autorise les scripts d'installation de `xcraft-core-bus`, `koffi` et `better-sqlite3`. Le script `test` n'est pas implémenté.

## Licence

Ce module est distribué sous [licence MIT](./LICENSE).

_Ce contenu a été généré par IA_

---

[xcraft-zog]: https://github.com/Xcraft-Inc/xcraft-zog
[xcraft-core-etc]: https://github.com/Xcraft-Inc/xcraft-core-etc
[xcraft-core-log]: https://github.com/Xcraft-Inc/xcraft-core-log
