
# UROC 2018 Project: Tweet Analysis with Spark

## Getting Started

Log in:

```sh
$ ssh datasci.cs.uwaterloo.ca
```

## Your First Spark Script

Take a look at the following file on HDFS that we're going to play with:

```sh
$ hadoop fs -cat /data/uroc2018/Shakespeare.txt | less
```

Start the Spark shell:

```sh
$ spark-shell --num-executors 2 --executor-cores 4
```

It'll take several seconds for the Spark shell to fire up. Wait until
you get a prompt. There may be a few warnings, but don't worry about
them.

Copy and paste the following following Spark word count program:

```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)

val textFile = sc.textFile("/data/uroc2018/Shakespeare.txt")
val counts = textFile.flatMap(line => line.split(" "))
                 .map(word => (word, 1))
                 .reduceByKey((x, y) => x + y)
counts.saveAsTextFile("Shakespeare-counts/")
```

Note that you use the `:paste` command to paste multi-line code into the
Spark shell.

In another shell, you can examine the output:

```sh
$ hadoop fs -ls Shakespeare-counts/
Found 3 items
-rw-r--r--   3 jimmylin jimmylin          0 2018-09-24 12:11 Shakespeare-counts/_SUCCESS
-rw-r--r--   3 jimmylin jimmylin     423214 2018-09-24 12:11 Shakespeare-counts/part-00000
-rw-r--r--   3 jimmylin jimmylin     423786 2018-09-24 12:11 Shakespeare-counts/part-00001
$ hadoop fs -cat Shakespeare-counts/part-00000 | less
...
```

If you want to re-run the above Spark script again, first delete the
existing output:

```sh
$ hadoop fs -rm -r Shakespeare-counts/
```

## Tweet Analysis with the Archives Unleashed Toolkit

[Archives Unleashed Toolkit
(UAT)](https://github.com/archivesunleashed/aut) is a platform that
some colleagues and I have been developing for analyzing web archives
and tweets.

To save you some time, the UAT jar as been pre-built and stored at:

```
/tmp/aut/aut-0.16.1-SNAPSHOT-fatjar.jar
```

We've gathered some tweets for you, stored on HDFS at `/data/uroc2018/tweets/`. To list them:

```sh
$ hadoop fs -ls /data/uroc2018/tweets/
```

To examine each individual file containing the tweets:

```sh
$ hadoop fs -cat /data/uroc2018/tweets/statuses.log.2016-11-01-00.gz | gunzip -c | less
```

It might be helpful when you are developing to just run over a few
files, e.g., `statuses.log.2016-11-01*`.

Let's get crunching! Fire up the Spark shell as follows:

```sh
$ spark-shell --jars /tmp/aut/aut-0.16.1-SNAPSHOT-fatjar.jar \
    --num-executors 4 --executor-cores 4 --executor-memory 24G --driver-memory 8G
```

Try the following script:

```scala
import io.archivesunleashed._
import io.archivesunleashed.matchbox._
import io.archivesunleashed.util.TweetUtils._

// Load tweets from HDFS
val tweets = RecordLoader.loadTweets("/data/uroc2018/tweets", sc)

// Count them
tweets.count()

// Extract some fields
val r = tweets.map(tweet => (tweet.id, tweet.createdAt, tweet.username, tweet.text, tweet.lang,
                             tweet.isVerifiedUser, tweet.followerCount, tweet.friendCount))

// Take a sample of 10 on console
r.take(10)

// Count the different number of languages
val s = tweets.map(tweet => tweet.lang).countItems().collect()

// Count the number of hashtags
// (Note we don't 'collect' here because it's too much data to bring into the shell)
val hashtags = tweets.map(tweet => tweet.text)
                     .filter(text => text != null)
                     .flatMap(text => {"""#[^ ]+""".r.findAllIn(text).toList})
                     .countItems()

// Take the top 10 hashtags
hashtags.take(10)
```

Here's an example of how you parse the `created_at` date field:

```scala
import io.archivesunleashed._
import io.archivesunleashed.matchbox._
import io.archivesunleashed.util.TweetUtils._
import java.text.SimpleDateFormat
import java.util.TimeZone

val tweets = RecordLoader.loadTweets("/data/uroc2018/tweets", sc)

val counts = tweets.map(tweet => tweet.createdAt)
  .mapPartitions(iter => {
      TimeZone.setDefault(TimeZone.getTimeZone("UTC"))
      val dateIn = new SimpleDateFormat("EEE MMM dd HH:mm:ss ZZZZZ yyyy")
      val dateOut = new SimpleDateFormat("yyyy-MM-dd")
    iter.map(d => try { dateOut.format(dateIn.parse(d)) } catch { case e: Exception => null })})
  .filter(d => d != null)
  .countItems()
  .sortByKey()
  .collect()
```

Let's put multiple elements above together and count daily mentions of [@HillaryClinton](https://twitter.com/HillaryClinton):

```scala
import io.archivesunleashed._
import io.archivesunleashed.matchbox._
import io.archivesunleashed.util.TweetUtils._
import java.text.SimpleDateFormat
import java.util.TimeZone

val tweets = RecordLoader.loadTweets("/data/uroc2018/tweets", sc)

val clintonCounts = tweets
  .filter(tweet => tweet.text != null && tweet.text.contains("@HillaryClinton"))
  .map(tweet => tweet.createdAt)
  .mapPartitions(iter => {
      TimeZone.setDefault(TimeZone.getTimeZone("UTC"))
      val dateIn = new SimpleDateFormat("EEE MMM dd HH:mm:ss ZZZZZ yyyy")
      val dateOut = new SimpleDateFormat("yyyy-MM-dd")
    iter.map(d => try { dateOut.format(dateIn.parse(d)) } catch { case e: Exception => null })})
  .filter(d => d != null)
  .countItems()
  .sortByKey()
  .collect()
```

Simple exercises you might want to try out:

+ Count daily mentions of [@realDonaldTrump](https://twitter.com/realDonaldTrump):
+ Visualize the results using something like [Rickshaw](https://shutterstock.github.io/rickshaw/)
+ Perform the same analysis on a minute-by-minute granularity, focusing on election night.

## Parsing JSON

What if you want to do more and access more data inside tweets?
Tweets are just JSON objects, see examples
[here](https://gist.github.com/hrp/900964) and
[here](https://gist.github.com/gnip/764239).  Twitter has [detailed
API documentation](https://dev.twitter.com/overview/api/tweets) that
tells you what all the fields mean.

The Archives Unleashed Toolkit internally uses
[json4s](https://github.com/json4s/json4s) to access fields in
JSON. You can manipulate fields directly to access any part of tweets.
Here are some examples:

```scala
import org.json4s._
import org.json4s.jackson.JsonMethods._

val sampleTweet = """  [insert tweet in JSON format here] """
val json = parse(sampleTweet)
```

The you can do something like:

```scala
implicit lazy val formats = org.json4s.DefaultFormats

// Extract id
(json \ "id_str").extract[String]

// Extract created_at
(json \ "created_at").extract[String]
```

