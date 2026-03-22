# FAQ — Questions Fréquentes Apache Spark Java

> [← Retour au sommaire](README.md)

---

## API et Concepts de Base

### Q : Quelle est la différence entre une transformation et une action ?

Les transformations (`filter`, `map`, `join`, `groupBy`) sont **paresseuses** — elles définissent un plan de calcul (DAG) sans rien exécuter. Les actions (`count`, `collect`, `show`, `write`) déclenchent l'exécution réelle. À chaque action, Spark ré-exécute toute la lignée depuis la source, sauf si des résultats intermédiaires sont mis en cache.

---

### Q : Qu'est-ce qu'un shuffle et pourquoi est-il coûteux ?

Un shuffle se produit quand Spark doit redistribuer les données entre partitions — typiquement lors d'un `groupBy`, `join`, `distinct`, `repartition` ou `orderBy`. Il implique l'écriture sur disque et des transferts réseau entre les nœuds, ce qui en fait l'opération la plus coûteuse dans Spark.

**Comment minimiser les shuffles :**
- Utiliser les broadcast joins pour les petites tables
- Utiliser `coalesce()` au lieu de `repartition()` quand possible
- Filtrer les données en amont (predicate pushdown)
- Ajuster `spark.sql.shuffle.partitions` selon la taille des données

---

### Q : Quelle est la différence entre `coalesce()` et `repartition()` ?

`coalesce(n)` réduit le nombre de partitions **sans shuffle complet** — c'est efficace mais peut créer des partitions déséquilibrées. À utiliser après avoir filtré un grand jeu de données.

`repartition(n)` effectue un **shuffle complet** pour créer `n` partitions équilibrées. À utiliser lorsque vous avez besoin d'un parallélisme équilibré ou d'un partitionnement par colonne spécifique.

```java
// Après un filtre qui réduit drastiquement les données
Dataset<Row> petit = grandDf.filter(col("annee").equalTo(2024));
petit.coalesce(10).write().parquet("sortie/2024/");  // bon : pas de shuffle

// Avant une opération coûteuse nécessitant un parallélisme équilibré
Dataset<Row> equilibre = df.repartition(200, col("client_id"));  // shuffle intentionnel
```

---

### Q : Comment créer un `Dataset<T>` à partir d'un bean Java ?

Votre bean doit être `Serializable`, avoir un constructeur sans argument, et des getters/setters publics.

```java
import java.io.Serializable;

public class EnregistrementVente implements Serializable {
    private static final long serialVersionUID = 1L;

    private long id;
    private String region;
    private double montant;

    public EnregistrementVente() {}  // constructeur sans arg obligatoire

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

// Créer un Dataset directement depuis une liste Java
List<EnregistrementVente> liste = Arrays.asList(
    new EnregistrementVente(1L, "EMEA", 150.0),
    new EnregistrementVente(2L, "APAC", 200.0)
);
Dataset<EnregistrementVente> dsCreated = spark.createDataset(liste, encodeur);
```

---

### Q : Comment effectuer une jointure en Java ?

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

// Jointure sur plusieurs colonnes
Dataset<Row> multiCle = factures.join(paiements,
    factures.col("facture_id").equalTo(paiements.col("facture_id"))
        .and(factures.col("devise").equalTo(paiements.col("devise"))));

// Éviter l'ambiguïté après jointure
Dataset<Row> propre = joint
    .drop(clients.col("id"))
    .withColumnRenamed("nom", "nom_client");
```

### Types de jointures disponibles

| Type | SQL équivalent | Description |
|------|---------------|-------------|
| `inner` | INNER JOIN | Lignes correspondant dans les deux tables |
| `left_outer` | LEFT OUTER JOIN | Toutes les lignes de gauche, nulls pour la droite |
| `right_outer` | RIGHT OUTER JOIN | Toutes les lignes de droite, nulls pour la gauche |
| `full_outer` | FULL OUTER JOIN | Toutes les lignes des deux tables |
| `left_semi` | LEFT SEMI JOIN | Lignes de gauche qui ont une correspondance à droite |
| `left_anti` | LEFT ANTI JOIN | Lignes de gauche sans correspondance à droite |
| `cross` | CROSS JOIN | Produit cartésien (attention : explosif !) |

---

### Q : Comment utiliser les fonctions de fenêtre (Window Functions) en Java ?

```java
import org.apache.spark.sql.expressions.Window;
import org.apache.spark.sql.expressions.WindowSpec;

WindowSpec fenetre = Window
    .partitionBy("region")
    .orderBy(col("date_event").desc());

Dataset<Row> resultat = df
    .withColumn("rang",           rank().over(fenetre))
    .withColumn("rang_dense",     dense_rank().over(fenetre))
    .withColumn("num_ligne",      row_number().over(fenetre))
    .withColumn("total_cumulé",   sum("montant").over(
        fenetre.rowsBetween(Window.unboundedPreceding(), Window.currentRow())))
    .withColumn("montant_prec",   lag("montant", 1).over(fenetre))
    .withColumn("montant_suiv",   lead("montant", 1).over(fenetre))
    .withColumn("max_region",     max("montant").over(
        Window.partitionBy("region")));  // sur toute la partition

// Garder uniquement le premier de chaque région (dédoublonnage)
Dataset<Row> top1ParRegion = resultat
    .filter(col("num_ligne").equalTo(1));
```

---

### Q : Qu'est-ce que l'AQE et comment l'activer ?

L'AQE (Adaptive Query Execution, Spark 3.0+) ré-optimise le plan de requête à l'exécution en se basant sur les statistiques réelles collectées. Il peut automatiquement :
- Fusionner de petites partitions de shuffle (moins de fichiers petits)
- Changer de stratégie de jointure (Sort-Merge → Broadcast si la table est petite)
- Gérer le déséquilibre des données (skew joins)

```java
SparkSession spark = SparkSession.builder()
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
    .config("spark.sql.adaptive.coalescePartitions.minPartitionSize", "64MB")
    .config("spark.sql.adaptive.skewJoin.enabled", "true")
    .config("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
    .getOrCreate();
```

---

### Q : Comment utiliser les accumulateurs et les variables broadcast en Java ?

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

// Ne pas oublier de détruire le broadcast quand plus nécessaire
broadcastRef.destroy();
```

> **Important :** Les accumulateurs ne sont incrémentés qu'une seule fois par tâche (même si la tâche est relancée). Pour les tâches qui échouent et sont réessayées, l'accumulateur peut être inexact.

---

### Q : Comment inspecter le schéma et les données d'un DataFrame ?

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

// Accéder à la valeur d'une colonne (depuis la liste ramenée avec collect)
Row premiereLigne = df.first();
String nom = premiereLigne.getAs("nom");
long id    = premiereLigne.getAs("id");
// Ou avec l'index :
String nom2 = premiereLigne.getString(0);
long id2    = premiereLigne.getLong(1);
```

---

## Performance et Optimisation

### Q : Mon job est lent — par où commencer le diagnostic ?

1. Ouvrir le **Spark UI** (`http://driver:4040`)
2. Regarder l'onglet **Stages** — chercher le stage le plus lent
3. Vérifier les métriques par tâche :
   - **GC Time > 10%** → problème mémoire
   - **Shuffle Read/Write élevé** → trop de shuffles
   - **Écart min/max task duration** → data skew
   - **Spill (disk)** → manque de mémoire
4. Utiliser `df.explain("formatted")` pour voir le plan physique
5. Activer l'AQE si ce n'est pas déjà fait

---

### Q : Quelle est la différence entre `cache()` et `persist()` ?

`cache()` est équivalent à `persist(StorageLevel.MEMORY_AND_DISK())` depuis Spark 3.x (auparavant `MEMORY_ONLY`). `persist()` vous permet de spécifier explicitement le niveau de stockage.

```java
df.cache();  // MEMORY_AND_DISK depuis Spark 3.x

// Équivalent explicite :
df.persist(StorageLevel.MEMORY_AND_DISK());
```

---

### Q : Pourquoi `collect()` est-il dangereux ?

`collect()` ramène **toutes** les données du cluster vers le Driver. Si les données dépassent la RAM du Driver, vous aurez un `OutOfMemoryError`.

```java
// ❌ Dangereux sur de grandes données
List<Row> toutesLesLignes = df.collectAsList();

// ✅ Alternatives sûres
df.write().parquet("sortie/");           // écrire dans le stockage
df.show(100);                            // afficher un échantillon
List<Row> echantillon = df.limit(1000).collectAsList();  // petit échantillon
df.foreach(row -> traiter(row));         // traiter en place sur les executors
```

---

### Q : Comment déboguer un job Spark sans cluster ?

```java
// Configuration locale pour le debug
SparkSession spark = SparkSession.builder()
    .master("local[1]")  // 1 thread = exécution séquentielle, plus facile à déboguer
    .config("spark.sql.shuffle.partitions", "2")
    .config("spark.ui.enabled", "false")
    .config("spark.sql.adaptive.enabled", "false")  // désactiver AQE pour voir le vrai plan
    .getOrCreate();

// Activer les logs détaillés
spark.sparkContext().setLogLevel("DEBUG");

// Afficher le plan à chaque étape
df.filter(col("statut").equalTo("actif")).explain();

// Breakpoints dans les lambdas — utiliser foreachPartition
df.foreachPartition(partition -> {
    partition.forEachRemaining(row -> {
        System.out.println(row.toString());  // point d'arrêt ici dans l'IDE
    });
});
```

---

### Q : Comment gérer les partitions Parquet lors de la lecture ?

```java
// Lire une partition spécifique (partition pruning)
Dataset<Row> df2024 = spark.read()
    .parquet("donnees/ventes/")
    .where("annee = 2024 AND mois = 3");  // Spark pousse ce filtre vers Parquet

// Forcer le partition pruning
spark.conf().set("spark.sql.sources.partitionOverwriteMode", "dynamic");

// Lire avec projection pushdown (Parquet lit seulement les colonnes nécessaires)
Dataset<Row> colonnesSelectionnees = spark.read()
    .parquet("donnees/")
    .select("id", "montant");  // seules ces colonnes sont lues depuis le fichier
```

---

[→ Suite : Problèmes & Solutions](06-problemes-solutions.md)
