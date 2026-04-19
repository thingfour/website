---
type: article
title: Alternative Thing DSL Parser
---
{{< intro >}}
One of main advantages of openHAB is its flexible provider mechanism which allow to use of various sources of configurations.
The very core concept of a Thing is covered by  `ThingProvider`, which have two primary implementations.
{{< /intro >}}

This article is intended to dive a bit deeper into how openHAB parses its `*.things` files and how they get processed internally.
Finally, we will bring a completely new implementation of such parser based on ANTLRv4 library.

## Introduction
The core of openHAB embeds several important libraries.
One of them is Xtext which provides very flexible Domain Specific Language (DSL) toolkit.
This toolkit is used to parse not only `*.things` files.
It is used also for `*.items` and `*.rules` (although with a bit deeper use of Xtext).

Once things files are parsed, they form intermediate model which is based on Eclipse Modeling Framework (EMF).
The EMF provides some generic methods to work with... well various sorts of models.
Then Xtend is used once again, this time to generate `GenericThingProvider` class.
This class transforms EMF model into openHAB core model - free of EMF dependencies.

Does it sound complex?
A bit.
Can we make it simpler?
Of course.

{{< problem-statement >}}
There are several troubles inherited through use of Xtext and EMF:

* The Xtext is based on fairly old ANTLR 3 version.
* The grammar syntax is intermediate - it is used to generate target ANTLR schema.
* There is a ANTLR lab which allows to test grammar online, by browser, but only for ANTLR 4.
* It depends on Eclipse Modeling Framework, which has its own weight.
* Generation of parser from Xtext requires use of fairly complex toolchain.
* There is very few people who can maintain Xtext / EMF stuff outside of Eclipse Foundation.

{{< /problem-statement >}}

## Making friendship with ANTLR v4
The [ANTLR](https://en.wikipedia.org/wiki/ANTLR) is a shortcut for **AN**other **T**ool for **L**anguage **R**ecognition.
ANTLR itself is a parser generator first developed by Terance Parr.
ANTLR consist also a runtime library which is needed by generated parser.
The core difference between Xtend and ANLTR is its role.
While Xtend is toolkit, ANTLR is just a tool.
Using second one requires more effort on our end, leaving us with some hard troubles to be solved.

In the actual `Things.g4` grammar, parsing starts from the `things` rule which accepts a sequence of `modelThing` and `modelBridge` elements until `EOF`.
Top-level and nested declarations are split on purpose: `modelThing` and `modelBridge` handle full UIDs, optional labels and optional parents, while `nestedThing` and `nestedBridge` handle shorter forms used inside bridge blocks.
Structure is then delegated to `thingDefinition` and `bridgeDefinition`, which cover optional location markers, bracketed property lists and brace-delimited bodies.
Inside these bodies, `modelChannels` and `modelChannel` describe both declared channels and typed channel references.
Property parsing is kept flat through `modelProperties`, `modelProperty` and `valueType`, which is enough for key-value configuration entries such as those used in example `*.things` files`.
The lexer side is equally direct: `STRING` strips quotes, `INTEGER` and `DECIMAL` are separated for clearer numeric handling, comments are pushed to the hidden channel, and `modelItemType` accepts not only standard item types such as `Switch` or `Number`, but also `Number:<dimension>` and plain `ID` values for dynamic item types resolved at runtime.

However, by looking at standard thing definition used by openHAB DSL, we can see it as rather basic.
Compared to rule DSL, its actually trivial, as it do no pose to be a fully fledged programming language.

The Thing DSL allows us to define:

1. Things
2. Bridges
3. Thing and Bridge configuration (key value paris within square brackets)
4. Channels within Things/Bridges
5. Channel configuration (again key value paris within square brackets)
6. Things within bridges

The only one hard part, at first look is nesting of Things within Bridges or Bridges within Bridges.
The ANTLR grammar syntax itself is rather simple, difficulty is really distinguishing the lexer and parser rules.
Both are evaluated at different stage of parsing process and have serious impact to how parser will behave.
Sometimes, improperly defined lexer rule, may not ever match.

## Practical use
The parser is used as a small component, not as a standalone demo.
The `thing4` stack exposes `Parser` and `Writer` interfaces for DSL resources.
This makes the parser usable in tooling and in runtime-oriented code.

The main modules are:

* `org.thing4.core.parser` for common `Parser` and `Writer` contracts.
* `org.thing4.core.model.facade` for access to official openHAB parsers through the same API.
* `org.thing4.core.parser.thing` for the lightweight parser with identifier `thing4`.
* `org.thing4.core.parser.thing.yaml` for YAML parsing and writing with identifier `yaml`.

This split keeps syntax handling separate from file watching, providers and output generation.

## Typical build tasks
The same parser can be used during a Maven build.
The `thing4-maven-plugin` provides three relevant goals:

* `parse-descriptors` reads `OH-INF/**/*.xml` descriptor files.
* `process-descriptors` renders descriptor data through templates.
* `parse-write` parses source files and can optionally write them in another format.

This is enough for several practical tasks:

* verify `docs/examples/**/*.things` files during build,
* convert `*.things` definitions to YAML,
* generate AsciiDoc from binding descriptors,
* check the same input with different parser implementations.

## Example: validate thing files
The simplest case is syntax validation of example files:

```xml
<plugin>
  <groupId>org.thing4.tools</groupId>
  <artifactId>thing4-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>parse-write</goal>
      </goals>
      <configuration>
        <includes>
          <include>docs/examples/**/*.things</include>
        </includes>
        <parser>thing4</parser>
        <type>org.openhab.core.thing.Thing</type>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Here the parser reads thing definitions and checks whether they can be mapped to `org.openhab.core.thing.Thing`.
No output writer is needed.

## Example: convert thing files to YAML
The same goal can write parsed definitions to another format:

```xml
<plugin>
  <groupId>org.thing4.tools</groupId>
  <artifactId>thing4-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>parse-write</goal>
      </goals>
      <configuration>
        <includes>
          <include>docs/examples/**/*.things</include>
        </includes>
        <parser>openhab</parser>
        <writer>yaml</writer>
        <type>org.openhab.core.thing.Thing</type>
      </configuration>
    </execution>
  </executions>
</plugin>
```

This configuration reads `*.things` files and emits matching YAML files.
The writer identifier becomes the output extension.

## Example: descriptor processing
The parser tooling is also useful next to binding descriptors:

```xml
<plugin>
  <groupId>org.thing4.tools</groupId>
  <artifactId>thing4-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>parse-descriptors</goal>
        <goal>process-descriptors</goal>
      </goals>
      <configuration>
        <outputExtension>adoc</outputExtension>
      </configuration>
    </execution>
  </executions>
</plugin>
```

This setup reads `OH-INF/thing/*.xml` and `OH-INF/config/*.xml`.
It can then render generated reference files such as bridge types, thing types, channel types and configuration descriptions.

## Example files
The examples below show the kinds of files handled by this tooling.
They include a `*.things` source, a nested bridge example, YAML output and descriptor XML.

{{< source-browser title="Example sources" glob="examples/**" >}}

## Why this matters
The merit of the ANTLR-based parser is simple.
It removes the Xtext and EMF dependency chain from one narrow task.
At the same time it stays useful in regular engineering work:

* build-time validation,
* format conversion,
* descriptor-driven documentation,
* and reuse in provider-oriented code.

That makes the parser easier to test, easier to embed and easier to maintain.
