# Hands-on Lab : Écriture de scripts shell HBase

## Contexte et Objectifs

Ce TP guidé vise à maîtriser l'automatisation des opérations HBase via des scripts Ruby/Shell, en s'appuyant sur l'environnement **CDH (Cloudera Distribution for Hadoop)**. À l'issue du lab, l'apprenant sera capable de :

- Lancer HBase Shell en mode non-interactif
- Écrire des scripts `.rb` et `.hbase` pour automatiser les opérations CRUD
- Utiliser des filtres avancés dans les scripts
- Enchaîner des opérations DDL/DML dans un fichier de script

***

## Prérequis

- Cluster CDH opérationnel avec HBase activé
- Accès SSH à un nœud edge ou master
- Interface Web HBase Master : `http://<master>:16010`
- Droits d'exécution sur `hbase shell`

***

## Partie 1 — Mode on-interactif : prise en main

Le HBase Shell étant basé sur JRuby, il supporte nativement l'exécution de scripts en mode non-interactif. C'est la base de toute automatisation.[^1]

**Étape 1 – Tester le mode non-interactif**

Créez un fichier `test_hbase.hbase` :

```bash
echo "status 'simple'" > /tmp/test_hbase.hbase
hbase shell /tmp/test_hbase.hbase
```

**Étape 2 – Passer une commande inline via pipe**

```bash
echo "list" | hbase shell
```

Vous pouvez aussi rediriger la sortie vers un fichier de résultats  :[^2]

```bash
echo "list" | hbase shell | grep -v "^[0-9]" > /tmp/tables.txt
```


***

## Partie 2 — Création d'une Table via Script

Créez le fichier `/tmp/create_bibliotheque.hbase` avec le contenu suivant :

```ruby
# Désactiver et supprimer la table si elle existe déjà
if exists('bibliotheque')
  disable 'bibliotheque'
  drop 'bibliotheque'
end

# Création de la table avec 3 familles de colonnes
create 'bibliotheque',
  {NAME => 'info',    VERSIONS => 3},
  {NAME => 'auteur',  VERSIONS => 1},
  {NAME => 'stats',   VERSIONS => 2, IN_MEMORY => true}

# Vérification
describe 'bibliotheque'
list
```

Exécutez le script  :[^1]

```bash
hbase shell /tmp/create_bibliotheque.hbase
```

> **Point pédagogique :** La directive `IN_MEMORY => true` sur la famille `stats` accélère les lectures fréquentes en maintenant les données dans le BlockCache.

***

## Partie 3 — Script d'Alimentation (PUT en masse)

Créez `/tmp/insert_bibliotheque.rb` :

```ruby
# Insertion de plusieurs livres
puts "=== Insertion des données ==="

table = HBase::Table.new('bibliotheque')

data = [
  ['livre:001', 'info:titre',      'Le Petit Prince'],
  ['livre:001', 'info:annee',      '1943'],
  ['livre:001', 'auteur:nom',      'Saint-Exupéry'],
  ['livre:001', 'auteur:pays',     'France'],
  ['livre:001', 'stats:emprunts',  '1250'],
  ['livre:002', 'info:titre',      'Dune'],
  ['livre:002', 'info:annee',      '1965'],
  ['livre:002', 'auteur:nom',      'Herbert'],
  ['livre:002', 'auteur:pays',     'USA'],
  ['livre:002', 'stats:emprunts',  '890'],
  ['livre:003', 'info:titre',      'Fondation'],
  ['livre:003', 'info:annee',      '1951'],
  ['livre:003', 'auteur:nom',      'Asimov'],
  ['livre:003', 'auteur:pays',     'USA'],
  ['livre:003', 'stats:emprunts',  '3400'],
]

data.each do |row, col, val|
  family, qualifier = col.split(':')
  put 'bibliotheque', row, col, val
end

puts "=== Insertion terminée ==="
scan 'bibliotheque'
```

Exécutez :

```bash
hbase shell /tmp/insert_bibliotheque.rb
```


***

## Partie 4 — Script de Requêtage avec Filtres

Créez `/tmp/query_bibliotheque.hbase` pour interroger les données  :[^3]

```ruby
puts "=== GET simple ==="
get 'bibliotheque', 'livre:001'

puts "\n=== GET multi-colonnes avec versions ==="
get 'bibliotheque', 'livre:001', {
  COLUMN => ['info:titre', 'stats:emprunts'],
  VERSIONS => 3
}

puts "\n=== SCAN complet ==="
scan 'bibliotheque'

puts "\n=== SCAN avec filtre sur pays auteur ==="
scan 'bibliotheque', {
  FILTER => "SingleColumnValueFilter('auteur', 'pays', =, 'binary:USA')"
}

puts "\n=== SCAN avec plage de rowkeys ==="
scan 'bibliotheque', {
  STARTROW => 'livre:001',
  STOPROW  => 'livre:002',
  COLUMNS  => ['info:titre', 'auteur:nom']
}
```


***

## Partie 5 — Script de Maintenance (DDL Avancé)

Créez `/tmp/maintenance_bibliotheque.hbase` :

```ruby
puts "=== Alter : ajout famille de colonnes ==="
disable 'bibliotheque'
alter 'bibliotheque', {NAME => 'media', VERSIONS => 1}
enable 'bibliotheque'

puts "=== Flush et Compaction ==="
flush 'bibliotheque'
major_compact 'bibliotheque'

puts "=== Récupération des splits ==="
get_splits 'bibliotheque'

puts "=== Count des lignes ==="
count 'bibliotheque', INTERVAL => 10
```

> **Point pédagogique :** Il est impératif de `disable` la table avant tout `alter` de structure. L'oubli de cette étape génère une erreur `TableNotDisabledException`.[^3]

***

## Partie 6 — Script d'Administration Bash + HBase Shell

Ce script bash orchestre plusieurs opérations HBase depuis le système  :[^4]

Créez `/tmp/admin_hbase.sh` :

```bash
#!/bin/bash
HBASE_CMD="hbase shell"
TABLE="bibliotheque"
BACKUP_DIR="/tmp/hbase_backup_$(date +%Y%m%d)"

echo "=== Vérification de la table $TABLE ==="
echo "exists '$TABLE'" | $HBASE_CMD

echo "=== Export du scan vers fichier ==="
mkdir -p $BACKUP_DIR
echo "scan '$TABLE'" | $HBASE_CMD \
  | grep "column=" > $BACKUP_DIR/scan_result.txt

echo "=== Résultat sauvegardé dans $BACKUP_DIR/scan_result.txt ==="
wc -l $BACKUP_DIR/scan_result.txt

echo "=== Suppression des données de test ==="
cat <<EOF | $HBASE_CMD
delete '$TABLE', 'livre:003', 'stats:emprunts'
EOF

echo "=== Opération terminée ==="
```

```bash
chmod +x /tmp/admin_hbase.sh
/tmp/admin_hbase.sh
```


***

## Tableau Récapitulatif des Commandes Clés

| Commande | Rôle | Contexte Script |
| :-- | :-- | :-- |
| `create` | Créer une table + CF | DDL initial |
| `put` | Insérer/Mettre à jour une cellule | Alimentation |
| `get` | Lire une ligne ou cellule | Requêtage ciblé |
| `scan` | Parcourir une plage de lignes | Requêtage large + filtres |
| `delete` | Supprimer une cellule | Maintenance |
| `alter` | Modifier la structure d'une table | DDL évolution |
| `flush` | Forcer l'écriture MemStore → HFile | Performance |
| `major_compact` | Fusion et nettoyage des HFiles | Optimisation [^5] |
| `count` | Compter les lignes | Monitoring |


***

## Livrables Attendus

1. Les **5 scripts** fonctionnels exécutés avec succès (captures de sortie)
2. Le fichier `scan_result.txt` généré par le script bash
3. La sortie de `describe 'bibliotheque'` après l'`alter` de la Partie 5
4. **Question de réflexion :** Quelle est la différence entre `delete` et `deleteall` dans un script ? Dans quel cas utilise-t-on `VERSIONS => N` dans un `get` ?
<span style="display:none">[^10][^11][^12][^13][^14][^15][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://docs.cloudera.com/runtime/7.3.1/accessing-hbase/topics/hbase-script-with-shell.html

[^2]: https://jdm.kr/blog/152

[^3]: https://www.labri.fr/perso/auber/BigDataGL/tds/hbase.html

[^4]: https://stackoverflow.com/questions/31126292/shell-script-for-executing-hbase-commands-deleting-all-hbase-tables

[^5]: https://github.com/pinpoint-apm/pinpoint/blob/master/hbase/scripts/README.md

[^6]: https://juvenal-chokogoue.developpez.com/tutoriels/apprendre-travailler-hbase/

[^7]: https://forge.in2p3.fr/projects/travaux-pratiques-publics/wiki/HBase

[^8]: https://blent.ai/blog/a/apache-hbase-base-nosql

[^9]: https://blog.stephane-robert.info/docs/admin-serveurs/linux/fondamentaux/efficace-shell/premier-script/

[^10]: http://insatunisia.github.io/TP-BigData/tp4/

[^11]: https://www.datacamp.com/fr/tutorial/how-to-write-bash-script-tutorial

[^12]: https://www.youtube.com/watch?v=jYi7-rAn-vQ

[^13]: https://www.youtube.com/watch?v=gBCGeLjYBgQ

[^14]: https://liora.io/hbase-guide-introductif-complet

[^15]: https://openclassrooms.com/forum/sujet/apprendre-a-ecrire-des-scripts

