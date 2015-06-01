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


.. note::

    * Jay Kreps and log centric architecture


More than Webserver Logs
========================

* Log output can be an event stream

* Processing (and reprocessing) these logs can recreate system state

* Logs are a canonical source of information about what happened

* For Example:
    * MongoDB: the oplog
    * MySQL: transaction logs
    * Nginx: access logs

.. note::

    * Could just be information, or could be more
    * Consider user event streams
    * Logging into site, activating integration


LinkedIn's lattice problem
==========================

.. rst-class:: spaced

    .. image:: ./_static/lattice.png
        :width: 100%
        :align: center

Over time, every system will want to consume data from every other system.


.. note::

    * Systems reacting to other systems
    * Monitoring, etc
    * User logs in, warm ES queries

Enter the unified log
=====================

.. rst-class:: spaced

    .. image:: ./_static/unified_log.png
        :width: 100%
        :align: center

.. note::

    * "How do I make this available?"
    * "Where do I go to get that information?"

Parse.ly is log-centric, too
============================

.. rst-class:: spaced

    .. image:: ./_static/parsely_log_arch.png
        :width: 80%
        :align: center


.. note::

    * Canonical: user events in nginx logs

Introducing Kafka
=================

=============== ==================================================================
Feature         Description
=============== ==================================================================
Speed           100's of megabytes of reads/writes per sec from 1000's of clients
Durability      Can use your entire disk to create a massive message backlog
Scalability     Cluster-oriented design allows for horizontal machine scaling
Availability    Cluster-oriented design allows for node failures without data loss
Multi-consumer  Many clients can read the same stream with no penalty
=============== ==================================================================

.. note::

    * What could handle that volume?
    * Terabytes of log data every day
    * Realtime distribution of data (no uploads to S3)
    * Kafka is...

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
=============== ==================================================================

Kafka layout
============

.. rst-class:: spaced

    .. image:: ./_static/kafka_topology.png
        :width: 80%
        :align: center

.. note::
    * Some of this is behind the scenes since 0.8
    * Zookeeper is...

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

.. note::

    * Almost zero ovhead for adding new consumers
    * vs RabbitMQ: No enqueue, task-in-progress, task-complete
    * vs pub/sub: disk backed


Introducing PyKafka
===================

* Formerly named samsa

* Completely refactored for Kafka 0.8.x

* High performance implementation of Kafka's binary protocol

* Includes implementation of a balancing consumer

* v1.0.0 released this morning!

* Currently in production use at Parse.ly

.. note::

    * ~20k in, ~100k out. 3 AWS instances.
    * (with replication) ~100/250mbps in/out
    * We swamped networking *first*


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

.. note::

    * Now that you have all your logs in this super fast service,
      what do you do with it?
    * You *can* do worker/queue, but ops is non-trivial


Worker problems
===============

* tedious to tune parallelism and load distribution
* no guaranteed processing for multi-stage pipelines
* no fault tolerance for individual stages
* difficult to do local / beta / staging environments
* dependencies between worker stages are unclear

.. note::

    * A series of surmountable headaches
    * Some of these are answered by Kafka

      * Data moving between stages

    * Everyone using it needs some basic understanding of the ops story.


Meanwhile, in Batch land...
===========================

... everything is **peachy**!

When I have all my data available, I can just run Map/Reduce jobs.

**Problem solved.**

We use Apache Pig, and I can get all the gurantees I need, and scale up on EMR.

... but, no ability to do this in real-time on the stream! :(

.. note::

    * I upload my logs to S3, I use spot pricing to keep the costs down and
      then I get the results sometime the following day.  :-\\
    * The code gets where it needs to be when it needs to be there.
    * Yarn is handling provisioning.
    * For the most part, it just works.
    * *I don't need to know ops in order to use it*.

Introducing Storm
=================

Storm is a **distributed real-time computation system**.

Hadoop provides a set of general primitives for doing batch processing.

Storm provides a set of general primitives for doing **real-time computation**.

.. note::

    * What if we could take the good parts of Pig/Hadoop and use them in a realtime system?
    * Storm is...

      * Distributed workers processing realtime streams
      * Provisioning of workers and distribution of code.

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

.. note::

    * Storm and Kafka, two great tastes that test great together
    * Kafka is fast enough to provide data to Storm
    * Storm is fast enough to process Kafka data


Storm features
==============

=============== ====================================================================
Feature         Description
=============== ====================================================================
Speed           1,000,000 tuples per second per node, using Kryo and Netty
Fault Tolerance Workers and Storm management daemons self-heal in face of failure
Parallelism     Tasks run on cluster w/ tuneable parallelism
Guaranteed Msgs Tracks lineage of data tuples, providing an at-least-once guarantee
Easy Code Mgmt  Several versions of code in a cluster; multiple languages supported
Local Dev       Entire system can run in "local mode" for end-to-end testing
=============== ====================================================================

.. note::

    * Tuples are...
    * Don't foget local dev!
        * Local dev line item is cut off in presenter view.

Storm core concepts
===================

=============== =======================================================================
Concept         Description
=============== =======================================================================
Tuple           An individual message passed between nodes in the topology.
Spout           A source of a stream of tuples; typically reading from Kafka
Bolt            Computation steps that consume streams and emits new streams
Topology        Directed Acyclic Graph (DAG) describing Spouts and Bolts
=============== =======================================================================

.. note::

    * Tuples *can* but Python tuples, but don't have to be. Just a list of values.

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

.. note::

    * We're working on replacing the Clojure


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

.. note::

    * This is part of a classic wordcount example
    * Just going to emit words and count them. I don't know why.


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


.. note::

    * Stop here for live demo


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

* http://parsely.com/code
* http://twitter.com/parsely

Me:

* keith@parsely.com
* Twitter: @kbourgoin

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
