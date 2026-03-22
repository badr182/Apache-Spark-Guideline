# Bonnes Pratiques — Apache Spark Java

> [← Retour au sommaire](README.md)

---

## 1. Initialiser correctement la SparkSession

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
            .config("spark.sql.adaptive.enabled", "true")
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

### Pattern Factory (recommandé pour les grands projets)

```java
public class SparkSessionFactory {

    private SparkSessionFactory() {}

    public static SparkSession create(String appName, boolean local) {
        SparkSession.Builder builder = SparkSession.builder()
            .appName(appName)
            .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
            .config("spark.sql.adaptive.enabled", "true")
            .config("spark.sql.adaptive.coalescePartitions.enabled", "true");

        if (local) {
            builder.master("local[*]")
                   .config("spark.ui.enabled", "false");
        }

        return builder.getOrCreate();
    }
}
```

---

## 2. Utiliser `Dataset<Row>` et `Dataset<T>` selon le besoin

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

## 3. Toujours Définir le Schéma Explicitement

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

### Centraliser les Schémas

```java
// SchemaUtils.java — un seul endroit pour tous les schémas
public class SchemaUtils {

    public static final StructType VENTES_SCHEMA = new StructType()
        .add("id",         DataTypes.LongType,   false)
        .add("nom",        DataTypes.StringType, true)
        .add("montant",    DataTypes.DoubleType, true)
        .add("date_event", DataTypes.DateType,   true)
        .add("statut",     DataTypes.StringType, true);

    public static final StructType CLIENTS_SCHEMA = new StructType()
        .add("id",         DataTypes.LongType,   false)
        .add("region",     DataTypes.StringType, true)
        .add("segment",    DataTypes.StringType, true);
}
```

---

## 4. Utiliser l'API Column et les Fonctions Intégrées

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
split(col("x"), ",")          // retourne ArrayType
array_contains(col("arr"), "val")

// Dates
year(col("date")), month(col("date")), dayofmonth(col("date"))
date_format(col("date"), "yyyy-MM-dd")
datediff(col("fin"), col("debut"))
to_date(col("str_date"), "yyyy-MM-dd")
date_add(col("date"), 7)      // ajouter 7 jours
trunc(col("date"), "month")   // tronquer au mois

// Conditionnelles
when(col("score").geq(90), lit("A"))
    .when(col("score").geq(80), lit("B"))
    .otherwise(lit("C"))

// Agrégations
sum("montant"), avg("montant"), count("*")
countDistinct(col("id")), max("montant"), min("montant")
collect_list(col("x"))        // agrège en Array
first(col("x"), true)         // premier non-null

// Gestion des nulls
col("x").isNull(), col("x").isNotNull()
coalesce(col("x"), col("y"), lit(0))
nvl(col("x"), lit("defaut"))  // alias de coalesce pour 2 args

// Tableaux et Maps
explode(col("array_col"))     // une ligne par élément
array(col("a"), col("b"))     // crée un tableau
map_keys(col("map_col"))
element_at(col("array_col"), 1)
```

---

## 5. Écrire des UDFs Java (quand c'est nécessaire)

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

## 6. Comprendre l'Évaluation Paresseuse (Lazy Evaluation)

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

### Éviter les Actions Multiples sur le Même DataFrame

```java
// ❌ Mauvais — la lignée est calculée deux fois
long total = df.count();
df.write().parquet("sortie/");

// ✅ Bon — mettre en cache avant de déclencher plusieurs actions
df.cache();
long total = df.count();  // déclenche le calcul et met en cache
df.write().parquet("sortie/");  // utilise le cache
df.unpersist();
```

---

## 7. Mettre en Cache de Façon Stratégique

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
| `MEMORY_ONLY_SER()` | Sérialisé (Kryo recommandé) — moins de mémoire, plus de CPU |
| `DISK_ONLY()` | Sur disque uniquement — pour les très grandes données |
| `MEMORY_AND_DISK_2()` | Répliqué sur 2 nœuds — pour la tolérance aux pannes |
| `OFF_HEAP()` | Hors du tas JVM — réduit la pression GC (nécessite configuration) |

**Quand mettre en cache :**
- Le DataFrame est utilisé dans plusieurs actions dans le même job.
- Algorithmes itératifs (boucles d'entraînement MLlib).
- Après des jointures ou agrégations coûteuses réutilisées en aval.

**Quand NE PAS mettre en cache :**
- DataFrames utilisés une seule fois.
- Données qui tiennent facilement en un seul scan.
- Si la mémoire est limitée et que le spill sur disque serait trop coûteux.

---

## 8. Utiliser les Broadcast Joins pour les Petites Tables

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

**Quand broadcaster :**
- Tables de référence (pays, codes postaux, produits)
- Dimensions dans un schéma étoile
- Tables < 10 MB (par défaut) ou < 50-100 MB selon la RAM des executors

**Quand NE PAS broadcaster :**
- Tables de plusieurs GB
- Jointures répétées où la table évolue entre les joins

---

## 9. Optimiser le Nombre de Partitions

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

// Repartitionner par colonne avec un nombre fixe de partitions
Dataset<Row> parPaysFixe = df.repartition(200, col("pays"));

// Configurer les partitions de shuffle globalement
spark.conf().set("spark.sql.shuffle.partitions", "400");

// Ou activer AQE pour un réglage automatique (Spark 3.x)
spark.conf().set("spark.sql.adaptive.enabled", "true");
spark.conf().set("spark.sql.adaptive.coalescePartitions.enabled", "true");
```

**Règle générale :** visez des partitions de 100 Mo à 200 Mo. Utilisez `df.rdd().getNumPartitions()` pour inspecter.

| Taille des données | Partitions recommandées |
|--------------------|------------------------|
| < 1 GB | 10-50 |
| 1-100 GB | 100-500 |
| 100 GB - 1 TB | 500-2000 |
| > 1 TB | 2000-10000 |

---

## 10. Lire et Écrire des Données

```java
// ===== LECTURE =====

// Parquet (format recommandé — columnar, compressé, schéma intégré)
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

// ORC
Dataset<Row> orcDf = spark.read().orc("donnees/orc/");

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

// Delta Lake (si disponible)
Dataset<Row> deltaDf = spark.read().format("delta").load("donnees/delta/");

// ===== ÉCRITURE =====

// Parquet avec partitionnement
resultat.write()
    .mode(SaveMode.Overwrite)      // Overwrite, Append, Ignore, ErrorIfExists
    .partitionBy("annee", "mois")
    .option("compression", "snappy")
    .parquet("sortie/ventes/");

// Parquet avec contrôle de la taille des fichiers
resultat.write()
    .option("maxRecordsPerFile", 1_000_000)
    .option("compression", "zstd")  // meilleure compression que snappy
    .parquet("sortie/ventes/");

// JDBC
resultat.write()
    .format("jdbc")
    .option("url", "jdbc:postgresql://hote:5432/mabase")
    .option("dbtable", "public.resultats")
    .option("user", "utilisateur")
    .option("password", "motdepasse")
    .option("batchsize", "10000")  // optimiser les inserts en batch
    .mode(SaveMode.Append)
    .save();

// Delta Lake (ACID, time travel, merge)
resultat.write()
    .format("delta")
    .mode(SaveMode.Overwrite)
    .partitionBy("annee")
    .save("sortie/delta/ventes/");
```

### Formats de Fichiers : Comparaison

| Format | Compression | Schema | Columnar | Lecture Partielle | Usage |
|--------|-------------|--------|----------|-------------------|-------|
| **Parquet** | Oui (snappy, zstd) | Oui | Oui | Oui | ETL, analytique |
| **ORC** | Oui | Oui | Oui | Oui | Hive, Hadoop |
| **Delta** | Oui | Oui | Oui | Oui | ACID, upserts |
| **Avro** | Oui | Oui | Non | Non | Kafka, streaming |
| **CSV** | Non | Non | Non | Non | Interop, debug |
| **JSON** | Non | Non | Non | Non | APIs, debug |

---

## 11. Utiliser Spark SQL pour la Lisibilité

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

// Vue globale (accessible depuis plusieurs SparkSessions)
df.createOrReplaceGlobalTempView("ventes_global");
spark.sql("SELECT * FROM global_temp.ventes_global LIMIT 10").show();
```

---

## 12. Gérer les Valeurs Nulles Explicitement

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

// Remplacer des valeurs spécifiques (ex: chaîne vide → null)
Dataset<Row> normalise = df
    .withColumn("nom", when(col("nom").equalTo(""), null).otherwise(col("nom")))
    .withColumn("montant", when(col("montant").lt(0), lit(0.0)).otherwise(col("montant")));
```

---

## 13. Structured Streaming en Java

```java
import org.apache.spark.sql.streaming.StreamingQuery;
import org.apache.spark.sql.streaming.Trigger;

// Lire depuis Kafka
Dataset<Row> flux = spark.readStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "topic-evenements")
    .option("startingOffsets", "latest")
    .option("maxOffsetsPerTrigger", "10000")  // contrôler le débit
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
    .outputMode("append")           // append, update, complete
    .format("parquet")
    .option("path", "sortie/flux")
    .option("checkpointLocation", "checkpoints/ma-requete")  // OBLIGATOIRE
    .trigger(Trigger.ProcessingTime("30 seconds"))
    // ou : Trigger.Once()     → traiter tout ce qui est disponible, une seule fois
    // ou : Trigger.Continuous("1 second") → latence ultra-faible (expérimental)
    .start();

try {
    requete.awaitTermination();
} catch (Exception e) {
    e.printStackTrace();
    requete.stop();
}
```

### Modes de Sortie (Output Modes)

| Mode | Description | Quand utiliser |
|------|-------------|----------------|
| `append` | Seules les nouvelles lignes finalisées sont écrites | Agrégations avec watermark, pas de groupBy |
| `update` | Seules les lignes modifiées depuis le dernier batch | Agrégations sans watermark |
| `complete` | Toute la table de résultats est écrite à chaque batch | Petits agrégats qui tiennent en mémoire |

---

## 14. Configuration des Ressources

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
  --conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:InitiatingHeapOccupancyPercent=35" \
  target/mon-app-shaded.jar
```

**Répartition de la mémoire par executor :**
```
spark.executor.memory (ex: 8g)
  ├── Mémoire unifiée (60%) → exécution (jointures, tris) + stockage (cache)
  └── Réservé / Utilisateur (40%) → overhead JVM, structures de données

spark.executor.memoryOverhead (ex: 2g) → hors tas JVM : metaspace, bibliothèques natives
```

### Règle de dimensionnement (executor cores)

```
# Recommandation : 4-5 cores par executor
# Éviter 1 core (pas de parallélisme interne) et > 5 cores (HDFS throughput bottleneck)

Exemple pour un nœud de 32 cores, 128 GB RAM :
  - Réserver 1 core + 1 GB pour l'OS et le Node Manager
  - Restant : 31 cores, 127 GB RAM
  - 5 executors × (5 cores + 25 GB) + overhead (2g) = 5 × (5c, 27g) = 25 cores, 135 GB
  - Ajuster si nécessaire selon les besoins du job
```

---

[→ Suite : FAQ](05-faq.md)
