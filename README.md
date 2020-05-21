kafka-1.1.1在鲲鹏920服务器上，默认的kafka包处理snappy请求时报错：

```
[2020-05-20 19:35:30,174] ERROR [ReplicaManager broker=0] Error processing append operation on partition iot_log_topic-3 (kafka.server.ReplicaManager)
org.apache.kafka.common.KafkaException: Failed to decompress record stream
        at org.apache.kafka.common.record.DefaultRecordBatch$1.readNext(DefaultRecordBatch.java:268)
        at org.apache.kafka.common.record.DefaultRecordBatch$RecordIterator.next(DefaultRecordBatch.java:563)
        at org.apache.kafka.common.record.DefaultRecordBatch$RecordIterator.next(DefaultRecordBatch.java:532)
        at org.apache.kafka.common.record.DefaultRecordBatch.iterator(DefaultRecordBatch.java:327)
        at scala.collection.convert.Wrappers$JIterableWrapper.iterator(Wrappers.scala:54)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at kafka.log.LogValidator$$anonfun$validateMessagesAndAssignOffsetsCompressed$1.apply(LogValidator.scala:267)
        at kafka.log.LogValidator$$anonfun$validateMessagesAndAssignOffsetsCompressed$1.apply(LogValidator.scala:259)
        at scala.collection.Iterator$class.foreach(Iterator.scala:891)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1334)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at kafka.log.LogValidator$.validateMessagesAndAssignOffsetsCompressed(LogValidator.scala:259)
        at kafka.log.LogValidator$.validateMessagesAndAssignOffsets(LogValidator.scala:70)
        at kafka.log.Log$$anonfun$append$2.liftedTree1$1(Log.scala:661)
        at kafka.log.Log$$anonfun$append$2.apply(Log.scala:660)
        at kafka.log.Log$$anonfun$append$2.apply(Log.scala:642)
        at kafka.log.Log.maybeHandleIOException(Log.scala:1696)
        at kafka.log.Log.append(Log.scala:642)
        at kafka.log.Log.appendAsLeader(Log.scala:612)
        at kafka.cluster.Partition$$anonfun$13.apply(Partition.scala:609)
        at kafka.cluster.Partition$$anonfun$13.apply(Partition.scala:597)
        at kafka.utils.CoreUtils$.inLock(CoreUtils.scala:250)
        at kafka.utils.CoreUtils$.inReadLock(CoreUtils.scala:256)
        at kafka.cluster.Partition.appendRecordsToLeader(Partition.scala:596)
        at kafka.server.ReplicaManager$$anonfun$appendToLocalLog$2.apply(ReplicaManager.scala:739)
        at kafka.server.ReplicaManager$$anonfun$appendToLocalLog$2.apply(ReplicaManager.scala:723)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
        at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:130)
        at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:130)
        at scala.collection.mutable.HashTable$class.foreachEntry(HashTable.scala:236)
        at scala.collection.mutable.HashMap.foreachEntry(HashMap.scala:40)
        at scala.collection.mutable.HashMap.foreach(HashMap.scala:130)
        at scala.collection.TraversableLike$class.map(TraversableLike.scala:234)
        at scala.collection.AbstractTraversable.map(Traversable.scala:104)
        at kafka.server.ReplicaManager.appendToLocalLog(ReplicaManager.scala:723)
        at kafka.server.ReplicaManager.appendRecords(ReplicaManager.scala:464)
        at kafka.server.KafkaApis.handleProduceRequest(KafkaApis.scala:466)
        at kafka.server.KafkaApis.handle(KafkaApis.scala:104)
        at kafka.server.KafkaRequestHandler.run(KafkaRequestHandler.scala:69)
        at java.lang.Thread.run(Thread.java:748)
Caused by: java.io.IOException: FAILED_TO_UNCOMPRESS(5)
        at org.xerial.snappy.SnappyNative.throw_error(SnappyNative.java:98)
        at org.xerial.snappy.SnappyNative.rawUncompress(Native Method)
        at org.xerial.snappy.Snappy.rawUncompress(Snappy.java:474)
        at org.xerial.snappy.Snappy.uncompress(Snappy.java:513)
        at org.xerial.snappy.SnappyInputStream.hasNextChunk(SnappyInputStream.java:439)
        at org.xerial.snappy.SnappyInputStream.read(SnappyInputStream.java:466)
        at java.io.DataInputStream.readByte(DataInputStream.java:265)
        at org.apache.kafka.common.utils.ByteUtils.readVarint(ByteUtils.java:168)
        at org.apache.kafka.common.record.DefaultRecord.readFrom(DefaultRecord.java:292)
        at org.apache.kafka.common.record.DefaultRecordBatch$1.readNext(DefaultRecordBatch.java:264)
        ... 42 more
```

原因是$KAFKA_HOME/libs/snappy-java-1.1.7.1.jar里默认带的aarch64的so库有问题。默认的Makefile里将arm64/aarch64默认指定为“Big Endian”，但鲲鹏920是“Little Endian的”：

```shell
$ lscpu | grep "Byte Order"
Byte Order:            Little Endian
```

根据社区相关[issue](https://github.com/xerial/snappy-java/pull/213)，虽然在1.1.7.2之后的新版本里合并了该patch，但kafka-1.1.1替换为snappy-java-1.1.7.2.jar之后的新版本之后报错`java.lang.NoClassDefFoundError: Could not initialize class org.xerial.snappy.Snappy`。故只有针对1.1.7.1源码修改后重新生成so库：`make clean-native native`。并增加kafka启动参数：

```
-Dorg.xerial.snappy.use.systemlib=true -Djava.library.path=/usr/local/kafka_2.11-1.1.1/native_libs/Linux/aarch64
```





snappy-java [![Build Status](https://travis-ci.org/xerial/snappy-java.svg?branch=master)](https://travis-ci.org/xerial/snappy-java) [![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.xerial.snappy/snappy-java/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.xerial.snappy/snappy-java/) [![Javadoc](http://javadoc-badge.appspot.com/org.xerial.snappy/snappy-java.svg?label=javadoc)](http://javadoc-badge.appspot.com/org.xerial.snappy/snappy-java) [![Reference Status](https://www.versioneye.com/java/org.xerial.snappy:snappy-java/reference_badge.svg?style=flat-square)](https://www.versioneye.com/java/org.xerial.snappy:snappy-java/references)
=== 

snappy-java is a Java port of the snappy
<http://code.google.com/p/snappy/>, a fast C++ compresser/decompresser developed by Google.

## Features
  * Fast compression/decompression around 200~400MB/sec.
  * Less memory usage. SnappyOutputStream uses only 32KB+ in default.
  * JNI-based implementation to achieve comparable performance to the native C++ version.
     * Although snappy-java uses JNI, it can be used safely with multiple class loaders (e.g. Tomcat, etc.).
  * Compression/decompression of Java primitive arrays (`float[]`, `double[]`, `int[]`, `short[]`, `long[]`, etc.)
     * To improve the compression ratios of these arrays, you can use a fast data-rearrangement implementation ([`BitShuffle`](https://oss.sonatype.org/service/local/repositories/releases/archive/org/xerial/snappy/snappy-java/1.1.3-M1/snappy-java-1.1.3-M1-javadoc.jar/!/org/xerial/snappy/BitShuffle.html)) before compression
  * Portable across various operating systems; Snappy-java contains native libraries built for Window/Mac/Linux (64-bit). snappy-java loads one of these libraries according to your machine environment (It looks system properties, `os.name` and `os.arch`).
  * Simple usage. Add the snappy-java-(version).jar file to your classpath. Then call compression/decompression methods in `org.xerial.snappy.Snappy`.
  * [Framing-format support](https://github.com/google/snappy/blob/master/framing_format.txt) (Since 1.1.0 version)
  * OSGi support
  * [Apache License Version 2.0](http://www.apache.org/licenses/LICENSE-2.0). Free for both commercial and non-commercial use.

## Performance
  * Snappy's main target is very high-speed compression/decompression with reasonable compression size. So the compression ratio of snappy-java is modest and about the same as `LZF` (ranging 20%-100% according to the dataset).

  * Here are some [benchmark results](https://github.com/ning/jvm-compressor-benchmark/wiki), comparing
 snappy-java and the other compressors
 `LZO-java`/`LZF`/`QuickLZ`/`Gzip`/`Bzip2`. Thanks [Tatu Saloranta @cotowncoder](http://twitter.com/#!/cowtowncoder) for providing the benchmark suite.
  * The benchmark result indicates snappy-java is the fastest compreesor/decompressor in Java: http://ning.github.com/jvm-compressor-benchmark/results/canterbury-roundtrip-2011-07-28/index.html
 * The decompression speed is twice as fast as the others: http://ning.github.com/jvm-compressor-benchmark/results/canterbury-uncompress-2011-07-28/index.html


## Download

 [![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.xerial.snappy/snappy-java/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.xerial.snappy/snappy-java/) [![Javadoc](http://javadoc-badge.appspot.com/org.xerial.snappy/snappy-java.svg?label=javadoc)](http://javadoc-badge.appspot.com/org.xerial.snappy/snappy-java)

 * [Release Notes](Milestone.md)

The current stable version is available from here:
  * Release version: http://central.maven.org/maven2/org/xerial/snappy/snappy-java/
  * Snapshot version (the latest beta version): https://oss.sonatype.org/content/repositories/snapshots/org/xerial/snappy/snappy-java/

### Using with Maven
  * Snappy-java is available from Maven's central repository:  <http://central.maven.org/maven2/org/xerial/snappy/snappy-java>

Add the following dependency to your pom.xml:

    <dependency>
      <groupId>org.xerial.snappy</groupId>
      <artifactId>snappy-java</artifactId>
      <version>(version)</version>
      <type>jar</type>
      <scope>compile</scope>
    </dependency>

### Using with sbt

```
libraryDependencies += "org.xerial.snappy" % "snappy-java" % "(version)"
```


## Usage
First, import `org.xerial.snapy.Snappy` in your Java code:

```java
import org.xerial.snappy.Snappy;
```

Then use `Snappy.compress(byte[])` and `Snappy.uncompress(byte[])`:

```java
String input = "Hello snappy-java! Snappy-java is a JNI-based wrapper of "
     + "Snappy, a fast compresser/decompresser.";
byte[] compressed = Snappy.compress(input.getBytes("UTF-8"));
byte[] uncompressed = Snappy.uncompress(compressed);

String result = new String(uncompressed, "UTF-8");
System.out.println(result);
```

In addition, high-level methods (`Snappy.compress(String)`, `Snappy.compress(float[] ..)` etc. ) and low-level ones (e.g. `Snappy.rawCompress(.. )`,  `Snappy.rawUncompress(..)`, etc.), which minimize memory copies, can be used.

### Stream-based API
Stream-based compressor/decompressor `SnappyOutputStream`/`SnappyInputStream` are also available for reading/writing large data sets. `SnappyFramedOutputStream`/`SnappyFramedInputStream` can be used for the [framing format](https://github.com/google/snappy/blob/master/framing_format.txt). 

 * See also [Javadoc API](https://oss.sonatype.org/service/local/repositories/releases/archive/org/xerial/snappy/snappy-java/1.1.3-M1/snappy-java-1.1.3-M1-javadoc.jar/!/index.html)

#### Compatibility Notes
 * `SnappyOutputStream` and `SnappyInputStream` use `[magic header:16 bytes]([block size:int32][compressed data:byte array])*` format. You can read the result of `Snappy.compress` with `SnappyInputStream`, but you cannot read the compressed data generated by `SnappyOutputStream` with `Snappy.uncompress`. Here is the data format compatibility matrix:
 * `SnappyHadoopCompatibleOutputStream` does not emit a file header but write out the current block size as a  preemble to each block

| Write\Read      | `Snappy.uncompress`   | `SnappyInputStream`  | `SnappyFramedInputStream` | `org.apache.hadoop.io.compress.SnappyCodec` |
| --------------- |:-------------------:|:------------------:|:-----------------------:|:-------------------------------------------:|
| `Snappy.compress` | ok | ok | x | x |
| `SnappyOutputStream`  | x | ok | x | x |
| `SnappyFramedOutputStream` | x | x | ok | x |
| `SnappyHadoopCompatibleOutputStream` | x | x | x | ok |

### BitShuffle API (Since 1.1.3-M2)

BitShuffle is an algorithm that reorders data bits (shuffle) for efficient compression (e.g., a sequence of integers, float values, etc.). To use BitShuffle routines, import `org.xerial.snapy.BitShuffle`:

```java
import org.xerial.snappy.BitShuffle;

int[] data = new int[] {1, 3, 34, 43, 34};
byte[] shuffledByteArray = BitShuffle.shuffle(data);
byte[] compressed = Snappy.compress(shuffledByteArray);
byte[] uncompressed = Snappy.uncompress(compressed);
int[] result = BitShuffle.unshuffleIntArray(uncompress);

System.out.println(result);
```

Shuffling and unshuffling of primitive arrays (e.g., `short[]`, `long[]`,  `float[]`, `double[]`, etc.) are supported. See [Javadoc](http://static.javadoc.io/org.xerial.snappy/snappy-java/1.1.3-M1/org/xerial/snappy/BitShuffle.html) for the details.

### Setting classpath
If you have snappy-java-(VERSION).jar in the current directory, use `-classpath` option as follows:

    $ javac -classpath ".;snappy-java-(VERSION).jar" Sample.java  # in Windows
    or
    $ javac -classpath ".:snappy-java-(VERSION).jar" Sample.java  # in Mac or Linux


## Public discussion group
Post bug reports or feature request to the Issue Tracker: <https://github.com/xerial/snappy-java/issues>

Public discussion forum is here: [Xerial Public Discussion Group](http://groups.google.com/group/xerial?hl=en)

## For developers

snappy-java uses sbt (simple build tool for Scala) as a build tool. Here is a simple usage

    $ ./sbt            # enter sbt console
    > ~test            # run tests upon source code change
    > ~test-only *     # run tests that matches a given name pattern  
    > publishM2        # publish jar to $HOME/.m2/repository
    > package          # create jar file
    > findbugs         # Produce findbugs report in target/findbugs
    > jacoco:cover     # Report the code coverage of tests to target/jacoco folder    

If you need to see detailed debug messages, launch sbt with `-Dloglevel=debug` option:

```
$ ./sbt -Dloglevel=debug
```

For the details of sbt usage, see my blog post: [Building Java Projects with sbt](http://xerial.org/blog/2014/03/24/sbt/)

### Building from the source code
See the [build instruction](https://github.com/xerial/snappy-java/blob/master/BUILD.md). Building from the source code is an option when your OS platform and CPU architecture is not supported. To build snappy-java, you need Git, JDK (1.6 or higher), g++ compiler (mingw in Windows) etc.

    $ git clone https://github.com/xerial/snappy-java.git
    $ cd snappy-java
    $ make

When building on Solaris, use `gmake`:

    $ gmake

A file `target/snappy-java-$(version).jar` is the product additionally containing the native library built for your platform.

## Miscellaneous Notes
### Using snappy-java with Tomcat 6 (or higher) Web Server

Simply put the snappy-java's jar to WEB-INF/lib folder of your web application. Usual JNI-library specific problem no longer exists since snappy-java version 1.0.3 or higher can be loaded by multiple class loaders.


### Configure snappy-java using property file

Prepare org-xerial-snappy.properties file (under the root path of your library) in Java's property file format.
Here is a list of the available properties:

 * org.xerial.snappy.lib.path   (directory containing a snappyjava's native library)
 * org.xerial.snappy.lib.name   (library file name)
 * org.xerial.snappy.tempdir    (temporary directory to extract a native library bundled in snappy-java)
 * org.xerial.snappy.use.systemlib  (if this value is true, use system installed libsnappyjava.so looking the path specified by java.library.path) 

----
Snappy-java is developed by [Taro L. Saito](http://www.xerial.org/leo). Twitter  [@taroleo](http://twitter.com/#!/taroleo)

