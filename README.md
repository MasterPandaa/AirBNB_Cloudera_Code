# Analisis Tren Harga Sewa Properti AirBNB Menggunakan Cloudera Hadoop

(https://www.kaggle.com/datasets/rupindersinghrana/airbnb-price-dataset)

Langkah 1: Memahami Dataset dan Tentukan Tujuan
--------------------------------------------------------------------------------------------------------------------------------------------------------------

Dataset Overview: Dataset ini berisi daftar Airbnb dengan kolom seperti id, log_price, property_type, room_type, amenities, accommodates, bathrooms, bed_type, cancellation_policy, cleaning_fee, city, neighbourhood, number_of_reviews, review_scores_rating, bedrooms, beds, dll.

Objective: Menganalisis faktor-faktor yang mempengaruhi log_price (e.g., property_type, room_type, amenities, neighbourhood, review_scores_rating) untuk merekomendasikan strategi harga yang kompetitif yang selaras dengan tren pasar dan anggaran pelanggan.

Metrik Utama: Korelasi antara fasilitas, akomodasi, kamar tidur, dan harga.

Langkah 2: Menyiapkan Cloudera Environment
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Install dan masuk ke Cloudera VM.
2. Lakukan pengecekan apakah Hadoop, Hive, dan Spark telah terinstal.
3. Membuat shared folder untuk berbagi file dengan komputer lokal.
4. Upload dataset ke Cloudera VM untuk diolah

Langkah 3: Unggah Dataset ke HDFS
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Upload dataset ke HDFS:

```
hdfs dfs -mkdir /user/cloudera/airbnb
hdfs dfs -put airbnb_data.csv /user/cloudera/airbnb/
```

2. Verifikasi file di HDFS:

```
hdfs dfs -ls /user/cloudera/airbnb
```

Langkah 4: Pre-processing Data
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Membuat database HIVE

```
hive -e "CREATE DATABASE airbnb_db;"
```

2. Membuat tabel eksternal untuk data CSV

```
CREATE EXTERNAL TABLE airbnb_db.listings (
    id STRING,
    log_price STRING,
    property_type STRING,
    room_type STRING,
    amenities STRING,
    accommodates INT,
    bathrooms STRING,
    bed_type STRING,
    cancellation_policy STRING,
    cleaning_fee BOOLEAN,
    city STRING,
    description STRING,
    first_review STRING,
    host_has_profile_pic STRING,
    host_identity_verified STRING,
    host_response_rate STRING,
    host_since STRING,
    instant_bookable STRING,
    last_review STRING,
    latitude STRING,
    longitude STRING,
    name STRING,
    neighbourhood STRING,
    number_of_reviews INT,
    review_scores_rating STRING,
    thumbnail_url STRING,
    zipcode STRING,
    bedrooms STRING,
    beds STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/cloudera/airbnb'
TBLPROPERTIES ('skip.header.line.count'='1');
```

3. Bersihkan 'log_price' (hapus koma, dan ubah menjadi float)

```
CREATE TABLE airbnb_db.listings_cleaned AS
SELECT
    id,
    CAST(REPLACE(log_price, ',', '') AS DOUBLE) AS log_price,
    property_type,
    room_type,
    amenities,
    accommodates,
    CAST(bathrooms AS FLOAT) AS bathrooms,
    bed_type,
    cancellation_policy,
    cleaning_fee,
    city,
    neighbourhood,
    number_of_reviews,
    CAST(review_scores_rating AS FLOAT) AS review_scores_rating,
    CAST(bedrooms AS FLOAT) AS bedrooms,
    CAST(beds AS FLOAT) AS beds
FROM airbnb_db.listings;
```

Langkah 5: Analisis Data (Menggunakan HIVE Queries)
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Harga Rata-rata berdasarkan Kota dan Lingkungan:

```
SELECT city, neighbourhood, AVG(log_price) AS avg_price
FROM airbnb_db.listings_cleaned
GROUP BY city, neighbourhood
ORDER BY avg_price DESC;
```

Hal ini mengidentifikasi lingkungan yang bernilai tinggi, sehingga membantu pemilik menetapkan harga yang kompetitif.

2. Dampak Fasilitas terhadap Harga

Mengurai 'Amenities' (membagi string menjadi beberapa fasilitas):

```
CREATE TABLE airbnb_db.amenities_expanded AS
SELECT id, log_price, amenity
FROM airbnb_db.listings_cleaned
LATERAL VIEW EXPLODE(SPLIT(REGEXP_REPLACE(amenities, '[{}"]', ''), ',')) AS amenity;
```

Hitung harga rata-rata berdasarkan 'amenities'

```
SELECT amenity, AVG(log_price) AS avg_price
FROM airbnb_db.amenities_expanded
GROUP BY amenity
ORDER BY avg_price DESC;
```

Ini menunjukkan fasilitas mana (misalnya, “AC”, “Dapur”) yang meningkatkan harga.

3. Korelasi Antara Fitur dan Harga

Gunakan Hive untuk menghitung rata-rata untuk daftar dengan peringkat tinggi atau lebih banyak kamar tidur

```
SELECT bedrooms, AVG(log_price) AS avg_price
FROM airbnb_db.listings_cleaned
WHERE review_scores_rating >= 90
GROUP BY bedrooms;
```

4. Export Hasil Analisis ke CSV

Dampak Fasilitas terhadap Harga

```
INSERT OVERWRITE DIRECTORY '/user/cloudera/airbnb/price_by_amenity'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n'
STORED AS TEXTFILE
SELECT amenity, ROUND(AVG(log_price), 2) AS avg_price
FROM airbnb_db.amenities_expanded
GROUP BY amenity
ORDER BY avg_price DESC;
```

Korelasi Antara Kamar Tidur dan Harga (Daftar Harga Tinggi)

```
INSERT OVERWRITE DIRECTORY '/user/cloudera/airbnb/price_by_bedrooms'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n'
STORED AS TEXTFILE
SELECT bedrooms, ROUND(AVG(log_price), 2) AS avg_price
FROM airbnb_db.listings_cleaned
WHERE review_scores_rating >= 90
GROUP BY bedrooms
ORDER BY bedrooms;
```

Verifikasi File CSV di HDFS

```
hdfs dfs -ls /user/cloudera/airbnb/price_by_amenity
hdfs dfs -ls /user/cloudera/airbnb/price_by_bedrooms
```

Gabungkan dan salin file HDFS ke direktori Home

```
hdfs dfs -getmerge /user/cloudera/airbnb/price_by_amenity /home/cloudera/price_by_amenity.csv
hdfs dfs -getmerge /user/cloudera/airbnb/price_by_bedrooms /home/cloudera/price_by_bedrooms.csv
```

Langkah 6: Visualisasi Insights
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Pindahkan file hasil analisis ke Shared Folder
2. Masuk ke dalam Google Colab, dan upload file hasil analisis
3. Visualisasikan harga rata-rata berdasarkan fasilitas (amenities)

```
import matplotlib.pyplot as plt

df_price_by_amenity['Amenity'] = df_price_by_amenity['Amenity'].astype(str)

plt.figure(figsize=(12, 6))
plt.bar(df_price_by_amenity['Amenity'], df_price_by_amenity['Average Price'].astype(float))
plt.xlabel('Fasilitas')
plt.ylabel('Harga Rata-rata')
plt.title('Harga Rata-rata berdasarkan Fasilitas')
plt.xticks(rotation=90)
plt.tight_layout()
plt.show()
```

4. Visualisasikan harga rata-rata berdasarkan jumlah kamar tidur (bedrooms)

```
import matplotlib.pyplot as plt

plt.figure(figsize=(10, 6))
plt.bar(df_price_by_bedrooms['Bedrooms'], df_price_by_bedrooms['Average Price'].astype(float))
plt.xlabel('Jumlah Kamar Tidur')
plt.ylabel('Harga Rata-rata')
plt.title('Harga Rata-rata berdasarkan Jumlah Kamar Tidur')
plt.show()
```

5. Visualisasi proporsi harga rata-rata berdasarkan fasilitas (amenities)

```
import matplotlib.pyplot as plt

df_price_by_amenity['Amenity'] = df_price_by_amenity['Amenity'].astype(str)
df_price_by_amenity['Average Price'] = df_price_by_amenity['Average Price'].astype(float)

total_price_amenity = df_price_by_amenity['Average Price'].sum()
df_price_by_amenity['Percentage'] = (df_price_by_amenity['Average Price'] / total_price_amenity) * 100

plt.figure(figsize=(10, 10))
plt.pie(df_price_by_amenity['Average Price'], labels=df_price_by_amenity['Amenity'], autopct='%1.1f%%', startangle=140)
plt.title('Proporsi Harga Rata-rata berdasarkan Fasilitas')
plt.axis('equal')
plt.show()
```

6. Visualisasi proporsi harga rata-rata berdasarkan jumlah kamar tidur (bedrooms)

```
import matplotlib.pyplot as plt

df_price_by_bedrooms['Bedrooms'] = df_price_by_bedrooms['Bedrooms'].astype(str)
df_price_by_bedrooms['Average Price'] = df_price_by_bedrooms['Average Price'].astype(float)

total_price_bedrooms = df_price_by_bedrooms['Average Price'].sum()
df_price_by_bedrooms['Percentage'] = (df_price_by_bedrooms['Average Price'] / total_price_bedrooms) * 100

plt.figure(figsize=(10, 10))
plt.pie(df_price_by_bedrooms['Average Price'], labels=df_price_by_bedrooms['Bedrooms'], autopct='%1.1f%%', startangle=140)
plt.title('Proporsi Harga Rata-rata berdasarkan Jumlah Kamar Tidur')
plt.axis('equal')
plt.show()
```

Langkah 7: Otomatisasi Pipeline (Oozie)
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Kembali ke Cloudera VM, lalu pastikan Oozie aktif

```
oozie admin -oozie http://localhost:11000/oozie -status
```

2. Membuat struktur workflow

```
mkdir -p ~/oozie_workflow
cd ~/oozie_workflow
```

3. Membuat file workflow.xml untuk mendefinisikan workflow Oozie yang akan menjalankan Hive script

```
gedit workflow.xml
```

4. Isi workflow.xml

```
<workflow-app name="price-data-processing" xmlns="uri:oozie:workflow:0.5">
    <global>
        <configuration>
            <property>
                <name>mapreduce.job.queuename</name>
                <value>default</value>
            </property>
        </configuration>
    </global>

    <start to="process-amenity-data"/>

    <action name="process-amenity-data">
        <map-reduce>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>mapred.mapper.class</name>
                    <value>org.apache.hadoop.mapred.lib.IdentityMapper</value>
                </property>
                <property>
                    <name>mapred.reducer.class</name>
                    <value>org.apache.hadoop.mapred.lib.IdentityReducer</value>
                </property>
                <property>
                    <name>mapred.input.dir</name>
                    <value>${amenityInputPath}</value>
                </property>
                <property>
                    <name>mapred.output.dir</name>
                    <value>${amenityOutputPath}</value>
                </property>
                 <property>
                    <name>mapred.job.name</name>
                    <value>ProcessAmenityData</value>
                </property>
            </configuration>
        </map-reduce>
        <ok to="process-bedrooms-data"/>
        <error to="fail"/>
    </action>

    <action name="process-bedrooms-data">
        <map-reduce>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>mapred.mapper.class</name>
                    <value>org.apache.hadoop.mapred.lib.IdentityMapper</value>
                </property>
                <property>
                    <name>mapred.reducer.class</name>
                    <value>org.apache.hadoop.mapred.lib.IdentityReducer</value>
                </property>
                <property>
                    <name>mapred.input.dir</name>
                    <value>${bedroomsInputPath}</value>
                </property>
                <property>
                    <name>mapred.output.dir</name>
                    <value>${bedroomsOutputPath}</value>
                </property>
                 <property>
                    <name>mapred.job.name</name>
                    <value>ProcessBedroomsData</value>
                </property>
            </configuration>
        </map-reduce>
        <ok to="end"/>
        <error to="fail"/>
    </action>

    <kill name="fail">
        <message>Workflow failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>

    <end name="end"/>
</workflow-app>
```

5. Membuat job.properties untuk menentukan konfigurasi Oozie (HDFS path, JobTracker, lokasi workflow).

```
gedit job.properties
```

6. Isi job.properties

```
nameNode=hdfs://quickstart.cloudera:8020
jobTracker=quickstart.cloudera:8032
queueName=default
examplesRoot=oozie_workflow

oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/workflow.xml

amenityInputPath=/user/${user.name}/oozie_data/price_by_amenity.csv
amenityOutputPath=/user/${user.name}/oozie_output/amenity_processed

bedroomsInputPath=/user/${user.name}/oozie_data/price_by_bedrooms.csv
bedroomsOutputPath=/user/${user.name}/oozie_output/bedrooms_processed

oozie.use.system.libpath=true
oozie.action.sharelib.for.mapreduce=mapreduce
```

7. Membuat direktori di HDFS untuk file input

```
hdfs dfs -mkdir /user/cloudera/oozie_data
```

8. Mengunggah file hasil analisis CSV ke HDFS

```
hdfs dfs -put /home/cloudera/price_by_amenity.csv /user/cloudera/oozie_data/
hdfs dfs -put /home/cloudera/price_by_bedrooms.csv /user/cloudera/oozie_data/
```

9. Membuat direktori di HDFS untuk file workflow dan job.properties

```
hdfs dfs -mkdir /user/cloudera/oozie_workflow
```

10. Mengunggah file workflow.xml dan job.properties ke HDFS

```
hdfs dfs -put /home/cloudera/oozie_workflow/workflow.xml /user/cloudera/oozie_workflow/
hdfs dfs -put /home/cloudera/oozie_workflow/job.properties /user/cloudera/oozie_workflow/
```

11. Menjalankan workflow Oozie

```
oozie job -oozie http://quickstart.cloudera:11000/oozie -config /home/cloudera/oozie_workflow/job.properties -run
```

Langkah 8: Performance Tuning
--------------------------------------------------------------------------------------------------------------------------------------------------------------

1. Menjalankan Spark shell pada terminal

```
spark-shell
```

2. Menjalankan wordcount dan hitung durasinya, dilanjutkan dengan menjalankan cache

price_by_amenity

```
val t0 = System.nanoTime()

val data = sc.textFile("hdfs:///user/cloudera/airbnb/price_by_amenity")

val words = data.flatMap(_.split(" "))

val counts = words.map(word => (word, 1)).reduceByKey(_ + _)

counts.count()

val t1 = System.nanoTime()

println("Durasi: " + (t1 - t0) / 1e9 + " detik")
```

Mengaktifkan cache

```
val cachedData = data.cache()
```

price_by_bedrooms

```
val t0 = System.nanoTime()

val data = sc.textFile("hdfs:///user/cloudera/airbnb/price_by_bedrooms")

val words = data.flatMap(_.split(" "))

val counts = words.map(word => (word, 1)).reduceByKey(_ + _)

counts.count()

val t1 = System.nanoTime()

println("Durasi: " + (t1 - t0) / 1e9 + " detik")
```

Mengaktifkan cache

```
val cachedData = data.cache()
```


3. Menjalankan spark kembali setelah caching, namun dengan konfigurasi tuning

```
spark-shell --executor-memory 2G --driver-memory 1G --conf spark.default.parallelism=4
```


4. Menjalankan job word count ulang

price_by_amenity

```
val t0 = System.nanoTime()

val data = sc.textFile("hdfs:///user/cloudera/airbnb/price_by_amenity")

val words = data.flatMap(_.split(" "))

val counts = words.map(word => (word, 1)).reduceByKey(_ + _)

counts.count()

val t1 = System.nanoTime()

println("Durasi: " + (t1 - t0) / 1e9 + " detik")
```

price_by_bedrooms

```
val t0 = System.nanoTime()

val data = sc.textFile("hdfs:///user/cloudera/airbnb/price_by_bedrooms")

val words = data.flatMap(_.split(" "))

val counts = words.map(word => (word, 1)).reduceByKey(_ + _)

counts.count()

val t1 = System.nanoTime()

println("Durasi: " + (t1 - t0) / 1e9 + " detik")
```

5. Bandingkan hasil sebelum dan sesudah tuning