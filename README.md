This project explains some of the dependency management tricks that
can be used to create libraries and apps that depend on newer versions
of a transitive dependency. The examples below uses
[reactor](https://projectreactor.io) as an example of such a
dependency because it is nearing a major new release (2.5.0) but
existing dependency management platforms (like Spring Boot 1.3)
declare a dependency on older versions (2.0.7). If you wanted to write
an app that depended on a new version of reactor through a transitive
dependency on a library, this is the situation you would be faced
with.

> NOTE: All the code examples below are in the
> [github repository](https:github.com/dsyer/dependency-hell). You
> should `mvn install` at the top level to get everything set up. The
> Maven projects are all laid out as a reactor build, but this is just
> for convenience. In principle all the projects could be
> independently built and installed (if the relative paths of their
> parents were fixed).

## The Problem

We have a parent pom that has dependency management for reactor
(2.0.7). It does this via a Maven property, i.e. in the parent pom:

```xml
	<properties>
		<reactor.version>2.0.7.RELEASE</reactor.version>
	</properties>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>io.projectreactor</groupId>
				<artifactId>reactor-core</artifactId>
				<version>${reactor.version}</version>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

Then we have a library with this parent that wants to use a newer
version of reactor, so it does this:

```
	<properties>
		<reactor.version>2.5.0.BUILD-SNAPSHOT</reactor.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>io.projectreactor</groupId>
			<artifactId>reactor-core</artifactId>
		</dependency>
	</dependencies>
```

Everyone is happy. Here's a summary of the relationships between the
artifacts:

```
parent (manages reactor:2.0.7)
\(child)- library
   \(depends on)- reactor:2.5.0
```

and the actual dependency:tree from Maven (3.3):

```
[INFO] com.example:example-library:jar:0.0.1-SNAPSHOT
[INFO] \- io.projectreactor:reactor-core:jar:2.5.0.BUILD-SNAPSHOT:compile
[INFO]    \- org.reactivestreams:reactive-streams:jar:1.0.0:compile
```

Then a user wants to write an app that depends on the library and
would like to re-use the parent, let's say for other features that we
haven't included in the simple sample. He does that and finds that
(boo, hoo), the reactor version is messed up. The relationships for
the app can be summarised like this:

```
parent
\(child)- app
  \(depends on)- library
```

and the Maven (3.3) dependency report looks like this:

```
[INFO] com.example:example-app:jar:0.0.1-SNAPSHOT
[INFO] \- com.example:example-library:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- io.projectreactor:reactor-core:jar:2.0.7.RELEASE:compile
[INFO]       +- org.reactivestreams:reactive-streams:jar:1.0.0:compile
[INFO]       \- org.slf4j:slf4j-api:jar:1.7.12:compile
```

(i.e. it has the wrong version of reactor). This is because the parent
has dependency management for reactor, and there is no explicit
dependency or dependency management of reactor in the app itself. The
parent always wins in this case and it doesn't help to add a BOM
(using Maven 3.3 at least) with the right reactor version - only an
explicit version of reactor itself will fix it.

This is the same structure (with fewer levels) as a user app generated
from [initializr](https://start.spring.io), with a dependency on a
libary that uses reactor 2.5.0. It has all the same problems, and the
same options for workarounds and fixes.

## Workarounds and Fixes

*1* Explicitly manage the reactor dependency in the app:

```
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>io.projectreactor</groupId>
				<artifactId>reactor-core</artifactId>
				<version>2.5.0.BUILD-SNAPSHOT</version>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

This is ugly and lots of lines of XML. Note that it doesn't help to
use a BOM that has this code in it with Maven 3.3 (but does with
Maven 3.4, see below).

```
$ cd app-1
$ ../mvn dependency:tree
...
[INFO] com.example:example-app-1:jar:0.0.1-SNAPSHOT
[INFO] \- com.example:example-library:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- io.projectreactor:reactor-core:jar:2.5.0.BUILD-SNAPSHOT:compile
[INFO]       \- org.reactivestreams:reactive-streams:jar:1.0.0:compile
...
```

> NOTE: the same amount of XML (or actually slightly less) can be used
> to explicitly list the same dependencies in the `<dependencies>`
> section of the POM.

*2* Explicitly manage only the reactor version in the app via a
 property:

```
	<properties>
		<reactor.version>2.5.0.BUILD-SNAPSHOT</reactor.version>
	</properties>
```

This seems fairly palatable, and it's a simple rule to follow: if your
project or one of your dependencies needs to override the version of a
transitive dependency that is managed by the parent POM, just add a
version property for that dependency. For this rule to work the parent
POM has to define version properties for all the dependencies that it
manages (the `spring-boot-starter-parent` does this).

```
$ cd app-2
$ ../mvnw dependency:tree
...
[INFO] com.example:example-app-2:jar:0.0.1-SNAPSHOT
[INFO] \- com.example:example-library:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- io.projectreactor:reactor-core:jar:2.5.0.BUILD-SNAPSHOT:compile
[INFO]       \- org.reactivestreams:reactive-streams:jar:1.0.0:compile
...
```

*3* Use a BOM with the new reactor version and Maven 3.4.

I.e. in the app:

```
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>com.example</groupId>
				<artifactId>example-bom</artifactId>
				<version>0.0.1-SNAPSHOT</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

> NOTE: the Spring Boot parallel BOM to this one is
> `spring-boot-dependencies`. Other Boot-based projects, e.g. various
> parts of Spring Cloud, define their own `*-dependencies` BOM that
> you can use in the same way.

Maven 3.4 is not released yet, but you can get it from
[the repo]https://repository.apache.org/content/repositories/snapshots/org/apache/maven/apache-maven/3.4.0-SNAPSHOT/)
e.g. by editing the `wrapper.properties` in the application
project. This strategy is nice because it fits the Maven dependency
management model quite well, but only works with a version of Maven
that isn't released yet.

```
$ cd app-3
$ ./mvnw dependency:tree # N.B. Maven 3.4
...
[INFO] com.example:example-app-3:jar:0.0.1-SNAPSHOT
[INFO] \- com.example:example-library:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- io.projectreactor:reactor-core:jar:2.5.0.BUILD-SNAPSHOT:compile
[INFO]       \- org.reactivestreams:reactive-streams:jar:1.0.0:compile
...
```

> NOTE: the `wrapper.properties` for this project has been set up to
> work at the time of writing. You might have to edit the version
> label to get it to work with the latest snapshot.

*4* Use gradle (instead of Maven) and a BOM with the new reactor
 version. There is no parent, since that is a Maven thing, and
 dependency management with the available BOMs can be applied using
 the `spring.io` plugin.

```
$ cd app-4
$ gradle dependencies
...
compile - Dependencies for source set 'main'.
\--- com.example:example-library:0.0.1-SNAPSHOT
     \--- io.projectreactor:reactor-core:2.5.0.BUILD-SNAPSHOT
          \--- org.reactivestreams:reactive-streams:1.0.0
...
```

*5* Don't use that parent, and adopt the `*-dependencies` model that
 Spring Cloud uses. If the parent has some plugin and property
 declarations that you want to re-use, copy those into a new parent,
 and use that as your application parent POM. The dependency
 management can be declared in a standalone BOM that can then be used
 in the `<dependencyManagement>` section of your application POM, and
 if it is declared first it will take precedence over other BOMs for
 anything it declares explicitly.

```
$ cd app-5
$ ../mvnw dependency:tree
...
[INFO] com.example:example-app-5:jar:0.0.1-SNAPSHOT
[INFO] \- com.example:example-library:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- io.projectreactor:reactor-core:jar:2.5.0.BUILD-SNAPSHOT:compile
[INFO]       \- org.reactivestreams:reactive-streams:jar:1.0.0:compile
...
```
