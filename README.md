# Elastic Sentiment Analysis (ElSA)

The *Elastic Sentiment Analysis* (ElSA) is a  Spark Streaming-based application written for the [DCOS](http://mesosphere.com/product/). It derives public opinions/sentiments on specified Twitter topics and is able to elastically scale its processing capacity, based on the volume of the topics' traffic, leveraging Apache Mesos and Marathon.

ElSA works as follows:

* It takes a list of words (called topics in the following), such as *Mesos*, *Docker*, *DCOS*, etc., as input and—using the Twitter firehose—pulls tweets containing these topics for processing.
* Based on the tweet content it performs a simple sentiment analysis in an ongoing fashion. This operation is implemented via [Spark Streaming](https://spark.apache.org/docs/latest/streaming-programming-guide.html).
* Last but not least, based on the activity in a certain topic the app scales elastically through leveraging the [Marathon REST API](https://mesosphere.github.io/marathon/docs/rest-api.html). This means that if, for example, a rapid increase of mentions of the topic *DCOS* is detected (tweets per time unit), then more instances are launched.

See below for the architecture and data flow as well as [deployment](#deployment) and [usage](#usage) instructions.

## Architecture

| components                            | flow                 |
| ------------------------------------- | -------------------- |
| ![Architecture](doc/architecture.png) |![Flow](doc/flow.png) |

Description TBD.

## Deployment

In the following, an Ubuntu 14.04 environment is assumed and in addition, ElSA depends on:

* Apache [Mesos 0.21.x](http://archive.apache.org/dist/mesos/0.21.0/) with [Marathon 0.7.6](https://github.com/mesosphere/marathon/releases/tag/v0.7.6)
* [marathon-python](https://github.com/thefactory/marathon-python)
* Apache [Spark 1.2.x](https://spark.apache.org/downloads.html)
* A Twitter account and an [app](https://apps.twitter.com/) that can be used for accessing the Twitter firehose

### ElSA to-go: Vagrant deployment 

**ElSA to-go** is a single-node Vagrant deployment based on the ingenious [Playa Mesos](https://github.com/mesosphere/playa-mesos). See details in [deployment/to-go](deployment/to-go) …


### ElSA Docker


### Digital Ocean deployment

IaaS deployment on [DO](https://cloud.digitalocean.com/)

### Google Compute deployment

IaaS deployment on [GCE](https://cloud.google.com/)

### AWS EC2 deployment

IaaS deployment on [EC2](https://console.aws.amazon.com/)


### Manual single node deployment

For Python packages we need `pip` so before anything else do:

     $ sudo apt-get install python-pip

Now we can start with the setup.

**Install Mesos**: simply use [Playa Mesos](https://github.com/mesosphere/playa-mesos) which contains an Marathon installation or follow the [step-by-step instructions](http://mesos.apache.org/gettingstarted/) from the Apache Mesos site and install Marathon on top of it.

Further, as a preparation for the ElSA app, we need a [Python package](https://github.com/thefactory/marathon-python) wrapping the Marathon [REST API](https://mesosphere.github.io/marathon/docs/rest-api.html) so let's do that right away:

    $ pip install marathon

**Install Spark**:

First we download the Spark source and make sure Java env is set up correctly:

    $ cd
    $ wget http://d3kbcqa49mib13.cloudfront.net/spark-1.2.0.tgz
    $ tar xzf spark-1.2.0.tgz && cd spark-1.2.0/
    $ sudo apt-get install default-jdk
    $ export JAVA_HOME=$(readlink -f /usr/bin/javac | sed "s:bin/javac::")

Now make sure the correct version of Maven (3.0.4 or higher) is available:

    $ sudo apt-get update
    $ sudo apt-get install maven
    $ mvn -version
    Apache Maven 3.0.5
    Maven home: /usr/share/maven
    Java version: 1.7.0_65, vendor: Oracle Corporation
    Java home: /usr/lib/jvm/java-7-openjdk-amd64/jre

OK, ready to build Spark. Note: right now is a good time to get a cup of tea or coffee, whatever floats your boat. As usual, Maven is downloading half of the Internet for the following and that might take, um, a while:

    $ export MAVEN_OPTS="-Xmx2g -XX:MaxPermSize=512M -XX:ReservedCodeCacheSize=512m"
    $ sudo mvn -DskipTests clean package

So, here we are. Next we package our newly built Spark distribution for the Mesos slaves to use:

    $ ./make-distribution.sh
    $ mv dist spark-1.2.0
    $ tar czf spark-1.2.0.tgz spark-1.2.0/
    $ cd conf/
    $ cp spark-env.sh.template spark-env.sh

Now open `../spark-1.2.0/conf/spark-env.sh` in your favorite editor and add the following at the end of the file:

    export MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
    export SPARK_EXECUTOR_URI=file:///home/vagrant/spark-1.2.0/spark-1.2.0.tgz
    export MASTER=mesos://127.0.1.1:5050

Note that if you've built Spark in a different directory (I did it in `/home/vagrant/`) then you'll have to change the setting for the `SPARK_EXECUTOR_URI` to point to the resulting `tgz` file from the previous step,  

Then, finally, we're ready to launch Spark:

    $ cd ..
    $ bin/spark-shell

**Install Elsa**:

    $ cd
    $ git clone https://github.com/mhausenblas/elsa.git
    $ cd elsa
    $ mvn clean package

## Usage

Assuming you've installed ElSA using one of the options above, you should now be in the position to launch it as described below.

Note: in order for ElSA to run you'll need to supply your Twitter credentials, that is, you `cp elsa.conf.example elsa.conf` and replace the `YOUR STUFF HERE` sections with the details you obtain from creating a Twitter application and generating the access token via the [app](https://apps.twitter.com/) interface.

Before you start, you might want to quickly check out this 3min walkthrough of ElSA op:

<a href="http://www.youtube.com/watch?feature=player_embedded&v=WfV2Qk-Xy5g" target="_blank"><img src="http://img.youtube.com/vi/WfV2Qk-Xy5g/0.jpg" alt="ElSA op video walkthrough on YouTUb" width="480" height="360" border="0" /></a>


### Launching ElSA manually

To launch ElSA manually (without elasticity, directly on Mesos), do the following:

    $ cd elsa
    $ ./launch-elsa.sh
    Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
    15/03/06 19:02:32 INFO ElsaHelper: Setting log level to [ERROR].
    
    In the past 5 seconds I found 0 tweet(s) containing your topics: Datacenter DCOS Docker Mesos Mesosphere devop microservice
    
    In the past 5 seconds I found 1 tweet(s) containing your topics: Datacenter DCOS Docker Mesos Mesosphere devop microservice
    ===
    RT @SoftLayer: Let’s talk software, specifically how to create a private @Docker registry on SoftLayer. ≡ http://t.co/UVpX1Anl4s http://t.c…
    ===

## Launching Elsa through Marathon

To launch the ElSA app (through Marathon) and automatically scale the number of instances used, depending on the increase/decrease of traffic detected for the specified topics, do the following (hint: stop app by hitting `CTRL+C`):

    $ cd elsa
    $ ./autoscale.py http://localhost:8080 elsa.conf
    Using /tmp/elsa/stats to monitor topic traffic
    Using traffic increase threshold of 10 and scale factor 5
    ElSA is deployed and running, waiting now 5 sec before starting auto-scale ...
    Difference in traffic in the past 10 seconds: 9
    Difference in traffic in the past 10 seconds: -9
    Resetting number of instances to 1
    Difference in traffic in the past 10 seconds: 11
    Increasing number of instances to 2
    Difference in traffic in the past 10 seconds: -14
    Resetting number of instances to 1
    ^CElSA has been stopped by user, halting app and rolling back deployment. Thanks and bye!

You should then see something like the following in [Marathon](http://10.141.141.10:8080/):

![ElSA Marathon deployment](doc/elsa-marathon-deploy.png)

## To do

- [x] Core business logic 
- [x] Single node deployment and launch
- [x] Single node elastic
- [x] Make all auto-scale parameter configurable via config
- [x] Improve SA (positive negative)
- [x] Video walkthrough
- [x] Vagrant file
- [ ] Docker image
- [ ] Cluster deployment DO
- [ ] Cluster deployment GCE
- [ ] Cluster deployment EC2
- [ ] Architecture and flow explanation

## Notes

Kudos to the Spark team for providing [the basis](https://github.com/apache/spark/blob/master/examples/src/main/scala/org/apache/spark/examples/streaming/TwitterPopularTags.scala) for the SA part and to Alexandre Rodrigues for helping me out concerning the initial development.

If you want to learn how to run Spark on Mesos, I suggest you try out the great [step-by-step tutorial](https://mesosphere.com/docs/tutorials/run-spark-on-mesos/) provided by the Mesosphere folks.

Lastly, apologies to all [Frozen](http://www.imdb.com/title/tt2294629/) fans, especially our kids, for hijacking the Elsa label in this context. I thought it's funny … 
