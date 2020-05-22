---
classes:
    - wide
excerpt: A quick recipe for getting a Maven project containing a test suite containerized, including all of its dependencies
---

The Challenge
=============

I recently ran into an interesting challenge.  My company maintains suites of automated API tests in a [Maven](https://maven.apache.org) project, which
we run using [JUnit](https://junit.org/junit4/).  We've been considering modernizing our Continuous Integration (CI) implementation and noticed that modern
CI systems now commonly use [Docker](https://www.docker.com/) containers to implement their steps.

As I considered how I would containerize our API test suites, I made the decision that I didn't want to abandon the functionality that Maven provides.
The [Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/), as one example, provides the framework of our entire test suite execution.
Our tests, developed over the course of a few years, made assumptions that this would be their execution environment, and we use the test suite reporting,
flow control, and test selection functionality that's part of the plugin.  We also rely on the parallel execution it supports.  I didn't want to replace
any of that functionality...instead, I wanted this all to happen quickly and easily in a Docker container.

As I began to build a container with our build, another use case presented itself to me:  Our platform runs both in a public cloud environment and in
on-premise installations.  Some of our test suites are useful for validating successfully-installed software on-prem, and it would be great to be able
to run this in those contexts.  I realized two problems with Maven's practice of assuming a connected environment when it executes:

- By default, my container downloads all of its dependencies each time it executes.  This wastes time and bandwidth.  When this container is used in an
  on-prem context, I would like to avoid the need for it to contact my private artifact repository to download dependencies at all.
- Depending on how dependency versions are specified (or not specified) in the POM for this project, it's possible a dependency could change between the
  time of building my container and its execution.

I decided it would be better to pre-package any needed dependencies inside the container so that each image is complete and doesn't have any external
dependencies (other than the services it tests).  Additionally, I was able to speed up the execution of this container from about 3 minutes on my laptop
down to 12 seconds.

Maven's Offline Mode
====================

Maven's dependency plugin has an [offline mode](https://maven.apache.org/plugins/maven-dependency-plugin/go-offline-mojo.html) that I thought would be
sufficient for making this happen.  To use it, you use the following goal:

```
$ mvn dependency:go-offline
```

When this happens, the dependency plugin resolves all of the dependencies in your local `pom.xml` file, downloads the needed artifacts, and stores them
in your local repository.  I added this command as a `RUN` step in my `Dockerfile` so that the container image would include all of the dependencies needed.
This way, I would be able to run a Maven build which triggers the Surefire tests, but not have it download all of the dependencies.

What wasn't initially clear for me after reading the dependency plugin's documentation was that the `dependency:go-offline` goal isn't completely
sufficient to operate completely offline.  Some Maven functionality requires dependencies, even if you don't have them specified in your POM.  One example
of this is a dependency named `maven-profile` that, for me, wasn't downloaded with the `dependency:go-offline` goal.

There were several of these missing dependencies, and I experimented with explicitly adding these dependencies into my POM.  This wasn't ideal, because
these are normally managed internally by Maven, and I didn't want to burden this project with these extra dependencies.

The Recipe
==========

Maven Build
-----------

I used [docker-maven-plugin](https://github.com/fabric8io/docker-maven-plugin)  to allow me to create the container image from my Maven build.  The
relevant section of the POM looks as follows:

```
    ...

    <properties>
       <docker.push.registry>[HOSTNAME]:[PORT]</docker.push.registry>
       <docker-maven-plugin.version>0.31.0</docker-maven-plugin.version>
       <docker.image.prefix>[IMAGE_PREFIX]</docker.image.prefix>
       <docker.image.name>${project.artifactId}</docker.image.name>
       <surefire.version>2.19.1</surefire.version>
    </properties>

    ...

    <profiles>
       <profile>
           <id>dockerize</id>
           <activation>
               <property>
                   <name>dockerize</name>
               </property>
           </activation>
           <build>
               <plugins>
                   <plugin>
                       <groupId>io.fabric8</groupId>
                       <artifactId>docker-maven-plugin</artifactId>
                       <version>${docker-maven-plugin.version}</version>
                       <configuration>
                           <images>
                               <image>
                                   <name>${docker.image.prefix}/${docker.image.name}</name>
                                   <build>
                                       <cacheFrom/>
                                       <tags>
                                           <tag>latest</tag>
                                       </tags>
                                       <assembly>
                                           <descriptorRef>artifact</descriptorRef>
                                           <basedir>/</basedir>
                                       </assembly>
                                       <contextDir>${project.basedir}</contextDir>
                                       <dockerFile>src/main/docker/Dockerfile</dockerFile>
                                   </build>
                               </image>
                           </images>
                       </configuration>
                       <executions>
                           <execution>
                               <id>build-image</id>
                               <phase>package</phase>
                               <goals>
                                   <goal>build</goal>
                               </goals>
                           </execution>
                           <execution>
                               <id>push-image</id>
                               <phase>install</phase>
                               <goals>
                                   <goal>push</goal>
                               </goals>
                           </execution>
                       </executions>
                   </plugin>
               </plugins>
           </build>
       </profile>
    </profiles>
    
    ...

    <dependencies>
        <dependency>
           <groupId>org.apache.maven.surefire</groupId>
           <artifactId>surefire-junit47</artifactId>
           <version>${surefire.version}</version>
       </dependency>
    </dependencies>

```

This configures `docker-maven-plugin` to build the image whenever the `package` phase is run.  And when the `install` phase runs, the image is pushed to
the repository configured in the properties.  Finally, the `contextDir` configuration attribute is set here specifically to allow us to include files
from the source tree to be included in the resulting image.  By default, `contextDir` is set to the directory containing the `Dockerfile`, and the plugin
prohibits the inclusion of files outside that directory.  The properties used include the following:

- `docker.push.registry` controls the Docker registry to which the image is pushed.  You will likely need to `docker login` to this registry for the push
  to work.
- `docker-maven-plugin.version` controls the version of `docker-maven-plugin` to use.
- `docker.image.prefix` controls the prefix associated with the image in the repository.  At my company, it's common practice to prefix all of our images
  with our company name.
- `docker.image.name` controls the name of the image.  I've just set it to the project's artifact ID.

For the tags assigned to the built image, I've just tagged the image with `latest` with each build, but you could do something better like including the artifact version.

Finally, in order to get my complete test execution working, I needed to add one dependency for the `surefire-junit47` artifact to the POM.  I'll explain
why below as I describe the Dockerfile and how the container is built.

Maven Settings
--------------

I include a `settings.xml` file in my repository which contains the following:

```

<?xml version="1.0" encoding="UTF-8"?>
<settings xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd" xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <localRepository>/root/.m2/repository</localRepository>
  <servers>

    ...

  </servers>
  <profiles>
    <profile>
      <repositories>

        ...

      </repositories>
      <pluginRepositories>

        ...

      </pluginRepositories>
      <id>[PROFILE_ID]</id>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>[PROFILE_ID]</activeProfile>
  </activeProfiles>
</settings>

```

This file contains a limited set of credentials for the private repo that has a few internal dependencies, and also specifies the location of the local
repository, `/root/.m2/repository`, within my container.  In this case, I can accept that individuals with access to this container will have the same
level of read-only access to the repo associated with these credentials.

Dockerfile
----------

An example Dockerfile appears below:

```

FROM maven:3-jdk-11

COPY pom.xml .
COPY src /src/
COPY target /target/

RUN mvn -B -s src/main/resources/settings.xml install -DskipTests

ENTRYPOINT mvn -B -s src/main/resources/settings.xml -o surefire:test

```

This builds an image that is based on the official Maven image.  I haven't bothered locking to a specific version, but you might find that appropriate.
The `COPY` steps add the POM and the `src` and `target` directories to the container image.  In my case, this includes the `settings.xml` I described above.

The `RUN` step invokes Maven and goes all the way through the `install` phase, skipping the execution of the tests.  I chose to use this approach rather
than manage a larger list of dependencies, some of which include 'hidden' plugins or artifacts with which a developer wouldn't normally concern themselves.
This approach does unnecessarily compile the project, however, it greatly simplified the dependency management.  Since this step happens only at container
build time, the extra work is not a problem.

The additional dependency I mentioned earlier, `surefire-junit47` is one of the hidden dependencies that would be transparently downloaded and installed
in the local Maven repository if I actually executed tests during the container build.  However, I had a strong preference *not* to execute these tests,
because they're not tests of the test suite itself, but tests of APIs in our platform.  I was willing to manage the one additional dependency to avoid
running these tests while building the container.

Finally, the `ENTRYPOINT` step triggers the execution of Maven when this container is run.  The `-o` command-line flag instructs Maven to run the build in
offline mode, relying on the local repository, which, in this case, lives in `/root/.m2/repository`.  The preceding `RUN` step insures that the local
repository has everything needed to complete the test cycle without downloading any additional artifacts.  The `-B` flag on the Maven command line tells
Maven that you're running the command in batch mode, and switches off Maven's beautiful colorized output.

Usage
=====

You could then run your container with the following command:

```

$ docker run mycompany/my-test-suite -PmyTestProfile

```

Our test suite uses a Maven profile to select the tests to run and the environment to run them against.  Using an `ENTRYPOINT` step in your `Dockerfile`
will cause any additional arguments you provide on the command line to be appended to the command line specified in the `ENTRYPOINT` step.

Conclusion
==========

I hope this recipe comes in handy for you.  I found it relatively easy to get this project built into a container that can run quickly and execute my
company's test suite.  For me, Maven has always felt like a 'heavy' tool, in large part because of the pattern of validating and downloading dependencies
at runtime.  Most of the time, that behavior is beneficial.  However, for this use case, I was pleasantly surprised at how easily I could disconnect it
from repository and dependency hell, and get it to do something very useful.

