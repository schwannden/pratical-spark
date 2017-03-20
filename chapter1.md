# Install on Ubuntu 16.04

The book only give instruction on installing spark 2.1 on Ubuntu 16.04, for other version of your machine, please find the guideline on Google.

```bash
wget http://d3kbcqa49mib13.cloudfront.net/spark-2.1.0-bin-hadoop2.7.tgz
tar -xzvf spark-2.1.0-bin-hadoop2.7.tgz
sudo mv spark/ /usr/lib/
```

Next since we are using Scala, install the Scala build tool

```bash
echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
sudo apt-get update
sudo apt-get install sbt
```

Since Scala is a JVM language, install Java 8 too

```bash
sudo apt-add-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

Finally add the following path to your bash startup script \(~/.bashrc\). Now the exact path of your JAVA\_\_HOME and SBT\_\_HOME might be different from mine even if you follow the exactg same step. Please look it up for sure

```bash
 export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
 export SBT_HOME=/usr/share/sbt-launcher-packaging/bin/sbt-launch.jar
 export SPARK_HOME=/usr/lib/spark
 export PATH=$PATH:$JAVA_HOME/bin
 export PATH=$PATH:$SBT_HOME/bin:$SPARK_HOME/bin:$SPARK_HOME/sbin
```



