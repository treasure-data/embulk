---
EEP: unnumbered
Title: The "Compact Core" Principle
Author: dmikurube
Status: Draft
Type: Standards Track
Created: 2023-01-16
---

Introduction
=============

This EEP describes the design principle of the Embulk core, especially about what kind of features should be implemented in the core, or in each plugin. Whenever a new feature is proposed for the Embulk core, the core team members would think about this at first.

Motivation
===========

Embulk has provided several features, librarires, and utilities for plugins from the core since its beginning. They have made it easier to implement a new plugin in its early age, however, they have brought unnecessary dependencies on the core, and often blocked making changes in the Embulk core.

Many of such existing features in the core have already been marked as deprecated through Embulk v0.10. The motivation of this EEP is to encourage keeping the Embulk core "compact", and to discourage future expansions that could bring unnecessary dependencies.

Examples
---------

There are several painful examples in the Embulk history.

### Configurations, and Jackson

### TempFileSpace

### TimestampFormatter / TimestampParser, and Joda-Time

### JsonParser, and msgpack-java

### JRuby

JRuby is the biggest pain through the Embulk history.

The Principle, and Guidelines
==============================

The Embulk core should keep "compact" to avoid plugins to have unnecessary dependencies on the core details.

When a feature is proposed for the Embulk core, consider the following points as guidelines.

* Can it be implemented just as a utility library for plugins?
    * If it can, go for the utility library way.
* Could plugins be impacted when the feature has a change in the future?
    * If future changes there could impact existing plugins, try minimizing the part to be implemented in the core.
    * One idea is to add only an interface in the core, and to have implementations in each plugin.
* Does the extended thing handed over from plugins to plugins?
    * If the proposal is a data structure conveyed from plugins to plugins, keep it simple, and limit its extensibility.
    * One idea is to implement the data structure as a `final` class only with simple features, which is opposite to the above.

Copyright / License
====================

This document is placed under the [CC0-1.0-Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.en) license, whichever is more permissive.
