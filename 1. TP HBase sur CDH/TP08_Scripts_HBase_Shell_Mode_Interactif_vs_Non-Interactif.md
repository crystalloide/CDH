## HBase Shell : mode interactif vs non-interactif

Les deux modes reposent sur le même moteur JRuby, mais diffèrent fondamentalement dans leur mode d'invocation, leur comportement et leurs cas d'usage.

***

## Mode Interactif

C'est le mode par défaut lorsqu'on lance simplement `hbase shell`. L'utilisateur tape des commandes une par une et obtient un retour immédiat.

**Caractéristiques :**

- Affiche un **prompt** `hbase(main):001:0>` entre chaque commande
- Charge les fichiers d'initialisation de l'environnement (`.irbrc`, variables JRuby)
- Supporte l'**historique des commandes** (touches ↑/↓) et l'auto-complétion via TAB
- Affiche les résultats formatés avec durée d'exécution (`Took X.XXXX seconds`)
- Permet des **corrections à la volée** et un débogage immédiat
- Produit des sorties parasites (bandeaux de démarrage, invites de commande) difficiles à parser

```bash
hbase shell
```

HBase Shell
Use "help" to get list of supported commands.

```bash
list
# ← résultat affiché ici
```


***

## Mode Non-Interactif

Le shell exécute un fichier de script ou des commandes pipées, puis se termine automatiquement. C'est la base de l'**automatisation**.

**Trois syntaxes possibles :**

```bash
# 1. Passer un fichier de script
hbase shell /tmp/mon_script.hbase
```
```bash
# 2. Passer une commande inline via pipe
echo "list" | hbase shell
```
```bash
# 3. Here-document dans un script Bash
hbase shell <<EOF
put 'ma_table', 'row1', 'cf:col', 'valeur'
scan 'ma_table'
EOF
```

**Caractéristiques :**

- **Pas de prompt** affiché, pas d'interactivité attendue
- Peut fonctionner **en arrière-plan** ou dans un cron job
- Retourne un **code de retour** (`$?`) exploitable dans un script Bash
- La sortie peut être **redirigée** vers un fichier ou parsée via `grep`/`awk`
- En cas d'erreur dans le script, l'exécution peut continuer ou s'arrêter selon la gestion des exceptions JRuby

***

## Tableau Comparatif

| Critère | Interactif | Non-Interactif |
| :-- | :-- | :-- |
| **Lancement** | `hbase shell` | `hbase shell script.rb` ou pipe |
| **Entrée** | Clavier (terminal tty) | Fichier ou stdin |
| **Prompt** | Affiché (`hbase(main):001:0>`) | Absent |
| **Sortie** | Formatée, avec bandeaux | Parseable, redirigeable |
| **Historique/TAB** | Activé | Désactivé  |
| **Fichiers init** | Chargés (.irbrc) | Partiellement ignorés  |
| **Cas d'usage** | Exploration, debug, TP ponctuel | Automatisation, cron, CI/CD |
| **Gestion d'erreur** | Visuelle et immédiate | À coder explicitement (`begin/rescue`) |


***

## Piège Fréquent : Filtrer les Sorties Parasites

En mode non-interactif, HBase Shell génère quand même des lignes de démarrage (version Java, warnings) qu'il faut filtrer  :

```bash
# Récupérer uniquement les noms de tables
echo "list" | hbase shell 2>/dev/null \
  | grep -v "^HBase" \
  | grep -v "^Use" \
  | grep -v "^[0-9]"
```


***

## Quand utiliser lequel ?

- **Interactif** → exploration d'un cluster inconnu, debug d'une requête, TP de découverte
- **Non-interactif** → scripts de provisioning, chargement de données en masse, tâches planifiées (cron), pipelines d'intégration, **scripts de TP automatisés** sur CDH
