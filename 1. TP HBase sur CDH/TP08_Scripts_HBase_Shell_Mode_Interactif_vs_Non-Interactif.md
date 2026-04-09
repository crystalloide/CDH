## HBase Shell : Mode Interactif vs Non-Interactif

Les deux modes reposent sur le même moteur JRuby, mais diffèrent fondamentalement dans leur mode d'invocation, leur comportement et leurs cas d'usage.

***

## Mode Interactif

C'est le mode par défaut lorsqu'on lance simplement `hbase shell`. L'utilisateur tape des commandes une par une et obtient un retour immédiat.[^5]

**Caractéristiques :**

- Affiche un **prompt** `hbase(main):001:0>` entre chaque commande
- Charge les fichiers d'initialisation de l'environnement (`.irbrc`, variables JRuby)[^6]
- Supporte l'**historique des commandes** (touches ↑/↓) et l'auto-complétion via TAB
- Affiche les résultats formatés avec durée d'exécution (`Took X.XXXX seconds`)
- Permet des **corrections à la volée** et un débogage immédiat
- Produit des sorties parasites (bandeaux de démarrage, invites de commande) difficiles à parser

```bash
$ hbase shell
HBase Shell
Use "help" to get list of supported commands.
hbase(main):001:0> list
# ← résultat affiché ici
```


***

## Mode Non-Interactif

Le shell exécute un fichier de script ou des commandes pipées, puis se termine automatiquement. C'est la base de l'**automatisation**.[^2]

**Trois syntaxes possibles :**

```bash
# 1. Passer un fichier de script
hbase shell /tmp/mon_script.hbase

# 2. Passer une commande inline via pipe
echo "list" | hbase shell

# 3. Here-document dans un script Bash
hbase shell <<EOF
put 'ma_table', 'row1', 'cf:col', 'valeur'
scan 'ma_table'
EOF
```

**Caractéristiques :**

- **Pas de prompt** affiché, pas d'interactivité attendue[^1]
- Peut fonctionner **en arrière-plan** ou dans un cron job[^2]
- Retourne un **code de retour** (`$?`) exploitable dans un script Bash
- La sortie peut être **redirigée** vers un fichier ou parsée via `grep`/`awk`
- En cas d'erreur dans le script, l'exécution peut continuer ou s'arrêter selon la gestion des exceptions JRuby

***

## Tableau Comparatif

| Critère | Interactif | Non-Interactif |
| :-- | :-- | :-- |
| **Lancement** | `hbase shell` | `hbase shell script.rb` ou pipe |
| **Entrée** | Clavier (terminal tty) | Fichier ou stdin [^5] |
| **Prompt** | Affiché (`hbase(main):001:0>`) | Absent |
| **Sortie** | Formatée, avec bandeaux | Parseable, redirigeable |
| **Historique/TAB** | Activé | Désactivé [^5] |
| **Fichiers init** | Chargés (.irbrc) | Partiellement ignorés [^6] |
| **Cas d'usage** | Exploration, debug, TP ponctuel | Automatisation, cron, CI/CD |
| **Gestion d'erreur** | Visuelle et immédiate | À coder explicitement (`begin/rescue`) |


***

## Piège Fréquent : Filtrer les Sorties Parasites

En mode non-interactif, HBase Shell génère quand même des lignes de démarrage (version Java, warnings) qu'il faut filtrer  :[^2]

```bash
# Récupérer uniquement les noms de tables
echo "list" | hbase shell 2>/dev/null \
  | grep -v "^HBase" \
  | grep -v "^Use" \
  | grep -v "^[0-9]"
```


***

## Quand Utiliser Lequel ?

- **Interactif** → exploration d'un cluster inconnu, debug d'une requête, TP de découverte
- **Non-interactif** → scripts de provisioning, chargement de données en masse, tâches planifiées (cron), pipelines d'intégration, **scripts de TP automatisés** sur CDH[^2]
<span style="display:none">[^3][^4][^7][^8]</span>

<div align="center">⁂</div>

[^1]: https://www.reddit.com/r/bash/comments/11osjrn/what_exactly_is_the_difference_between_an/

[^2]: http://pierrellensa.free.fr/dev/bash/www.bsdbooks.net/shells/scripting/fr/intandnonint.html

[^3]: https://www.n0tes.fr/2022/10/25/Langage-Bash-Shell-interactif/

[^4]: https://www.reddit.com/r/commandline/comments/11osne8/what_exactly_is_the_difference_between_an/

[^5]: https://notes.kodekloud.com/docs/Advanced-Bash-Scripting/Introduction/Interactive-vs-non-interactive-shell/page

[^6]: https://dev.to/lionthehoon/understanding-linux-shells-interactive-non-interactive-and-rc-files-3eli

[^7]: https://www.youtube.com/watch?v=8dujSlLHtnQ

[^8]: https://www.mislavjuric.com/linux-tutorial-series-84-interactive-vs-non-interactive-and-login-vs-non-login-shells/

