# Apache Spark pour Java — Guide du Développeur

> Un référentiel complet couvrant les bonnes pratiques, la FAQ et les problèmes courants avec leurs solutions pour développer des pipelines de données avec Apache Spark en Java.

---

## Table des Matières

| # | Section | Description |
|---|---------|-------------|
| 1 | [Vue d'ensemble](01-vue-densemble.md) | Concepts fondamentaux, composants, API Java vs autres langages |
| 2 | [Installation & Maven](02-installation-maven.md) | Dépendances `pom.xml`, Fat JAR, `spark-submit` |
| 3 | [Architecture](03-architecture.md) | Driver, Executors, DAG, Cluster Managers |
| 4 | [Bonnes Pratiques](04-bonnes-pratiques.md) | 14 pratiques essentielles avec exemples de code |
| 5 | [FAQ](05-faq.md) | Questions fréquentes sur l'API Java Spark |
| 6 | [Problèmes & Solutions](06-problemes-solutions.md) | Diagnostic et résolution des erreurs courantes |
| 7 | [Ressources](07-ressources.md) | Liens vers la documentation officielle et ressources d'apprentissage |

---

## Démarrage Rapide

### Pré-requis

- Java 11+ (Java 17 recommandé pour Spark 3.5+)
- Maven 3.6+
- Apache Spark 3.5.x

### Exemple Minimal

```java
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

public class HelloSpark {
    public static void main(String[] args) {
        SparkSession spark = SparkSession.builder()
            .appName("HelloSpark")
            .master("local[*]")
            .getOrCreate();

        Dataset<Row> df = spark.read()
            .option("header", "true")
            .csv("donnees/exemple.csv");

        df.printSchema();
        df.show(10);

        spark.stop();
    }
}
```

### Commande `spark-submit`

```bash
spark-submit \
  --class com.exemple.HelloSpark \
  --master yarn \
  --deploy-mode cluster \
  --executor-memory 4g \
  --executor-cores 2 \
  target/mon-app-shaded.jar
```

---

## Navigation Rapide par Thème

### Performance
- [Mise en cache stratégique](04-bonnes-pratiques.md#7-mettre-en-cache-de-façon-stratégique)
- [Broadcast joins](04-bonnes-pratiques.md#8-utiliser-les-broadcast-joins-pour-les-petites-tables)
- [Optimisation des partitions](04-bonnes-pratiques.md#9-optimiser-le-nombre-de-partitions)
- [Configuration des ressources](04-bonnes-pratiques.md#14-configuration-des-ressources)

### Erreurs Courantes
- [Task Not Serializable](06-problemes-solutions.md#-problème--exception-task-not-serializable)
- [OutOfMemoryError Driver](06-problemes-solutions.md#-problème--outofmemoryerror-sur-le-driver)
- [Data Skew](06-problemes-solutions.md#-problème--job-lent--une-tâche-prend-beaucoup-plus-de-temps-déséquilibre--skew)

### Streaming
- [Structured Streaming](04-bonnes-pratiques.md#13-structured-streaming-en-java)
- [Requête streaming silencieuse](06-problemes-solutions.md#-problème--la-requête-streaming-sarrête-silencieusement)
