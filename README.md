# Zog SHell

## Utilisation

Le module `zog` fournit **`zogsh`**, le lanceur du **Zog Shell** : un client en ligne de commande pour les serveurs Xcraft. Il se connecte à un serveur local ou distant, via ses ports de commandes et d'événements, pour l'administrer, l'inspecter ou le piloter.

Deux modes d'utilisation sont possibles :

- **mode commande** : une ou plusieurs commandes du bus Xcraft sont données après le séparateur `--`. Elles sont exécutées directement, puis le programme rend la main.
- **mode interactif** : aucune commande n'est donnée après `--`, et le shell Zog s'ouvre avec un prompt. Les commandes du bus peuvent alors y être saisies et enchaînées au cours d'une même session.

## Exécution

### Installation

Les dépendances (dont [xcraft-zog]) s'installent depuis la racine du module :

```bash
npm install
```

### Démarrage

Le binaire `zogsh` est déclaré dans le champ `bin` du `package.json`. Il peut être lancé de plusieurs manières :

- directement depuis le module, avec `node bin/zogsh` ou `npx zogsh` ;
- par son nom, `zogsh`, si le module est installé globalement ou lié (`npm link`).

## Paramètres

La ligne de commande se compose de deux parties séparées par `--` :

```text
zogsh [options du shell] -- [commande du bus] [arguments de la commande]
```

| Élément                                               | Rôle                                                                                                                                                                                     |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Options du shell** (avant `--`)                     | Elles configurent le shell Zog, notamment la connexion au serveur. Elles sont interprétées par [xcraft-zog].                                                                             |
| `--connect <hôte>:<port-commandes>:<port-événements>` | Se connecte à un serveur Xcraft. L'hôte est son adresse (par exemple `localhost`). Le port de commandes sert à envoyer les commandes, et le port d'événements à recevoir les événements. |
| `--`                                                  | Sépare les options du shell des commandes du bus. Il est ajouté automatiquement s'il est absent.                                                                                         |
| **Commande du bus** (après `--`)                      | Nom d'une commande exposée sur le bus Xcraft du serveur (par exemple `activity.status`). Toute commande déclarée par les modules chargés par le serveur peut être appelée.               |
| **Arguments de la commande**                          | Arguments propres à la commande appelée, saisis à la suite de son nom.                                                                                                                   |

Les options du shell autres que `--connect`, ainsi que les arguments propres à chaque commande, dépendent de [xcraft-zog] et du serveur ciblé. Leur détail ne relève pas de ce module.

### Configuration locale

Le fichier `etc.js` à la racine du module sert à surcharger la configuration par défaut des modules Xcraft, sans toucher à leurs sources. Dans sa version actuelle, il active la journalisation de [xcraft-core-log] (option `journalize`). Il suffit d'y ajouter une entrée par module pour modifier un autre comportement.

## Exemples

### Exécuter une commande sur un serveur

```bash
zogsh --connect localhost:35400:35800 -- activity.status
```

- `--connect localhost:35400:35800` : connexion à un serveur Xcraft local, avec le port de commandes `35400` et le port d'événements `35800`.
- `--` : séparateur entre les options du shell et la commande.
- `activity.status` : commande du bus qui liste les activités en cours sur le serveur.

Les ports dépendent de la configuration du serveur ciblé.

### Ouvrir le shell interactif

```bash
zogsh --connect localhost:35400:35800
```

Aucune commande ne suit le `--`, donc le shell Zog s'ouvre avec son prompt. On peut y saisir les mêmes commandes qu'en mode commande, par exemple `activity.status`, et en enchaîner plusieurs au cours d'une session.

### Ouvrir le shell sans préciser de serveur

```bash
zogsh
```

Le shell s'ouvre en mode interactif sans option de connexion. Le comportement de connexion par défaut est géré par [xcraft-zog].

### Se connecter à un serveur distant

```bash
zogsh --connect serveur.example.org:35400:35800 -- activity.status
```

Le principe est le même qu'en local. Seule l'adresse change, et les ports de commandes et d'événements doivent être joignables depuis le poste client.

_Ce contenu a été généré par IA_

---

[xcraft-zog]: https://github.com/Xcraft-Inc/xcraft-zog
[xcraft-core-etc]: https://github.com/Xcraft-Inc/xcraft-core-etc
[xcraft-core-log]: https://github.com/Xcraft-Inc/xcraft-core-log
