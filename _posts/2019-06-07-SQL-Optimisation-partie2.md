---
toc: true
layout: post
description: Optimisation de requêtes MySQL - Partie 2
categories: [markdown]
title: Optimisation de requêtes MySQL par indexation - Partie 2
---

# Optimisation de requêtes MySQL par indexation - Partie 2

Dans cette seconde partie, nous allons voir l'importance des moteurs de stockage dans l'indexation, la spécificité de leur architecture, et leurs cas d'usage.

Nous allons étudier les deux principaux moteurs de stockage (storage engine), MyISAM et InnoDb, ainsi que trois méthodes d'indexation (B-tree, B+tree et Hash).

## Méthodes d'indexation

Pour comprendre les méthodes d'indexation, introduisons le concept d'ordre physique et logique.
L'ordre physique est l'ordre de la variable que l'on observe dans la table, et l'ordre logique celui l'ordre enregistrée dans une table qui référence chacun des index de la table disponible pour l'utilisateur.
Dans le diagramme ci-dessous, chaque index (Col1) de la table MySQL est présent dans une table ordonnée pointant vers la table MySQL (grâce au pointeur qui correspond à l'"adresse" de la ligne). 

![physque_vs_logique](/images/physque_vs_logique.png)

Les méthodes d'indexation utilisent cette logique. Nous allons étudier les trois principales: B-tree, B+tree et Hash.

### B-tree

La méthode B-tree permet d'avoir un index désordonné dans la table Mysql, comme c'est le cas dans l'illustration au dessus.
Dans les sous-tables (feuilles) créées par la méthodes, les index sont ordonnés, et pointent vers les lignes de la table visible. Un niveau supplémentaire (nœud interne) pointe vers les feuilles (nœud final), la valeur de l'index de ce niveau correspond au dernier index (maximum dans le cas numérique) de chaque feuille.
Avec cette architecture, MySQL peut rapidement trouver l'adresse de la/les lignes demandée(s) par l'utilisateur. 

![btree_diag](/images/btree_diag.png)

Exemple:

- l'utilisateur recherche la valeur 34 à partir d'une requête
- Le nœud interne ne contient pas la valeur recherchée, mais indique à MySQL de rechercher la valeur dans le second nœud (34 se situant entre 28 et 72)
- Une fois la valeur trouvée après le parcours de l'arbre, la ligne de la table est renvoyée à partir de son adresse (pointeur)

:warning: Si la clé n'est pas unique, ou que utilisateur requête une fourchette de valeurs (RANGE), l'algorithme doit parcourir toute la feuille (nœud final) pour s'assurer de trouver toutes les occurrences. Il doit par ailleurs reparcourir l'arbre si l'index "déborde" sur plusieurs feuilles. Plus une clé non-unique comporte de valeurs répétés, plus l'exécution risque d'être longue. De même pour les RANGE, l'exécution sera d'autant plus lente que plus la fourchette de valeurs est large.

### B+tree

Dans la méthode B+tree, les index sont ordonnées dans la table visible de l'utilisateur. Par ailleurs, les feuilles sont connectées entre elles, et chacune pointe vers la suivante. S'affranchissant du besoin de reparcourir l'arbre, cette méthode est donc plus efficace pour la recherche d'une fourchette de valeurs, ou d'un index non-unique.

![bptree_diag](/images/bptree_diag.png)

:grey_exclamation:A la différence de la méthode B-tree, il n'y a pas de données enregistrées dans le nœud interne (branche). Le parcours peut donc être plus long pour la recherche d'une observation particulière, puisqu'il faut atteindre la nœud final pour obtenir l'adresse de la ligne.

### Hash

Dans cette méthode, c'est une fonction de *hashing* qui donne l'adresse de la ligne (pointeur).
L'adresse de la ligne est donc simplement obtenue en passant la valeur recherchée par la fonction, dont le résultat sera l'adresse (pointeur) de la ligne.
Cette méthode est très efficace pour la recherche d'une observation particulière. 
Elle est en revanche très limité pour une fourchette de valeurs, qui consiste en de multiples recherches individuelles, les résultats des fonctions de *hasing* étant aléatoires et non ordinales.

## Moteurs de stockage

MySQL intègre plusieurs moteurs de stockage. Les deux principaux sont MyISAM, InnoDB. Chacun de ces moteurs dispose de sa propre implémentation des méthode d'indexation.

### MyISAM

MyISAM utilise la structure B-tree pour les clés primaires, les clés uniques, ainsi que les index secondaires.
Pour les index secondaires, un pointeur vers la clé primaire est enregistré (id ligne recherchée).

:warning: Les index sont gérés en mémoire, il est important que le paramètre *key_buffer_size* soit bien défini (sujet non traité dans cet article)

### InnoDB

Le moteur InnoDB utilise la méthode B+tree pour la clé primaire.
Pour les clés secondaires, c'est la méthode B-tree qui est utilisée. A la différence de MyISAM, la clé primaire et non son pointeur est enregistrée avec la clé secondaire.

:grey_exclamation: Le pointeur étant enregistré avec la clé secondaire dans MyISAM, il n'est pas nécessaire de reparcourir un arbre pour retrouver la ligne à renvoyer, contrairement à InnoDB.

:warning: Plus la valeur de la clé primaire est grande, plus la feuille prendra d'espace sur le disque.

:warning: InnoDB peut décider d'utiliser la méthode Hash pour la gestion des index. Voir la configuration de la variable *innodb_adaptive_hash_index*.
