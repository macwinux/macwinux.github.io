---
layout: posts
title: PySpark en Scala
---

He querido comenzar este blog con una investigación que tuve que hacer durante un proyecto en el que trabajé. La dificultad radicaba en que queriamos hacer un producto sobre Spark que corriese código embebido sql, scala o python en un yaml.

Casi toda la lógica se escribía en Spark SQL y cuando querías hacer algo más complejo se solía utilizar los UDFs de Scala.
El problema es que un determinado caso de uso necesitaba el uso de una librería de python, por lo tanto modificamos el código de tal manera que fuese capaz de procesar tambien udfs en python (aunque no era muy eficiente).

Como ejemplo he dejado este repositorio de ejemplo: [repositorio](https://github.com/macwinux/pyspark-in-scala).

Dentro de la carpeta main se puede ver que hay dos carpetas, la carpeta con el código python y la carpeta src del código scala que arranca la session de Spark. 

Vamos a explicar el ciclo de procesamiento de los dos códigos por partes:

1. El código de scala es bastante sencillo. Primero se definen ciertas variables mutables como el `JavaSparkContext` y el `SparkConf`. Se declaran asi en vez de inmutables porque luego se acceden a ellas con los metodos `getJsc()` y `getConf()`, que los necesitaremos en python, de ahi que se deba declarar todo fuera del main.

``` scala
def getJsc(): JavaSparkContext = jsc
def getConf(): SparkConf = conf
var jsc: JavaSparkContext = _ 
var conf: SparkConf = _
```

2. Seguidamente en el main instanciamos la `SparkSession` que vamos a utilizar y con el contexto de esta sesión creamos el contexto Java, y con este su `SparkConf` y se asignan ambos a las variables antes instanciadas. Despues vamos a crear una Vista temporal en spark SQL con datos randoms:

``` scala
val spark = SparkSession.builder().appName("Spark Python Runner")
                    .master("local[1]")
                    .getOrCreate()
jsc = new JavaSparkContext(spark.sparkContext)
conf = jsc.getConf
import spark.implicits._
val columns = Seq("language","users_count")
val data = Seq(("Java", "20000"), ("Python", "100000"), ("Scala", "3000"))
val rdd = spark.sparkContext.parallelize(data)
val df = rdd.toDF(columns:_*)
df.createOrReplaceTempView("table")
```

3. Una vez creada la vista ejecutamos el código de python:

``` scala
PythonRunner.main(Array(
      "src/main/python/example.py",
      "src/main/python/example.py"
    ))
```

4. La parte de pyspark que busca la sesión activa por el código scala:

``` python
gateway = pyspark.java_gateway.launch_gateway()
java_import(gateway.jvm, "com.example.pysparkscala.PysparkScala")
jsc = gateway.jvm.PysparkScala.getJsc()
jconf = gateway.jvm.PysparkScala.getConf()
conf = pyspark.conf.SparkConf(True, gateway.jvm, jconf)
sc = pyspark.SparkContext(gateway=gateway, jsc=jsc, conf=conf)
spark = SparkSession(sc)
```

Esta parte se comunica con el código scala de spark mediante el java_gateway. Recoge la sesión y se instancia esta sesión con `spark = SparkSession(sc)`.
Ya puede usar pyspark como siempre, lo bueno es que ahora tanto la sesión de scala como la python es la misma, y ahora lo veremos.

5. Leemos la tabla "table" que creamos anteriormente en scala. Como comparten la sesión se puede acceder a ella desde python. Creamos una nueva columna y reemplazamos la vista:

``` python
df = spark.sql("SELECT * FROM table")
df_processed = df.withColumn("len", length('language').alias('len'))
df_processed.createOrReplaceTempView("table")
```

6. Finalmente, creamos un udf que simplemente mide la longitud del string con pandas_udf y la registramos en spark SQL:

``` python
@pandas_udf(IntegerType())
def slen(s: pd.Series) -> pd.Series:
    return s.str.len()

spark.udf.register("slen", slen)
```

7. Al estar registrada esta última parte se puede hacer tanto en python como en scala, aunque nosotros la hemos hecho en python. Simplemente llamamos a la nueva función con Spark SQL y reemplazamos de nuevo la vista:

``` python
df_udf = spark.sql("SELECT language, users_count, len, slen(language) as udf_len FROM table")
df_udf.createOrReplaceTempView("table")
```

8. Ahora la última parte del ejemplo es volver a hacer un select con scala para ver que la tabla contiene todos los cambios realizados con python:

``` scala
spark.sql("SELECT * FROM table").show()
```

De esta manera hemos visto como podemos interactuar con código en distintos lenguajes compartiendo la misma Session en Spark.