# TP09 : Maîtriser le splitting dans HBase

## Présentation

Nous allons voir l'intérêt et la gestion des splits de tables dans HBase.

---

## 1. Introduction & Concept

Dans HBase, la scalabilité horizontale repose sur les **Régions**. 

Une table est découpée en plages de clés (**RowKeys**). 

Lorsqu'une région devient trop volumineuse, HBase la "splitte" (fractionne) en deux pour répartir la charge sur le cluster.

**Objectifs du lab :**
* Configurer une table pour observer un split automatique.
* Utiliser `LoadTestTool` pour simuler une charge massive.
* Implémenter le pre-splitting pour optimiser les performances d'écriture.

---

## 2. Prérequis

* Un cluster HBase fonctionnel (ou mode standalone).
* Accès au Shell HBase.
* Accès à la ligne de commande du serveur (pour lancer les outils Java).

---

## 3. Exercice 1 : Observation du Split Automatique

Par défaut, les régions sont volumineuses (10 Go+). 

Pour ce lab, nous allons forcer HBase à splitter dès **64 Mo** afin d'observer le phénomène + rapidement.

### Étape 1 : Création de la table

#### Ouvrez le shell HBase :

```bash
hbase shell
```


#### Créez une table avec un paramètre MAX_FILESIZE réduit : Extrait de code
```bash
create 'lab_split_auto', 'data', {METHOD => 'table_att', MAX_FILESIZE => '67108864'}
```

### Étape 2 : 

Injection de données avec LoadTestToolSortez du shell (exit) 

et utilisation de l'utilitaire natif de HBase pour injecter des données. 

Nous allons écrire 500 000 lignes de 1 Ko chacune.

```bash
hbase org.apache.hadoop.hbase.util.LoadTestTool -tn lab_split_auto -write 1:1024:10 -num_keys 500000
```

 -tn : Nom de la table.
 -write : 1 thread, 1024 octets par valeur, 10 colonnes par ligne.
 -num_keys : Nombre total de lignes à insérer.


Décomposition des commandes
-tn lab_split_autoSpécifie le nom de la table cible. Si la table n'existe pas, l'outil tentera de la créer.

-write 1:1024:10: Ceci définit la configuration de la charge de travail d'écriture au format <avg_payload_size>:<max_payload_size>:<num_threads>.

1 : Taille minimale de la charge utile (1 octet).

1024 : Taille maximale de la charge utile (1 Ko).

10 : L'outil utilisera 10 threads parallèles pour écrire les données.

-num_keys 500000: Indique à l'outil d'écrire un total de 500 000 lignes uniques (clés).

À quoi s'attendre pendant l'exécution
Création de la table : si lab_split_autoelle n’existe pas, HBase la créera avec des familles de colonnes par défaut (généralement f1).

Fractionnement des régions : Le nom de votre table incluant « split_auto », vous testez probablement la fonctionnalité de fractionnement automatique des régions de HBase . À mesure que les 500 000 clés sont écrites et que les fichiers HFile augmentent, le nombre de régions de cette table devrait s’accroître dans l’interface utilisateur du serveur HBase (généralement sur le port 16010).

Rapports d'avancement : La console affichera des mises à jour d'état périodiques indiquant le nombre de clés écrites et le débit d'écriture actuel (clés/seconde).

Problèmes courants à surveiller
Charge du serveur de régions : Avec 10 threads et 500 000 clés, vous constaterez un pic d’utilisation du processeur et des E/S disque. Si le serveur de régions MemStoreest saturé trop rapidement, des messages « Mises à jour bloquantes » peuvent apparaître dans les journaux.

Délai d'expiration de session Zookeeper : si le cluster manque de ressources, la charge d'écriture importante peut entraîner la perte de la communication entre Zookeeper et un RegionServer, ce qui peut provoquer un plantage du serveur.
 

### Étape 3 : Vérification des régions

Retournez dans le shell ou utilisez l'interface Web (port 16010) : 

Dans le shell :
```bash
list_regions 'lab_split_auto'
```

Observation : 

Vous devriez voir plusieurs régions avec des Start Key et End Key différentes. 
Notez comment le RegionName a été généré après le fractionnement.

## 4. Exercice 2 : Stratégie de Pre-splitting
Le split "à la volée" provoque des pics de latence (compactage). 
Pour les applications à haute performance, on définit les régions à la création.

### Étape 1 : 

Création avec points de splitImaginons que vos RowKeys soient des IDs d'utilisateurs allant de 0 à 1000. 

Créons une table déjà découpée en 4 zones :
Extrait de codecreate 'lab_presplit', 'cf', SPLITS => ['250', '500', '750']

### Étape 2 : Vérification immédiateExtrait de codelist_regions 'lab_presplit'

Résultat : La table possède déjà 4 régions réparties, avant même d'avoir inséré une seule donnée. 
C'est la méthode idéale pour éviter le Hotspotting (surcharge d'un seul serveur en début de projet).

5. Exercice 3 : Maintenance et Équilibrage
Même avec des splits automatiques, le cluster peut devenir déséquilibré (un RegionServer gère plus de régions que les autres).Étape 1 : Forcer un split manuelSi une région est trop sollicitée (lectures intensives sur une plage précise), on peut la couper manuellement :Extrait de code# Syntaxe : split 'nom_de_la_region' ou 'nom_de_la_table'
split 'lab_presplit'

Étape 2 : Lancer le Balancer

HBase déplace les régions entre les serveurs pour équilibrer la charge globale du cluster.
Extrait de code
balancer


6. Synthèse des commandes utiles :

____
Action						Commande Shell
____
Créer avec seuil de split	create 't1', 'f1', {MAX_FILESIZE => '67108864'}
Pré-splitter (Manuel)		create 't1', 'f1', SPLITS => ['A', 'B']
Lister les régions			list_regions 'nom_table'
Split manuel				split 'nom_table_ou_region'
Vérifier l'état du cluster	status 'detailed'


7. Conclusion

Le splitting est le moteur de la distribution dans HBase. 
Un bon administrateur préférera toujours le pre-splitting pour maîtriser la topologie de ses données 
et éviter que le cluster ne "transpire" en réorganisant les fichiers lors des pics de charge.




#### Complément ;
 
La hiérarchie des configurations dans HBase suit une logique de "spécificité croissante". 
Plus on se rapproche de l'objet (la table ou la famille de colonnes), plus la règle devient prioritaire.


Voici l'ordre de priorité (du plus fort au plus faible) utilisé par HBase pour déterminer le MAX_FILESIZE :


1. Propriété de la Table (Le plus prioritaire)	

C’est ce que nous avons fait dans le lab précédent avec la commande create 'table', {METHOD => 'table_att', MAX_FILESIZE => '...'}.
Si ce paramètre est défini explicitement au niveau de la table, HBase ignore totalement les configurations globales. 
C'est idéal pour isoler une table très volumineuse ou, au contraire, une petite table de référence.


2. Configuration Globale (hbase-site.xml)

Si rien n'est spécifié au niveau de la table, HBase cherche la clé **hbase.hregion.max.filesize** dans votre fichier de configuration cluster.
Valeur par défaut (HBase 2.x+) : 10 Go.
Cette valeur s'applique à toutes les tables du cluster qui n'ont pas de paramètre spécifique.


3. La politique de Split (RegionSplitPolicy)

C'est ici que ça se complique un peu (et où l'on "s'emmêle les pinceaux"). 
HBase n'utilise pas toujours une taille fixe. 
La politique par défaut (IncreasingToUpperBoundRegionSplitPolicy) calcule dynamiquement le seuil.
Le split se déclenche selon cette formule :$$\min(R^3 \cdot \text{flush\_size}, \text{max\_filesize})$$
(Où $R$ est le nombre de régions de la table sur le serveur)

Cela signifie que pour une nouvelle table, HBase va splitter très tôt (ex: 128 Mo, puis 1 Go...) 
jusqu'à atteindre la limite finale fixée par le MAX_FILESIZE.

Résumé de la priorité d'application :

Niveau		Source									Priorité

Table		ALTER ou CREATE via Shell				1 (Maximale)
Cluster		hbase-site.xml							2 (Moyenne)
Défaut		Valeur codée en dur (HBase Default)		3 (Faible)


Si un administrateur modifie le fichier hbase-site.xml et redémarre le cluster, 
mais qu'il ne voit aucun changement sur sa table, c'est probablement parce que :
- La table a été créée avec un MAX_FILESIZE spécifique (qui "écrase" la config globale).
- La politique de split dynamique (le calcul au cube) a splitté la région avant même d'atteindre la limite globale.

Astuce d'expert : 
Pour forcer une table à n'obéir qu'à une taille fixe et simple, 
on peut changer sa politique en ConstantSizeRegionSplitPolicy, 
mais c'est rarement recommandé en production car cela empêche une montée en charge progressive.



### Piège classique pour les débutants : 

On règle le MAX_FILESIZE à 10 Go, mais on voit les régions se couper à 128 Mo. 

Voici pourquoi : 

La politique par défaut de HBase (depuis la version 0.94) est la IncreasingToUpperBoundRegionSplitPolicy. 
Elle est conçue pour être "agressive" au début (pour répartir les données sur plusieurs serveurs rapidement) et plus stable ensuite.

La Formule MagiqueLe seuil de split pour une table donnée sur un RegionServer se calcule ainsi :

$$\text{Seuil} = \min(R^3 \cdot (2 \cdot \text{hbase.hregion.memstore.flush.size}), \text{hbase.hregion.max.filesize})$$

$R$ : Le nombre de régions de cette table présentes sur ce RegionServer.

Memstore Flush Size : Généralement 128 Mo par défaut.

Max Filesize : Le plafond (ex: 10 Go).


#### Exemple de calcul (Pas à pas)

Imaginons une table fraîchement créée sur un serveur. 

Le Memstore Flush Size est à 128 Mo et le Max Filesize à 10 Go.

Au début ($R = 1$ région) :
- Calcul : $1^3 \cdot (2 \cdot 128\text{ Mo}) = 256\text{ Mo}$
- Résultat : La région splitte dès qu'elle atteint 256 Mo.

Après le premier split ($R = 2$ régions) :
- Calcul : $2^3 \cdot (2 \cdot 128\text{ Mo}) = 8 \cdot 256\text{ Mo} = 2\,048\text{ Mo}$ (2 Go)
- Résultat : La prochaine région splittera à 2 Go.

Après le deuxième split ($R = 3$ régions) :
- Calcul : $3^3 \cdot (2 \cdot 128\text{ Mo}) = 27 \cdot 256\text{ Mo} = 6\,912\text{ Mo}$ (6,9 Go)
- Résultat : Le prochain split se fera à 6,9 Go.

Après le troisième split ($R = 4$ régions) :
- Calcul : $4^3 \cdot (2 \cdot 128\text{ Mo}) = 64 \cdot 256\text{ Mo} = 16\,384\text{ Mo}$ (16 Go)
- Comparaison : Comme 16 Go est supérieur au MAX_FILESIZE (10 Go), c'est la limite des 10 Go qui s'applique.
- Résultat : Désormais, tous les futurs splits se feront à 10 Go.

Cela permet d'expliquer que HBase est "intelligent" : Il crée vite des régions au début pour utiliser tous les serveurs disponibles.

Le comportement change avec le temps : Plus la table est grosse, plus elle devient "calme" et attend d'atteindre sa taille nominale pour splitter.

Le Pre-splitting annule cet effet : Si vous pré-splittez votre table en 10 régions dès le départ, 
$R$ est déjà grand, et HBase utilisera directement le MAX_FILESIZE.

Note de l'expert : Si vous constatez des splits bizarres, il faut vérifier la valeur de hbase.hregion.memstore.flush.size dans la config. 
Si elle a été augmentée, le split arrivera encore plus tard !
