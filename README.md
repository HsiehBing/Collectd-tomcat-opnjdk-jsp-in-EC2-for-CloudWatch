## 緣起: 
Java Web的使用會透過JVM啟動，但是在CloudWatch中無法取得JVM中的虛擬參數(CPU, RAM.....)， \
但是可以透過node-exporter串接prometheus及grafana並輸出至CloudWatch中取得其數據，在2022八月AWS官方釋出文件[1] \
介紹可以直接透過CollectD將JVM的數據直接傳至CloudWatch。

<font color="Red">但是在文件中有許多細節並未說明清楚，因此將過程中會遇到的問題進行統整，以便之後能夠更方便上手</font>

### 測試環境：
開一台t2.micro + tomcat + openjdk

然後弄個jsp檔案上去，用jmeter做壓力測試

目標是能把metrics輸出到cloudwath log

## 相關設定

### 步驟概述：本小節中會單純描述各個流程，詳細流程可以參考下方製作過程說明
1. 在AWS開啟EC2主機AMI為Linux
2. 在環境中安裝openjdk ```sudo yum -y install openjdk ```
3. 以wget下載tomcat
4. 下載collectd  ```sudo yum -y install collectd collectd-java collectd-genetic-jmx ```
5. 下載 ``` sudo yum -y install amazon-clouwatch-agent  ``` 
6. 至CloudWatch查看結果
7. Jmeter並進行壓力測試


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
  ```142.251.222.46|142.251.222.46```

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

### CloudWatch相關設定

* CloudWatch 需要特別建立IAM rule or user [4]， 一定要！！！ 

* 加上IAM role[5]
* 安裝詳細流程可以參考[6]
1. 安裝CloudWatch ```$ sudo yum -y install amazon-cloudwatch-agent```
2. 執行agent \
 ```$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard```
3. 安裝後啟動、停止
 ```
sudo apt-get update && sudo apt-get install collectd -y // 安裝 collectd
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard // 使用精靈產生設定檔或是直接放入設定檔到 /opt/aws/amazon-cloudwatch-agent/bin/config.json
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s // 重新啟動 CloudWatch Agent 並且使用本地 json 設定檔
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:configuration-parameter-store-name -s // 啟動 CloudWatch Agent 並且使用本地 SSM 設定檔
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a start // 開啟
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop // 停止
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status // 檢查狀態
 ```
 4. ```$ sudo vim /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d``` 
 ```
        "metrics_collected": {
            "collectd": { "metrics_aggregation_interval": 60,
                "name_prefix": "petsearch_", "collectd_security_level": "none"
            },
            "disk": {
                "measurement": [
                    "used_percent"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "mem": {
                "measurement": [
                    "mem_used_percent"
                ],
                "metrics_collection_interval": 60
            }
 ``` 
 ** 加入 "collectd": { "metrics_aggregation_interval": 60,
                "name_prefix": "petsearch_", "collectd_security_level": "none" **  \
 
 5. ```$ sudo vim /opt/aws/amazon-cloudwatch-agent/etc/amazon-clouwatch-agent.toml``` \
 在文件中 將[[inputs.socket_listener]]置於[[inputs]]下
 ```
  [[inputs.socket_listener]]
    collectd_auth_file = "/etc/collectd/auth_file"
    collectd_security_level = "none"
    collectd_typesdb = ["/usr/share/collectd/types.db"]
    data_format = "collectd"
    name_prefix = "myapp_"
    service_address = "udp://127.0.0.1:25826"
    [inputs.socket_listener.tags]
      "aws:AggregationInterval" = "60s"
      metricPath = "metrics" 
 ```
 
 



### Jmeter


### 參考資料
[1] Deliver Java JMX statistics to Amazon CloudWatch using the CloudWatch Agent and CollectD \
 https://aws.amazon.com/tw/blogs/mt/deliver-java-jmx-statistics-to-amazon-cloudwatch-using-the-cloudwatch-agent-and-collectd/ \
[2] centos 7 安裝tomcat 8.5 \
 https://blog.yslifes.com/archives/2413 \
[3] Collectd官方文件 \
 https://collectd.org \
[4] Create IAM roles and users for use with CloudWatch agent\
 https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-iam-roles-for-cloudwatch-agent-commandline.html\
[5] IAM roles for Amazon EC2\
 https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#attach-iam-role \
[6] CloudWatch 非官方超明確指南\
https://blog.clarence.tw/2019/08/10/use-cloudwatch-agent-add-ec2-instances-monitor-installation-and-teaching/ \\
[7] \

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
