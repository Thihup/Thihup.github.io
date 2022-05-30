---
title: "Create Your Own Incubating Java Modules"
date: 2022-05-30T13:14:44Z
draft: true
---

Java Modules have some secret features, that are mostly used by the JDK itself.
One of these features is the incubating modules.

In the [JEP 11: Incubator Modules](https://openjdk.java.net/jeps/11) it is described how
it should work

If we use the `jdk.incubator.concurrent` module as an example - 
from [JEP 428: Structured Concurrency (Incubator)](https://openjdk.java.net/jeps/428) -
we can observe that:
- This module is listed in the `java --list-modules`
- It is not resolved by default
- It warns in the console of the use of the incubating module

One way of resolving an incubating module is by adding a `requires some.incubator.module` in a `module-info.java`
or by including the `--add-modules=some.incubator.module` to the JVM command-line flag.

However, [JEP 11](https://openjdk.java.net/jeps/11) only defines how it should work with JMod files.
While JMods are great for use with JLink, it is not the current deploy format for 
Maven Central. 

Looking through the JDK source-code, though, I have found out that not only `JMod` tool
supports the creation of incubating modules, but also the `Jar` tool.

> **WARNING** These command-line flags and the class file attribute are not in the specification.
> They may be changed or removed without any notice!

There are two main flags:
- [--do-not-resolve-by-default](https://github.com/openjdk/jdk/blob/master/src/jdk.jartool/share/classes/sun/tools/jar/GNUStyleOptions.java#L170)
- [--warn-if-resolved=value](https://github.com/openjdk/jdk/blob/master/src/jdk.jartool/share/classes/sun/tools/jar/GNUStyleOptions.java#L177)
(where `value` can be one of):
    - [incubating](https://github.com/openjdk/jdk/blob/master/src/jdk.jartool/share/classes/sun/tools/jar/GNUStyleOptions.java#L187)
    - [deprecated-for-removal](https://github.com/openjdk/jdk/blob/master/src/jdk.jartool/share/classes/sun/tools/jar/GNUStyleOptions.java#L185)
    - [deprecated](https://github.com/openjdk/jdk/blob/master/src/jdk.jartool/share/classes/sun/tools/jar/GNUStyleOptions.java#L183)


Using the `Javap` tool, it is possible to see that the `module-info.class` is
changed to include a (unspecified) attribute: `ModuleResolution` that has a few valid values:

```java
ModuleResolution:
  1     //  DO_NOT_RESOLVE_BY_DEFAULT
ModuleResolution:
  2     //  WARN_DEPRECATED
ModuleResolution:
  4     //  WARN_DEPRECATED_FOR_REMOVAL
ModuleResolution:
  8     //  WARN_INCUBATING
```

Only the first option (`DO_NOT_RESOLVE_BY_DEFAULT`) can be combined with any of ther other options.

> *Note* Due a bug, it is needed to specify the
> `--warn-if-resolved` flag before `--do-not-resolve-by-default`

In Maven, you can achive this by updating the jar after it has been already created,
like this:

```xml
<plugins>
    <plugin>
        <groupId>io.github.wiverson</groupId>
        <artifactId>jtoolprovider-plugin</artifactId>
        <version>1.0.34</version>
        <executions>
            <execution>
                <phase>package</phase>
                <goals>
                    <goal>java-tool</goal>
                </goals>
                <configuration>
                    <toolName>jar</toolName>
                    <writeOutputToLog>true</writeOutputToLog>
                    <writeErrorsToLog>true</writeErrorsToLog>
                    <failOnError>true</failOnError>
                    <args>
                        <arg>--update</arg>
                        <arg>--warn-if-resolved=incubating</arg>                        
                        <arg>--do-not-resolve-by-default</arg>
                        <arg>--file</arg>
                        <arg>${project.build.directory}/${project.build.finalName}.jar</arg>
                        <!-- It seems to be needed to include twice -->
                        <arg>${project.build.directory}/${project.build.finalName}.jar</arg>
                    </args>
                </configuration>
            </execution>
        </executions>
    </plugin>
</plugins>
```

You can checkout a full example here: https://github.com/Thihup/incubator-module-example