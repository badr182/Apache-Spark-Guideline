# Vue d'ensemble — Apache Spark pour Java

> [← Retour au sommaire](README.md)

---

## Qu'est-ce qu'Apache Spark ?

Apache Spark est un moteur de traitement de données distribué, open source, conçu pour la rapidité et la flexibilité. Il permet de traiter des volumes de données massifs (de quelques Go à plusieurs Po) en parallèle sur un cluster de machines.

**Caractéristiques clés :**
- **Traitement en mémoire** : jusqu'à 100x plus rapide que MapReduce sur disque
- **API unifiée** : batch, streaming, SQL, ML et graphes dans un seul framework
- **Multi-langage** : Scala (natif), Java, Python, R et SQL
- **Tolérance aux pannes** : récupération automatique via le DAG et les checkpoints

---

## L'API Java dans Spark

Bien que Scala soit le langage natif de Spark, l'API Java est **mature et complète**. Elle couvre l'ensemble des fonctionnalités de Spark sans compromis fonctionnel.

### Comparaison des langages

| Aspect | Java | Scala | Python (PySpark) |
|--------|------|-------|-----------------|
| Typage | Statique (compile-time) | Statique (compile-time) | Dynamique (runtime) |
| Verbosité | Élevée | Faible | Faible |
| Performance | Haute (JVM native) | Haute (JVM native) | Overhead sérialisation |
| Interop Spark | Totale | Totale | Totale (via Py4J) |
| Ecosystème enterprise | Excellent | Bon | Très bon |
| Débogage | Facile (IDEs Java) | Modéré | Modéré |

> **Recommandation :** Si votre équipe est Java, restez en Java. Ne migrez pas vers Scala uniquement pour Spark — les performances sont équivalentes.

### Points clés de l'API Java

- `Dataset<Row>` est le DataFrame non typé (équivalent à un tableau SQL).
- `Dataset<T>` avec un `Encoder<T>` permet des opérations typées et sûres à la compilation.
- Les lambdas Java fonctionnent avec l'API fonctionnelle de Spark (`MapFunction`, `FilterFunction`, `FlatMapFunction`, etc.).
- L'API Java est plus verbeuse mais offre la **sécurité du typage à la compilation** et une intégration IDE excellente.

---

## Composants Principaux de Spark

| Composant | Rôle | Cas d'usage |
|-----------|------|-------------|
| **Spark Core** | Ordonnancement des tâches, gestion mémoire, I/O de base | Fondation de tous les autres composants |
| **Spark SQL** | Données structurées via DataFrames et Datasets | ETL, Data Warehousing, requêtes analytiques |
| **Structured Streaming** | Traitement de données en temps réel (micro-batches ou continu) | Alertes en temps réel, dashboards live |
| **MLlib** | Machine learning distribué | Entraînement et inférence à grande échelle |
| **GraphX** | Calcul sur graphes (API Scala uniquement, Spark ≤ 3.x) | Analyse de réseaux sociaux, détection de fraude |

---

## Dataset vs DataFrame vs RDD

### RDD (Resilient Distributed Dataset)
L'API originale de Spark. Collection distribuée d'objets Java.

```java
// Évitez les RDDs dans le code nouveau — préférez Dataset/DataFrame
JavaRDD<String> lignes = spark.sparkContext()
    .textFile("fichier.txt")
    .toJavaRDD();
```

**Quand utiliser les RDDs :**
- Interopérabilité avec du code legacy Spark 1.x
- Traitement de données non structurées (binaire, texte brut complexe)
- Contrôle fin sur la sérialisation ou la localité des données

### DataFrame (`Dataset<Row>`)
Représentation de données structurées avec un schéma. Optimisé par le moteur Catalyst.

```java
Dataset<Row> df = spark.read().parquet("données/");
df.filter(col("montant").gt(100)).groupBy("region").agg(sum("montant")).show();
```

### Dataset typé (`Dataset<T>`)
Combine les avantages du DataFrame (optimisation Catalyst) et du typage Java.

```java
Dataset<Commande> commandes = df.as(Encoders.bean(Commande.class));
commandes.filter(c -> c.getMontant() > 100.0);  // lambda typé
```

### Recommandation

```
Pour la majorité des cas : Dataset<Row> (DataFrame)
Pour la logique métier complexe : Dataset<T>
Pour du legacy ou non structuré : RDD (uniquement si nécessaire)
```

---

## Le Cycle de Vie d'un Job Spark

```
1. Code Java → SparkSession.builder().getOrCreate()
2. Définir les transformations (lazy) → construction du DAG
3. Déclencher une action (count, show, write, collect)
4. Catalyst optimise le plan logique → plan physique
5. Le Driver envoie les tâches aux Executors via le Cluster Manager
6. Les Executors exécutent les tâches en parallèle sur les partitions
7. Les résultats sont retournés au Driver ou écrits dans le stockage
```

### Transformations vs Actions

**Transformations (lazy — rien ne s'exécute immédiatement) :**

| Transformation | Description |
|----------------|-------------|
| `filter()` | Filtre les lignes selon une condition |
| `select()` | Sélectionne des colonnes |
| `withColumn()` | Ajoute ou modifie une colonne |
| `join()` | Jointure entre deux Datasets |
| `groupBy()` | Regroupement (suivi d'une agrégation) |
| `orderBy()` / `sort()` | Tri (déclenche un shuffle) |
| `distinct()` | Suppression des doublons |
| `union()` / `unionByName()` | Union de deux Datasets |
| `repartition()` / `coalesce()` | Modification du partitionnement |
| `map()` / `flatMap()` | Transformations par ligne (Dataset typé) |

**Actions (déclenchent l'exécution) :**

| Action | Description |
|--------|-------------|
| `count()` | Nombre de lignes |
| `show()` | Affiche les N premières lignes |
| `collect()` | Ramène toutes les données dans le Driver ⚠️ |
| `first()` / `head()` | Première ligne |
| `take(n)` | N premières lignes |
| `write()` | Écriture dans le stockage |
| `foreach()` / `foreachPartition()` | Effet de bord par ligne/partition |

---

## Le Moteur Catalyst

Catalyst est l'optimiseur de requêtes de Spark SQL. Il transforme votre code en un plan d'exécution optimal.

```
Votre Code Java
     ↓
Plan Logique Non Résolu
     ↓  (résolution des colonnes, types)
Plan Logique Résolu
     ↓  (règles d'optimisation : push down, élimination colonnes...)
Plan Logique Optimisé
     ↓  (sélection stratégies : hash join, sort-merge join, broadcast...)
Plan Physique
     ↓  (génération de bytecode via Tungsten)
Exécution
```

**Pourquoi c'est important :**
- Les UDFs Java **contournent Catalyst** → moins d'optimisations possibles
- Les fonctions intégrées (`functions.*`) sont **optimisées par Catalyst**
- Utiliser `explain()` pour voir le plan généré :

```java
df.filter(col("statut").equalTo("actif"))
  .groupBy("region")
  .agg(sum("montant"))
  .explain(true);  // affiche le plan logique et physique
```

---

## Compatibilité des Versions

| Spark | Java | Scala | Notes |
|-------|------|-------|-------|
| 3.5.x | 8, 11, 17 | 2.12, 2.13 | Recommandé — support long terme |
| 3.4.x | 8, 11, 17 | 2.12, 2.13 | Stable |
| 3.3.x | 8, 11 | 2.12, 2.13 | End of support |
| 3.2.x | 8, 11 | 2.12, 2.13 | End of support |

> Spark 4.0 (à venir) abandonnera le support Java 8 — migrez vers Java 11 ou 17.

---

[→ Suite : Installation & Maven](02-installation-maven.md)
