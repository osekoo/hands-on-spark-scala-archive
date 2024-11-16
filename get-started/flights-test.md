# **Guide Complet pour la Mise en Place, le Test et le Déploiement d'une Application Spark**

## **Table des Matières**
1. [Mise en place du projet](#mise-en-place-du-projet)
2. [Architecture des données en local](#architecture-des-données-en-local)
3. [Transfert des données sur HDFS](#transfert-des-données-sur-hdfs)
4. [Exécution de l'application](#exécution-de-lapplication)

---

## **Mise en place du projet**

### **1. Créer un nouveau projet Scala/SBT**
Commencez par créer un nouveau projet Spark/Scala/Sbt :

### **2. Configuration du fichier `build.sbt`**
Créez un fichier nommé `build.sbt` et copiez-y le contenu suivant :

```sbt
name := "flights" // Nom du projet
version := "0.1" // Version de l'application
scalaVersion := "2.12.18" // Version de Scala
organization := "com.miasd.dev"

val sparkVersion = "3.5.2"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % sparkVersion,
  "org.apache.spark" %% "spark-sql" % sparkVersion,
  "org.apache.spark" %% "spark-mllib" % sparkVersion % "provided"
)
```

### **3. Implémenter le code principal (`Main.scala`)**
Créez le fichier `src/main/scala/Main.scala` avec le contenu suivant :

```scala
import org.apache.spark.sql.SparkSession

object Main {
  def main(args: Array[String]): Unit = {
    val spark: SparkSession = SparkSession.builder()
      .appName("flights")
      .getOrCreate()

    spark.sparkContext.setLogLevel("ERROR")

    println("Arguments reçus : " + args.mkString(", "))
    if (args.length < 1) {
      println("Usage: Main <data-root-dir>")
      System.exit(1)
    }

    val dataRootDir = args(0)
    println("Répertoire des données : " + dataRootDir)

    val flightPath = dataRootDir + "/flights/*.csv"
    val weatherPath = dataRootDir + "/weather/*.txt"
    val airportPath = dataRootDir + "/airport/*.csv"

    println("Loading flight data from: " + flightPath)
    val flightData = spark.read.option("header", "true").option("inferSchema", "true").csv(flightPath)
    flightData.show(10)

    println("Loading weather data from: " + weatherPath)
    val weatherData = spark.read.option("header", "true").option("inferSchema", "true").csv(weatherPath)
    weatherData.show(10)

    println("Loading airport data from: " + airportPath)
    val airportData = spark.read.option("header", "true").option("inferSchema", "true").csv(airportPath)
    airportData.show(10)

    println("Saving flight data to parquet format")
    flightData.write.mode("overwrite").parquet(dataRootDir + "/flights.parquet")

    println("Saving weather data to parquet format")
    weatherData.write.mode("overwrite").parquet(dataRootDir + "/weather.parquet")

    println("Saving airport data to parquet format")
    airportData.write.mode("overwrite").parquet(dataRootDir + "/airport.parquet")

    println("Session terminée.")
    spark.stop()
  }
}
```

### **4. Mettre en place l'exécuteur (`spark-run`)**

#### **a) Version locale (Windows)**
Créez un fichier `spark-run.bat` avec le contenu suivant :

```batch
spark-submit ^
    --deploy-mode client ^
    --master "local[*]" ^
    --executor-cores "4" ^
    --executor-memory "1G" ^
    --num-executors "1" ^
    --class "Main" ^
    "target/scala-2.12/flights_2.12-0.1.jar" ^
    "data"
```

#### **b) Version locale (Linux/Mac)**
Créez un fichier `spark-run.sh` :

```bash
#!/bin/bash
spark-submit \
    --deploy-mode "client" \
    --master "local[*]" \
    --executor-cores 2 \
    --executor-memory 2G \
    --num-executors 2 \
    --class "Main" \
    "target/scala-2.12/flights_2.12-0.1.jar" \
    "data"
```

#### **c) Version pour le cluster**
Créez un autre script pour l'exécution sur un cluster :

```bash
#!/bin/bash
spark-submit \
    --deploy-mode "client" \
    --master "spark://vmhadoopmaster.cluster.lamsade.dauphine.fr:7077" \
    --executor-cores 2 \
    --executor-memory 2G \
    --num-executors 2 \
    --class "Main" \
    "flights_2.12-0.1.jar" \
    "data"
```
- Copiez ce fichier sur le serveur.

### **5. Compiler le projet**
Dans le terminal, exécutez :

```bash
sbt package
```

- Le fichier JAR sera généré dans `target/scala-2.12/`.
- Copiez le ficher JAR sur le cluster dans le même répertoire que `spark-run.sh`.

---

## **Architecture des données en local et sur le cluster**

1. Créez dans le répertoire où vous avez copié les fichiers `spark-run.sh` et le `JAR` un répertoire pour stocker les données :

    ```bash
    mkdir -p data/flights data/weather data/airport
    ```

2. Copiez vos fichiers de données dans les répertoires appropriés :

   - `data/flights/` : fichiers CSV de vols.
   - `data/weather/` : fichiers TXT de météo.
   - `data/airport/` : fichiers CSV d'aéroports.
  
3. Copiez ces données sur le cluster en conservant la structure des répertoires
4. La structure de votre espace espace de travail doit ressembler à:
```bash
   flights_project/
       - flights_2.12-0.1.jar
       - spark-run.sh
       - data/
           - airport/
           - fligths/
           - weather/
```
---

## **Transfert des données sur HDFS**

### **1. Créer le répertoire sur HDFS**
Sur le cluster, exécutez :

```bash
hdfs dfs -mkdir -p data/
```

### **2. Uploader les fichiers**
Sur le cluster, exécutez :

```bash
hdfs dfs -put data/* data/
```

### **3. Vérifier les données sur HDFS**
Assurez-vous que les fichiers ont été correctement transférés :

```bash
hdfs dfs -ls data
hdfs dfs -ls data/flights/
hdfs dfs -ls data/weather/
hdfs dfs -ls data/airport/
```

---

## **Exécution de l'application**

### **1. Sur machine locale (Windows)**
Lancez le script :

```bash
./spark-run.bat
```

### **2. Sur machine locale (Linux/Mac)**
Lancez le script :

```bash
./spark-run.sh
```

### **3. Sur le cluster**
Connectez-vous au nœud principal du cluster et exécutez :

```bash
./spark-run.sh
```
