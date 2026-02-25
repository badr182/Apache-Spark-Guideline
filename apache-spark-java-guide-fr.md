# Apache Spark pour Java — Guide du Développeur

> Un référentiel complet couvrant les bonnes pratiques, la FAQ et les problèmes courants avec leurs solutions pour développer des pipelines de données avec Apache Spark en Java.

---

## Table des Matières

1. [Vue d'ensemble](#vue-densemble)
2. [Installation & Dépendances Maven](#installation--dépendances-maven)
3. [Architecture](#architecture)
4. [Bonnes Pratiques](#bonnes-pratiques)
5. [FAQ](#faq)
6. [Problèmes & Solutions](#problèmes--solutions)
7. [Ressources Supplémentaires](#ressources-supplémentaires)

---

## Vue d'ensemble

Apache Spark fournit une API Java complète et mature. Bien que Scala soit le langage natif de Spark, l'API Java couvre l'ensemble des fonctionnalités : Dataset/DataFrame, Spark SQL, Structured Streaming, MLlib et GraphX.

**Points clés de l'API Java :**
- `Dataset<Row>` est utilisé pour les DataFrames non typés (équivalent à un tableau SQL).
- `Dataset<T>` avec un `Encoder<T>` permet des opérations typées et sûres à la compilation.
- Les lambdas Java fonctionnent avec l'API fonctionnelle de Spark (`MapFunction`, `FilterFunction`, etc.).
- L'API Java est plus verbeuse que Scala ou Python, mais offre la sécurité du typage à la compilation.

**Composants principaux :**

| Composant | Rôle |
|-----------|------|
| Spark Core | Ordonnancement des tâches, gestion mémoire, I/O |
| Spark SQL | Données structurées via DataFrames et Datasets |
| Structured Streaming | Traitement de données en temps réel |
| MLlib | Machine learning distribué |
| GraphX | Calcul sur graphes |

---

## Installation & Dépendances Maven

### Fichier `pom.xml`

```xml
<properties>
    <spark.version>3.5.0</spark.version>
    <java.version>11</java.version>
</properties>

<dependencies>
    <!-- Spark Core -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.12</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope> <!-- "provided" lors du déploiement sur cluster -->
    </dependency>

    <!-- Spark SQL (inclut l'API DataFrame/Dataset) -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.12</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope>
    </dependency>

    <!-- Spark MLlib (optionnel) -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-mllib_2.12</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope>
    </dependency>

    <!-- Spark Structured Streaming (optionnel) -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.12</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

> **Remarque :** Utilisez `scope=provided` lors du déploiement sur un cluster (les JARs Spark sont déjà présents). Utilisez `scope=compile` pour le développement local ou les fat JARs.

### Créer un Fat JAR avec Maven Shade Plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.exemple.SparkApp</mainClass>
                    </transformer>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Soumettre au cluster :
```bash
spark-submit \
  --class com.exemple.SparkApp \
  --master yarn \
  --deploy-mode cluster \
  --executor-memory 8g \
  --executor-cores 4 \
  --num-executors 10 \
  target/mon-app-shaded.jar
```

---

## Architecture

```
┌─────────────────────────────────────────────┐
│             Driver (JVM)                    │
│  SparkSession / SparkContext                │
│  - Construit le DAG des transformations     │
│  - Coordonne avec le Cluster Manager        │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────▼──────────┐
        │   Cluster Manager   │  (YARN / Kubernetes / Standalone)
        └──────────┬──────────┘
         ┌─────────┼──────────┐
    ┌────▼───┐ ┌───▼────┐ ┌───▼────┐
    │Executor│ │Executor│ │Executor│
    │  JVM   │ │  JVM   │ │  JVM   │  ← exécutent les tâches, stockent le cache
    └────────┘ └────────┘ └────────┘
```

| Terme | Description |
|-------|-------------|
| **Driver** | Votre méthode `main()` Java ; orchestre le job |
| **Executor** | Processus JVM sur les nœuds workers ; exécute les tâches |
| **Tâche (Task)** | Unité de travail — une par partition |
| **Étape (Stage)** | Groupe de tâches sans frontière de shuffle |
| **Job** | Déclenché par une action (`count()`, `collect()`, `write()`) |
| **DAG** | Graphe orienté acyclique — le plan d'exécution logique |

---

## Bonnes Pratiques

### 1. Initialiser correctement la SparkSession

`SparkSession` est le point d'entrée unique depuis Spark 2.0. Il remplace `SparkContext`, `SQLContext` et `HiveContext`.

```java
import org.apache.spark.sql.SparkSession;

public class SparkApp {
    public static void main(String[] args) {

        SparkSession spark = SparkSession.builder()
            .appName("MonApplicationSpark")
            .master("local[*]")  // à retirer lors du déploiement sur cluster
            .config("spark.sql.shuffle.partitions", "200")
            .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
            .getOrCreate();

        try {
            executer(spark);
        } finally {
            spark.stop();  // toujours arrêter proprement
        }
    }

    private static void executer(SparkSession spark) {
        // votre logique ici
    }
}
```

> Appelez toujours `spark.stop()` dans un bloc `finally` pour libérer les ressources du cluster proprement.

---

### 2. Utiliser `Dataset<Row>` et `Dataset<T>` selon le besoin

```java
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoders;

// DataFrame non typé — Dataset<Row>
Dataset<Row> df = spark.read()
    .option("header", "true")
    .schema(schema)
    .csv("donnees/ventes.csv");

// Dataset typé — Dataset<MonBean>
// Le bean doit être Serializable avec getters/setters
Dataset<EnregistrementVente> ds = df.as(Encoders.bean(EnregistrementVente.class));

// Filtre typé
Dataset<EnregistrementVente> filtre = ds.filter(
    (FilterFunction<EnregistrementVente>) enreg -> enreg.getMontant() > 100.0
);
```

**Règle :** Utilisez `Dataset<Row>` pour la majorité des transformations ETL et des requêtes SQL. Utilisez `Dataset<T>` lorsque la sécurité de type Java est importante pour votre logique métier complexe.

---

### 3. Toujours Définir le Schéma Explicitement

Ne jamais utiliser `inferSchema` en production — cela déclenche un scan complet supplémentaire et peut produire des types incorrects.

```java
import org.apache.spark.sql.types.*;

StructType schema = new StructType(new StructField[]{
    DataTypes.createStructField("id",          DataTypes.LongType,    false),
    DataTypes.createStructField("nom",         DataTypes.StringType,  true),
    DataTypes.createStructField("montant",     DataTypes.DoubleType,  true),
    DataTypes.createStructField("date_event",  DataTypes.DateType,    true),
    DataTypes.createStructField("statut",      DataTypes.StringType,  true),
});

Dataset<Row> df = spark.read()
    .schema(schema)
    .option("header", "true")
    .csv("donnees/ventes.csv");
```

---

### 4. Utiliser l'API Column et les Fonctions Intégrées

Évitez les UDFs Java autant que possible. Les fonctions intégrées de Spark sont optimisées par le moteur Catalyst.

```java
import static org.apache.spark.sql.functions.*;

Dataset<Row> resultat = df
    .filter(col("statut").equalTo("actif").and(col("montant").gt(0)))
    .withColumn("remise",        col("montant").multiply(0.1))
    .withColumn("nom_propre",    lower(trim(col("nom"))))
    .withColumn("annee",         year(col("date_event")))
    .withColumn("libelle_complet", concat(col("nom"), lit(" - "), col("statut")))
    .select("id", "nom_propre", "montant", "remise", "annee");
```

**Fonctions couramment utilisées :**

```java
// Chaînes de caractères
lower(col("x")), upper(col("x")), trim(col("x")), length(col("x"))
concat(col("a"), lit("-"), col("b"))
substring(col("x"), 1, 5)
regexp_replace(col("x"), "\\s+", "_")

// Dates
year(col("date")), month(col("date")), dayofmonth(col("date"))
date_format(col("date"), "yyyy-MM-dd")
datediff(col("fin"), col("debut"))
to_date(col("str_date"), "yyyy-MM-dd")

// Conditionnelles
when(col("score").geq(90), lit("A"))
    .when(col("score").geq(80), lit("B"))
    .otherwise(lit("C"))

// Agrégations
sum("montant"), avg("montant"), count("*")
countDistinct(col("id")), max("montant"), min("montant")

// Gestion des nulls
col("x").isNull(), col("x").isNotNull()
coalesce(col("x"), col("y"), lit(0))
```

---

### 5. Écrire des UDFs Java (quand c'est nécessaire)

Si les fonctions intégrées ne couvrent pas votre logique, créez un UDF.

```java
import org.apache.spark.sql.api.java.UDF1;
import org.apache.spark.sql.types.DataTypes;

// Définir l'UDF
UDF1<String, String> masquerEmail = (String email) -> {
    if (email == null) return null;
    int indexAt = email.indexOf('@');
    if (indexAt <= 1) return email;
    return email.charAt(0) + "***" + email.substring(indexAt);
};

// Enregistrer auprès de la SparkSession
spark.udf().register("masquer_email", masquerEmail, DataTypes.StringType);

// Utiliser dans les opérations DataFrame
Dataset<Row> resultat = df.withColumn("email_masque",
    callUDF("masquer_email", col("email")));

// Utiliser en SQL
spark.sql("SELECT masquer_email(email) FROM utilisateurs");
```

> **Note de performance :** Les UDFs Java contournent l'optimisation Catalyst. Ne les utilisez que si les fonctions intégrées ne peuvent vraiment pas répondre au besoin.

---

### 6. Comprendre l'Évaluation Paresseuse (Lazy Evaluation)

Les transformations sont **paresseuses** — rien ne s'exécute avant qu'une **action** soit appelée.

```java
// Rien ne s'exécute ici — Spark construit seulement le DAG
Dataset<Row> filtre    = df.filter(col("statut").equalTo("actif"));
Dataset<Row> jointure  = filtre.join(clientsDf, "client_id");
Dataset<Row> agregation = jointure.groupBy("region").agg(sum("montant").alias("total"));

// L'exécution est déclenchée par une ACTION
long nombre = agregation.count();                         // action
agregation.show(20);                                      // action
agregation.write().parquet("sortie/resultat");            // action
```

> Chaque action ré-exécute toute la lignée depuis le début, sauf si des DataFrames intermédiaires sont mis en cache.

---

### 7. Mettre en Cache de Façon Stratégique

```java
import org.apache.spark.storage.StorageLevel;

// Mise en cache en mémoire avec débordement sur disque (recommandé par défaut)
Dataset<Row> jointureCouteuse = grandDf.join(moyenDf, "cle");
jointureCouteuse.persist(StorageLevel.MEMORY_AND_DISK());

// Déclencher le cache avec une action
jointureCouteuse.count();

// Réutiliser plusieurs fois sans recalcul
jointureCouteuse.filter(col("region").equalTo("EMEA")).show();
jointureCouteuse.filter(col("region").equalTo("APAC")).write().parquet("sortie/apac");

// Toujours libérer le cache quand ce n'est plus nécessaire
jointureCouteuse.unpersist();
```

**Niveaux de stockage disponibles :**

| Niveau | Description |
|--------|-------------|
| `MEMORY_ONLY()` | En mémoire JVM désérialisé — lecture rapide, haute consommation mémoire |
| `MEMORY_AND_DISK()` | Déborde sur disque si la mémoire est pleine — **bon choix par défaut** |
| `MEMORY_ONLY_SER()` | Sérialisé — moins de mémoire, plus de CPU |
| `DISK_ONLY()` | Sur disque uniquement — pour les très grandes données |
| `MEMORY_AND_DISK_2()` | Répliqué sur 2 nœuds — pour la tolérance aux pannes |

**Quand mettre en cache :**
- Le DataFrame est utilisé dans plusieurs actions dans le même job.
- Algorithmes itératifs (boucles d'entraînement MLlib).
- Après des jointures ou agrégations coûteuses réutilisées en aval.

**Quand NE PAS mettre en cache :**
- DataFrames utilisés une seule fois.
- Données qui tiennent facilement en un seul scan.

---

### 8. Utiliser les Broadcast Joins pour les Petites Tables

```java
import static org.apache.spark.sql.functions.broadcast;

// Forcer une broadcast join pour une petite table de référence
Dataset<Row> resultat = grandDf.join(broadcast(petiteTableRef), "produit_id");

// Seuil de broadcast automatique (10 Mo par défaut)
spark.conf().set("spark.sql.autoBroadcastJoinThreshold",
    String.valueOf(50 * 1024 * 1024)); // 50 Mo

// Désactiver le broadcast automatique (utile pour le débogage)
spark.conf().set("spark.sql.autoBroadcastJoinThreshold", "-1");
```

---

### 9. Optimiser le Nombre de Partitions

```java
// Vérifier le nombre de partitions actuel
int nbPartitions = df.rdd().getNumPartitions();
System.out.println("Nombre de partitions : " + nbPartitions);

// Réduire les partitions sans shuffle (après un filtre)
Dataset<Row> petit = df.coalesce(20);

// Repartitionner avec shuffle (pour équilibrer la charge)
Dataset<Row> equilibre = df.repartition(100);

// Repartitionner par colonne (pour les écritures partitionnées)
Dataset<Row> parPays = df.repartition(col("pays"));

// Configurer les partitions de shuffle globalement
spark.conf().set("spark.sql.shuffle.partitions", "400");

// Ou activer AQE pour un réglage automatique (Spark 3.x)
spark.conf().set("spark.sql.adaptive.enabled", "true");
spark.conf().set("spark.sql.adaptive.coalescePartitions.enabled", "true");
```

**Règle générale :** visez des partitions de 100 Mo à 200 Mo. Utilisez `df.rdd().getNumPartitions()` pour inspecter.

---

### 10. Lire et Écrire des Données

```java
// ===== LECTURE =====

// Parquet (format recommandé)
Dataset<Row> parquetDf = spark.read().parquet("donnees/ventes/");

// CSV avec toutes les options
Dataset<Row> csvDf = spark.read()
    .schema(schema)
    .option("header", "true")
    .option("delimiter", ";")
    .option("nullValue", "NULL")
    .option("dateFormat", "dd/MM/yyyy")
    .option("mode", "DROPMALFORMED") // PERMISSIVE, DROPMALFORMED, FAILFAST
    .csv("donnees/fichier.csv");

// JSON
Dataset<Row> jsonDf = spark.read()
    .schema(schema)
    .json("donnees/evenements/");

// JDBC (base de données relationnelle)
Dataset<Row> jdbcDf = spark.read()
    .format("jdbc")
    .option("url", "jdbc:postgresql://hote:5432/mabase")
    .option("dbtable", "public.commandes")
    .option("user", "utilisateur")
    .option("password", "motdepasse")
    .option("numPartitions", "10")
    .option("partitionColumn", "id")
    .option("lowerBound", "1")
    .option("upperBound", "10000000")
    .load();

// ===== ÉCRITURE =====

// Parquet avec partitionnement
resultat.write()
    .mode(SaveMode.Overwrite)      // Overwrite, Append, Ignore, ErrorIfExists
    .partitionBy("annee", "mois")
    .option("compression", "snappy")
    .parquet("sortie/ventes/");

// JDBC
resultat.write()
    .format("jdbc")
    .option("url", "jdbc:postgresql://hote:5432/mabase")
    .option("dbtable", "public.resultats")
    .option("user", "utilisateur")
    .option("password", "motdepasse")
    .mode(SaveMode.Append)
    .save();
```

---

### 11. Utiliser Spark SQL pour la Lisibilité

```java
// Enregistrer les DataFrames comme vues temporaires
df.createOrReplaceTempView("ventes");
clientsDf.createOrReplaceTempView("clients");

// Exécuter une requête SQL
Dataset<Row> resultat = spark.sql(
    "SELECT c.region, " +
    "       SUM(v.montant)             AS total_ventes, " +
    "       COUNT(DISTINCT v.client_id) AS clients_uniques " +
    "FROM ventes v " +
    "JOIN clients c ON v.client_id = c.id " +
    "WHERE v.statut = 'termine' " +
    "  AND v.date_event >= '2024-01-01' " +
    "GROUP BY c.region " +
    "ORDER BY total_ventes DESC"
);

resultat.show();
```

---

### 12. Gérer les Valeurs Nulles Explicitement

Spark ne plante pas sur les nulls — ils se propagent silencieusement. Traitez-les délibérément.

```java
import java.util.HashMap;
import java.util.Map;

// Supprimer les lignes contenant au moins un null
Dataset<Row> propre = df.na().drop();

// Supprimer seulement si certaines colonnes sont nulles
Dataset<Row> propre2 = df.na().drop(new String[]{"id", "montant"});

// Remplir les nulls avec des valeurs par défaut
Map<String, Object> valeurs = new HashMap<>();
valeurs.put("montant", 0.0);
valeurs.put("statut", "inconnu");
valeurs.put("nom", "N/A");
Dataset<Row> rempli = df.na().fill(valeurs);

// Gestion conditionnelle avec coalesce
Dataset<Row> avecDefaut = df.withColumn("montant",
    coalesce(col("montant"), lit(0.0)));
```

---

### 13. Structured Streaming en Java

```java
import org.apache.spark.sql.streaming.StreamingQuery;
import org.apache.spark.sql.streaming.Trigger;

// Lire depuis Kafka
Dataset<Row> flux = spark.readStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "topic-evenements")
    .option("startingOffsets", "latest")
    .load();

// Analyser le payload JSON
Dataset<Row> parse = flux
    .selectExpr("CAST(value AS STRING) as json_str")
    .select(from_json(col("json_str"), schema).alias("data"))
    .select("data.*");

// Agrégation par fenêtre temporelle
Dataset<Row> agrege = parse
    .withWatermark("temps_event", "10 minutes")
    .groupBy(
        window(col("temps_event"), "5 minutes"),
        col("region")
    )
    .agg(sum("montant").alias("total"));

// Écrire le flux
StreamingQuery requete = agrege.writeStream()
    .outputMode("append")
    .format("parquet")
    .option("path", "sortie/flux")
    .option("checkpointLocation", "checkpoints/ma-requete")
    .trigger(Trigger.ProcessingTime("30 seconds"))
    .start();

try {
    requete.awaitTermination();
} catch (Exception e) {
    e.printStackTrace();
    requete.stop();
}
```

---

### 14. Configuration des Ressources

```bash
spark-submit \
  --class com.exemple.SparkApp \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 4g \
  --executor-memory 8g \
  --executor-cores 4 \
  --num-executors 20 \
  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
  --conf spark.sql.shuffle.partitions=400 \
  --conf spark.sql.adaptive.enabled=true \
  --conf spark.dynamicAllocation.enabled=true \
  --conf spark.dynamicAllocation.minExecutors=2 \
  --conf spark.dynamicAllocation.maxExecutors=50 \
  --conf "spark.executor.extraJavaOptions=-XX:+UseG1GC" \
  target/mon-app-shaded.jar
```

**Répartition de la mémoire par executor :**
```
spark.executor.memory (ex: 8g)
  ├── Mémoire unifiée (60%) → exécution (jointures, tris) + stockage (cache)
  └── Réservé / Utilisateur (40%) → overhead JVM, structures de données

spark.executor.memoryOverhead (ex: 2g) → hors tas JVM : metaspace, bibliothèques natives
```

---

## FAQ

**Q : Quelle est la différence entre une transformation et une action ?**

Les transformations (`filter`, `map`, `join`, `groupBy`) sont **paresseuses** — elles définissent un plan de calcul (DAG) sans rien exécuter. Les actions (`count`, `collect`, `show`, `write`) déclenchent l'exécution réelle. À chaque action, Spark ré-exécute toute la lignée depuis la source, sauf si des résultats intermédiaires sont mis en cache.

---

**Q : Qu'est-ce qu'un shuffle et pourquoi est-il coûteux ?**

Un shuffle se produit quand Spark doit redistribuer les données entre partitions — typiquement lors d'un `groupBy`, `join`, `distinct`, `repartition` ou `orderBy`. Il implique l'écriture sur disque et des transferts réseau entre les nœuds, ce qui en fait l'opération la plus coûteuse dans Spark. Minimisez les shuffles inutiles et ajustez `spark.sql.shuffle.partitions` selon la taille de vos données.

---

**Q : Quelle est la différence entre `coalesce()` et `repartition()` ?**

`coalesce(n)` réduit le nombre de partitions **sans shuffle complet** — c'est efficace mais peut créer des partitions déséquilibrées. À utiliser après avoir filtré un grand jeu de données.

`repartition(n)` effectue un **shuffle complet** pour créer `n` partitions équilibrées. À utiliser lorsque vous avez besoin d'un parallélisme équilibré ou d'un partitionnement par colonne spécifique.

---

**Q : Comment créer un `Dataset<T>` à partir d'un bean Java ?**

Votre bean doit être `Serializable`, avoir un constructeur sans argument, et des getters/setters publics.

```java
import java.io.Serializable;

public class EnregistrementVente implements Serializable {
    private long id;
    private String region;
    private double montant;

    public EnregistrementVente() {}

    public long getId()             { return id; }
    public void setId(long id)      { this.id = id; }
    public String getRegion()       { return region; }
    public void setRegion(String r) { this.region = r; }
    public double getMontant()      { return montant; }
    public void setMontant(double m){ this.montant = m; }
}

// Convertir un DataFrame en Dataset typé
Encoder<EnregistrementVente> encodeur = Encoders.bean(EnregistrementVente.class);
Dataset<EnregistrementVente> ds = df.as(encodeur);
```

---

**Q : Comment effectuer une jointure en Java ?**

```java
// Jointure interne (par défaut)
Dataset<Row> joint = commandes.join(clients,
    commandes.col("client_id").equalTo(clients.col("id")));

// Spécifier le type de jointure
Dataset<Row> gaucheExt = commandes.join(clients,
    commandes.col("client_id").equalTo(clients.col("id")),
    "left_outer"); // inner, left_outer, right_outer, full_outer, left_semi, left_anti

// Jointure sur une colonne de même nom (les colonnes sont fusionnées)
Dataset<Row> simple = commandes.join(clients, "client_id");

// Éviter l'ambiguïté après jointure
Dataset<Row> propre = joint
    .drop(clients.col("id"))
    .withColumnRenamed("nom", "nom_client");
```

---

**Q : Comment utiliser les fonctions de fenêtre (Window Functions) en Java ?**

```java
import org.apache.spark.sql.expressions.Window;
import org.apache.spark.sql.expressions.WindowSpec;

WindowSpec fenetre = Window
    .partitionBy("region")
    .orderBy(col("date_event").desc());

Dataset<Row> resultat = df
    .withColumn("rang",           rank().over(fenetre))
    .withColumn("num_ligne",      row_number().over(fenetre))
    .withColumn("total_cumulé",   sum("montant").over(
        fenetre.rowsBetween(Window.unboundedPreceding(), Window.currentRow())))
    .withColumn("montant_prec",   lag("montant", 1).over(fenetre))
    .withColumn("montant_suiv",   lead("montant", 1).over(fenetre));
```

---

**Q : Qu'est-ce que l'AQE et comment l'activer ?**

L'AQE (Adaptive Query Execution, Spark 3.0+) ré-optimise le plan de requête à l'exécution en se basant sur les statistiques réelles collectées. Il peut automatiquement fusionner de petites partitions de shuffle, changer de stratégie de jointure et gérer le déséquilibre des données.

```java
SparkSession spark = SparkSession.builder()
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
    .config("spark.sql.adaptive.skewJoin.enabled", "true")
    .getOrCreate();
```

---

**Q : Comment utiliser les accumulateurs et les variables broadcast en Java ?**

```java
import org.apache.spark.util.LongAccumulator;
import org.apache.spark.broadcast.Broadcast;

// Accumulateur — compteur partagé entre les tâches
LongAccumulator nbErreurs = spark.sparkContext().longAccumulator("nbErreurs");

df.foreach((ForeachFunction<Row>) ligne -> {
    if ("ERREUR".equals(ligne.getAs("statut"))) {
        nbErreurs.add(1);
    }
});
System.out.println("Total erreurs : " + nbErreurs.value());

// Variable broadcast — partager des données en lecture seule efficacement
Map<String, String> tableRef = chargerTableReference();
Broadcast<Map<String, String>> broadcastRef =
    spark.sparkContext().broadcast(tableRef,
        scala.reflect.ClassTag$.MODULE$.apply(Map.class));

Dataset<Row> resultat = df.map(
    (MapFunction<Row, Row>) ligne -> {
        String cle = ligne.getAs("code");
        String valeur = broadcastRef.value().getOrDefault(cle, "inconnu");
        return RowFactory.create(ligne.getAs("id"), valeur);
    },
    RowEncoder.apply(schemaResultat)
);
```

---

**Q : Comment inspecter le schéma et les données d'un DataFrame ?**

```java
// Afficher le schéma
df.printSchema();

// Afficher les 20 premières lignes (défaut)
df.show();
df.show(50, false);   // 50 lignes, ne tronque pas les longues chaînes

// Compter les lignes
long nbLignes = df.count();

// Statistiques basiques
df.describe("montant", "score").show();

// Valeurs distinctes
df.select("statut").distinct().show();

// Taille des partitions
df.groupBy(spark_partition_id().alias("partition_id"))
  .count()
  .orderBy(col("count").desc())
  .show();
```

---

## Problèmes & Solutions

### ❌ Problème : Exception `Task Not Serializable`

**Symptôme :** `org.apache.spark.SparkException: Task not serializable`

**Cause :** Un objet non sérialisable (logger, connexion BD, classe de service) est capturé dans un lambda envoyé aux executors.

**Solution :**

```java
// ❌ Incorrect — le Logger est capturé dans le lambda, non sérialisable
Logger log = LoggerFactory.getLogger(MonApp.class);
df.foreach((ForeachFunction<Row>) ligne -> {
    log.info(ligne.toString());  // capture du logger → non sérialisable
});

// ✅ Créer à l'intérieur du lambda (par tâche)
df.foreach((ForeachFunction<Row>) ligne -> {
    Logger log = LoggerFactory.getLogger(MonApp.class); // créé dans l'executor
    log.info(ligne.toString());
});

// ❌ Incorrect — connexion BD créée dans le driver, capturée dans le lambda
Connection conn = DriverManager.getConnection(url);
df.foreach((ForeachFunction<Row>) ligne -> conn.execute(ligne));

// ✅ Utiliser foreachPartition — une connexion par partition, pas par ligne
df.foreachPartition((ForeachPartitionFunction<Row>) lignes -> {
    Connection conn = DriverManager.getConnection(url);
    while (lignes.hasNext()) {
        conn.execute(lignes.next());
    }
    conn.close();
});
```

---

### ❌ Problème : `OutOfMemoryError` sur le Driver

**Symptôme :** `java.lang.OutOfMemoryError: Java heap space` dans le processus driver.

**Cause :** Appel de `collect()` ou `collectAsList()` sur un grand Dataset.

**Solution :**

```java
// ❌ Dangereux sur de grandes données
List<Row> toutesLignes = df.collectAsList();

// ✅ Écrire dans le stockage
df.write().mode(SaveMode.Overwrite).parquet("sortie/resultat");

// ✅ Échantillonner uniquement ce dont vous avez besoin
List<Row> echantillon = df.limit(1000).collectAsList();

// ✅ Utiliser show() pour le débogage — ne ramène que N lignes
df.show(20);

// Augmenter la mémoire du driver dans spark-submit
// --driver-memory 8g
```

---

### ❌ Problème : `OutOfMemoryError` sur les Executors

**Symptôme :** Erreurs OOM dans les executors ; les tâches échouent et se relancent.

**Solutions :**

```bash
# Augmenter la mémoire des executors
--executor-memory 16g
--conf spark.executor.memoryOverhead=4g

# Utiliser le GC G1 pour les grands tas mémoire
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:InitiatingHeapOccupancyPercent=35"
```

```java
// Réduire la taille des partitions (moins de lignes par tâche)
spark.conf().set("spark.sql.shuffle.partitions", "800");

// Utiliser un stockage sérialisé pour réduire la pression mémoire
df.persist(StorageLevel.MEMORY_AND_DISK_SER());

// Libérer le cache des DataFrames inutilisés
df.unpersist();
```

---

### ❌ Problème : Job Lent — Une Tâche Prend Beaucoup Plus de Temps (Déséquilibre / Skew)

**Symptôme :** Dans l'interface Spark, la plupart des tâches se terminent en secondes, mais 1 ou 2 tâches tournent pendant des minutes.

**Solution :**

```java
// Activer la gestion automatique du skew via AQE (Spark 3.x)
spark.conf().set("spark.sql.adaptive.enabled", "true");
spark.conf().set("spark.sql.adaptive.skewJoin.enabled", "true");
spark.conf().set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB");

// Solution manuelle : salage (salting) de la clé de jointure déséquilibrée
import static org.apache.spark.sql.functions.*;

int nbSels = 50;

// Saler la grande table
Dataset<Row> grandeSalee = grandDf
    .withColumn("sel",
        (floor(rand().multiply(nbSels))).cast(DataTypes.StringType))
    .withColumn("cle_salee",
        concat(col("cle_jointure"), lit("_"), col("sel")));

// Éclater la petite table pour correspondre à tous les sels
Column tableauSels = array(java.util.stream.IntStream.range(0, nbSels)
    .mapToObj(i -> lit(String.valueOf(i)))
    .toArray(Column[]::new));

Dataset<Row> petiteSalee = petitDf
    .withColumn("sel", explode(tableauSels))
    .withColumn("cle_salee",
        concat(col("cle_jointure"), lit("_"), col("sel")));

Dataset<Row> resultat = grandeSalee.join(petiteSalee, "cle_salee");
```

---

### ❌ Problème : Colonne Ambiguë Après Jointure

**Symptôme :** `AnalysisException: Reference 'nom_colonne' is ambiguous`

**Solution :**

```java
// ❌ Ambigu — les deux DataFrames ont une colonne "id"
Dataset<Row> joint = commandes.join(clients,
    commandes.col("client_id").equalTo(clients.col("id")));
joint.select("id"); // lequel des deux "id" ?

// ✅ Utiliser la référence du DataFrame pour lever l'ambiguïté
joint.select(commandes.col("id"), clients.col("id").alias("id_client"));

// ✅ Ou supprimer la colonne dupliquée après la jointure
Dataset<Row> propre = joint.drop(clients.col("id"));

// ✅ Ou utiliser des alias avant la jointure
Dataset<Row> c = commandes.alias("c");
Dataset<Row> cl = clients.alias("cl");
Dataset<Row> joint2 = c.join(cl, col("c.client_id").equalTo(col("cl.id")));
joint2.select("c.id", "cl.nom");
```

---

### ❌ Problème : Trop de Petits Fichiers en Sortie

**Symptôme :** Le répertoire de sortie contient des milliers de petits fichiers (quelques Ko chacun).

**Cause :** Trop de partitions au moment de l'écriture.

**Solution :**

```java
// Réduire les partitions sans shuffle (recommandé après filtrage)
df.coalesce(20).write()
    .mode(SaveMode.Overwrite)
    .parquet("sortie/resultat");

// Repartitionner avec shuffle pour un équilibrage parfait
df.repartition(50).write()
    .mode(SaveMode.Overwrite)
    .parquet("sortie/resultat");

// Limiter le nombre d'enregistrements par fichier de sortie
df.write()
    .option("maxRecordsPerFile", 1_000_000)
    .parquet("sortie/resultat");
```

---

### ❌ Problème : `AnalysisException` — Colonne Introuvable

**Symptôme :** `AnalysisException: Resolved attribute(s) missing from ...`

**Causes et solutions :**

```java
// ❌ Référencer une colonne d'un ancien DataFrame
Dataset<Row> df2 = df.filter(col("statut").equalTo("actif"));
Dataset<Row> df3 = df2.select(df.col("nom")); // référence à l'ancien df !

// ✅ Référencer le DataFrame courant
Dataset<Row> df3 = df2.select(df2.col("nom"));
// ou simplement
Dataset<Row> df3 = df2.select("nom");

// ❌ Colonne créée dynamiquement non reconnue dans la même expression
// ✅ Chaîner les withColumn séparément — cela fonctionne correctement
df.withColumn("x", col("a").plus(col("b")))
  .withColumn("y", col("x").multiply(2)); // "x" est disponible ici
```

---

### ❌ Problème : `GC Overhead Limit Exceeded` sur les Executors

**Symptôme :** Pauses fréquentes du garbage collector ; les tâches échouent avec des erreurs GC.

**Solutions :**

```bash
# Passer au GC G1 (meilleur pour les grands tas)
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:G1HeapRegionSize=16m"

# Utiliser le sérialiseur Kryo (plus rapide et plus compact)
--conf spark.serializer=org.apache.spark.serializer.KryoSerializer
```

```java
// Utiliser le cache sérialisé pour réduire la pression mémoire
df.persist(StorageLevel.MEMORY_AND_DISK_SER());

// Augmenter le nombre de partitions pour réduire les objets par tâche
spark.conf().set("spark.sql.shuffle.partitions", "800");

// Enregistrer les classes métier avec Kryo
spark.conf().set("spark.kryo.classesToRegister",
    "com.exemple.EnregistrementVente,com.exemple.Client");
```

---

### ❌ Problème : Un UDF Retourne Toujours Null

**Symptôme :** L'UDF est enregistré et appelé correctement, mais la colonne de sortie est toujours null.

**Causes courantes :**

```java
// ❌ Type de retour déclaré incorrect — déclaré IntegerType mais retourne Long
UDF1<String, Integer> monUdf = (String s) -> s.length();
spark.udf().register("mon_udf", monUdf, DataTypes.LongType); // incompatibilité de type !

// ✅ Faire correspondre exactement le type déclaré
spark.udf().register("mon_udf", monUdf, DataTypes.IntegerType);

// ❌ L'UDF lance une exception sur les valeurs nulles — retourne null silencieusement
UDF1<String, String> mauvaisUdf = (String s) -> s.toUpperCase(); // NullPointerException si s est null !

// ✅ Toujours gérer les entrées nulles dans les UDFs
UDF1<String, String> bonUdf = (String s) -> {
    if (s == null) return null;
    return s.toUpperCase();
};
```

---

### ❌ Problème : La Requête Streaming S'arrête Silencieusement

**Symptôme :** La requête Structured Streaming se termine sans message d'erreur clair.

**Solution :**

```java
StreamingQuery requete = df.writeStream()
    .format("parquet")
    .option("path", "sortie/flux")
    .option("checkpointLocation", "checkpoints/requete1") // OBLIGATOIRE
    .start();

// Toujours attendre avec gestion des exceptions
try {
    requete.awaitTermination();
} catch (StreamingQueryException e) {
    System.err.println("Requête échouée : " + e.getMessage());
    e.printStackTrace();
} finally {
    spark.stop();
}

// Vérifier l'état de la requête
if (!requete.isActive()) {
    Optional<Throwable> exception =
        Optional.ofNullable(requete.exception().getOrElse(null));
    exception.ifPresent(Throwable::printStackTrace);
}

// Surveiller la progression
System.out.println(requete.lastProgress());
System.out.println(requete.status());
```

---

### ❌ Problème : `java.io.IOException: No space left on device`

**Symptôme :** Les executors échouent avec des erreurs de disque plein pendant les shuffles.

**Solutions :**

```bash
# Changer les répertoires de shuffle vers des disques avec plus d'espace
--conf spark.local.dir=/mnt/disque-rapide/tmp,/mnt/disque2/tmp

# Activer la compression du shuffle
--conf spark.shuffle.compress=true
--conf spark.io.compression.codec=lz4

# Nettoyage plus agressif des données de shuffle anciennes
--conf spark.cleaner.periodicGC.interval=1min
```

---

## Ressources Supplémentaires

- [Documentation officielle Apache Spark](https://spark.apache.org/docs/latest/)
- [Javadoc API Spark Java](https://spark.apache.org/docs/latest/api/java/index.html)
- [Spark: The Definitive Guide (O'Reilly)](https://www.oreilly.com/library/view/spark-the-definitive/9781491912201/)
- [Guide de Tuning des Performances Spark](https://spark.apache.org/docs/latest/tuning.html)
- [Documentation AQE](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution)
- [Guide Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Databricks Academy (cours gratuits)](https://www.databricks.com/learn/training/catalog)
