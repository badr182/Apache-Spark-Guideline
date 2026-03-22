# Problèmes & Solutions — Apache Spark Java

> [← Retour au sommaire](README.md)

---

## ❌ Problème : Exception `Task Not Serializable`

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
    try (Connection conn = DriverManager.getConnection(url)) {
        PreparedStatement stmt = conn.prepareStatement("INSERT INTO ...");
        while (lignes.hasNext()) {
            Row ligne = lignes.next();
            stmt.setLong(1, ligne.getAs("id"));
            stmt.addBatch();
        }
        stmt.executeBatch();
    }
});
```

**Diagnostic :** La trace de la pile contiendra souvent le nom de la classe non sérialisable. Cherchez `Caused by: java.io.NotSerializableException: ...`.

**Checklist :**
- [ ] Les classes de service/config référencées dans les lambdas sont-elles `Serializable` ?
- [ ] Les connexions BD/HTTP sont-elles créées à l'intérieur des lambdas ?
- [ ] Les classes internes anonymes capturent-elles `this` (la classe englobante) ?

---

## ❌ Problème : `OutOfMemoryError` sur le Driver

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

// ✅ Traiter ligne par ligne sans tout ramener
df.foreach((ForeachFunction<Row>) row -> processRow(row));
```

```bash
# Augmenter la mémoire du driver dans spark-submit si nécessaire
--driver-memory 8g
--conf spark.driver.maxResultSize=4g  # limite la taille des collect()
```

---

## ❌ Problème : `OutOfMemoryError` sur les Executors

**Symptôme :** Erreurs OOM dans les executors ; les tâches échouent et se relancent (`FAILED`, retries épuisés).

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

// Éviter les opérations qui accumulent en mémoire
// ❌ collect_list sur des millions de lignes
df.groupBy("cle").agg(collect_list(col("valeur")));

// ✅ Utiliser des agrégations scalaires
df.groupBy("cle").agg(count("valeur"), sum("montant"));
```

---

## ❌ Problème : Job Lent — Data Skew (Déséquilibre des Données)

**Symptôme :** Dans l'interface Spark, la plupart des tâches se terminent en secondes, mais 1 ou 2 tâches tournent pendant des minutes (visible dans l'onglet Stages > Task Duration).

**Diagnostic :**

```java
// Inspecter la distribution des données par clé de jointure
df.groupBy("cle_jointure")
  .count()
  .orderBy(col("count").desc())
  .show(20);
```

**Solution 1 : AQE (automatique, Spark 3.x)**

```java
spark.conf().set("spark.sql.adaptive.enabled", "true");
spark.conf().set("spark.sql.adaptive.skewJoin.enabled", "true");
spark.conf().set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB");
spark.conf().set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5");
```

**Solution 2 : Salting (salage) manuel**

```java
import static org.apache.spark.sql.functions.*;

int nbSels = 50;

// Saler la grande table : ajouter un suffixe aléatoire à la clé
Dataset<Row> grandeSalee = grandDf
    .withColumn("sel",
        floor(rand().multiply(nbSels)).cast(DataTypes.StringType))
    .withColumn("cle_salee",
        concat(col("cle_jointure"), lit("_"), col("sel")));

// Éclater la petite table : répliquer pour chaque valeur de sel possible
Column tableauSels = array(java.util.stream.IntStream.range(0, nbSels)
    .mapToObj(i -> lit(String.valueOf(i)))
    .toArray(Column[]::new));

Dataset<Row> petiteSalee = petitDf
    .withColumn("sel", explode(tableauSels))
    .withColumn("cle_salee",
        concat(col("cle_jointure"), lit("_"), col("sel")));

// Jointure sur la clé salée
Dataset<Row> resultat = grandeSalee.join(broadcast(petiteSalee), "cle_salee")
    .drop("sel", "cle_salee");
```

---

## ❌ Problème : Colonne Ambiguë Après Jointure

**Symptôme :** `AnalysisException: Reference 'nom_colonne' is ambiguous, could be: ...`

**Solution :**

```java
// ❌ Ambigu — les deux DataFrames ont une colonne "id"
Dataset<Row> joint = commandes.join(clients,
    commandes.col("client_id").equalTo(clients.col("id")));
joint.select("id"); // lequel des deux "id" ?

// ✅ Option 1 : Utiliser la référence du DataFrame pour lever l'ambiguïté
joint.select(commandes.col("id"), clients.col("id").alias("id_client"));

// ✅ Option 2 : Supprimer la colonne dupliquée après la jointure
Dataset<Row> propre = joint.drop(clients.col("id"));

// ✅ Option 3 : Utiliser des alias avant la jointure
Dataset<Row> c  = commandes.alias("c");
Dataset<Row> cl = clients.alias("cl");
Dataset<Row> joint2 = c.join(cl, col("c.client_id").equalTo(col("cl.id")));
joint2.select("c.id", "cl.nom", "c.montant");

// ✅ Option 4 : Jointure sur colonne de même nom (fusionne automatiquement)
Dataset<Row> joint3 = commandes.join(clients, "client_id"); // pas d'ambiguïté
```

---

## ❌ Problème : Trop de Petits Fichiers en Sortie

**Symptôme :** Le répertoire de sortie contient des milliers de petits fichiers (quelques Ko chacun). Cela dégrade les performances de lecture future.

**Cause :** Trop de partitions au moment de l'écriture.

**Solution :**

```java
// Option 1 : Réduire les partitions sans shuffle (recommandé après filtrage)
df.coalesce(20).write()
    .mode(SaveMode.Overwrite)
    .parquet("sortie/resultat");

// Option 2 : Repartitionner avec shuffle pour un équilibrage parfait
df.repartition(50).write()
    .mode(SaveMode.Overwrite)
    .parquet("sortie/resultat");

// Option 3 : Limiter le nombre d'enregistrements par fichier de sortie
df.write()
    .option("maxRecordsPerFile", 1_000_000)
    .parquet("sortie/resultat");

// Option 4 (Spark 3.x) : Activer AQE pour la coalescence automatique
spark.conf().set("spark.sql.adaptive.coalescePartitions.enabled", "true");
spark.conf().set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "128MB");
```

**Règle :** Visez des fichiers Parquet de 128 MB à 512 MB.

---

## ❌ Problème : `AnalysisException` — Colonne Introuvable

**Symptôme :** `AnalysisException: Resolved attribute(s) missing from child ...`

**Causes et solutions :**

```java
// ❌ Référencer une colonne d'un ancien DataFrame
Dataset<Row> df2 = df.filter(col("statut").equalTo("actif"));
Dataset<Row> df3 = df2.select(df.col("nom")); // référence à l'ancien df !

// ✅ Référencer le DataFrame courant
Dataset<Row> df3 = df2.select(df2.col("nom"));
// ou simplement par nom
Dataset<Row> df3b = df2.select("nom");

// ❌ Colonne avec espace ou caractère spécial sans backticks
df.select(col("ma colonne"));  // erreur
df.filter("ma colonne > 100"); // erreur en SQL inline

// ✅ Utiliser les backticks
df.select(col("`ma colonne`"));
spark.sql("SELECT `ma colonne` FROM ma_vue");

// ✅ Renommer les colonnes problématiques en amont
df.withColumnRenamed("ma colonne", "ma_colonne");

// Les colonnes créées avec withColumn sont disponibles dans la suite de la chaîne
df.withColumn("x", col("a").plus(col("b")))
  .withColumn("y", col("x").multiply(2));  // "x" est disponible ici ✅
```

---

## ❌ Problème : `GC Overhead Limit Exceeded` sur les Executors

**Symptôme :** Pauses fréquentes du garbage collector ; les tâches échouent avec `java.lang.OutOfMemoryError: GC overhead limit exceeded`.

**Solutions :**

```bash
# Passer au GC G1 (meilleur pour les grands tas)
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:InitiatingHeapOccupancyPercent=35"

# Utiliser le sérialiseur Kryo (plus rapide et plus compact que Java serialization)
--conf spark.serializer=org.apache.spark.serializer.KryoSerializer
```

```java
// Utiliser le cache sérialisé pour réduire la pression mémoire
df.persist(StorageLevel.MEMORY_AND_DISK_SER());

// Augmenter le nombre de partitions pour réduire les objets par tâche
spark.conf().set("spark.sql.shuffle.partitions", "800");

// Enregistrer les classes métier avec Kryo pour une meilleure sérialisation
spark.conf().set("spark.kryo.classesToRegister",
    "com.exemple.EnregistrementVente,com.exemple.Client");
spark.conf().set("spark.kryo.registrationRequired", "false");  // false = permissif
```

---

## ❌ Problème : Un UDF Retourne Toujours Null

**Symptôme :** L'UDF est enregistré et appelé correctement, mais la colonne de sortie est toujours null.

**Causes courantes :**

```java
// ❌ Type de retour déclaré incorrect — déclaré LongType mais Java retourne Integer
UDF1<String, Integer> monUdf = (String s) -> s.length();  // retourne Integer
spark.udf().register("mon_udf", monUdf, DataTypes.LongType);  // incompatibilité de type → null !

// ✅ Faire correspondre exactement le type déclaré
spark.udf().register("mon_udf", monUdf, DataTypes.IntegerType);  // ✅

// ❌ L'UDF lance une exception sur les valeurs nulles — silencieusement null
UDF1<String, String> mauvaisUdf = (String s) -> s.toUpperCase(); // NullPointerException si s est null !

// ✅ Toujours gérer les entrées nulles dans les UDFs
UDF1<String, String> bonUdf = (String s) -> {
    if (s == null) return null;
    return s.toUpperCase();
};
spark.udf().register("mon_udf_str", bonUdf, DataTypes.StringType);

// ❌ UDF qui modifie des données partagées (effets de bord)
List<String> resultats = new ArrayList<>();
UDF1<String, String> mauvaisUdf2 = (String s) -> {
    resultats.add(s);  // concurrent, non thread-safe dans les executors
    return s;
};

// ✅ Les UDFs doivent être des fonctions pures (sans état partagé)
```

---

## ❌ Problème : La Requête Streaming S'arrête Silencieusement

**Symptôme :** La requête Structured Streaming se termine sans message d'erreur clair dans les logs.

**Solution :**

```java
StreamingQuery requete = df.writeStream()
    .format("parquet")
    .option("path", "sortie/flux")
    .option("checkpointLocation", "checkpoints/requete1") // OBLIGATOIRE pour la reprise
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

// Surveiller la progression (dans un thread séparé)
new Thread(() -> {
    while (requete.isActive()) {
        System.out.println(requete.lastProgress());
        System.out.println("Statut : " + requete.status());
        try { Thread.sleep(10_000); } catch (InterruptedException e) { break; }
    }
}).start();

requete.awaitTermination();
```

**Causes fréquentes d'arrêt silencieux :**
- Pas de `checkpointLocation` configuré — le checkpoint est en mémoire et perdu
- Exception dans un `foreachBatch` mal gérée
- Source Kafka vide pendant trop longtemps avec `failOnDataLoss=true`
- Timeout du YARN ApplicationMaster

---

## ❌ Problème : `java.io.IOException: No space left on device`

**Symptôme :** Les executors échouent avec des erreurs de disque plein pendant les shuffles.

**Solutions :**

```bash
# Changer les répertoires de shuffle vers des disques avec plus d'espace
--conf spark.local.dir=/mnt/disque-rapide/tmp,/mnt/disque2/tmp

# Activer la compression du shuffle
--conf spark.shuffle.compress=true
--conf spark.io.compression.codec=lz4        # rapide, moins de CPU
# ou :
--conf spark.io.compression.codec=zstd       # meilleure compression, plus de CPU

# Nettoyage plus agressif des données de shuffle anciennes
--conf spark.cleaner.periodicGC.interval=1min

# Augmenter les partitions shuffle pour réduire la taille par fichier
--conf spark.sql.shuffle.partitions=800
```

---

## ❌ Problème : Job Lent à Cause des Petits Fichiers en Entrée

**Symptôme :** Des milliers de petits fichiers en entrée créent autant de tâches, ce qui génère un overhead énorme.

**Solution :**

```java
// Option 1 : Fusionner les fichiers à la lecture
spark.conf().set("spark.sql.files.maxPartitionBytes",
    String.valueOf(128 * 1024 * 1024)); // 128 MB par partition (défaut)
spark.conf().set("spark.sql.files.openCostInBytes",
    String.valueOf(4 * 1024 * 1024));  // coût d'ouverture de fichier = 4 MB

// Option 2 : Augmenter la taille cible des partitions
spark.conf().set("spark.sql.files.minPartitionNum", "10");

// Option 3 : Repartitionner immédiatement après la lecture
Dataset<Row> df = spark.read().parquet("donnees/beaucoup-de-petits-fichiers/")
    .repartition(200);

// Option 4 (long terme) : Compacter les fichiers existants
Dataset<Row> compact = spark.read().parquet("donnees/vieux-petits-fichiers/");
compact.write()
    .option("maxRecordsPerFile", 2_000_000)
    .mode(SaveMode.Overwrite)
    .parquet("donnees/compacte/");
```

---

## ❌ Problème : Erreur `WARN HeartbeatReceiver: Removing executor X with no recent heartbeats`

**Symptôme :** Des executors sont retirés du cluster pendant l'exécution, causant des relances de tâches.

**Causes et solutions :**

```bash
# Augmenter le timeout du heartbeat (défaut : 120s)
--conf spark.network.timeout=600s
--conf spark.executor.heartbeatInterval=30s

# Augmenter le timeout des tâches longues
--conf spark.sql.broadcastTimeout=600

# Si le problème est lié à des tâches très longues sur peu de données
# → Augmenter les partitions pour réduire la durée par tâche
```

---

[→ Suite : Ressources](07-ressources.md)
