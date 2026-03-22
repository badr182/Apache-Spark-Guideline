# Architecture Apache Spark

> [← Retour au sommaire](README.md)

---

## Vue d'Ensemble de l'Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Cliente                      │
│                  (votre code Java / main())                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Driver Program (JVM)                      │
│                                                             │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────┐   │
│  │ SparkSession│   │ SparkContext │   │ DAGScheduler   │   │
│  │             │──▶│              │──▶│                │   │
│  │ (depuis 2.0)│   │ (RDD API)    │   │ TaskScheduler  │   │
│  └─────────────┘   └──────────────┘   └───────┬────────┘   │
└──────────────────────────────────────────────┬─────────────┘
                                               │
                              ┌────────────────▼────────────────┐
                              │         Cluster Manager          │
                              │  YARN / Kubernetes / Standalone  │
                              └────────────┬────────────────────┘
                      ┌─────────────────────┼──────────────────────┐
           ┌──────────▼──────────┐ ┌────────▼──────────┐ ┌────────▼──────────┐
           │    Worker Node 1     │ │   Worker Node 2    │ │   Worker Node N    │
           │  ┌───────────────┐  │ │ ┌───────────────┐  │ │ ┌───────────────┐ │
           │  │  Executor JVM │  │ │ │  Executor JVM │  │ │ │  Executor JVM │ │
           │  │ ┌───────────┐ │  │ │ │ ┌───────────┐ │  │ │ │ ┌───────────┐│ │
           │  │ │  Task 1   │ │  │ │ │ │  Task 3   │ │  │ │ │ │  Task 5   ││ │
           │  │ │  Task 2   │ │  │ │ │ │  Task 4   │ │  │ │ │ │  Task 6   ││ │
           │  │ └───────────┘ │  │ │ │ └───────────┘ │  │ │ │ └───────────┘│ │
           │  │   Cache/Disk  │  │ │ │   Cache/Disk  │  │ │ │   Cache/Disk │ │
           │  └───────────────┘  │ │ └───────────────┘  │ │ └───────────────┘ │
           └─────────────────────┘ └───────────────────-─┘ └───────────────────┘
```

---

## Glossaire des Concepts Clés

| Terme | Description |
|-------|-------------|
| **Driver** | La JVM qui exécute votre `main()`. Orchestre tout le job, construit le DAG, coordonne avec le Cluster Manager |
| **Executor** | Processus JVM sur les nœuds workers. Exécute les tâches, stocke les données en cache |
| **Tâche (Task)** | Unité atomique de travail — traite une seule partition de données |
| **Étape (Stage)** | Groupe de tâches pouvant s'exécuter en parallèle sans échange de données entre nœuds |
| **Job** | Déclenché par une action (`count()`, `collect()`, `write()`). Composé de plusieurs stages |
| **DAG** | Graphe Orienté Acyclique — le plan d'exécution logique de votre code |
| **Partition** | Unité de parallélisme — une fraction du Dataset traitée par une tâche |
| **Shuffle** | Redistribution des données entre partitions, nécessite des I/O disque et réseau |
| **Slot** | Emplacement d'exécution dans un executor (= un thread = 1 core CPU) |

---

## Le DAG et les Stages

### Comment Spark découpe un Job

```
Action : df.groupBy("region").agg(sum("montant")).write().parquet("sortie/")

Stage 1 (Map) :
  Tâche 1 → lire partition 0, filtrer, projeter
  Tâche 2 → lire partition 1, filtrer, projeter
  Tâche N → lire partition N, filtrer, projeter
         ↓
      SHUFFLE (écriture sur disque, transfert réseau)
         ↓
Stage 2 (Reduce) :
  Tâche 1 → agréger partitions region="EMEA"
  Tâche 2 → agréger partitions region="APAC"
  Tâche M → agréger partitions region="AMER"
         ↓
      Écriture Parquet
```

### Inspecter le Plan d'Exécution

```java
// Plan physique simple
df.explain();

// Plan logique + optimisé + physique détaillé
df.explain(true);

// Format étendu (Spark 3.x)
df.explain("extended");  // logique non résolu → résolu → optimisé
df.explain("cost");      // avec statistiques de coût
df.explain("formatted"); // plus lisible
```

---

## Cluster Managers

### YARN (Yet Another Resource Negotiator)

Le gestionnaire de ressources standard dans les environnements Hadoop/CDH/HDP.

```bash
# Mode client — le Driver tourne sur la machine de soumission
spark-submit --master yarn --deploy-mode client ...

# Mode cluster — le Driver tourne sur un nœud du cluster (recommandé en production)
spark-submit --master yarn --deploy-mode cluster ...
```

**Avantages :** intégration native avec HDFS, HIVE, HBase.

### Kubernetes

De plus en plus adopté pour les architectures cloud-native.

```bash
spark-submit \
  --master k8s://https://k8s-api:6443 \
  --deploy-mode cluster \
  --conf spark.kubernetes.container.image=spark:3.5.0 \
  ...
```

**Avantages :** isolation, scaling automatique, intégration CI/CD.

### Standalone

Cluster Spark autonome sans dépendance à YARN ou K8s.

```bash
# Démarrer le master
$SPARK_HOME/sbin/start-master.sh

# Démarrer un worker
$SPARK_HOME/sbin/start-worker.sh spark://master:7077

# Soumettre
spark-submit --master spark://master:7077 ...
```

---

## Gestion de la Mémoire

### Modèle Mémoire Spark (Unified Memory Manager)

```
spark.executor.memory (ex: 8g)
├── Reserved Memory (300 MB fixe — pour Spark interne)
└── Usable Memory (reste)
    ├── Spark Memory (spark.memory.fraction = 0.6 par défaut)
    │   ├── Execution Memory → jointures, tris, shuffles, agrégations
    │   └── Storage Memory  → cache DataFrame (persist/cache)
    │   (les deux partagent dynamiquement l'espace)
    └── User Memory (0.4) → structures de données utilisateur, métadonnées

spark.executor.memoryOverhead (ex: 2g)
    → Mémoire hors-tas JVM : bytecode généré, bibliothèques natives, threads
```

### Paramètres Importants

```properties
# Fraction de la mémoire dédiée à Spark (exécution + stockage)
spark.memory.fraction=0.6

# Fraction du Spark memory allouée au stockage (cache)
# L'exécution peut emprunter la partie non utilisée
spark.memory.storageFraction=0.5

# Overhead hors-tas (10% de executor.memory, min 384 MB)
spark.executor.memoryOverhead=2g

# Pour les applications PySpark ou utilisant des bibliothèques natives
spark.executor.pyspark.memory=1g
```

---

## Partitionnement

### Comment les Données sont Partitionnées

```
Fichier HDFS (128 MB par bloc) :
bloc 0 → Partition 0 → Tâche 0 → Executor A
bloc 1 → Partition 1 → Tâche 1 → Executor B
bloc 2 → Partition 2 → Tâche 2 → Executor A
...

Après un shuffle (ex: groupBy) :
Les données sont redistribuées en spark.sql.shuffle.partitions partitions
(200 par défaut — ajuster selon la taille des données)
```

### Inspecter et Ajuster le Partitionnement

```java
// Voir le nombre actuel de partitions
System.out.println("Partitions: " + df.rdd().getNumPartitions());

// Voir la taille de chaque partition (en bytes)
df.rdd().mapPartitions(it -> {
    long count = 0;
    while (it.hasNext()) { it.next(); count++; }
    return Collections.singletonList(count).iterator();
}, true /* preservesPartitioning */)
.collect()
.forEach(System.out::println);

// Règle pratique : viser 100-200 MB par partition
// Pour 1 TB de données → 5000 à 10000 partitions
```

---

## Localité des Données (Data Locality)

Spark essaie d'exécuter chaque tâche sur le nœud qui contient physiquement les données. Niveaux de localité (du mieux au moins bien) :

| Niveau | Description | Performance |
|--------|-------------|-------------|
| `PROCESS_LOCAL` | Données dans la JVM du même executor | Optimal |
| `NODE_LOCAL` | Données sur le même nœud (HDFS local) | Très bon |
| `NO_PREF` | Aucune préférence (S3, GCS, ADLS) | Variable |
| `RACK_LOCAL` | Données sur le même rack réseau | Acceptable |
| `ANY` | Données sur un autre rack ou nœud | Lent (réseau) |

> **Note :** Sur S3/GCS/ADLS (stockage objet), la localité n'existe pas — toutes les lectures sont `ANY`. C'est pourquoi les formats columnar (Parquet, ORC) et les prédicats pushdown sont critiques sur le cloud.

---

## L'Interface Web Spark (Spark UI)

Accessible sur `http://driver-host:4040` pendant l'exécution.

| Onglet | Informations |
|--------|-------------|
| **Jobs** | Liste des jobs, statut, durée |
| **Stages** | Détail par stage, tâches, durée, shuffle |
| **Storage** | DataFrames en cache, niveaux de stockage |
| **Environment** | Configuration Spark active |
| **Executors** | Mémoire, CPU, GC par executor |
| **SQL** | Plans d'exécution des requêtes SQL/DataFrame |

### Métriques Clés à Surveiller

```
Dans l'onglet Stages :
- "Shuffle Read/Write" élevé → beaucoup de shuffles, vérifier les jointures
- "GC Time" > 10% de la durée totale → problème de mémoire
- Écart important entre min/médian/max task duration → data skew
- "Spill (Memory)" ou "Spill (Disk)" > 0 → manque de mémoire

Dans l'onglet Executors :
- "Task Time (GC)" > 20% → augmenter executor.memory ou changer GC
- "Shuffle Spill" → augmenter shuffle.partitions
```

---

[→ Suite : Bonnes Pratiques](04-bonnes-pratiques.md)
