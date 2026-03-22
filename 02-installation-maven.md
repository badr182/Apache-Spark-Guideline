# Installation & Dépendances Maven

> [← Retour au sommaire](README.md)

---

## Fichier `pom.xml` Complet

```xml
<project>
  <groupId>com.exemple</groupId>
  <artifactId>mon-app-spark</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <spark.version>3.5.0</spark.version>
    <scala.binary.version>2.12</scala.binary.version>
    <java.version>11</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
  </properties>

  <dependencies>
    <!-- Spark Core -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-core_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
      <scope>provided</scope>
    </dependency>

    <!-- Spark SQL (DataFrame / Dataset) -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
      <scope>provided</scope>
    </dependency>

    <!-- Spark MLlib (optionnel) -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-mllib_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
      <scope>provided</scope>
    </dependency>

    <!-- Spark Streaming (optionnel) -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-streaming_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
      <scope>provided</scope>
    </dependency>

    <!-- Connecteur Kafka pour Structured Streaming (optionnel) -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql-kafka-0-10_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
    </dependency>

    <!-- Delta Lake (optionnel) -->
    <dependency>
      <groupId>io.delta</groupId>
      <artifactId>delta-spark_${scala.binary.version}</artifactId>
      <version>3.1.0</version>
    </dependency>

    <!-- Tests -->
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_${scala.binary.version}</artifactId>
      <version>${spark.version}</version>
      <classifier>tests</classifier>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.10.0</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

> **`scope=provided` vs `scope=compile` :**
> - `provided` : le JAR Spark est fourni par le cluster (YARN, K8s) — ne l'inclut pas dans le fat JAR
> - `compile` (défaut) : inclut dans le fat JAR — nécessaire pour les tests locaux ou les applications standalone

---

## Créer un Fat JAR avec Maven Shade Plugin

Le Fat JAR (ou Uber JAR) regroupe votre code et toutes ses dépendances dans un seul fichier JAR déployable.

```xml
<build>
  <plugins>
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
                  <!-- Exclure les signatures qui causeraient des conflits -->
                  <exclude>META-INF/*.SF</exclude>
                  <exclude>META-INF/*.DSA</exclude>
                  <exclude>META-INF/*.RSA</exclude>
                  <!-- Exclure les JARs Spark (scope provided) -->
                  <exclude>META-INF/versions/*/module-info.class</exclude>
                </excludes>
              </filter>
            </filters>
            <transformers>
              <!-- Définir la classe principale -->
              <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                <mainClass>com.exemple.SparkApp</mainClass>
              </transformer>
              <!-- Fusionner les fichiers META-INF/services (IMPORTANT pour Spark) -->
              <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
              <!-- Fusionner les fichiers reference.conf (pour Akka/Spark) -->
              <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                <resource>reference.conf</resource>
              </transformer>
            </transformers>
            <!-- Renommer les packages pour éviter les conflits de version -->
            <relocations>
              <relocation>
                <pattern>com.google.guava</pattern>
                <shadedPattern>shaded.com.google.guava</shadedPattern>
              </relocation>
            </relocations>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### Construire le JAR

```bash
mvn clean package -DskipTests

# Le fat JAR se trouve dans :
ls -lh target/*-shaded.jar
# ou selon la configuration :
ls -lh target/mon-app-spark-1.0.0.jar
```

---

## Soumettre au Cluster avec `spark-submit`

### YARN (le plus courant en entreprise)

```bash
spark-submit \
  --class com.exemple.SparkApp \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 4g \
  --executor-memory 8g \
  --executor-cores 4 \
  --num-executors 10 \
  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
  --conf spark.sql.shuffle.partitions=400 \
  --conf spark.sql.adaptive.enabled=true \
  --files config/application.properties \
  target/mon-app-spark-1.0.0.jar \
  --input hdfs:///data/input \
  --output hdfs:///data/output
```

### Kubernetes

```bash
spark-submit \
  --master k8s://https://<kubernetes-api-server>:6443 \
  --deploy-mode cluster \
  --name mon-app-spark \
  --class com.exemple.SparkApp \
  --conf spark.executor.instances=5 \
  --conf spark.kubernetes.container.image=mon-registre/spark-java:3.5.0 \
  --conf spark.kubernetes.namespace=spark-jobs \
  local:///opt/spark/jars/mon-app-spark-1.0.0.jar
```

### Mode Local (développement)

```bash
spark-submit \
  --class com.exemple.SparkApp \
  --master local[4] \
  target/mon-app-spark-1.0.0.jar
```

---

## Structure de Projet Recommandée

```
mon-app-spark/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/com/exemple/
│   │   │   ├── SparkApp.java          ← point d'entrée (main)
│   │   │   ├── jobs/
│   │   │   │   ├── VentesJob.java
│   │   │   │   └── ClientsJob.java
│   │   │   ├── transforms/
│   │   │   │   ├── VentesTransform.java
│   │   │   │   └── EnrichissementTransform.java
│   │   │   ├── models/
│   │   │   │   ├── Vente.java         ← beans Serializable
│   │   │   │   └── Client.java
│   │   │   └── utils/
│   │   │       ├── SparkSessionFactory.java
│   │   │       └── SchemaUtils.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/com/exemple/
│           ├── jobs/VentesJobTest.java
│           └── transforms/VentesTransformTest.java
└── target/
    └── mon-app-spark-1.0.0.jar
```

---

## Tests Unitaires avec SparkSession

```java
import org.apache.spark.sql.SparkSession;
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class VentesJobTest {

    private static SparkSession spark;

    @BeforeAll
    static void setupSpark() {
        spark = SparkSession.builder()
            .appName("TestSpark")
            .master("local[2]")
            .config("spark.sql.shuffle.partitions", "2")  // petit pour les tests
            .config("spark.ui.enabled", "false")           // désactiver l'UI de test
            .getOrCreate();
    }

    @AfterAll
    static void teardownSpark() {
        if (spark != null) spark.stop();
    }

    @Test
    void testFiltreVentesActives() {
        // Créer des données de test en mémoire
        Dataset<Row> input = spark.createDataFrame(
            Arrays.asList(
                RowFactory.create(1L, "actif",  150.0),
                RowFactory.create(2L, "inactif", 50.0),
                RowFactory.create(3L, "actif",  200.0)
            ),
            new StructType()
                .add("id",      DataTypes.LongType)
                .add("statut",  DataTypes.StringType)
                .add("montant", DataTypes.DoubleType)
        );

        Dataset<Row> resultat = VentesTransform.filtrerActifs(input);

        assertEquals(2, resultat.count());
        assertEquals(350.0, resultat.agg(sum("montant")).first().getDouble(0), 0.001);
    }
}
```

---

## Variables d'Environnement Utiles

```bash
# Répertoire d'installation de Spark
export SPARK_HOME=/opt/spark

# Mémoire du driver en mode local
export SPARK_DRIVER_MEMORY=4g

# Options JVM du driver
export SPARK_DRIVER_OPTS="-XX:+UseG1GC -Dlog4j.configuration=file:log4j.properties"

# Répertoire des logs
export SPARK_LOG_DIR=/var/log/spark

# PySpark (si utilisé conjointement)
export PYSPARK_PYTHON=python3
```

---

[→ Suite : Architecture](03-architecture.md)
