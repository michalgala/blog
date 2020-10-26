# Optimisation de requêtes MySQL par indexation - Partie 1

L'indexation de variables est le principal levier d'amélioration de la performance des requêtes SQL. Dans cette série d'article nous allons présenter l'utilité des index, leur utilisation par MySQL pour l'exécution de requêtes, et leurs implications sur la performance.

Dans cette première partie, à l'aide de la fonction EXPLAIN, nous allons voir comment MySQL exécute une requête, et étudier l'impact des index sur leur vitesse d'exécution.

##### Données

Les données utilisées sont issues du playground Kaggle "predict future sales", auxquelles j'ai rajouté une clé unique auto-incrémentée pour les tables sans identifiant unique de ligne.

## Debug de requêtes 

MySQL utilise les index pour optimiser de nombreuses opérations: les requêtes simples, les jointures, les tris, les groupements et agrégations.

En l'absence d'index, l'optimiseur parcourt toute la colonne à la recherche du critère indiqué dans la requête.
Prenons l’exemple d'une requête basée sur la variable 'index' (clé primaire et donc index de la table sales_train), et de la variable item_id (simple colonne de la table, non indexée).

```sql
SELECT *
FROM sales_train
WHERE `index` = 2000000

SELECT *
FROM sales_train
WHERE item_id = 22169
```

La première requête s'exécute quasi-instantanément (**4ms**), et la seconde 225 fois plus longtemps (**900ms**).
Mysql donne la possibilité d'étudier le schéma d'exécution grâce à la fonction **EXPLAIN**, à ajouter en préfixe de la requête.

```sql
EXPLAIN SELECT * FROM sales_train WHERE `index` = 2000000
EXPLAIN SELECT * FROM sales_train WHERE item_id = 22169
```

|              | table       | type  | possible_keys | key     | rows    |
| ------------ | ----------- | ----- | ------------- | ------- | ------- |
| 1ère requête | sales_train | const | PRIMARY       | PRIMARY | 1       |
| 2de requête  | sales_train | ALL   | NULL          | NULL    | 2722275 |

Chaque requête précédée d'EXPLAIN, renvoie une table dont les colonnes qui nous concernent sont présentées dans le tableau ci-dessus.

- Colonne **rows**: On y voit que la première requête examine 1 seule ligne, alors que la seconde parcourt toute la table, à savoir les 2 722 275 lignes correspondant au nombre total d'observations. (note: la requête n'étant pas exécutée, il s'agit en réalité d'approximations)
- Colonnes **key**: On peut voir que la seconde requête n'utilise pas d'index pour son exécution, contrairement à la première
- Colonne **type**: ALL dans cette colonne, et NULL dans la colonne key nous indique un scan de toute la table

## Optimisation par l'ajout d'index

### Requête simple

```sql
-- Dupliquer la table et ajouter indexer item_id
CREATE TABLE sales_train_test LIKE sales_train;
INSERT INTO sales_train_test SELECT * FROM sales_train;  
ALTER TABLE sales_train_test ADD INDEX `item_id` (`item_id`);

-- Requête
SELECT * FROM sales_train_test WHERE item_id = 22169;
EXPLAIN SELECT * FROM sales_train_test WHERE item_id = 22169;
```

Dans le premier bloc requête, une nouvelle table 'sales_train_test' est créée à partir de la table utilisée précédemment, et la variable 'item_id' y est indexée.
Enfin, nous exécutons la même requête que dans la partie précédente, et nous voyons que cette fois-ci la requête s'exécute en **6ms** (contre **900ms** auparavant).

|            | table            | type | possible_keys | key     | rows    |
| ---------- | ---------------- | ---- | ------------- | ------- | ------- |
| Sans index | sales_train      | ALL  | NULL          | NULL    | 2722275 |
| Avec index | sales_train_test | ref  | item_id       | item_id | 1       |

La fonction EXPLAIN nous indique que l'optimiseur prend bien en compte l'index pour exécuter la requête, qui ne parcourt plus toutes les lignes à la recherche du critère.

:warning: L'ajout d'index n'améliore pas systématiquement la performance d'une requête. Nous en expliquerons les raisons dans les parties suivantes.

### Jointure

Nous allons tester 3 cas de jointures. Le premier avec 'item_id' indexé dans les deux tables, le second dans une seule des deux tables, et enfin sans que la variable ne soit indexée.
Le tableau ci dessous est le résultat de la fonction EXPLAIN, ainsi que le temps d'exécution pour un INNER JOIN des tables 'sales_train' (T1) et 'items' (T2) avec et sans index pour la variable de jointure 'item_id'

|                            | Temps     | type            | key               | rows                  |
| -------------------------- | --------- | --------------- | ----------------- | --------------------- |
| T1_index<br />T2_index     | **7ms**   | ref<br />ALL    | item_id<br />NULL | 124<br />22 418       |
| T1_noindex<br />T2_index   | **8ms**   | ALL<br />eq_ref | NULL<br />PRIMARY | 2 722 275<br />1      |
| T1_index<br />T2_noindex   | **7ms**   | ref<br />ALL    | item_id<br />NULL | 124<br />21 432       |
| T1_noindex<br />T2_noindex | **258ms** | ALL<br />ALL    | NULL<br />NULL    | 2 722 275<br />21 432 |

On observe qu'en l'absence d'index la requête met près de 40 fois plus de temps à s'exécuter. 


### Tri, groupements, agrégations

L'utilisation d'index se révèle particulièrement efficace pour les opérations de tri et de groupement (qui trient les données afin de pouvoir former les groupes). Nous pouvons voir dans le tableau produit par la fonction EXPLAIN, que l'utilisation de l'index est beaucoup plus efficace que l'utilisation d'une table temporaire pour grouper les données, qui par ailleurs est enregistrée sur le disque si elle ne peut être contenue en mémoire (colonne **Extra**)[^1]

```sql
-- Table originale
SELECT item_id, count(*)
FROM sales_train
GROUP BY item_id

EXPLAIN SELECT ...

-- Table avec item_id en index
SELECT item_id, count(*)
FROM sales_train_test
GROUP BY item_id

EXPLAIN SELECT ...
```

|                         | type | possible_keys | key     | rows    | Extra                               |
| ----------------------- | ---- | ------------- | ------- | ------- | ----------------------------------- |
| Sans index (**1630ms**) | ALL  | NULL          | NULL    | 2722275 | Using temporary<br />Using filesort |
| Avec index (**7ms**)    | ref  | item_id       | item_id | 1       | Using index                         |

:warning: Avoir un index en variable de regroupement/tri ne garantit pas son utilisation par l'optimiseur



[^1]: http://s.petrunia.net/blog/?p=24
[^2]: "*Effective MySQL: Optimizing SQL Statements*"  Ronald Bradford