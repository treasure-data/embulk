---
EEP: unnumbered
Title: JSON Column Type
Author: dmikurube
Status: Draft
Type: Standards Track
Created: 2023-01-17
---

Introduction
=============

This EEP describes the history and the design decisions of Embulk's "JSON" column type. It looks back the history for what the "JSON" column type was introduced, then discusses the long-standing confusion by the past decision on the "JSON" column type.

This EEP discusses how the confusion could be resolved, and proposes the long-term migration process to realize the resolution.

Motivation
===========

Beginning of the "JSON" column type
------------------------------------

The "JSON" column type has been introduced in Embulk since v0.8.0 with the pull request: https://github.com/embulk/embulk/pull/353

The motivation os the "JSON" column type was to avoid losing information from data sources that provide schema-less data, and to cover flexible conversions from schema-less data records into schemaful data records by filter plugins. See the related GitHub Issue at: https://github.com/embulk/embulk/issues/306

At the same time, the original authors decided to represent "JSON" values as `org.msgpack.value.Value` from [`msgpack-java`](https://github.com/msgpack/msgpack-java).

Representation of "JSON" values
--------------------------------

The decision to use `msgpack-java` has brought a long-standing confusion in Embulk, unfortunately. "JSON" values have been represented as `org.msgpack.value.Value` since Embulk v0.8.0 while Embulk has never used any packer nor unpacker of MessagePack. `msgpack-java` has been there only for `org.msgpack.value.Value` just to represent JSON-like data structures.

On the other hand, the value classes and the packer/unpacker classes are unsplittable in `msgpack-java`. Once the Embulk core has `msgpack-java` in it, its packer and unpacker are involed together in the Embulk core while they are not used.

The "confusion" is revealed when a plugin needs the `msgpack-java` packer or unpacker, for example, [`embulk-parser-msgpack`](https://github.com/embulk/embulk-parser-msgpack). Such a plugin depends on `msgpack-java`, which has been included in the Embulk core. Then, the plugin would not be able to use a later version of `msgpack-java` only by itself. Oppositely, the Embulk core cannot upgrade the `msgpack-java` casually because a later version of `msgpack-java` may have some possible incompatibility. It's the dependency hell deadlock.

Having said that, just dropping `msgpack-java` is uneasy because `org.msgpack.value.Value` has been used in method signatures of the Embulk plugin SPI, for example, `void PageBuilder#setJson(Column, org.msgpack.value.Value)` and `org.msgpack.value.Value PageReader#getJson(Column)`.

The purpose of this EEP is to show the long-term migration process to get `msgpack-java` dropped from the Embulk plugin SPI.

Alternative of `msgpack-java`
==============================

New "JSON" value classes
-------------------------

To drop `msgpack-java` from the Embulk plugin SPI, Embulk at least needs an alternative set of classes to represent "JSON" values. This EEP proposes to add a brand-new set of classes.

The other possible approach could be utilizing some existing libraries to represent JSON values, such as [Jackson](https://github.com/FasterXML/jackson), [Gson](https://github.com/google/gson), or the [GlassFish JSON Processing](https://glassfish.org/) as-is. But, any of them would bring the same dependency hell. That's no choice.

Basic design of the new classes
--------------------------------

First of all, the new "JSON" value classes would need to have almost the same features with the value classes of `msgpack-core`. Otherwise, plugins would have a hard time to migrate. They would extend the base interface `JsonValue`, and consist of classes that implement `JsonValue`.

* `JsonNull` for JSON `null`, corresponding to msgpack-java [`ImmutableNilValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableNilValueImpl.html)
* `JsonBoolean` for JSON `true` and JSON `false`, corresponding to msgpack-java [`ImmutableBooleanValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableBooleanValueImpl.html)
* `JsonLong` for JSON integral numbers, corresponding to msgpack-java [`ImmutableLongValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableLongValueImpl.html)
    * The integral numbers are represented by Java `long` that is also used for Embulk's `LONG` columns.
* `JsonDouble` for JSON decimal numbers, corresponding to msgpack-java [`ImmutableDoubleValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableDoubleValueImpl.html)
    * The decimal numbers are represented by Java `double` that is also used for Embulk's `DOUBLE` columns.
* `JsonString` for JSON strings, corresponding to msgpack-java [`ImmutableStringValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableStringValueImpl.html)
* `JsonArray` for JSON arrays, corresponding to msgpack-java [`ImmutableArrayValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableArrayValueImpl.html)
* `JsonObject` for JSON objects, corresponding to msgpack-java [`ImmutableMapValueImpl`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/impl/ImmutableMapValueImpl.html)

`JsonLong` and `JsonDouble` would have a common interface `JsonNumber`, corresponding to msgpack-java [`ImmutableNumberValue`](https://www.javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/ImmutableNumberValue.html).

Second, method signatures of the [Java(TM) EE JSON Processing API `javax.json` classes](https://javaee.github.io/jsonp/), or the [Jakarta EE JSON Processing API `jakarta.json` classes](https://jakarta.ee/specifications/jsonp/2.0/), would be a good reference. They have been the semi standard (not the de-facto standard though). They provide just interfaces. It would be reasonable to follow their method signatures, including their naming convention. Note that the "JSON" value classes would not "implement" their interfaces now because [Java(TM) EE has been transitioned to Jakarta EE, their package names have changed, and the situation does not seem very stbale](https://blogs.oracle.com/javamagazine/post/transition-from-java-ee-to-jakarta-ee).

Last, the new "JSON" value classes would not expect nor allow arbitrary extension by plugin developers. In other words, even if a plugin developer defines a new class such as `MyJsonBigDecimal` that implements `JsonValue`, it would not work as the developer expects.

`BigDecimal` and `BigInteger`
------------------------------

One big consideration is JSON numbers. The JSON specification does not limit the range and precision of numbers, but at the same time, it also "allows" implementations to set the limits. See [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259#section-6).

Also from Embulk's viewpoint, it has had the `LONG` and `DOUBLE` coulmn types since its beginning, but no column type to represent unlimited numbers. Unlimited numbers could cause another problem about interoperability with the existing `LONG` and `DOUBLE` column types, for example, how [`embulk-filter-flatten_json`](https://github.com/civitaspo/embulk-filter-flatten_json) would flatten a big JSON number into a `LONG` column.

We'd say it's somehow reasonable to limit its number representations to Java `long` and `double`.

That being said, however, users sometimes want such a thing in the real world. Embulk has received [such a request](https://github.com/embulk/embulk/issues/775) in the past, indeed.

We are going to leave some room for the future to extend the "JSON" value classes for `BigDecimal` and `BigInteger`. New classes, that would be like `JsonBigDecimal` or `JsonBigInteger`, can be defined by implementing `JsonNumber` in the Embulk SPI, and by re-defining their equality also with existing number classes, `JsonLong` and `JsonDouble`.

Migration
==========

This change eventually brings some SPI incompatibility. The migration would go lazily in the long-term with a sufficient migration period. The migration steps are similar to the `TIMESTAMP` column type migrated to `java.time.Instant` from `org.embulk.spi.time.Timestamp`.

In Embulk v0.10.42
-------------------

1. Upgrades `msgpack-core` to v0.8.24 from v0.8.11.
    * This v0.8.24 is the latest version of `msgpack-core` v0.8.
    * Upgrading from v0.8.11 to v0.8.24 is considered compatible.
2. Adds the new "JSON" value classes in `org.embulk.spi.json`.
3. Modifies the new "JSON" value classes to represent their inner values with `msgpack-java` classes.
    * These classes should keep behaving the same.
4. Adds new Embulk plugin SPI methods with the new "JSON" value classes in parallel to the existing methods.
    * Existing:
        * `void PageBuilder#setJson(int, org.msgpack.value.Value)`
        * `void PageBuilder#setJson(Column, org.msgpack.value.Value)`
        * `org.msgpack.value.Value PageReader#getJson(int)`
        * `org.msgpack.value.Value PageReader#getJson(Column)`
    * New:
        * `void PageBuilder#setJsonValue(int, org.embulk.spi.json.JsonValue)`
        * `void PageBuilder#setJsonValue(Column, org.embulk.spi.json.JsonValue)`
        * `org.embulk.spi.json.JsonValue PageReader#getJsonValue(int)`
        * `org.embulk.spi.json.JsonValue PageReader#getJsonValue(Column)`
    * The new methods are forwarded to the existing methods with `msgpack-java` classes.
5. Replaces Embulk's internal representation of "JSON" values to the new "JSON" value classes.
6. Deprecates the existing older SPI methods with `msgpack-java` classes.

During Embulk v0.11
--------------------

Embulk plugins are expected to start using the new "JSON" value classes and the new SPI methods to handle the "JSON" column type.

At some point after Embulk v1.0.0
----------------------------------

Finally, this would be the "incompatible" change.

* The older SPI methods with `msgpack-core` classes are removed.
* The new "JSON" values are back to be represented without `msgpack-core` classes.
* `msgpack-java` is removed from the Embulk core.

Plugin SPI Changes
===================

`JsonValue` classes
--------------------

It adds the following `org.embulk.spi.json.JsonValue` classes and interfaces.

### interface `JsonValue`

```java
package org.embulk.spi.json;

public interface JsonValue {
    public static enum EntityType {
        NULL,
        BOOLEAN,
        LONG,
        DOUBLE,
        STRING,
        ARRAY,
        OBJECT;

        public boolean isNull();
        public boolean isBoolean();
        public boolean isLong();
        public boolean isDouble();
        public boolean isString();
        public boolean isArray();
        public boolean isObject();
    }

    EntityType getEntityType();
    boolean isJsonNull();
    boolean isJsonBoolean();
    boolean isJsonLong();
    boolean isJsonDouble();
    boolean isJsonString();
    boolean isJsonArray();
    boolean isJsonObject();

    JsonNull asJsonNull();
    JsonBoolean asJsonBoolean();
    JsonLong asJsonLong();
    JsonDouble asJsonDouble();
    JsonString asJsonString();
    JsonArray asJsonArray();
    JsonObject asJsonObject();

    boolean equals(Object other);
    String toJson();
}
```

### interface `JsonNumber`

```java
package org.embulk.spi.json;

import java.math.BigDecimal;
import java.math.BigInteger;

public interface JsonNumber extends JsonValue {
    boolean isIntegral();

    boolean isByteValue();
    boolean isShortValue();
    boolean isIntValue();
    boolean isLongValue();

    byte byteValue();
    byte byteValueExact();
    short shortValue();
    short shortValueExact();
    int intValue();
    int intValueExact();
    long longValue();
    long longValueExact();
    BigInteger bigIntegerValue();
    BigInteger bigIntegerValueExact();

    float floatValue();
    double doubleValue();
    BigDecimal bigDecimalValue();
}
```

### final class `JsonNull`

```java
package org.embulk.spi.json;

public final class JsonNull implements JsonValue {
    public static JsonNull of();  // return the singleton instance of JsonNull

    @Override public String toJson();  // just return "null"
    @Override public String toString();  // just return "null"
    @Override public boolean equals(final Object otherObject);
    @Override public int hashCode()
}
```

### final class `JsonBoolean`

```java
```

### final class `JsonLong`

```java
```

### final class `JsonDouble`

```java
```

### final class `JsonString`

```java
```

### final class `JsonArray`

```java
```

### final class `JsonObject`

```java
```


Backwards Compatibility
========================


Copyright / License
====================

This document is placed under the [CC0-1.0-Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.en) license, whichever is more permissive.
