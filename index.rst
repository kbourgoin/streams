========================================
Using Python with Apache Storm and Kafka
========================================

Keith Bourgoin |br|
Backend Lead @ Parse.ly

.. rst-class:: logo

    .. image:: ./_static/parsely.png
        :width: 40%
        :align: right

.. |br| raw:: html

    <br />

Agenda
======

* Who we are
* Organizing around logs (Kafka)
* Aggregating the stream (Storm)
* Real-time vs Batch tensions

What is Parse.ly?
=================

Analytics for digital storytellers.

    .. image:: ./_static/banner_01.png
        :align: center
    .. image:: ./_static/banner_02.png
        :align: center
    .. image:: ./_static/banner_03.png
        :align: center
    .. image:: ./_static/banner_04.png
        :align: center

We Combine Many Streams
=======================

Audience data:
    * visits
    * sessions

Engagement data:
    * views / time spent
    * social shares

Crawl data:
    * keywords / topics
    * author / section / tag


To Create a Cohesive Picture
============================

.. rst-class:: spaced

    .. image:: ./_static/glimpse.png
        :width: 100%
        :align: center



======================
Organizing around logs
======================

More than Webserver Logs
========================

* Log output can be an event stream

* Processing (and reprocessing) these logs can recreate system state

* Logs are a canonical source of information about what happened

* For Example:
    * MongoDB: the oplog
    * MySQL: transaction logs
    * Nginx: access logs


LinkedIn's lattice problem
==========================

.. rst-class:: spaced

    .. image:: ./_static/lattice.png
        :width: 100%
        :align: center

Over time, every system will want to consume data from every other system.

Enter the unified log
=====================

.. rst-class:: spaced

    .. image:: ./_static/unified_log.png
        :width: 100%
        :align: center

Log-centric is simpler
======================

.. rst-class:: spaced

    .. image:: ./_static/log_centric.png
        :width: 65%
        :align: center

Parse.ly is log-centric, too
============================

.. rst-class:: spaced

    .. image:: ./_static/parsely_log_arch.png
        :width: 80%
        :align: center

Introducing Kafka
=================

=============== ==================================================================
Feature         Description
=============== ==================================================================
Speed           100's of megabytes of reads/writes per sec from 1000's of clients
Durability      Can use your entire disk to create a massive message backlog
Scalability     Cluster-oriented design allows for horizontal machine scaling
Availability    Cluster-oriented design allows for node failures without data loss (in 0.8+)
Multi-consumer  Many clients can read the same stream with no penalty
=============== ==================================================================

Kafka concepts
==============

=============== ==================================================================
Concept         Description
=============== ==================================================================
Topic           A group of related messages (a stream)
Producer        Procs that publish msgs to stream
Consumer        Procs that subscribe to msgs from stream
Broker          An individual node in the Cluster
Cluster         An arrangement of Brokers & Zookeeper nodes
Offset          Coordinated state between Consumers and Brokers (in Zookeeper)
=============== ==================================================================

Kafka layout
============

.. rst-class:: spaced

    .. image:: ./_static/kafka_topology.png
        :width: 80%
        :align: center

Kafka is a "distributed log"
============================

Topics are **logs**, not queues.

Consumers **read into offsets of the log**.

Consumers **do not "eat" messages**.

Logs are **maintained for a configurable period of time**.

Messages can be **"replayed"**.

Consumers can **share identical logs easily**.

Multi-consumer
==============

.. rst-class:: spaced

    .. image:: ./_static/multiconsumer.png
        :width: 60%
        :align: center

Even if Kafka's availability and scalability story isn't interesting to you,
the **multi-consumer story should be**.


Introducing PyKafka
===================

* Formerly named samsa

* Completely refactored for Kafka 0.8.x

* High performance implementation of Kafka's binary protocol

* Includes implementation of a balancing consumer

<TODO: Insert Benchmark Data> |br|
(sorry, it's not done for tonight's presentation)

PyKafka in Action
=================

.. sourcecode:: python

    from pykafka import KafkaClient

    client = KafkaClient()
    topic = client.topics['server_logs']
    producer = topic.get_producer()
    for i in xrange(10000):
        producer.produce('message {}'.format(i))

.. sourcecode:: python

    from pykafka import KafkaClient

    client = KafkaClient()
    topic = client.topics['server_logs']
    consumer = topic.get_balanced_consumer()
    for msg in consumer:
        print msg


======================
Aggregating the stream
======================

So, what do you do with the Logs?
=================================

You could use RabbitMQ or another worker/queue system.

.. rst-class:: spaced

    .. image:: /_static/tech_stack.png
        :width: 70%
        :align: center

We tried that.


Worker problems
===============

* no control for parallelism and load distribution
* no guaranteed processing for multi-stage pipelines
* no fault tolerance for individual stages
* difficult to do local / beta / staging environments
* dependencies between worker stages are unclear

Meanwhile, in Batch land...
===========================

... everything is **peachy**!

When I have all my data available, I can just run Map/Reduce jobs.

**Problem solved.**

We use Apache Pig, and I can get all the gurantees I need, and scale up on EMR.

... but, no ability to do this in real-time on the stream! :(

Introducing Storm
=================

Storm is a **distributed real-time computation system**.

Hadoop provides a set of general primitives for doing batch processing.

Storm provides a set of **general primitives** for doing **real-time computation**.

Hadoop primitives
=================

**Durable** Data Set, typically from **S3**.

**HDFS** used for inter-process communication.

**Mappers** & **Reducers**; Pig's **JobFlow** is a **DAG**.

**JobTracker** & **TaskTracker** manage execution.

**Tuneable parallelism** + built-in **fault tolerance**.

Storm primitives
================

**Streaming** Data Set, typically from **Kafka**.

**Netty** used for inter-process communication.

**Bolts** & **Spouts**; Storm's **Topology** is a **DAG**.

**Nimbus** & **Workers** manage execution.

**Tuneable parallelism** + built-in **fault tolerance**.

Storm features
==============

=============== ====================================================================
Feature         Description
=============== ====================================================================
Speed           1,000,000 tuples per second per node, using Kyro and Netty
Fault Tolerance Workers and Storm management daemons self-heal in face of failure
Parallelism     Tasks run on cluster w/ tuneable parallelism
Guaranteed Msgs Tracks lineage of data tuples, providing an at-least-once guarantee
Easy Code Mgmt  Several versions of code in a cluster; multiple languages supported
Local Dev       Entire system can run in "local mode" for end-to-end testing
=============== ====================================================================

Storm core concepts
===================

=============== =======================================================================
Concept         Description
=============== =======================================================================
Stream          Unbounded sequence of data tuples with named fields
Spout           A source of a Stream of tuples; typically reading from Kafka
Bolt            Computation steps that consume Streams and emits new Streams
Grouping        Way of partitioning data fed to a Bolt; for example: by field, shuffle
Topology        Directed Acyclic Graph (DAG) describing Spouts, Bolts, & Groupings
=============== =======================================================================

Wired Topology
==============

.. rst-class:: spaced

    .. image:: ./_static/topology.png
        :width: 80%
        :align: center


Enter Streamparse
=================

Avoid Java, use Python!

* Pure python bolt/spout implementation
* Clojure for topology definition
* Includes tools for submitting and managing topologies

.. sourcecode:: bash

    # Run the topology locally
    $ sparse run -n my_topology

    # Submit to a remote cluster
    $ sparse submit -n my_topology

    # List/kill running topologies
    $ sparse list
    $ sparse kill -n my_topology


A Simple Spout
==============


.. sourcecode:: python

    import itertools
    from streamparse.spout import Spout

    class WordSpout(Spout):

        def initialize(self, stormconf, context):
            self.words = itertools.cycle(['dog', 'cat',
                                          'zebra', 'elephant'])

        def next_tuple(self):
            word = next(self.words)
            self.emit([word])


A Simple Bolt
=============

.. sourcecode:: python

    from collections import Counter
    from streamparse.bolt import Bolt


    class WordCounter(Bolt):

        def initialize(self, conf, ctx):
            self.counts = Counter()

        def process(self, tup):
            word = tup.values[0]
            self.counts[word] += 1
            self.emit([word, self.counts[word]])
            self.log('%s: %d' % (word, self.counts[word]))


The Topology Definition
=======================

.. sourcecode:: clojure

    (ns wordcount
      (:use     [streamparse.specs])
      (:gen-class))

    (defn wordcount [options]
       [{"word-spout" (python-spout-spec
              options
              "spouts.words.WordSpout"
              ["word"]
              )
        },
        {"count-bolt" (python-bolt-spec
              options
              {"word-spout" :shuffle}
              "bolts.wordcount.WordCounter"
              ["word" "count"]
              :p 2
              )
        }]
    )


Putting It All Together
=======================


.. rst-class:: spaced

    .. image:: ./_static/quickstart.gif
        :width: 95%
        :align: center


Not Just for Simple Tasks!
==========================

* Most of the Parse.ly stack is built on streamparse

* Performant, stable and mature

* Supports:

  * streams
  * time-based batching bolts
  * all multilang features Storm exposes



Questions?
==========

Go forth and stream!

Parse.ly:

* http://parse.ly
* http://twitter.com/parsely

Projects

* http://github.com/parsely/pykafka
* http://github.com/parsely/streamparse
* http://www.parsely.com/code/

Me:

* http://twitter.com/kbourgoin

.. ifnotslides::

    .. raw:: html

        <script>
        $(function() {
            $("body").css("width", "1080px");
            $(".sphinxsidebar").css({"width": "200px", "font-size": "12px"});
            $(".bodywrapper").css("margin", "auto");
            $(".documentwrapper").css("width", "880px");
            $(".logo").removeClass("align-right");
        });
        </script>

.. ifslides::

    .. raw:: html

        <script>
        $("tr").each(function() { 
            $(this).find("td:first").css("background-color", "#eee"); 
        });
        </script>
