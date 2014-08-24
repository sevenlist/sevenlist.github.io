---
layout: post
title: "Spring @Transactional with AspectJ"
date: 2014-08-24 12:05:00 +0200
comments: true
categories: Spring @Transactional AspectJ
description: Spring @Transactional with AspectJ - For private methods use @EnableTransactionManagement(mode = AdviceMode.ASPECTJ) and aspectj-maven-plugin.   
keywords: spring @transactional with aspectj, no sources found skipping aspectj compile, aspectj-maven-plugin, @transactional, aspectj
---

If we want to use the `@Transactional` annotation on private methods, we cannot use Spring's default proxy mode -
it only works for public methods that are called from client code.

For self-invoked methods we need to use _AspectJ_.
<!--more-->
### An Example
This may be the case e.g. when we want to save incoming data and send a letter using that data within a transaction.
If saving the data fails we do not want to send the letter.
If sending the letter fails we do not want to loose the data that we received.
Hence, we want to save the data in a separate transaction, by annotating our saveData(..) method with
`@Transactional(propagation = Propagation.REQUIRES_NEW)`.

### Configuring Spring To Use AspectJ
On a `@Configuration` class we just need to set the AspectJ advice mode for the transaction management configuration:
``` groovy
@Configuration
@ComponentScan
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)
class ApplicationConfiguration {
    // ...
}
```

### Configuring Maven To Run The AspectJ Compiler
Next we need to use the AspectJ compiler, so let's add its needed dependencies:
``` xml
<dependencies>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjrt</artifactId>
        <version>1.7.4</version>
    </dependency>

    <!-- In this example we use JPA for persistence. -->
    <dependency>
        <groupId>org.hibernate.javax.persistence</groupId>
        <artifactId>hibernate-jpa-2.1-api</artifactId>
        <version>1.0.0.Final</version>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>4.0.6.RELEASE</version>
    </dependency>
</dependencies>
```

The AspectJ compiler itself can then be enabled with:
``` xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>aspectj-maven-plugin</artifactId>
            <version>1.6</version>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <!-- <goal>test-compile</goal> -->
                    </goals>
                </execution>
            </executions>
            <configuration>
                <complianceLevel>1.7</complianceLevel>
                <forceAjcCompile>true</forceAjcCompile>
                <weaveDirectories>
                    <weaveDirectory>${project.build.outputDirectory}</weaveDirectory>
                </weaveDirectories>
                <aspectLibraries>
                    <aspectLibrary>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring-aspects</artifactId>
                    </aspectLibrary>
                </aspectLibraries>
            </configuration>
        </plugin>
    </plugin>
</build>
```
If we do not use `<forceAjcCompile>` and do not set the `<weaveDirectory>` AspectJ will not weave the transaction code into
our generated Groovy bytecode. Instead, it will warn us with _No sources found skipping aspectJ compile_ when Maven runs.
That is as the aspectj-maven-plugin processes the src/main/java directory by default but not src/main/groovy.

### How The AspectJ Generated Code Looks Like
Just for completeness, let's look how AspectJ has wrapped the saveData(..) method with a transaction. Using the
[Procyon](https://bitbucket.org/mstrobel/procyon/wiki/Java%20Decompiler) decompiler we can see:
``` groovy
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveData(final SomeData data) {
    ((AbstractTransactionAspect)AnnotationTransactionAspect.aspectOf()).ajc$around$org_springframework_transaction_aspectj_AbstractTransactionAspect$1$2a73e96c((Object)this, (AroundClosure)new DataService$AjcClosure3(new Object[] { this, data }), DataService.ajc$tjp_1);
}
```
