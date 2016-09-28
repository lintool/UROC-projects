
# UROC 2016 Project: Tweet Analysis with Spark

## Getting Started

We'll be working with a Hadoop cluster thanks to the generous support of
Altiscale, which is a "Hadoop-as-a-service" provider. You'll be
getting an email directly from Altiscale with account information.

Follow the instructions from the email:

+ Set up your web profile at the [Altiscale Portal](http://portal.altiscale.com/)
+ Follow these instructions to upload your ssh keys: [Uploading and Managing Your Public Key](https://documentation.altiscale.com/uploading-public-key)
+ Follow these instructions to ssh into the "workspace": [Connecting to the Workbench Using SSH](https://documentation.altiscale.com/connecting-with-ssh). The workspace is the node from which you submit Spark jobs; it's also where you'll check out code, inspect HDFS data, etc.
+ Follow these instructions to access the cluster webapps: [Accessing Web UIs Through a SOCKS Proxy](https://documentation.altiscale.com/accessing-web-uis-socks). In particular, you'll need to access the Resource Manager webapp to examine the status of your running jobs at [`http://rm-ia.s3s.altiscale.com:8088/cluster/`](http://rm-ia.s3s.altiscale.com:8088/cluster/)

**The TL;DR version.** Configure your `~/.ssh/config` as follows:

```
Host altiscale
User YOUR_USERNAME
Hostname ia.z42.altiscale.com
Port 1395
IdentityFile ~/.ssh/id_rsa
Compression yes
ServerAliveInterval 15
DynamicForward localhost:1080
TCPKeepAlive yes
Protocol 2,1
```

And you should be able to ssh into the workspace:

```
$ ssh altiscale
```

Once you ssh into the workspace, to properly set up your environment,
add the following lines to your `.bash_profile`:

```
PATH=$PATH:$HOME/bin

export PATH
export SCALA_HOME=/opt/scala
export YARN_CONF_DIR=/etc/hadoop/
export SPARK_HOME=/opt/spark/

cd $SPARK_HOME/test_spark && ./init_spark.sh
cd
```

Running Spark on Altiscale:

In your workspace home directory, you should have
a `bin/` directory. Create a script there
called `my-spark-shell` with the following:

```
#!/bin/bash

/opt/spark/bin/spark-shell --master yarn \
  --driver-class-path $(find /opt/hadoop/share/hadoop/mapreduce/lib/hadoop-lzo-* | head -n 1) "$@"
```

Then `chmod` so that it's executable.

## Your First Spark Script

Take a look at the following file on HDFS that we're going to play with:

```
$ hadoop fs -cat /shared/cs489/uroc2016/Shakespeare.txt | less
```

Start the Spark shell:

```
$ my-spark-shell --num-executors 2 --executor-cores 4 
```

It'll take a few seconds for the Spark shell to fire up. Wait until
you get a prompt. There may be a few warnings, but don't worry about
them.

Copy and paste the following following Spark word count program:

```
scala> :p
// Entering paste mode (ctrl-D to finish)

val textFile = sc.textFile("/shared/cs489/uroc2016/Shakespeare.txt")
val counts = textFile.flatMap(line => line.split(" "))
                 .map(word => (word, 1))
                 .reduceByKey((x, y) => x + y)
counts.saveAsTextFile("Shakespeare-counts/")
```

Note that you use the `:p` command to paste multi-line code into the
Spark shell.

The script should run very quickly.

In another shell, you can examine the output:

```
$ hadoop fs -ls Shakespeare-counts/
Found 3 items
-rw-r--r--   3 jimmylin users          0 2016-09-28 20:24 Shakespeare-counts/_SUCCESS
-rw-r--r--   3 jimmylin users     423214 2016-09-28 20:24 Shakespeare-counts/part-00000
-rw-r--r--   3 jimmylin users     423786 2016-09-28 20:24 Shakespeare-counts/part-00001
$ hadoop fs -cat Shakespeare-counts/part-00000 | less
...
```

If you want to re-run the above Spark script again, first delete the
existing output:

```
$ hadoop fs -rm -r Shakespeare-counts/
```