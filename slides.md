---
# theme: apple-basic  
lineNumbers: true
drawings:
  persist: false
routerMode: history
selectable: true
remoteAssets: true
colorSchema: 'dark'
layout: cover
background: /cover.png
canvasWidth: 800
---

# Crafting the Ultimate Docker Image for Spring Applications

<img src="/bellsoft.png" width="200px" class="absolute right-10px bottom-10px"/>

---
layout: image-right
image: 'avatar.jpg'
---
# `whoami`

<v-clicks>

- <div v-after>Pasha Finkelshteyn</div>
- Dev <noto-v1-avocado /> at BellSoft
- ≈10 years in JVM. Mostly <logos-java /> and <logos-kotlin-icon />
- And <logos-spring-icon />
- <logos-twitter /> asm0di0
- <logos-mastodon-icon /> @asm0dey@fosstodon.org

</v-clicks>

---
layout: two-cols-header
---

# BellSoft

::left::

* Vendor of Liberica JDK
* Contributor to the OpenJDK
* Author of ARM32 support in JDK
* Own base images
* Own Linux: Alpaquita

Liberica is the JDK officially recommended by <logos-spring-icon />

<v-click><b>We know our stuff!</b></v-click>

::right::

<img src="/news.png" class="invert rounded self-center"/>


---

# So, what is ultimate?

- Smallest
- Fatsest startup
- Something inbetween

---
layout: statement
---

# So, let's look at all of them!

---

# How do we create an image?

Let's start from trivial.

```docker {none|1|3|4|all}
FROM bellsoft/liberica-runtime-container:jdk-25-musl

COPY . /app
RUN cd /app && ./gradlew build
ENTRYPOINT     java -jar /app/build/libs/spring-petclinic*.jar
```

---

# `Dockerfile` directives

1. Each directive creates a layer of the image.
2. Layers are *immutable*
3. Some layers are zero-sized <span v-click="3">(for example `RUN rm -rf /app`)</span>
<div v-click="1">

```docker {1|3|4}{at:2}
FROM bash

COPY . /app
RUN rm -rf /app
```

</div>
<v-click at="4">

4. We would like image to be light, but it's not :(

</v-click>  
---

# Our `Dockerfile`

```docker {1|3|4|5}
FROM bellsoft/liberica-runtime-container:jdk-25-musl

COPY . /app
RUN cd /app && ./gradlew build
ENTRYPOINT java -jar /app/build/libs/spring-petclinic*.jar
```

---

# Result

```text {all|3|4|5}{maxHeight:'300px'}
Cmp   Size  Command
     10 MB  FROM blobs
    109 MB  cd /app && ./gradlew build -xtest
     85 MB  (missing)
    591 MB  (missing)
```

<span v-click="1">109 MB of Java</span>

<span v-click="2">85 MB of the app</span>

<span v-click="3">591 MB of Java build caches</span>

---
layout: two-cols-header
---

# 600+ MB are changed on every build!

::left::

Why do we care? We care because:

1. `push` takes longer time to start <br/>(update is longer)
2. `pull` takes longer <br/> (update takes longer & scaling takes longer)

Also, more disk psace is inefficiently used

::right::

<img src="/clocks.png" class="max-h-330px rounded self-center">

---

# Optimizing. Round 1.

Let's build it outside of container

````md magic-move
```docker {3-5}
FROM bellsoft/liberica-runtime-container:jdk-25-musl

COPY . /app
RUN cd /app && ./gradlew build
ENTRYPOINT java -jar /app/build/libs/spring-petclinic*.jar
```
```docker {3-4|all}
FROM bellsoft/liberica-runtime-container:jdk-25-musl

COPY build/libs/spring-petclinic-3.3.0.jar /app/app.jar
CMD java -jar /app/app.jar
```
````

<span v-click="2">Just copying the prebuilt `jar` file</span>

---

# Results

````md magic-move
```plain {4-5}
Cmp   Size  Command
     10 MB  FROM blobs
    109 MB  cd /app && ./gradlew build -xtest
     85 MB  (missing)
    591 MB  (missing)
```
```plain{4}
Cmp   Size  Command
     10 MB  FROM blobs
    109 MB  (missing)
     69 MB  (missing)
```
````

No Gradle caches and build dir!

---
layout: statement
---

# Saved 600+ MB

## But the build is not clean now :(

<v-click>We have to build everything outside, what if environment affects the build?</v-click>

---
layout: cover
background: cover2.png
---

# Enter build stages

---

# Multi-stage builds

Holy grail of pure builds

* Allow clean builds
* Allow optimal packaging
* Allow different base images

---

# Staged example

```docker {none|1|3|4|6|8}
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-25-musl as runner

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
```

---

# Result

<div></div>

+ jar - same size

+ JRE - 48.28

---
layout: statement
---

# That's all folks!

<h2 v-click>Or is it?</h2>

---

# What's these 60 MiB?

We have to pull 60 MiB of petclinic on every small commit!

Is there a way to optimize it?

---

# Layers!

```docker {1-4|6|8,9|10|12,6|15-18|14}{maxHeight:'180px'}
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-slim-musl as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/liberica-runtime-container:jre-slim-musl as runner

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
```

- Build image

<v-click at="1">

- Introduce new "optimizer" stage

</v-click>
<v-click at="3">

- Extract the jar to layered structure

</v-click>
<v-click at="5">

- Copy layers

</v-click>

---

# Layers

Because Spring Boot jar is complex!

```plain      
Usage:
  java -Djarmode=tools -jar my-app.jar

Available commands:
  extract      Extract the contents from the jar
  list-layers  List layers from the jar that can be extracted
```

---

# Layers

```plain {1|104|103|4-102|3,2}{maxHeight:'300px'}
extracted
├── application
│   └── spring-petclinic-4.0.0-SNAPSHOT.jar
├── dependencies
│   └── lib
│       ├── angus-activation-2.0.2.jar
│       ├── antlr4-runtime-4.13.2.jar
│       ├── aspectjweaver-1.9.24.jar
│       ├── attoparser-2.0.7.RELEASE.jar
│       ├── bootstrap-5.3.8.jar
│       ├── byte-buddy-1.17.7.jar
│       ├── cache-api-1.1.1.jar
│       ├── caffeine-3.2.2.jar
│       ├── checker-qual-3.49.3.jar
│       ├── classmate-1.7.0.jar
│       ├── commons-logging-1.3.5.jar
│       ├── context-propagation-1.2.0-M1.jar
│       ├── error_prone_annotations-2.40.0.jar
│       ├── font-awesome-4.7.0.jar
│       ├── h2-2.3.232.jar
│       ├── HdrHistogram-2.2.2.jar
│       ├── hibernate-core-7.1.1.Final.jar
│       ├── hibernate-models-1.0.1.jar
│       ├── hibernate-validator-9.0.1.Final.jar
│       ├── HikariCP-7.0.2.jar
│       ├── istack-commons-runtime-4.1.2.jar
│       ├── jackson-annotations-2.20.jar
│       ├── jackson-core-3.0.0-rc9.jar
│       ├── jackson-databind-3.0.0-rc9.jar
│       ├── jakarta.activation-api-2.1.4.jar
│       ├── jakarta.annotation-api-3.0.0.jar
│       ├── jakarta.inject-api-2.0.1.jar
│       ├── jakarta.persistence-api-3.2.0.jar
│       ├── jakarta.transaction-api-2.0.1.jar
│       ├── jakarta.validation-api-3.1.1.jar
│       ├── jakarta.xml.bind-api-4.0.2.jar
│       ├── jaxb-core-4.0.5.jar
│       ├── jaxb-runtime-4.0.5.jar
│       ├── jboss-logging-3.6.1.Final.jar
│       ├── jspecify-1.0.0.jar
│       ├── jul-to-slf4j-2.0.17.jar
│       ├── LatencyUtils-2.0.3.jar
│       ├── log4j-api-2.24.3.jar
│       ├── log4j-to-slf4j-2.24.3.jar
│       ├── logback-classic-1.5.18.jar
│       ├── logback-core-1.5.18.jar
│       ├── micrometer-commons-1.16.0-M3.jar
│       ├── micrometer-core-1.16.0-M3.jar
│       ├── micrometer-jakarta9-1.16.0-M3.jar
│       ├── micrometer-observation-1.16.0-M3.jar
│       ├── micrometer-tracing-1.6.0-M3.jar
│       ├── mysql-connector-j-9.4.0.jar
│       ├── postgresql-42.7.7.jar
│       ├── slf4j-api-2.0.17.jar
│       ├── snakeyaml-2.5.jar
│       ├── spring-aop-7.0.0-M9.jar
│       ├── spring-aspects-7.0.0-M9.jar
│       ├── spring-beans-7.0.0-M9.jar
│       ├── spring-boot-4.0.0-M3.jar
│       ├── spring-boot-actuator-4.0.0-M3.jar
│       ├── spring-boot-actuator-autoconfigure-4.0.0-M3.jar
│       ├── spring-boot-autoconfigure-4.0.0-M3.jar
│       ├── spring-boot-cache-4.0.0-M3.jar
│       ├── spring-boot-data-commons-4.0.0-M3.jar
│       ├── spring-boot-data-jpa-4.0.0-M3.jar
│       ├── spring-boot-health-4.0.0-M3.jar
│       ├── spring-boot-hibernate-4.0.0-M3.jar
│       ├── spring-boot-http-converter-4.0.0-M3.jar
│       ├── spring-boot-jackson-4.0.0-M3.jar
│       ├── spring-boot-jdbc-4.0.0-M3.jar
│       ├── spring-boot-jpa-4.0.0-M3.jar
│       ├── spring-boot-micrometer-metrics-4.0.0-M3.jar
│       ├── spring-boot-micrometer-observation-4.0.0-M3.jar
│       ├── spring-boot-micrometer-tracing-4.0.0-M3.jar
│       ├── spring-boot-persistence-4.0.0-M3.jar
│       ├── spring-boot-servlet-4.0.0-M3.jar
│       ├── spring-boot-sql-4.0.0-M3.jar
│       ├── spring-boot-thymeleaf-4.0.0-M3.jar
│       ├── spring-boot-tomcat-4.0.0-M3.jar
│       ├── spring-boot-tx-4.0.0-M3.jar
│       ├── spring-boot-validation-4.0.0-M3.jar
│       ├── spring-boot-webmvc-4.0.0-M3.jar
│       ├── spring-boot-web-server-4.0.0-M3.jar
│       ├── spring-context-7.0.0-M9.jar
│       ├── spring-context-support-7.0.0-M9.jar
│       ├── spring-core-7.0.0-M9.jar
│       ├── spring-data-commons-4.0.0-M6.jar
│       ├── spring-data-jpa-4.0.0-M6.jar
│       ├── spring-expression-7.0.0-M9.jar
│       ├── spring-jdbc-7.0.0-M9.jar
│       ├── spring-orm-7.0.0-M9.jar
│       ├── spring-tx-7.0.0-M9.jar
│       ├── spring-web-7.0.0-M9.jar
│       ├── spring-webmvc-7.0.0-M9.jar
│       ├── thymeleaf-3.1.3.RELEASE.jar
│       ├── thymeleaf-spring6-3.1.3.RELEASE.jar
│       ├── tomcat-embed-core-11.0.11.jar
│       ├── tomcat-embed-el-11.0.11.jar
│       ├── tomcat-embed-websocket-11.0.11.jar
│       ├── txw2-4.0.5.jar
│       ├── unbescape-1.1.6.RELEASE.jar
│       └── webjars-locator-lite-1.1.1.jar
├── snapshot-dependencies
└── spring-boot-loader

6 directories, 98 files
```

---

# Layered image structure

```bash {all|2|5}{maxHeight:'180px'}
> du -h --max-depth=1 extracted
66M     extracted/dependencies
0       extracted/spring-boot-loader
0       extracted/snapshot-dependencies
892K    extracted/application
66M     extracted
```

<v-click at="1">

- Dependencies: ~66MiB

</v-click>

<v-click at="2">

- Application: ~892K

</v-click>

---
layout: statement
image: /cap.png
---

# 66.0MiB > 60Mib!

## 😱

---

# What are we optimizing?

Pull size!

Pull size here is usually only around 1MiB!

---
layout: statement
---

# We just reinvented how the BellSoft's buildpack works!

And it is amazing

---

# Diversion: buildpacks

https://paketo.io/

Trivial usage:

```bash {1|2|3|4}
/usr/sbin/pack build petclinic \
  --builder bellsoft/buildpacks.builder:musl \
  --path . \
  -e BP_JVM_VERSION=21
```

---

# Result

``` {7|5|4|3}
ID         TAG              SIZE      COMMAND                                                                               │
2774e3f214 petclinic:latest 0B        Buildpacks Process Types                                                              │
<missing>                   1.44kiB   Buildpacks Launcher Config                                                            │
<missing>                   2.44MiB   Buildpacks Application Launcher                                                       │
<missing>                   59.17MiB  Application Layer                                                                     │
<missing>                   729.21kiB Software Bill-of-Materials                                                            │
<missing>                   97.87MiB  Layer: 'jre', Created by buildpack: bellsoft/buildpacks/liberica@1.1.0                │
<missing>                   200.00B   Layer: 'java-security-properties', Created by buildpack: bellsoft/buildpacks/liberica@│
<missing>                   4.27MiB   Layer: 'helper', Created by buildpack: bellsoft/buildpacks/liberica@1.1.0             │
<missing>                   2.97kiB                                                                                         │
<missing>                   7.48MiB                                                                                         │
```

---

# Buildpacks

Why do we need them?

> * Buildpacks transform your application source code into container images
> * The Paketo open source project provides production-ready buildpacks for the most popular languages and frameworks
> * Use Paketo Buildpacks to easily build your apps and keep them updated 

- Reminds of s2i images
- Build better than default
- Spring-aware

---
layout: statement
---

# Is this all?

We've optimized pull/push size, right?

---

# Optimization is multidimensional

When startup time is more important…

There are several solutions for the Spring application startup time

1. Class Data Sharing (CDS)
1. Project Leyden
2. Coordinated Restore at Checkpoint
3. Native Image

---

# Class Data Sharing (CDS)

* **When**: Java 5 (!)
* **Why**: reduce the startup and memory footprint of multiple JVM instances running on the same host
* **How**: stores data in an archive

How does it help us?

<v-click>It does not! <twemoji-troll /></v-click>
<v-click>But</v-click>

---

# Application Class Data Sharing  

* **When**: Java 10
* **Why**: add application classes to the archive

And then:

* **When**: Java 13, [JEP 350](https://openjdk.org/jeps/350)
* **Why**: allow addition of classes into the archive upon app exit!

And this helps! How?

---

# How to use AppCDS with Spring?

* `-XX:ArchiveClassesAtExit=application.jsa` to create an archive
* `-Dspring.context.exit=onRefresh` to start and immediately exit the application

NB:
1. Use the same JDK
2. Use the same arguments

---

# And even better!

Spring AOT

Add `-Dspring.aot.enabled=true`

Even more classes!!!

---

# Practice

````md magic-move
```docker {all|12}{maxHeight:'100px'}
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-slim-musl as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --launcher

FROM bellsoft/liberica-runtime-container:jre-slim-musl as runner

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
```
```docker {12|14-17}{maxHeight:'100px'}
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-slim-musl as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted
FROM bellsoft/liberica-runtime-container:jre-cds-slim-musl as runner

ENTRYPOINT ["java", "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
```
```docker {9-12|4}{maxHeight:'100px'}
#omitted
FROM bellsoft/liberica-runtime-container:jre-cds-slim-musl as runner

ENTRYPOINT ["java", "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
RUN java -Dspring.aot.enabled=true \
  -XX:ArchiveClassesAtExit=./application.jsa \
  -Dspring.context.exit=onRefresh \
  -jar app.jar
```
```docker {4-7|all|2}{maxHeight:'100px'}
#omitted
FROM bellsoft/liberica-runtime-container:jre-cds-slim-musl as runner

ENTRYPOINT ["java", \
            "-Dspring.aot.enabled=true", \
            "-XX:SharedArchiveFile=application.jsa", \
            "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies / ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
RUN java -Dspring.aot.enabled=true \
  -XX:ArchiveClassesAtExit=./application.jsa \
  -Dspring.context.exit=onRefresh \
  -jar app.jar
```
```docker {2|12-15|4-7|all}{maxHeight:'100px'}
#omitted
FROM bellsoft/liberica-runtime-container:jre-slim-musl as runner

ENTRYPOINT ["java", \
            "-Dspring.aot.enabled=true", \
            "-XX:AOTCache=app.aot", \
            "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies / ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
RUN java -Dspring.aot.enabled=true \
  -XX:AOTCacheOutput=app.aot \
  -Dspring.context.exit=onRefresh \
  -jar app.jar
```
````

---

# What does it cost ?

```
111M Nov  4 23:47 app.aot
```

Which is not small at all!

<v-click>But you're trading pull speed for startup speed</v-click>

---
layout: image
image: "/startup times.png"
---

---

# Pushing further with CRaC

CRaC: Coordinated Restore at Checkpoint

> The CRaC (Coordinated Restore at Checkpoint) Project researches coordination of Java programs with mechanisms to checkpoint (make an image of, snapshot) a Java instance while it is executing. Restoring from the image could be a solution to some of the problems with the start-up and warm-up times. The primary aim of the Project is to develop a new standard mechanism-agnostic API to notify Java programs about the checkpoint and restore events. Other research activities will include, but will not be limited to, integration with existing checkpoint/restore mechanisms and development of new ones, changes to JVM and JDK to make images smaller and ensure they are correct.

https://openjdk.org/projects/crac/

---

# In a perfect world it should be:

```docker {6,12|9,10|14-16}
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Dspring.context.checkpoint=onRefresh -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```

---

# But in reality

```
#12 6.051       Suppressed: java.lang.RuntimeException: Native checkpoint failed.
#12 6.051               at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:114) ~[na:na]
#12 6.051               at java.base/jdk.crac.Core.checkpointRestore1(Core.java:192) ~[na:na]
#12 6.051               at java.base/jdk.crac.Core.checkpointRestore(Core.java:299) ~[na:na]
#12 6.051               at java.base/jdk.crac.Core.checkpointRestore(Core.java:278) ~[na:na]
#12 6.051               at java.base/javax.crac.Core.checkpointRestore(Core.java:73) ~[na:na]
#12 6.051               at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:103) ~[na:na]
#12 6.051               at java.base/java.lang.reflect.Method.invoke(Method.java:580) ~[na:na]
#12 6.051               at org.crac.Core$Compat.checkpointRestore(Core.java:141) ~[crac-1.4.0.jar!/:na]
#12 6.051               ... 17 common frames omitted
```

---

# CRaC is hard!

Let's try to fix it with arcane magic

````md magic-move
```docker
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Dspring.context.checkpoint=onRefresh -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
```docker {all|1|11}
# syntax=docker/dockerfile:1-labs
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN java -Dspring.context.checkpoint=onRefresh -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
```docker {11,12}
# syntax=docker/dockerfile:1-labs
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN --security=insecure java -Dspring.context.checkpoint=onRefresh \
  -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
```docker {11,12}
# syntax=docker/dockerfile:1-labs
FROM bellsoft/liberica-runtime-container:jdk-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-3.3.0.jar /app/app.jar
WORKDIR /app
RUN --security=insecure java -Dspring.context.checkpoint=onRefresh \
  -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar || true

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
````


---

# And this is not all!

Did you hear about `buildx`?

````md magic-move
```bash {1,2|3}
docker buildx create --buildkitd-flags \
    '--allow-insecure-entitlement security.insecure' \
    --name insecure-builder
```
```bash
docker buildx use insecure-builder
```
```bash {1|2|3,4}
docker buildx build \
    --allow security.insecure \
    -f Dockerfile.crac \
    -t pet-crac --output type=docker .
```
```bash
docker run --rm -it --privileged pet-crac
```
```plain  {all|4}
Restarting Spring-managed lifecycle beans after JVM restore
HikariPool-1 - Thread starvation or clock leap detected (housekeeper delta=1d19h37m42s102ms798µs806ns).
Tomcat started on port 8080 (http) with context path '/'
Restored PetClinicApplication in 0.186 seconds (process running for 0.19)
```
````

---

# Is it a good solution?

It depends

<v-clicks>

* If "It Works" is enough for you <span v-click=2>-> **YES**</span>
* If you need more predictable and maintanable thing <span v-click=3>-> **NO**</span>

</v-clicks>

---

# How to make it better?

<v-clicks depth="2">

1. Build JAR (in docker or not)
2. Create new image that will run the JAR with CRaC arguments in `ENTRYPOINT`
3. Run the container with capabilities:
    1. CAP_SYS_PTRACE
    2. CAP_CHECKPOINT_RESTORE
4. Container will run and stop
5. Commit the container like
    ```bash
    docker commit container-id new-tag
    ```

</v-clicks>

---
layout: two-cols
---

# Pros and Cons

## Pros:

1. Does not require arcane magic
2. Works more predictably
3. Does not depend on unstable features
4. Does not require privileged containers in 

::right::

# <br/>

## Cons:

1. Organization-specific
2. Requires more steps

---
layout: statement
---

# And now we have all flavours of ultimate docker images!


---

# Quick summary?

1. Use layers for faster deployment
2. Use CDS for faster startup without many compromises
3. Use CRaC for a *lightning-fast* startup (with compromises)
4. Use native image if you're prepared to sacrifice some build time

---
layout: two-cols-header
---

# Thank you!

::left::

Find me @


- <logos-firefox /> https://asm0dey.site
- <logos-bluesky /> @asm0dey.site
- <logos-mastodon-icon /> @asm0dey@fosstodon.org
- <logos-twitter /> asm0di0
- <logos-google-gmail /> me@asm0dey.site
- <logos-linkedin-icon /> <logos-telegram /> <logos-whatsapp-icon /> <logos-facebook /> asm0dey

::right::

<img src="/news.png" class="invert rounded self-center"/>

---


---
layout: end
---
