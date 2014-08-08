.. SparkConfig documentation master file, created by
   sphinx-quickstart on Fri Aug  8 10:18:15 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

.. toctree::
   :maxdepth: 2


Spark性能相关参数配置
=====================

Contents:

概述
----

随着Spark的逐渐成熟完善, 越来越多的可配置参数被添加到Spark中来, 在Spark的官方文档 http://spark.apache.org/docs/latest/configuration.html 中提供了这些可配置参数中相当大一部分的说明.

但是文档的更新总是落后于代码的开发的, 还有一些配置参数没有来得及被添加到这个文档中, 此外在这个文档中,对于许多的参数也只能简单的介绍它所代表的内容的字面含义, 如果没有一定的实践基础或者对其背后原理的理解, 往往无法真正理解该如何针对具体应用场合进行合理配置.

本文试图通过阐述这其中部分参数的工作原理和配置思路, 和大家一起学习探讨一下如何根据实际场合对Spark进行配置优化. 

由于本文主要针对和性能相关的一些配置参数进行阐述，所以不会覆盖其它和性能没有直接关系的配置。


* :doc:`./general`
* :doc:`./shuffle`


Indices and tables
------------------

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

