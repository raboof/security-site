---
title: "Reliably avoid dangerous MySQL JDBC parameters"
author: Arnout Engelen
date: 2024-01-22
description: The MySQL Connector/J JDBC driver has parameters that should only be exposed to trusted users. This post describes how to override those reliably
---

There are a number of Apache projects that allow users of their web-based
interface to define database connections through
[JDBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity) URLs.
Such URLs specify which JDBC driver to use, which database to connect to, and
additional parameters.

The [MySQL Connector/J](https://github.com/mysql/mysql-connector-j) driver
implements this API for MySQL databases. This driver assumes the JDBC URL it
receives can be fully trusted. Notably:

* The `autoDeserialize` option enabled automatically using Java deserialization to construct arbitrary objects. When used with untrusted input, this can [lead to arbitrary code execution](https://docs.oracle.com/javase/8/docs/technotes/guides/serialization/filters/serialization-filtering.html). This option has been [removed in version 8.2.0 of the driver](https://dev.mysql.com/doc/relnotes/connector-j/en/news-8-2-0.html).
* The `allowLoadLocalInfile` parameter lets the database server request an arbitrary file from the client machine.
* The `allowLoadLocalInfileInPath` parameter lets the database server request arbitrary files from the specified directory on the client machine.

In some cases, users are trusted to make database connections but
not to read arbitrary files from the server filesystem, let alone execute arbitrary code.
In such cases we should restrict the available options.

Validating or sanitizing the URL parameters, however, is challenging to do reliably:
the driver is fairly flexible in how these parameters can be passed in, which means you
would have to account for different case, URL-encodings, whitespace and more.

Luckily, there is a better way: instead of using `DriverManager.getConnection(url)`
or `DriverManager.getConnection(url, user, password)`, you can use [DriverManager.getConnection(url, info)](https://docs.oracle.com/en/java/javase/11/docs/api/java.sql/java/sql/DriverManager.html#getConnection(java.lang.String,java.util.Properties)):

    Properties info = new Properties();
    info.put("user", user);
    info.put("password", password);
    info.put("autoDeserialize", "false");
    info.put("allowLoadLocalInfile", "false");
    info.put("allowLoadLocalInfileInPath", "");
    info.put("useConfigs", "");
    return DriverManager.getConnection(url, info);

When you use this approach, the parameters you provide in the `Properties` object will
always override the parameters found in the URL, regardless of normalization.
This reliably avoids giving additional privileges to users who are trusted to define database
connections.
