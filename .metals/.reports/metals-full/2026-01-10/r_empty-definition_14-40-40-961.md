error id: file:///C:/Users/rubae/IdeaProjects/scalaTest/src/main/scala/SearchEngine.scala:`<none>`.
file:///C:/Users/rubae/IdeaProjects/scalaTest/src/main/scala/SearchEngine.scala
empty definition using pc, found symbol in pc: `<none>`.
empty definition using semanticdb
empty definition using fallback
non-local guesses:
	 -totalCount.
	 -totalCount#
	 -totalCount().
	 -scala/Predef.totalCount.
	 -scala/Predef.totalCount#
	 -scala/Predef.totalCount().
offset: 1904
uri: file:///C:/Users/rubae/IdeaProjects/scalaTest/src/main/scala/SearchEngine.scala
text:
```scala
// Ruba Egbaria: 12457003

import org.apache.spark.SparkContext

import scala.io.Source
import java.io.{File, PrintWriter}
import org.apache.spark.sql.SparkSession

import javax.management.Query
import scala.io.StdIn

//@main
object SearchEngine {

  def main(args: Array[String]): Unit = {
    println("In the main")

    // starting a spark session
    val spark = SparkSession
      .builder()
      .appName("SearchEngine")
      .master("local[*]")
      .config("spark.some.config.option", "")
      .getOrCreate()
    // spark context
    val sc = spark.sparkContext

    sc.setLogLevel("ERROR")

    // generating data out of a hugging face API, each doc file is a prompt, stored in data folder within the src folder
    generateData()

    // loading the data into the RDD
    val docsRDD = sc.parallelize(
      new File("src/data").listFiles
        .filter(_.isFile)
        .toList
        .map(f => (f.getName, Source.fromFile(f).mkString))
    )
    docsRDD.take(5).foreach(println)

    // cleaning up all non chars, then convert everything to small letters for easier search
    val cleanSmallLetters = (str: String) => {
      str.replaceAll("[^a-zA-Z]", "")
        .toLowerCase
        .trim
    }

    // invertedIndex, considering this format XX, 2, x1.txt, x20.txt
    val invertedIndexRDD = docsRDD.flatMap { case (fileName, content) =>
        content
          .split("\\s+")
          .map(cleanSmallLetters)
          .filter(_.nonEmpty)
          .map(word => (word, fileName))
      }
      .aggregateByKey((0, Set.empty[String]))(
        //    don't save the .txt part, unneeded use of resources
        { case ((count, files), fileName) => (count + 1, files + fileName.replaceAll(".txt", ""))},
        { case ((count1, files1), (count2, files2)) => (count1 + count2, files1 ++ files2)}
      )
      .map { case (word, (totalCount, filesSet)) =>
        (word, totalCoun@@t, filesSet.toList.sorted.mkString(","))
      }
      .sortBy(_._1)

    // saving the file
    val outFile = new PrintWriter(new File("wholeInvertedIndex.txt"))
    invertedIndexRDD.collect().foreach { case (word, count, files) =>
      outFile.println(s"$word, $count, $files")
    }
    outFile.close()

    // user input
    println("Search Engine Ready for Use")
    println("What are you searching for, Provide your Sentence:")
    val userInput = StdIn.readLine()

    println(s"Searching for $userInput ...")

    val query = userInput.split(" ").map(_.toLowerCase)

    processingResult(query,sc)
    spark.stop()
}

//  cleaning the user input & preparing the outcome
  private def processingResult(query: Array[String], sc : SparkContext): Unit = {
  //    cleaning the query
    val lowerCasing = (str: String ) => {
      str.toLowerCase.trim
    }
    val lowerCaseQuery = query.map(lowerCasing).filter(_.nonEmpty)

    if (lowerCaseQuery.isEmpty) {
      println("Rerun the code please, your input is invalid")
      return
    } else {
      // reading the file using RDD
//      reading occurs only if the input is valid (saving resources)
      val indexRDD = sc.textFile("wholeInvertedIndex.txt")

//      if it is only one word
      if (lowerCaseQuery.length == 1) {
        val word = lowerCaseQuery.head

        val count = indexRDD.filter(_.startsWith(word + ","))
          .flatMap(_.split(",").slice(1, 2))
          .first()

        val results = indexRDD.filter(_.startsWith(word + ","))
          .flatMap(_.split(",").drop(2))
          .collect()

        if (results.isEmpty) {
          println(s"Search Engine Output: $word not found")
        } else {
          println(s"Search Engine Output: \n'$word', $count, ${results.sorted.mkString(", ")}")
        }
      }
      // more than 1 word
      else {
        // word by word get their existence files,
        // then where they have a join then count the files
        val docSets = lowerCaseQuery.filter(_.nonEmpty).map { word =>
          val docs = indexRDD
            .filter(_.startsWith(word + ","))
            .flatMap(_.split(",").drop(2))
            .collect()
            .toSet
          (word, docs)
        }

        val cleanedDocSets =
        docSets.map { case (word, docs) =>
          word -> docs.map(_.trim)
        }

        val emptyTerms = cleanedDocSets.filter(_._2.isEmpty)
        if (emptyTerms.nonEmpty) {
          println("Search Engine Output: not found")
        } else {
          val intersection = cleanedDocSets
            .map(_._2)
            .reduce(_ intersect _)

          if (intersection.isEmpty) {
            println("Search Engine Output: words do not intersect in any document")
          } else {
            val combinedWords = lowerCaseQuery.mkString(" ")
            println(
              s"Search Engine Output:\n'$combinedWords', ${intersection.size}, ${intersection.toList.sorted.mkString(", ")}"
            )
          }
        }
      }
    }
  }

  //  creating the data folder and the docs
  private def generateData(): Unit = {
  //  create the src/data folder (if it does not exist) so we can generate the docs
  val dataPath = new File("src/data")
    if(!dataPath.exists())
      dataPath.mkdirs()

    //  src/data folder is empty? then create files
    if(dataPath.list().isEmpty) {
      // loading the data from the API
      //  this data contains 'act' and 'prompt' cols, where we gonna pull all the prompts into docs
      //  prompt1 -> doc1.txt...
      val promptsUrlCsv = "https://huggingface.co/datasets/fka/awesome-chatgpt-prompts/resolve/main/prompts.csv"
      val source = Source.fromURL(promptsUrlCsv)("UTF-8")

      try {
        // removing the col name (first row)
        val lines = source.getLines().drop(1)
        var counter = 1

        // creating docs out of the data rows: prompt1 -> doc1.txt
        lines.foreach { line =>
          val columns = line.split(",", 2).map(_.trim)
          if (columns.length > 1) {
            val prompt = columns(1).stripPrefix("\"").stripSuffix("\"")
            val fileName = s"src/data/doc$counter.txt"
            val writer = new PrintWriter(new File(fileName))
            writer.write(prompt)
            writer.close()
            counter += 1
          }
        }
        println("Data and the docs are generated and ready")
      } catch {
        case e: Exception =>
          Console.err.println(s"Request failed: ${e.getMessage}")
          sys.exit(1)
      }
    }
  }
}
```


#### Short summary: 

empty definition using pc, found symbol in pc: `<none>`.