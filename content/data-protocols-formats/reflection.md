---
title: "Reflection format"
linkTitle: "Reflection format"
weight: 10
---

## Overview

The "reflection" representation is used to save certain Vertica data items, such as data collector records or catalog objects, in an ASCII/UTF-8 text format. The data model is hierarchical, similar to JSON.

## Format

### Basic Object Structure

The first line of any represented object consists of a colon (`:`) followed by the name of the object type.

The object body consists of self-describing name value pairs, one per line. Simple properties are of the form `name:value`. There is no quoting applied to values.

All objects end with a period(`.`) on a line by itself. For example:

     :ExampleClass
     field1:value1
     field2:value2
     .

### Compound Object Structure

If a property has a compound/nested value, it is introduced by its property name (`name:`), and its contents are placed in subsequent lines. Its fields are, in turn, represented as name value pairs, until a line containing a period(`.`) is reached. The compound object may be indented.

    :MainClass
    subclassField:
     :B
     a:
     b:Here is b member of subclass
     .
    nullableField1:
     :ClassNameB
     a:value for a
     b:value for b \n has a newline
     .
    nullableField2:null
    .

### Containers / Lists

All lists / maps / other containers are represented as element lists. These are encoded as an open curly brace on its own line (`{`), followed by zero or more element objects, followed by a close curly brace (`}`) on its own line.

    testMap:
    {
     :string_pair
     first:1
     second:first element
     .
     :string_pair
     first:2
     second:second element
     .
     :string_pair
     first:3
     second:third element
     .
    }

### Escaping Rules

Because newlines are used to indicate the end of a property value, they are escaped with the sequence `\n`. Zero bytes must be escaped as `\0`. Because of these escape sequences, backslash itself must be escaped as `\\`.

### Field Formats

Most field formats, such as those for integers and strings, are self explanatory. Timestamps are saved in microseconds from Jan 1, 2000, 00:00 UTC.

## Forward / Backward Compatibility

While not part of the format per se, the name+value pair structure simplifies compatibility concerns. Unrecognized / obsolete property names may be ignored (perhaps with a warning), and missing properties can be filled in with default values.
