### 緣起: 
Java Web的使用會透過JVM啟動，但是在CloudWatch中無法取得JVM中的虛擬參數(CPU, RAM.....)， \
但是可以透過node-exporter串接prometheus及grafana並輸出至CloudWatch中取得其數據，在2022八月AWS官方釋出文件[1](https://aws.amazon.com/tw/blogs/mt/deliver-java-jmx-statistics-to-amazon-cloudwatch-using-the-cloudwatch-agent-and-collectd/ "Title")， \
介紹可以直接透過CollectD將JVM的數據直接傳至CloudWatch。

<font color="Red">但是在文件中有許多細節並未說明清楚，因此將過程中會遇到的問題進行統整，以便之後能夠更方便上手</font>
This is [an example](http://example.com/ "Title") inline link.
<font color=Blue>Test</font>
## 相關設定

### 步驟概述：本小節中會單純描述各個流程，詳細流程可以參考下方製作過程說明
1. 在AWS開啟EC2主機AMI為Linux
2. 在環境中安裝openjdk ```sudo yum -y install openjdk ```
3. 以wget下載tomcat
4. 下載collectd  ```sudo yum -y install collectd ```
5. 下載 ``` sudo yum -y install amazon-clouwatch-agent  ``` 
6. 至


### jsp製作


### openjdk設定

### Tomcat設定

### collectd設定

### CloudWatch相關設定

### 參考資料
[1] https://aws.amazon.com/tw/blogs/mt/deliver-java-jmx-statistics-to-amazon-cloudwatch-using-the-cloudwatch-agent-and-collectd/
[2]
