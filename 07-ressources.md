# Ressources Supplémentaires

> [← Retour au sommaire](README.md)

---

## Documentation Officielle

| Ressource | Description |
|-----------|-------------|
| [Documentation Apache Spark](https://spark.apache.org/docs/latest/) | Documentation officielle — référence complète |
| [Javadoc API Spark](https://spark.apache.org/docs/latest/api/java/index.html) | Référence complète de l'API Java |
| [Guide de Programmation SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html) | DataFrames, Datasets, SQL |
| [Guide Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) | Traitement de flux en Java |
| [Guide MLlib](https://spark.apache.org/docs/latest/ml-guide.html) | Machine learning avec Spark |
| [Guide de Tuning](https://spark.apache.org/docs/latest/tuning.html) | Optimisation des performances |
| [Documentation AQE](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution) | Adaptive Query Execution |
| [Configuration Spark](https://spark.apache.org/docs/latest/configuration.html) | Toutes les propriétés configurables |

---

## Livres Recommandés

| Livre | Auteurs | Description |
|-------|---------|-------------|
| **Spark: The Definitive Guide** | Chambers & Zaharia (O'Reilly) | La référence complète — couvre Scala, Java et Python |
| **Learning Spark, 2nd Edition** | Damji et al. (O'Reilly) | Mise à jour pour Spark 3.x — exemples Java/Scala/Python |
| **High Performance Spark** | Karau & Warren (O'Reilly) | Optimisation avancée des performances |
| **Stream Processing with Apache Spark** | Karau & Learner (O'Reilly) | Structured Streaming et DStreams |

---

## Formations en Ligne

| Plateforme | Cours | Niveau |
|------------|-------|--------|
| [Databricks Academy](https://www.databricks.com/learn/training/catalog) | Apache Spark Developer | Débutant → Avancé (gratuit) |
| [Coursera — Big Data Analysis with Scala and Spark](https://www.coursera.org/learn/scala-spark-big-data) | EPFL — Spark avec Scala | Intermédiaire |
| [Udemy — Apache Spark with Java](https://www.udemy.com/topic/apache-spark/) | Plusieurs cours disponibles | Débutant → Avancé |
| [LinkedIn Learning](https://www.linkedin.com/learning/topics/apache-spark) | Apache Spark Essential Training | Débutant |

---

## Outils et Écosystème

### Formats de Données

| Outil | Usage |
|-------|-------|
| [Apache Parquet](https://parquet.apache.org/) | Format columnar — standard pour Spark |
| [Apache Avro](https://avro.apache.org/) | Format row-based — idéal pour Kafka |
| [Delta Lake](https://delta.io/) | Tables ACID sur Parquet — upserts, time travel |
| [Apache Iceberg](https://iceberg.apache.org/) | Format de table ouvert — partitionnement avancé |
| [Apache Hudi](https://hudi.apache.org/) | Upserts et incremental queries |

### Stockage et Infrastructure

| Outil | Usage |
|-------|-------|
| [Apache Hadoop HDFS](https://hadoop.apache.org/) | Stockage distribué on-premise |
| [Apache Kafka](https://kafka.apache.org/) | Source/sink pour Structured Streaming |
| [Apache Hive](https://hive.apache.org/) | Metastore et tables SQL |
| AWS S3 / GCS / ADLS | Stockage objet cloud |

### Monitoring

| Outil | Usage |
|-------|-------|
| [Spark UI](http://localhost:4040) | Interface web intégrée — jobs, stages, executors |
| [Apache Spark History Server](https://spark.apache.org/docs/latest/monitoring.html) | Historique des jobs après exécution |
| [Ganglia](http://ganglia.sourceforge.net/) | Monitoring cluster |
| [Prometheus + Grafana](https://spark.apache.org/docs/latest/monitoring.html#executor-metrics) | Métriques Spark → Prometheus |

### IDEs et Outils de Développement

| Outil | Usage |
|-------|-------|
| [IntelliJ IDEA](https://www.jetbrains.com/idea/) | IDE Java/Scala recommandé pour Spark |
| [Apache Zeppelin](https://zeppelin.apache.org/) | Notebooks interactifs avec Spark |
| [Jupyter + Toree](https://toree.apache.org/) | Notebooks Jupyter avec kernel Spark |
| [spark-shell](https://spark.apache.org/docs/latest/quick-start.html) | REPL Scala/Python pour tests rapides |

---

## Communauté et Support

| Ressource | Description |
|-----------|-------------|
| [Stack Overflow — apache-spark](https://stackoverflow.com/questions/tagged/apache-spark) | Q&A communauté |
| [Apache Spark Users Mailing List](https://spark.apache.org/community.html) | Liste de diffusion officielle |
| [Databricks Community Edition](https://community.cloud.databricks.com/) | Cluster Spark gratuit dans le cloud |
| [GitHub — apache/spark](https://github.com/apache/spark) | Code source, issues, PRs |
| [Spark Summit / Data + AI Summit](https://databricks.com/dataaisummit/) | Conférence annuelle |

---

## Certifications

| Certification | Organisme | Description |
|---------------|-----------|-------------|
| [Databricks Certified Associate Developer for Apache Spark](https://www.databricks.com/learn/certification/apache-spark-developer-associate) | Databricks | Certification officielle Spark 3.x |
| [Cloudera Data Engineer](https://www.cloudera.com/about/training/certification/cloudera-certified-professional-data-engineer.html) | Cloudera | Inclut Spark sur CDH/CDP |

---

## Cheat Sheets

### Commandes `spark-submit` Essentielles

```bash
# Local (développement)
spark-submit --master local[4] --class com.MonApp app.jar

# YARN cluster
spark-submit \
  --master yarn --deploy-mode cluster \
  --driver-memory 4g --executor-memory 8g \
  --executor-cores 4 --num-executors 10 \
  --class com.MonApp app.jar

# Avec allocation dynamique
spark-submit \
  --master yarn --deploy-mode cluster \
  --conf spark.dynamicAllocation.enabled=true \
  --conf spark.dynamicAllocation.minExecutors=2 \
  --conf spark.dynamicAllocation.maxExecutors=100 \
  --class com.MonApp app.jar

# Kubernetes
spark-submit \
  --master k8s://https://k8s-api:6443 \
  --deploy-mode cluster \
  --conf spark.executor.instances=5 \
  --conf spark.kubernetes.container.image=my-spark:3.5.0 \
  local:///opt/spark/jars/app.jar
```

### Configurations de Performance Fréquentes

```properties
# Sérialisation (toujours activer)
spark.serializer=org.apache.spark.serializer.KryoSerializer

# Shuffles (adapter à la taille des données)
spark.sql.shuffle.partitions=200       # défaut, peut aller jusqu'à 2000+

# AQE (Spark 3.x — activer par défaut)
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
spark.sql.adaptive.skewJoin.enabled=true

# Broadcast join threshold
spark.sql.autoBroadcastJoinThreshold=50MB   # défaut: 10MB

# GC (pour les grand tas mémoire)
spark.executor.extraJavaOptions=-XX:+UseG1GC

# Allocation dynamique
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.minExecutors=2
spark.dynamicAllocation.maxExecutors=50
spark.dynamicAllocation.initialExecutors=5
```

### Import Java Spark Fréquents

```java
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.SaveMode;
import org.apache.spark.sql.types.*;
import org.apache.spark.sql.expressions.Window;
import org.apache.spark.storage.StorageLevel;
import org.apache.spark.broadcast.Broadcast;
import org.apache.spark.util.LongAccumulator;
import org.apache.spark.sql.streaming.StreamingQuery;
import org.apache.spark.sql.streaming.Trigger;

import static org.apache.spark.sql.functions.*;

// Interfaces fonctionnelles Java
import org.apache.spark.api.java.function.FilterFunction;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.ForeachFunction;
import org.apache.spark.api.java.function.ForeachPartitionFunction;
import org.apache.spark.api.java.function.ReduceFunction;

// UDFs
import org.apache.spark.sql.api.java.UDF1;
import org.apache.spark.sql.api.java.UDF2;
```

---

[← Retour au sommaire](README.md)
