---
title: Hadoop 설치, 설정 - Ubuntu 18.04
category: Record
date: 2018-06-20T12:00:00Z
lastmod: 2018-06-20T12:00:00Z
comment: true
adsense: true
---

### 1. 설치 환경

* Ubuntu 18.04 LTS 64bit, root user
* Java openjdk version "1.8.0_171"
* Hadoop 3.0.3

### 2. sshd 설치, 설정

* Haddop 설치모든 Hadoop Node에 root User로 Password 없이 Login 할 수 있도록 설정한다.
* sshd를 설치한다.

~~~
# apt update
# apt install -y openssh-server
# apt install -y pdsh
~~~

* /etc/ssh/sshd_config 파일에 아래와 같이 수정하여 root Login 허용한다.

<figure>
{% highlight text %}
...
#LoginGraceTime 2m
PermitRootLogin yes
#StrictModes yes
...
{% endhighlight %}
<figcaption class="caption">[파일 1] /etc/ssh/sshd_config</figcaption>
</figure>

* sshd 재시작 및 ssh 접속시 password가 불필요하도록 설정한다.

~~~
# service sshd restart
# ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
# cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
# chmod 0600 ~/.ssh/authorized_keys
# echo "ssh" > /etc/pdsh/rcmd_default
# ssh localhost

...
Are you sure you want to continue connecting (yes/no)? yes
~~~

### 3. Java 설치 

* Java Package를 설치한다.

~~~
# apt update
# apt install -y openjdk-8-jdk
~~~

### 4. Hadoop 설치, 설정

* Hadoop Binary를 Download 한다.

~~~
# cd ~
# wget http://mirror.navercorp.com/apache/hadoop/common/hadoop-3.0.3/hadoop-3.0.3.tar.gz
# tar zxvf hadoop-3.0.3.tar.gz
~~~

* ~/hadoop-3.0.3/etc/hadoop/hadoop-env.sh 파일을 아래와 같이 수정한다.

<figure>
{% highlight text %}
# The java implementation to use. By default, this environment
# variable is REQUIRED on ALL platforms except OS X!
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
{% endhighlight %}
<figcaption class="caption">[파일 2] ~/hadoop-3.0.3/etc/hadoop/hadoop-env.sh</figcaption>
</figure>

* ~/hadoop-3.0.3/etc/hadoop/core-site.xml 파일을 아래와 같이 수정한다.

<figure>
{% highlight xml %}
<configuration>
	<property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
{% endhighlight %}
<figcaption class="caption">[파일 3] ~/hadoop-3.0.3/etc/hadoop/core-site.xml</figcaption>
</figure>

* ~/hadoop-3.0.3/etc/hadoop/core-site.xml 파일을 아래와 같이 수정한다.

<figure>
{% highlight xml %}
<configuration>
	<property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
{% endhighlight %}
<figcaption class="caption">[파일 4] ~/hadoop-3.0.3/etc/hadoop/core-site.xml</figcaption>
</figure>

* ~/.bashrc 파일에 아래의 환경변수를 추가한다.

<figure>
{% highlight text %}
...
export HADOOP_HOME="/root/hadoop-3.0.3"
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME

export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"
{% endhighlight %}
<figcaption class="caption">[파일 5] ~/.bashrc</figcaption>
</figure>

* HDFS Format 및 HDFS을 시작한다.

~~~
# hdfs namenode -format
# start-dfs.sh
~~~

* HDFS 동작을 확인한다.
  * Web Browser에서 http://localhost:9870 접속한다.

### 5. YARN 설치, 설정

* root user 폴더를 생성한다.

~~~
# cd ~/hadoop-3.0.
# bin/hdfs dfs -mkdir /user
# bin/hdfs dfs -mkdir /user/root
~~~

* ~/hadoop-3.0.3/etc/hadoop/mapred-site.xml 파일을 아래와 같이 수정한다.

<figure>
{% highlight xml %}
<configuration>
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
	<property>
		<name>yarn.app.mapreduce.am.env</name>
		<value>HADOOP_MAPRED_HOME=/root/hadoop-3.0.3</value>
	</property>
	<property>
		<name>mapreduce.map.env</name>
		<value>HADOOP_MAPRED_HOME=/root/hadoop-3.0.3</value>
	</property>
	<property>
		<name>mapreduce.reduce.env</name>
		<value>HADOOP_MAPRED_HOME=/root/hadoop-3.0.3</value>
	</property>
</configuration>
{% endhighlight %}
<figcaption class="caption">[파일 6] ~/hadoop-3.0.3/etc/hadoop/mapred-site.xml</figcaption>
</figure>

* ~/hadoop-3.0.3/etc/hadoop/yarn-site.xml 파일을 아래와 같이 수정한다.

<figure>
{% highlight xml %}
<configuration>
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
	<property>
		<name>yarn.nodemanager.vmem-check-enabled</name>
		<value>false</value>
	</property>
</configuration>
{% endhighlight %}
<figcaption class="caption">[파일 7] ~/hadoop-3.0.3/etc/hadoop/yarn-site.xml</figcaption>
</figure>

* YARN을 시작한다. 

~~~
# start-yarn.sh
~~~

* YARN 동작을 확인한다. 
  * Web Browser에서 http://localhost:8088 접속한다.

### 6. 동작 확인

* 6개의 JVM 동작을 확인한다.

~~~
# jps
3988 NameNode
5707 Jps
5355 NodeManager
4203 DataNode
4492 SecondaryNameNode
5133 ResourceManager
~~~

* Example을 구동한다.

~~~
# cd ~/hadoop-3.0.3
# yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.0.3.jar pi 16 1000
...
Estimated value of Pi is 3.14250000000000000000
~~~

### 7. Issue 해결

* There are 0 datanode(s) Error 발생시 아래와 같이 수행한다.

~~~
# stop-yarn.sh
# stop-dfs.sh
# rm -rf /tmp/*
# start-dfs.sh
# start-yarn.sh
~~~

### 8. 참조

* [http://www.admintome.com/blog/installing-hadoop-on-ubuntu-17-10/](http://www.admintome.com/blog/installing-hadoop-on-ubuntu-17-10/)
* [https://data-flair.training/blogs/installation-of-hadoop-3-x-on-ubuntu/](https://data-flair.training/blogs/installation-of-hadoop-3-x-on-ubuntu/)
