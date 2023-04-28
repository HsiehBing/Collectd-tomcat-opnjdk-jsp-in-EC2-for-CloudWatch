## 緣起: 
Java Web的使用會透過JVM啟動，但是在CloudWatch中無法取得JVM中的虛擬參數(CPU, RAM.....)， \
但是可以透過node-exporter串接prometheus及grafana並輸出至CloudWatch中取得其數據，在2022八月AWS官方釋出文件[1] \
介紹可以直接透過CollectD將JVM的數據直接傳至CloudWatch。

<font color="Red">但是在文件中有許多細節並未說明清楚，因此將過程中會遇到的問題進行統整，以便之後能夠更方便上手</font>

### 測試環境：
開一台t2.micro + tomcat + openjdk

然後弄個jsp檔案上去，用jmeter去壓一

主要是看能不能把metrics輸出到cloudwath log

## 相關設定

### 步驟概述：本小節中會單純描述各個流程，詳細流程可以參考下方製作過程說明
1. 在AWS開啟EC2主機AMI為Linux
2. 在環境中安裝openjdk ```sudo yum -y install openjdk ```
3. 以wget下載tomcat
4. 下載collectd  ```sudo yum -y install collectd collectd-java collectd-genetic-jmx ```
5. 下載 ``` sudo yum -y install amazon-clouwatch-agent  ``` 
6. 下載Jmeter並進行測試
7. 至CloudWatch查看結果


### jsp製作


### openjdk設定
``` sudo yum install java- ```

### Tomcat設定 參考並修改[2]
1. 使用wget取得資料
```$ cd /tmp & wget -c https://downloads.apache.org/tomcat/tomcat-9/v9.0.74/bin/apache-tomcat-9.0.74.tar.gz``
下載前請先至https://downloads.apache.org/tomcat/tomcat-9/ 確認版本，並在更改後進行下載

2. 解壓縮並修改目錄user及group
```
$ sudo tar -zxf /tmp/apache-tomcat-*.tar.gz -C /opt
$ sudo ln -s /opt/apache-tomcat-* /opt/tomcat
$ sudo chown -hR tomcat: /opt/tomcat /opt/apache-tomcat-*
```
3.參考文章中是建立systemd，但是這邊只是為了測試，所以直接啟動， \
啟動前需要至做三件事情
 * 建立啟動環境
 ```
$ vim /opt/tomcat/apache-tomcat-9.0.74/bin/setenv.sh 
 ```
 設定JMX環境
 ```  
-Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false  
```

```
$ chown tomcat:tomcat /opt/tomcat/apache-tomcat-9.0.74/bin/setenv.sh
```

 * 增加使用者
```
$ vim /opt/tomcat/apache-tomcat-9.0.74

```
在最後</tomcat-users>上方新增
```
<role rolename="tomcat"/>
<user username="tomcat" password="s3cret" roles="manager-gui, manager-script, manager-jmx, manager-status"/>
```

 * 增加IP位址
 ```
 $ vim opt/tomcat/apache-tomcat-9.0.74/webapps/manager/META-INF/tomcat-user.xml
 ```
 到最後幾列尋找<Valve className=....> 下的allow=""，將自己的public IP加入，如果有多組IP可由｜分隔 \
 設定完後就可以啟動tomcat中並以port8080開啟Tomcat
  例如 \
  ```142.251.222.46:8080```

### collectd設定[3]
  
```
$ vim /etc/collectd.conf
```
修改
```
<Plugin logfile>
        LogLevel info
        File "/var/log/messages"
        Timestamp true
        PrintSeverity false
</Plugin>  
  
<Plugin syslog>
        LogLevel info
</Plugin>
 ```
 在60 61行
``` 
LoadPlugin logfile
LoadPlugin syslog 
 ```
 Load Plugin logfile要寫在前面
 另外要在最後加上附錄[1]
### Jmeter

### CloudWatch相關設定

### 參考資料
[1] https://aws.amazon.com/tw/blogs/mt/deliver-java-jmx-statistics-to-amazon-cloudwatch-using-the-cloudwatch-agent-and-collectd/ \
[2] https://blog.yslifes.com/archives/2413 \
[3] https://collectd.org \
[4]

### 最後編輯時間
2023/4/27

### 待做事項
1. 驗證openjdk安裝程序

2. Tomcat systemd設定

3. Tomcat 啟動教學

 
#### 附錄
 [1]置於/etc/collectd/collectd.conf最後(路徑也有可能是/etc/collectd.conf)
 ```
 LoadPlugin network
<Plugin network>
    Server "127.0.0.1" "25826"
</Plugin>

LoadPlugin java
<Plugin java>
# JVMArg "-verbose:jni"
  JVMArg "-Djava.class.path=/usr/share/collectd/java/collectd-api.jar:/usr/share/collectd/java/generic-jmx.jar"
  LoadPlugin "org.collectd.java.GenericJMX"

  <Plugin "GenericJMX">
    <MBean "Memory">
      ObjectName "java.lang:type=Memory"
      InstancePrefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_memory_heapmemoryusage_"
        Table true
        Attribute "HeapMemoryUsage"
      </Value>
    </MBean>

    <MBean "OperatingSystem">
      ObjectName "java.lang:type=OperatingSystem"
      InstancePrefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_operatingsystem_maxfiledescriptorcount"
        Attribute "MaxFileDescriptorCount"
      </Value>
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_operatingsystem_openfiledescriptorcount"
        Attribute "OpenFileDescriptorCount"
      </Value>
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_operatingsystem_freephysicalmemorysize"
        Attribute "FreePhysicalMemorySize"
      </Value>
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_operatingsystem_freeswapsizespace"
        Attribute "FreeSwapSpaceSize"
      </Value>
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_operatingsystem_committedvirtualmemorysize"
        Attribute "CommittedVirtualMemorySize"
      </Value>
    </MBean>

    <MBean "Threading">
      ObjectName "java.lang:type=Threading"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_threading_threadcount"
        Attribute "ThreadCount"
      </Value>
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_threading_daemonthreadcount"
        Attribute "DaemonThreadCount"
      </Value>
    </MBean>

    <MBean "ClassLoading">
      ObjectName "java.lang:type=ClassLoading"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_classloading_loadedclasscount"
        Attribute "LoadedClassCount"
      </Value>
    </MBean>

    <MBean "GCCollectiontimeCopy">
      ObjectName "java.lang:name=Copy,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_copy"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimePSScavenge">
      ObjectName "java.lang:name=PS Scavenge,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_ps_scavenge"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimeParNew">
      ObjectName "java.lang:name=ParNew,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_parnew"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimeMarkSweepCompact">
      ObjectName "java.lang:name=MarkSweepCompact,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_marksweepcompact"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimePSMarkSweep">
      ObjectName "java.lang:name=PS MarkSweep,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_ps_marksweep"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimeConcurrentMarkSweep">
      ObjectName "java.lang:name=ConcurrentMarkSweep,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_concurrentmarksweep"
        Attribute "CollectionTime"
      </Value>
    </MBean>
 
    <MBean "GCCollectiontimeG1Young">
      ObjectName "java.lang:name=G1 Young Generation,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "derive"
        InstancePrefix "java_lang_garbagecollector_collectiontime_g1_young_generation"
        Attribute "CollectionTime"
      </Value>
    </MBean>
 
    <MBean "GCCollectiontimeG1Old">
      ObjectName "java.lang:name=G1 Old Generation,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_g1_old_generation"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <MBean "GCCollectiontimeG1Mixed">
      ObjectName "java.lang:name=G1 Mixed Generation,type=GarbageCollector"
      Instanceprefix "java"
      <Value>
        Type "gauge"
        InstancePrefix "java_lang_garbagecollector_collectiontime_g1_mixed_generation"
        Attribute "CollectionTime"
      </Value>
    </MBean>

    <Connection>
      Host "localhost"
      ServiceURL "service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi"
      Collect "Memory"
      Collect "OperatingSystem"
      Collect "Threading"
      Collect "ClassLoading"
      Collect "GCCollectiontimeCopy"
      Collect "GCCollectiontimePSScavenge"
      Collect "GCCollectiontimeParNew"
      Collect "GCCollectiontimeMarkSweepCompact"
      Collect "GCCollectiontimePSMarkSweep"
      Collect "GCCollectiontimeConcurrentMarkSweep"
      Collect "GCCollectiontimeG1Young"
      Collect "GCCollectiontimeG1Old"
      Collect "GCCollectiontimeG1Mixed"
    </Connection>
  </Plugin>
</Plugin>
```
