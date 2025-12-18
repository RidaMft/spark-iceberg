# 🐳 Docker Image : Spark + Iceberg + Jupyter Notebook

Une image Docker autonome intégrant **Apache Spark**, **Apache Iceberg**, et **Jupyter Notebook**, conçue pour le développement local ou en environnement cloud (AWS / Azure / GCP). Idéale pour manipuler des tables Iceberg dans des notebooks Python (PySpark), avec connectivité native vers S3, ADLS et GCS.

---

## 🧰 Technologies intégrées

| Composant | Version |
|----------|---------|
| **Base** | `python:3.11.4-bullseye` |
| **OpenJDK** | 17 |
| **Spark** | `3.5.7` (Hadoop 3) |
| **Iceberg Runtime** | `1.10.0` (Spark 3.5 + Scala 2.12) |
| **Hadoop-AWS** | `3.3.4` |
| **AWS Java SDK** | `1.12.353` |
| **Bundles Cloud** | `iceberg-aws-bundle`, `iceberg-gcp-bundle`, `iceberg-azure-bundle` (v1.10.0) |
| **Jupyter + PySpark** | Via `spylon-kernel` et `pyspark` |
| **AWS CLI** | v2 (installé globalement) |


---

## 📦 Fonctionnalités

- ✅ Exécution interactive de notebooks PySpark avec support Iceberg (`CREATE TABLE`, `MERGE INTO`, etc.)
- ✅ Connexion native aux stockages :
  - **AWS S3** via `s3a://`
  - **Azure Data Lake** (ADLS Gen2) via `abfss://`
  - **Google Cloud Storage** via `gs://`
- ✅ Préchargement des JARs nécessaires (pas besoin de `--packages`)
- ✅ Interface Jupyter accessible sans token/mot de passe (mode dev uniquement ✅)
- ✅ Commandes `notebook` / `pyspark-notebook` pour lancer Spark en mode driver notebook
- ✅ Dossiers montables pour notebooks, warehouse locale, etc.

---

## 🚀 Utilisation rapide

### Construire l’image

```bash
docker build -t spark-iceberg-jupyter:latest .
```

### Lancer localement

```bash
docker run -it \
  -p 8888:8888 \
  -p 4040:4040 \
  -v $(pwd)/notebooks:/home/iceberg/notebooks \
  -v $(pwd)/warehouse:/home/iceberg/warehouse \
  spark-iceberg-jupyter:latest
```

➡️ Ouvrez [http://localhost:8888](http://localhost:8888) dans votre navigateur.

> 🔐 **Sécurité** : En production, désactivez `--NotebookApp.token=''` et ajoutez un mot de passe.

---

Voici une section **« 🐳 Déploiement sur Docker Hub »** à ajouter à votre `README.md`, compatible avec les bonnes pratiques et les contraintes de votre environnement (ex: préférence pour les branches `dev` avant `main`, scripts automatisés, etc.) :

---

## 🐳 Déploiement sur Docker Hub

### 1. Construire l’image avec un *tag* sémantique

```bash
# Exemple : tag de dev + date
TAG="dev-$(date +%Y%m%d)"
docker build -t <votre_dockerhub_id>/spark-iceberg-jupyter:${TAG} .

# Tag optionnel pour latest (à utiliser avec prudence)
docker tag <votre_dockerhub_id>/spark-iceberg-jupyter:${TAG} <votre_dockerhub_id>/spark-iceberg-jupyter:latest
```

> 🔔 **Bonnes pratiques**  
> - Utilisez toujours un tag explicite (ex: `v1.2.0`, `dev-20251218`) plutôt que `latest` en CI/CD.  
> - Pour les PRs ou branches de dev, préférez `dev-<branch>-<sha>`.

---

### 2. Pousser sur Docker Hub

```bash
docker login

docker push <votre_dockerhub_id>/spark-iceberg-jupyter:${TAG}
docker push <votre_dockerhub_id>/spark-iceberg-jupyter:latest  # si nécessaire
```
---

### 3. Utilisation depuis Docker Hub

Une fois poussée, tout utilisateur peut exécuter :

```bash
docker run -it -p 8888:8888 \
  -v $(pwd)/notebooks:/home/iceberg/notebooks \
  rmeftah/spark-iceberg:3.5.7-1.10.0
```


---

## 📁 Structure des dossiers

| Chemin dans le conteneur | Usage |
|--------------------------|-------|
| `/home/iceberg/notebooks` | Dossier par défaut des notebooks |
| `/home/iceberg/warehouse` | Warehouse locale (peut être montée) |
| `/home/iceberg/localwarehouse` | Alternative pour tests locaux |
| `/home/iceberg/spark-events` | Pour le monitoring Spark UI (à activer via conf) |
| `/opt/spark/conf/spark-defaults.conf` | Fichier de configuration inclus |
| `/opt/spark/jars/` | Tous les JARs Iceberg + Hadoop-AWS préinstallés |

---

## ⚙️ Configuration

Le fichier `spark-defaults.conf` est copié à la construction. Exemple minimal recommandé :

```properties
spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog
spark.sql.catalog.spark_catalog.type=hive
spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.local.type=hadoop
spark.sql.catalog.local.warehouse=file:///home/iceberg/warehouse

# S3 (optionnel)
# spark.hadoop.fs.s3a.access.key=...
# spark.hadoop.fs.s3a.secret.key=...
# spark.hadoop.fs.s3a.aws.credentials.provider=...
```

---

## 🔌 Exemples d’usage dans un notebook

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("Iceberg-Jupyter") \
    .getOrCreate()

# Créer une table Iceberg
spark.sql("""
CREATE TABLE IF NOT EXISTS local.db.test_table (
    id BIGINT,
    data STRING
) USING iceberg
""")

df = spark.createDataFrame([(1, "a"), (2, "b")], ["id", "data"])
df.writeTo("local.db.test_table").append()
spark.table("local.db.test_table").show()
```

---

## 🛠️ Personnalisation

- **Changer les versions** : Éditez les `ENV` dans le `Dockerfile`.
- **Ajouter des dépendances Python** : Modifiez `requirements.txt`.
- **Ajouter des JARs** : Ajoutez des `RUN curl …` avant la copie de `spark-defaults.conf`.
- **Mode cluster** : Cette image est conçue pour le **mode standalone local** (`local[*]`). Pour Spark Standalone ou Kubernetes, adaptez l’entrypoint.

---

## 📝 Notes importantes

- Le **serveur Spark UI** est accessible sur `http://localhost:4040` après exécution d’une action Spark.
- L’image **ne démarre pas Spark Master/Worker** par défaut — elle est centrée sur le mode *local notebook driver*.
- Pour AWS/GCP/Azure : configurez les identifiants via variables d’environnement ou fichiers montés (ex: `~/.aws/credentials`).

---

## 📄 Licence

L’image hérite des licences Apache 2.0 (Spark, Iceberg, Hadoop), MIT/BSD (Python, Jupyter), etc.

---

> 💡 **Astuce** : Utilisez cette image comme base pour vos pipelines CI/CD ou vos environnements dev/test EC2 (ex: `t3a.xlarge`), en montant vos notebooks via volumes.
```