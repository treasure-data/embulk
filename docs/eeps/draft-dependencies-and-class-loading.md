---
EEP: unnumbered
Title: Dependencies and Class Loading
Author: dmikurube
Status: Draft
Type: Standards Track
Created: 2023-01-18
---

Introduction
=============

This EEP describes

Problem: Dependency Conflicts
==============================

The Embulk core is not very trivial by itself, then it had several library dependencies for purposes. For example, [SnakeYAML](https://bitbucket.org/snakeyaml/snakeyaml/) for parsing user's configuration YAML, [Jackson](https://github.com/FasterXML/jackson) for parsing JSON data, and representing internal configuration data, [Guice](https://github.com/google/guice) for managing object life cycles, [Guava](https://github.com/google/guava) for several utilities, [SLF4J](https://www.slf4j.org/) and [Logback](https://logback.qos.ch/) for logging, ...

On the other hand, Embulk plugins usually need their own library dependencies, and more transitive dependencies, for their purposes. Some plugins may also need [Jackson](https://github.com/FasterXML/jackson) and its extention librarires for various data structures. Some AWS-related plugins need [AWS SDK for Java](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/welcome.html), and [its v1 needs Jackson transitively for itself](https://central.sonatype.dev/artifact/com.amazonaws/aws-java-sdk-core/1.12.385/dependencies). Some plugins may need [Guava](https://github.com/google/guava) for itself. Some Google Cloud-related plugins need [Google Cloud Client Library for Java](https://github.com/googleapis/google-cloud-java), and [it needs Jackson and Guava transitively for itself](https://central.sonatype.dev/artifact/com.google.cloud/google-cloud-storage/2.17.1/dependencies).

Jackson
--------

Jackson had been causing the biggest and heaviest dependency conflicts. Then let's start from Jackson.

Apache BVal / Validation API
-----------------------------

SnakeYAML
----------

Guice / Guava
--------------

JRuby
------

Joda-Time
----------

Findbugs
---------

MessagePack for Java
---------------------

SLF4J + Logback
----------------


Class Loader Tweaks
====================

Alternatives considered
------------------------

Migration
==========


Plugin SPI Changes
===================


Backwards Compatibility
========================


Copyright / License
====================

This document is placed under the [CC0-1.0-Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.en) license, whichever is more permissive.
