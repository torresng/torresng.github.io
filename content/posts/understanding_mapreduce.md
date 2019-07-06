---
title: "理解MapReduce"
date: 2019-07-01T21:18:06+08:00
draft: false
---

<!--more-->

## 概念

假如我們有一份很大的數據需要處理，只用單進程單線程需要大量的運行時間，為了減少執行時間，我們可以用多進程多線程并發執行，把一個大的數據划分為獨立較小的數據，然後交給這些進程獨立執行，當這些較小的數據處理完後，我們可以很容易把它們合并成最終結果，這是典型的分治算法，也是MapReduce的主要思想。

當然MapReduce要處理的是大數據的問題，需要解決的事比上面遠大得多。因為單台計算機最後也會出現性能瓶頸，不單擴展單台計算機比較困難且昂貴，而且當單台計算機Crash後，之前所處理的數據全都白費了，所以MapReduce的架構是主要是為了能在多個計算機上實現并行化計算。通過MapReduce可以在大量普通的計算機上實現高性能計算，而且具有可擴展性和高可性的特點。

MapReduce執行部分簡單可以分為單個**Master**和多個**Worker**，**Master**就像公司里的Leader，負責管理，調度和分配任務給小弟，而**Worker**就是拼了命工作的那些小弟，為了防了單台計算機，這些**Worker**分布在集群中不同的計算機上執行，當一個計算機Crash後，**Master**可以將那台計算機所執行的任務分配給其他**Worker**，這樣實現了高可用。那Master死後怎麼辦？那只能呵呵呵了，當然有各種方式可以解決，其中一種就是通知用戶**Master** Crash了，讓用戶決定處理方法。

金![Google Reduce](/media/understanding_mapreduce/Google_MapReduce.jpg)

## map 和 reduce

通過MapReduce，我們可以不用處理分布式上遇到的各種問題，MapReduce為我們處理了并行編程中分布存儲、工作調度、負載均衡、容錯處理及網絡通信等複雜問題，MapReduce任務分為兩個阶段：**map阶段**和**reduce阶段**。每阶段都以鍵值對作為輸入和輸出，其類型由程序員選擇。程序員還需要寫兩個函數：**map函數**和**reduce函數**。**map函數**負責把每個分片進行處理，**reduce函數**負責把map處理後的結果汇總起來。需要注意的是，用MapReduce來處理的數據集（或任務）必須具備這樣的特點：待處理的數據集可以分解成計多小的數據集，而且每一個小數據集都可以完全并行地處理。

## 分片

Hadoop將MapReduce的輸入數據划分成等長的小數據塊，稱為**輸入分片**或**分片**。Hadoop為每個分片构建一個map任務，并由該任務來運行用戶自定義的**map函數**從而處理分片中的每條記錄。

最佳分片的大小應該與塊大小相同：因為它是確保可以存儲在單個節點上的最大輸入塊的大小。如果分片跨越兩個數據塊，那麼對於任何一個 HDFS 節點，基本上都不可能同時存儲這兩個數據塊。

## 數據流

map任務將其輸出寫入本地硬盤，而非HDFS。因為map的輸出是中間結果，如果把它存儲在HDFS中并實現備份，難免有些小題大做。如果運行map任務的節點在將map中間結果傳送給reduce任務之前失敗，Hadoop將在另一個節點上重新運行這個map任務以再次构建map中結果。

而reduce任務不具備數據本地化的優勢，單個reduce任務的輸入通常來自於所有mapper的輸出。reduce的輸出通常存儲在HDFS中以實現可靠存儲，對於reduce輸出的每個HDFS塊，第一個複本存儲在本地節點上，其他複本出於可靠性考慮存儲在其他機架的節點中。因此，將reduce的輸出寫入HDFS確實需要占用網絡帶寬，但這與正常的HDFS管線寫入的消耗一樣。

如果有好多個reduce任務，每個map任務就會針對輸出進行分區，即為每個reduce任務建一個分區。每個分區有許多鍵（及其對應的值），但每個鍵對應的鍵-值對記錄都在同一分區中。分區可由用戶定義的分區函數控制，但通常用默認的partitioner通過哈希函數來分區，很高效。map任務和reduce任務之間的數據流稱為**shuffle**，因為每個reduce任務的輸入都來自許多map任務。

![Google Reduce](/media/understanding_mapreduce/MapReduce_data_flow_with multiple_reduce_tasks.png)

當數據處理可以完全并行（即無需shuffle時），可能會出現無reduce任務的情況。在這種情況下，唯一的非本地節點數據傳輸是map任務將結果寫入HDFS。

![Google Reduce](/media/understanding_mapreduce/MapReduce_data_flow_with_no_reduce_tasks.png)

## combiner函數

集群上的可用帶寬限制了MapReduce作業數量，因此䀆量避免map和reduce任務之間的數據傳輸是有利的（例如傳輸較小的數據）。Hadoop允許用戶針對map任務的輸出指定一個**combiner**（就像mapper和reduce一樣），combiner函數的輸出作為reduce函數的輸入。由於cmobiner屬於優化方案，所以Hadoop無法確定要對一個指定的map任務輸出記錄調用多少次combiner（如果需要）。換而言之，不管調用combiner多少次，0次，1次或多次，reducer的輸出結果都是一樣的。

## 運行代碼

安裝 Java 和 Hadoop 可以參考我另一篇文章 [Install Hadoop 2.7 in CentOS 7](http://www.torresng.com/posts/install_hadoop_2.7_in_centos_7/)。

這里我用 Hadoop權威指南 第二章的代碼做測試，源代碼可以從[這里下載](https://github.com/tomwhite/hadoop-book)。這個代碼計算每年的最高氣溫。

用 git 下載源碼下來後，將數據 sample.txt 複製到源碼路徑下，然後編譯運行。最後輸出結果。

```shell
$ git clone https://github.com/tomwhite/hadoop-book.git
$ cd hadoop-book
$ cp input/ncdc/sample.txt ch02-mr-intro/src/main/java/
$ cd ch02-mr-intro/src/main/java/
$ javac MaxTemperature.java                                                          
Note: MaxTemperature.java uses or overrides a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
$ java MaxTemperature sample.txt output
$ cat output/*
1949    111
1950    22
```

### 代碼

#### 用例數據

sample.txt

```
0067011990999991950051507004+68750+023550FM-12+038299999V0203301N00671220001CN9999999N9+00001+99999999999
0043011990999991950051512004+68750+023550FM-12+038299999V0203201N00671220001CN9999999N9+00221+99999999999
0043011990999991950051518004+68750+023550FM-12+038299999V0203201N00261220001CN9999999N9-00111+99999999999
0043012650999991949032412004+62300+010750FM-12+048599999V0202701N00461220001CN0500001N9+01111+99999999999
0043012650999991949032418004+62300+010750FM-12+048599999V0202701N00461220001CN0500001N9+00781+99999999999
```

每行的第15個字符到第19個字符為年份，第87到第92個字符為溫度，第93個字符為質量代碼。

#### map函數

MaxTemperatureMapper.java

```java

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class MaxTemperatureMapper
  extends Mapper<LongWritable, Text, Text, IntWritable> {

  private static final int MISSING = 9999;
  
  @Override
  public void map(LongWritable key, Text value, Context context)
      throws IOException, InterruptedException {
    
    String line = value.toString();
    String year = line.substring(15, 19);
    int airTemperature;
    if (line.charAt(87) == '+') { // parseInt doesn't like leading plus signs
      airTemperature = Integer.parseInt(line.substring(88, 92));
    } else {
      airTemperature = Integer.parseInt(line.substring(87, 92));
    }
    String quality = line.substring(92, 93);
    if (airTemperature != MISSING && quality.matches("[01459]")) {
      context.write(new Text(year), new IntWritable(airTemperature));
    }
  }
}
```

這個 Mapper 類是一個泛型類型，它有四個形參類型，分別指定 map 函數的輸入鍵、輸入值、輸出鍵和輸出的類型。這個例子中，輸入鍵是一個長整數偏移量，輸入值是一行文本，輸出鍵是年份，輸出值是氣溫（整數）。

#### reduce函數

MaxTemperatureReducer.java

```java
import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class MaxTemperatureReducer
  extends Reducer<Text, IntWritable, Text, IntWritable> {
  
  @Override
  public void reduce(Text key, Iterable<IntWritable> values,
      Context context)
      throws IOException, InterruptedException {
    
    int maxValue = Integer.MIN_VALUE;
    for (IntWritable value : values) {
      maxValue = Math.max(maxValue, value.get());
    }
    context.write(key, new IntWritable(maxValue));
  }
}
```

同樣，reduce 函數也有四個形式參數類型用於指定輸入和輸出類型。reduce 函數的輸入類型必須匹配 map 函數的輸出類型：即 Text 類型和 IntWritable 類型。

#### 負責運行 MapReduce 作業

MaxTemperature.java

```java
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class MaxTemperature {

  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperature <input path> <output path>");
      System.exit(-1);
    }
    
    Job job = new Job();
    job.setJarByClass(MaxTemperature.class);
    job.setJobName("Max temperature");

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

通過 **setMapperClass()** 和 **setReducerClass()** 方法指定要用的 map 類型和  reduce 類型。

**setOutputKeyClass()** 和 **setOutputValueClass()** 方法控制 reduce 函數的輸出類型，并且必須和 Reduce 類產生的相匹配。map 函數的輸出類型默認情況下和 reduce 函數是相同的，因此如果 mapper 產生出和 reducer 相同的類型時（如同本例所示），不需要單獨設置。否則，必須通過 **setMapOutputKeyClass()** 和 **setMapOutputValueClass()** 方法來設置 map 函數的輸出類型。

輸入的類型通過輸入格式來控制，我們的例子中沒有設置，因為使用的是默認的 **TextInputFormat**（文字輸入格式）。