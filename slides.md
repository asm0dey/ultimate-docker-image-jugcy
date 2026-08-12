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

- Smallest — fewest bytes on disk, smallest attack surface
- Fastest startup — 4 seconds or 0.1?
- Something in between — the smallest *pull* per commit

---
layout: statement
---

# So, let's look at all of them!

---

# How do we create an image?

Let's start from trivial.

```docker {none|1|3|4|all}
FROM bellsoft/liberica-runtime-container:jdk-26-musl

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
FROM bellsoft/liberica-runtime-container:jdk-26-musl

COPY . /app
RUN cd /app && ./gradlew build
ENTRYPOINT java -jar /app/build/libs/spring-petclinic*.jar
```

---

# Result

```text {all|2|3|4}{maxHeight:'300px'}
Cmp   Size  Command
    112 MB  FROM blobs
   1.9 MB   COPY . /app
    590 MB  cd /app && ./gradlew build -xtest
```

<span v-click="1">112 MB of Java</span>

<span v-click="2">1.9 MB of sources</span>

<span v-click="3">590 MB of the app *and* Gradle build caches</span>

<span v-click="4">713 MB total</span>

---
layout: two-cols-header
---

# 590 MB are changed on every build!

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
FROM bellsoft/liberica-runtime-container:jdk-26-musl

COPY . /app
RUN cd /app && ./gradlew build
ENTRYPOINT java -jar /app/build/libs/spring-petclinic*.jar
```
```docker {3-4|all}
FROM bellsoft/liberica-runtime-container:jdk-26-musl

COPY build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
CMD java -jar /app/app.jar
```
````

<span v-click="2">Just copying the prebuilt `jar` file</span>

---

# Results

````md magic-move
```plain {3-4}
Cmp   Size  Command
    112 MB  FROM blobs
   1.9 MB   COPY . /app
    590 MB  cd /app && ./gradlew build -xtest
```
```plain{3}
Cmp   Size  Command
    112 MB  FROM blobs
   66.4 MB  COPY build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar
```
````

No Gradle caches and build dir! 713 MB → 187 MB

---
layout: statement
---

# Saved 526 MB

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
FROM bellsoft/liberica-runtime-container:jdk-26-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-26-musl as runner

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
```

---

# Result

<div></div>

+ jar — same size, 66.4 MB

+ JDK layer 112 MB → JRE layer 144 MB <span v-click="1">🤨</span>

<v-click at="2">

+ But we *pull* compressed layers: **91 MB → 51 MB**

+ The JRE has fewer modules (48 vs 68) — it only looks bigger because it ships `lib/modules` uncompressed

</v-click>

---
layout: statement
---

# That's all folks!

<h2 v-click>Or is it?</h2>

---

# What's these 66 MB?

We have to pull 66 MB of petclinic on every small commit!

Is there a way to optimize it?

---

# Layers!

```docker {1-4|6|8,9|10|12,6|15-18|14}{maxHeight:'180px'}
FROM bellsoft/liberica-runtime-container:jdk-26-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-26-slim-musl as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/liberica-runtime-container:jre-26-slim-musl as runner

ENTRYPOINT ["java", "-jar", "app.jar"]
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

```plain {1|101|99,98|5-97|3,2}{maxHeight:'300px'}
extracted
├── application
│   └── spring-petclinic-4.0.0-SNAPSHOT.jar
├── dependencies
│   └── lib
│       ├── angus-activation-2.0.3.jar
│       ├── antlr4-runtime-4.13.2.jar
│       ├── aspectjweaver-1.9.25.1.jar
│       ├── attoparser-2.0.7.RELEASE.jar
│       ├── bootstrap-5.3.8.jar
│       ├── byte-buddy-1.18.10.jar
│       ├── cache-api-1.1.1.jar
│       ├── caffeine-3.2.4.jar
│       ├── classmate-1.7.3.jar
│       ├── commons-logging-1.3.6.jar
│       ├── error_prone_annotations-2.49.0.jar
│       ├── font-awesome-4.7.0.jar
│       ├── h2-2.4.240.jar
│       ├── HdrHistogram-2.2.2.jar
│       ├── hibernate-core-7.4.1.Final.jar
│       ├── hibernate-models-1.1.1.jar
│       ├── hibernate-validator-9.1.0.Final.jar
│       ├── HikariCP-7.0.2.jar
│       ├── istack-commons-runtime-4.1.2.jar
│       ├── jackson-annotations-2.21.jar
│       ├── jackson-core-3.1.4.jar
│       ├── jackson-databind-3.1.4.jar
│       ├── jakarta.activation-api-2.1.4.jar
│       ├── jakarta.annotation-api-3.0.0.jar
│       ├── jakarta.inject-api-2.0.1.jar
│       ├── jakarta.persistence-api-3.2.0.jar
│       ├── jakarta.transaction-api-2.0.1.jar
│       ├── jakarta.validation-api-3.1.1.jar
│       ├── jakarta.xml.bind-api-4.0.5.jar
│       ├── jaxb-core-4.0.9.jar
│       ├── jaxb-runtime-4.0.9.jar
│       ├── jboss-logging-3.6.3.Final.jar
│       ├── jspecify-1.0.0.jar
│       ├── jul-to-slf4j-2.0.18.jar
│       ├── log4j-api-2.25.4.jar
│       ├── log4j-to-slf4j-2.25.4.jar
│       ├── logback-classic-1.5.34.jar
│       ├── logback-core-1.5.34.jar
│       ├── micrometer-commons-1.17.0.jar
│       ├── micrometer-core-1.17.0.jar
│       ├── micrometer-jakarta9-1.17.0.jar
│       ├── micrometer-observation-1.17.0.jar
│       ├── mysql-connector-j-9.7.0.jar
│       ├── postgresql-42.7.11.jar
│       ├── slf4j-api-2.0.18.jar
│       ├── snakeyaml-2.6.jar
│       ├── spring-aop-7.0.8.jar
│       ├── spring-aspects-7.0.8.jar
│       ├── spring-beans-7.0.8.jar
│       ├── spring-boot-4.1.0.jar
│       ├── spring-boot-actuator-4.1.0.jar
│       ├── spring-boot-actuator-autoconfigure-4.1.0.jar
│       ├── spring-boot-autoconfigure-4.1.0.jar
│       ├── spring-boot-cache-4.1.0.jar
│       ├── spring-boot-data-commons-4.1.0.jar
│       ├── spring-boot-data-jpa-4.1.0.jar
│       ├── spring-boot-health-4.1.0.jar
│       ├── spring-boot-hibernate-4.1.0.jar
│       ├── spring-boot-http-converter-4.1.0.jar
│       ├── spring-boot-jackson-4.1.0.jar
│       ├── spring-boot-jdbc-4.1.0.jar
│       ├── spring-boot-jpa-4.1.0.jar
│       ├── spring-boot-micrometer-metrics-4.1.0.jar
│       ├── spring-boot-micrometer-observation-4.1.0.jar
│       ├── spring-boot-persistence-4.1.0.jar
│       ├── spring-boot-servlet-4.1.0.jar
│       ├── spring-boot-sql-4.1.0.jar
│       ├── spring-boot-thymeleaf-4.1.0.jar
│       ├── spring-boot-tomcat-4.1.0.jar
│       ├── spring-boot-transaction-4.1.0.jar
│       ├── spring-boot-validation-4.1.0.jar
│       ├── spring-boot-webmvc-4.1.0.jar
│       ├── spring-boot-web-server-4.1.0.jar
│       ├── spring-context-7.0.8.jar
│       ├── spring-context-support-7.0.8.jar
│       ├── spring-core-7.0.8.jar
│       ├── spring-data-commons-4.1.0.jar
│       ├── spring-data-jpa-4.1.0.jar
│       ├── spring-expression-7.0.8.jar
│       ├── spring-jdbc-7.0.8.jar
│       ├── spring-orm-7.0.8.jar
│       ├── spring-tx-7.0.8.jar
│       ├── spring-web-7.0.8.jar
│       ├── spring-webmvc-7.0.8.jar
│       ├── thymeleaf-3.1.5.RELEASE.jar
│       ├── thymeleaf-spring6-3.1.5.RELEASE.jar
│       ├── tomcat-embed-core-11.0.22.jar
│       ├── tomcat-embed-el-11.0.22.jar
│       ├── tomcat-embed-websocket-11.0.22.jar
│       ├── txw2-4.0.9.jar
│       ├── unbescape-1.1.6.RELEASE.jar
│       └── webjars-locator-lite-1.1.3.jar
├── snapshot-dependencies
└── spring-boot-loader

6 directories, 93 files
```

---

# Layered image structure

```bash {all|2|5}{maxHeight:'180px'}
> du -h --si --max-depth=1 extracted
66M     extracted/dependencies
0       extracted/spring-boot-loader
0       extracted/snapshot-dependencies
1.1M    extracted/application
67M     extracted
```

<v-click at="1">

- Dependencies: ~66 MB

</v-click>

<v-click at="2">

- Application: ~1 MB

</v-click>

---
layout: statement
image: /cap.png
---

# 65.1 MB + 1.1 MB = 66.2 MB

## 😱

### The same 66 MB jar, just cut in two

---

# What are we optimizing?

Pull size!

Every pull after the first one is only ~1 MB — the application layer!

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
pack build petclinic \
  --builder bellsoft/buildpacks.builder:musl \
  --path build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar \
  -e BP_JVM_VERSION=26
```

<v-click>Point it at the **jar**, not at `.` — that is what unlocks the layering</v-click>

---

# Result

``` {none|9|7|5|4}
ID         TAG              SIZE      COMMAND                                                                               │
a3f1c07d21 petclinic:latest 0B        Buildpacks Process Types                                                              │
<missing>                   1.59kiB   Buildpacks Launcher Config                                                            │
<missing>                   2.79MiB   Buildpacks Application Launcher                                                       │
<missing>                   1.38MiB   Application Slice: 4                                                                  │
<missing>                   392.58kiB Application Slice: 2                                                                  │
<missing>                   62.08MiB  Application Slice: 1                                                                  │
<missing>                   1.06MiB   Software Bill-of-Materials                                                            │
<missing>                   105.86MiB Layer: 'jre', Created by buildpack: bellsoft/buildpacks/liberica@20260810             │
<missing>                   5.10MiB   Layer: 'helper', Created by buildpack: bellsoft/buildpacks/liberica@20260810          │
```

<div class="relative h-20 mt-2">
<div v-click="[1,2]" class="absolute">The <b>JRE</b> — 111 MB, shared by every app built with this builder</div>
<div v-click="[2,3]" class="absolute"><b>Slice 1</b> — 65.1 MB of dependencies, untouched between commits</div>
<div v-click="[3,4]" class="absolute"><b>Slice 4</b> — 1.45 MB of your classes: the only thing you re-push</div>
<div v-click="[4,5]" class="absolute">The <b>launcher</b> — 2.93 MB of process types and entrypoint wiring</div>
<div v-click="5" class="absolute">226 MB total — and the buildpack sliced it for us</div>
</div>

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
layout: statement
---

# But we promised the *smallest* one

## So far we only made the *pull* small

---

# Where do those 207 MB live?

```text {all|2|3|4}
  Size  Layer
138 MB  FROM jre-26-slim-musl
 67 MB  COPY dependencies/
1.4 MB  COPY application/
```

<v-click at="1">The JRE is now the biggest thing in the image</v-click>

<v-click at="3">Our code is 0.7% of it</v-click>

---

# Do we need a *whole* JRE?

`jlink` — build a runtime with only the modules you use. Since Java 9.

Ask `jdeps` what the app actually touches:

```bash
jdeps --ignore-missing-deps --print-module-deps -q \
  --multi-release 26 -cp 'extracted/dependencies/lib/*' \
  --recursive extracted/application/app.jar
```

<v-click>

```text
java.base,java.compiler,java.desktop,java.instrument,java.net.http,
java.prefs,java.rmi,java.scripting,java.security.jgss,
java.sql.rowset,jdk.jfr,jdk.management,jdk.unsupported
```

</v-click>

---

# Linking it

````md magic-move
```docker {1}
FROM bellsoft/liberica-runtime-container:jdk-26-musl as builder

RUN jlink --add-modules "$(jdeps ... app.jar)" \
      --strip-debug --no-header-files --no-man-pages \
      --compress zip-9 --output /javaruntime
```
```plain
Error: This JDK does not support linking from the current run-time image
```
```docker {1}
FROM bellsoft/liberica-runtime-container:jdk-all-26-musl as builder

RUN jlink --add-modules "$(jdeps ... app.jar)" \
      --strip-debug --no-header-files --no-man-pages \
      --compress zip-9 --output /javaruntime
```
````

<v-click at="2">The slim JDK ships without `jmods` — `jlink` needs the `-all` one</v-click>

---

# The runtime we got

| | Modules | Size |
|---|---|---|
| JDK | 68 | 217 MB |
| JRE | 48 | 138 MB |
| **our `jlink` runtime** | **21** | **74 MB** |

<v-click>Then put it on a base that has nothing else in it</v-click>

---

# Distroless

```docker {1|2|4}
FROM bellsoft/hardened-base:distroless-musl
COPY --from=builder /javaruntime /opt/java
WORKDIR /app
ENTRYPOINT ["/opt/java/bin/java", "-jar", "app.jar"]
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./
```

<v-click at="1">0.6 MB of base image. That's the whole OS.</v-click>

<v-click at="3">

```bash
> docker run --rm --entrypoint /bin/sh pet-jlinked -c id
exec: "/bin/sh": stat /bin/sh: no such file or directory
```

</v-click>

---

# Result

**207 MB → 147 MB** on disk. 60 MB gone.

<v-click>And it still starts in 2.8 s and still serves pets 🐕</v-click>

---
layout: statement
---

# But what about the pull?

<h2 v-click>😱</h2>

---

# The pull

| Image | On disk | Pulled |
|---|---|---|
| slim JRE | 207 MB | 108.9 MB |
| distroless JRE | 206 MB | 108.4 MB |
| `jlink` + distroless | **147 MB** | 109.3 MB |

<v-click>We cut 60 MB from disk and pull **0.4 MB more**</v-click>

<v-click>`lib/modules` was already compressed — and the 61.6 MB of dependencies didn't move</v-click>

---

# So was it pointless?

<v-clicks>

* Pull size: yes, pointless — your dependencies *are* the image
* Disk and page cache: 60 MB per node, per version
* Attack surface: 48 modules → 21, no shell, no package manager
* And you don't need `jlink` for that last one:<br/>`bellsoft/hardened-liberica-runtime-container:jre-26-distroless-musl`

</v-clicks>

---

# Optimization is multidimensional

When startup time is more important…

There are several solutions for the Spring application startup time

1. Class Data Sharing (CDS)
2. Project Leyden — the AOT cache ([JEP 483](https://openjdk.org/jeps/483), [JEP 514](https://openjdk.org/jeps/514))
3. Coordinated Restore at Checkpoint
4. Native Image

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
```docker {all|5}
#omitted: builder stage
FROM bellsoft/liberica-runtime-container:jre-25-slim-musl as optimizer
COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --launcher

FROM bellsoft/liberica-runtime-container:jre-25-slim-musl as runner
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
```
```docker {5|7,8}
#omitted: builder stage
FROM bellsoft/liberica-runtime-container:jre-25-slim-musl as optimizer
COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
WORKDIR /app
RUN java -Djarmode=tools -jar /app/app.jar extract --layers --destination extracted

FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl as runner
ENTRYPOINT ["java", "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
```
```docker {8-11|3}
#omitted: builder and optimizer stages
FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl as runner
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
```docker {3-6|all}
#omitted: builder and optimizer stages
FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl as runner
ENTRYPOINT ["java", \
            "-Dspring.aot.enabled=true", \
            "-XX:SharedArchiveFile=application.jsa", \
            "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
COPY --from=optimizer /app/extracted/spring-boot-loader/ ./
COPY --from=optimizer /app/extracted/snapshot-dependencies/ ./
COPY --from=optimizer /app/extracted/application/ ./
RUN java -Dspring.aot.enabled=true \
  -XX:ArchiveClassesAtExit=./application.jsa \
  -Dspring.context.exit=onRefresh \
  -jar app.jar
```
```docker {2|11-14|3-6|all}
#omitted: builder and optimizer stages
FROM bellsoft/liberica-runtime-container:jre-25-slim-musl as runner
ENTRYPOINT ["java", \
            "-Dspring.aot.enabled=true", \
            "-XX:AOTCache=app.aot", \
            "-jar", "app.jar"]
COPY --from=optimizer /app/extracted/dependencies/ ./
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
> ls -lh --si app.aot
117M Aug 11 13:47 app.aot
```

Which is not small at all!

<v-click>But you're trading pull speed for startup speed</v-click>

---

# Startup, measured

5× faster for 117 MB on disk — same host, same layered image

| Image | Startup |
|---|---|
| plain JRE, no cache | 3.7 – 4.0 s |
| `-XX:SharedArchiveFile` (AppCDS) | 1.66 – 1.86 s |
| `-XX:AOTCache` | 1.12 – 1.18 s |
| `-XX:AOTCache` + `-Dspring.aot.enabled=true` | **0.69 – 0.79 s** |

---

# Pushing further with CRaC

CRaC: Coordinated Restore at Checkpoint

> …coordination of Java programs with mechanisms to checkpoint (make an image of, snapshot) a Java
> instance while it is executing. Restoring from the image could be a solution to some of the
> problems with the start-up and warm-up times.

https://openjdk.org/projects/crac/

<v-click>

Two things to know before we start:

* `jre-crac-slim` is **JDK 21** — there is no CRaC build of 25 or 26 yet
* the app needs `org.crac:crac` on the classpath for `spring.context.checkpoint` to fire

</v-click>

---

# In a perfect world it should be:

```docker {6,12|9,10|14-16}
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
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
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
WORKDIR /app
RUN java -Dspring.context.checkpoint=onRefresh -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
```docker {all|1|11}
# syntax=docker/dockerfile:1-labs
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
WORKDIR /app
RUN java -Dspring.context.checkpoint=onRefresh -XX:CRaCCheckpointTo=/checkpoint -jar /app/app.jar

FROM bellsoft/liberica-runtime-container:jre-crac-slim as runner

ENTRYPOINT java -XX:CRaCRestoreFrom=/checkpoint
COPY --from=optimizer /app/app.jar /app/app.jar
COPY --from=optimizer /checkpoint /checkpoint
```
```docker {11,12}
# syntax=docker/dockerfile:1-labs
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
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
FROM bellsoft/liberica-runtime-container:jdk-25-musl as builder

COPY . /app
RUN cd /app && ./gradlew build -xtest

FROM bellsoft/liberica-runtime-container:jre-crac-slim as optimizer

COPY --from=builder /app/build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar /app/app.jar
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
HikariPool-1 - Thread starvation or clock leap detected (housekeeper delta=1h6m48s576ms).
Tomcat started on port 8080 (http) with context path '/'
Restored PetClinicApplication in 0.104 seconds (process running for 0.104)
```
````

---

# What does *this* cost?

```text
443 MB  /checkpoint
 66 MB  app.jar
199 MB  JDK 21 CRaC JRE
------
708 MB  total
```

<v-click>0.1 s startup — and the biggest image in this whole talk</v-click>

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

1. Use layers for faster deployment — 66 MB pulled once, ~1 MB per commit after
2. Use `jlink` + distroless for the smallest, quietest image — 207 MB → 147 MB, 21 modules, no shell
3. Use CDS or the AOT cache for faster startup without many compromises — 3.9 s → 0.7 s
4. Use CRaC for a *lightning-fast* startup — 0.1 s, for a 708 MB image and privileged builds
5. Use native image if you're prepared to sacrifice some build time

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
