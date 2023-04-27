### 緣起: 
Java Web的使用會透過JVM啟動，但是在CloudWatch中無法取得JVM中的虛擬參數(CPU, RAM.....)， \
但是可以透過node-exporter串接prometheus及grafana並輸出至CloudWatch中取得其數據，在2022八月AWS官方釋出文件[1] \
介紹可以直接透過CollectD將JVM的數據直接傳至CloudWatch。

<font color="Red">但是在文件中有許多細節並未說明清楚，因此將過程中會遇到的問題進行統整，以便之後能夠更方便上手</font>

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
 
### Jmeter

### CloudWatch相關設定

### 參考資料
[1] https://aws.amazon.com/tw/blogs/mt/deliver-java-jmx-statistics-to-amazon-cloudwatch-using-the-cloudwatch-agent-and-collectd/
[2] https://blog.yslifes.com/archives/2413
[3] https://collectd.org
[4]

### 最後編輯時間


### 待做事項
1. 驗證openjdk安裝程序

### 編輯記錄
