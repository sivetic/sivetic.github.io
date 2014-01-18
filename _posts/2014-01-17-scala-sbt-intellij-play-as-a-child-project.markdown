---
layout: post
title: Scala, SBT, IntelliJ, and Play! project as a child project
---

# Motivation

I've spent the last few days figuring out ways to organize a new Scala project, built using SBT, with IntelliJ IDEA 13.0.1 acting as the IDE of choice.  The project structure is composed of a REST API implemented using a Play! Framework and a separate JSF UI project that will make calls to the API.  The solution also contains several libraries, some of which are only referenced by one of the two projects, and a commons project referenced by both the Play and JSF projects.  Most sites explaining how to set up a Play! project do not deal with child dependencies not located in the root Play directory.

Note: my approach may not be the best approach.  Integration between IntelliJ, SBT, and Play! has improved as of IntelliJ IDEA v13, however it is still far from flawless.  After doing a fair bit of research (primarily via Google), this is the best solution I've managed to arrive at.  I long for the days of Visual Studio, .NET development, and the excellent IDE integration, but I digress.

# Prerequisites

The following packages will need to be installed before continuing:
  
  - [IntelliJ IDEA 13.0](http://www.jetbrains.com/idea/) with the latest Scala plugin
  - [SBT 0.13.0+](http://www.scala-sbt.org/)
  - [Play! Framework 2.2.2-RC1+](http://www.playframework.com/download)

# Project Structure

Our final project structure will look similar to the following:

```
- root
  - webservices (play rest api)
  - ui (jsf or other ui application)
  - db (database layer, referenced by webservices only)
  - util (various util methods used by db/webservices projects)
  - commons (referenced by webservices and ui)
```

## Folder/file structure

Initially, we want to create a folder structure containing all projects except the Play! REST API project.  We will create the Play! project and copy it to our project structure in the next step.  For purposes of this post, we will call our root project `my-project`.

```
- my-project
  - project
    * plugins.sbt
  - db
    - src
      - main
        - scala
      - test
        - scala
    * build.sbt
  - commons
    - src
      - main
        - scala
      - test
        - scala
    * build.sbt
  - ui
    - src
      - main
        - java
      - test
        - java
    * build.sbt
* build.sbt
```

## Create Play Project

Play! Framework includes a `play` command that can be used to initialize a new project.  Unfortunately, I ran into issues creating a project within an existing SBT project directory, which meant I had to create the play project as a separate step and copy it to the above project structure.  To do so, open a terminal/command prompt, and execute the following command to create our REST API web services project:

`play new webservices`

Press enter to accept webservices as the project name, and select `Create a simple Scala application` to use the Scala template.  Once Play creates the project successfully, copy the entire `webservices` directory to `my-project` directory to include it in our solution.

# Base build.sbt

Once we have our project structure defined, we need to start working on our SBT build files.  The first build file we need to edit is `build.sbt` located in our root directory (`my-project/build.sbt`).  I've included a starting point for our root build.sbt project definition, which defines our root project along with all the child projects and their dependencies.  Refer to [SBT documentation](http://www.scala-sbt.org/release/docs/index.html) for further details on SBT project structures and build files.

One important detail to note is that properties in root `build.sbt` file are not shared across child projects unless specifically desired.  Use `in ThisBuild` in order to share individual properties to all child projects that the root project builds (see aggregate in root project definition)

```scala
import com.typesafe.sbt.packager.Keys._

name := "my-project"

// Using 'in ThisBuild' recursively sets the properties of all projects the root projects builds (aggregates)
version in ThisBuild := "1.0-SNAPSHOT"

organization in ThisBuild := "ca.ivetic"

scalaVersion in ThisBuild := "2.10.3"

scalacOptions in ThisBuild ++= Seq("-deprecation", "-unchecked", "-feature")

scalacOptions in (ThisBuild, Test) ++= Seq("-Yrangepos")

javacOptions in ThisBuild ++= Seq("-source", "1.6", "-target", "1.6", "-Xlint:deprecation", "-Xlint:unchecked")

// Remove snapshots if you do not want the ability to depend on snapshot releases found on Typesafe/Sonatype repositories
// Maven repository is added by default
resolvers in ThisBuild ++= Seq(
  "Typesafe releases" at "http://repo.typesafe.com/typesafe/releases/",
  "Typesafe snapshots" at "http://repo.typesafe.com/typesafe/snapshots/",
  "Sonatype releases" at "http://oss.sonatype.org/content/repositories/releases/",
  "Sonatype snapshots" at "http://oss.sonatype.org/content/repositories/snapshots/"
)

// withSources() enables auto-downloading sources instead of depending on IntelliJ plugins (which I have not found yet) to download sources
// Side-effect of using withSources() is that the initial compile after adding a new dependency takes slightly longer
libraryDependencies in ThisBuild ++= Seq(
  "joda-time" % "joda-time" % "1.6.2" withSources(),
  "log4j" % "log4j" % "1.2.16" withSources(),
  "com.typesafe" % "config" % "1.0.2" withSources(),
  "org.specs2" %% "specs2" % "2.3.7" withSources()
)

// Publish to local maven repository by default instead of to ivy.  Useful if integrating with Maven builds
publishTo in ThisBuild := Some(Resolver.file("file",  new File(Path.userHome.absolutePath+"/.m2/repository")))

publishMavenStyle in ThisBuild := true

parallelExecution in (ThisBuild, Test) := true

// Define the root project, and make it compile all child projects
lazy val root = project.in(file(".")).aggregate(
  commons,
  db,
  util,
  webservices,
  ui
)

// Define individual projects, the directories they reside in, and other projects they depend on

// Commons is a shared project that does not depend on any of our projects
lazy val commons = project.in(file("commons"))

// Util project depends on commons
lazy val util = project.in(file("util")).dependsOn(commons)

// DB project depends on commons
lazy val db = project.in(file("db")).dependsOn(commons)

// WebServices depends on db and util (both of which in turn depend on commons, though we do not define that dependency here)
lazy val webservices = project.in(file("webservices")).dependsOn(db, util)

// UI project only depends on commons
lazy val ui = project.in(file("ui")).dependsOn(commons)

// Add any command aliases that may be useful as shortcuts
addCommandAlias("cc", ";clean;compile")
```

# Child projects' build.sbt files

Each child project will also need a build.sbt file to define several properties, such as name, and specific library dependencies not defined in root build file:

```scala
import sbt.Keys._

// Project name
name := "db"

// If required, you can override properties defined in parent projects, such as version
// version := "1.1"

scalacOptions in (Compile, doc) ++= Opts.doc.title("DB")

// Add any library dependencies not defined in parent projects
libraryDependencies ++= Seq(
  "com.typesafe.play" %% "play-json" % "2.2.2-RC1" withSources()
)
```

Above `build.sbt` needs to be replicated for every project, with exception of webservices which should already have a build.sbt created by the `play` command run earlier.


# Testing project structure

At this point it is a good idea to test compile the project an ensure our structure was created successfully.  Open a new command prompt, `cd` to `my-project` root directory, and start the SBT console by executing the `sbt` command.  Assuming project structure was created correctly, SBT should load the project without any errors.  Any errors encountered must be corrected before continuing.


# Generating IntelliJ IDEA project files

Once the project structure is set up, we will need to generate our IntelliJ IDEA project strucutre.  A useful sbt plugin called `sbt-idea` can be used to generate the project structure.

## Edit `project/plugins.sbt` to add required plugins

Open `project/plugins.sbt` file and edit to reflect the following:

```scala
// Comment to get more information during initialization
logLevel := Level.Warn

// The Typesafe/Sonatype repositories
resolvers += "Typesafe repository" at "http://repo.typesafe.com/typesafe/releases/"

resolvers += "Sonatype snapshots" at "http://oss.sonatype.org/content/repositories/snapshots/"

// Use the Play sbt plugin for Play projects
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.2.2-RC1")

// IntelliJ IDEA plugin
addSbtPlugin("com.github.mpeltonen" % "sbt-idea" % "1.6.0-SNAPSHOT")
```

## Generate IntelliJ IDEA files

Start SBT console, and execute `gen-idea` task.  Assuming no errors are present in project structure, task should complete successfully after a few seconds.  Once complete, the generated project can be opened in IntelliJ IDEA.  Open IntelliJ IDEA, click File-Open, and select folder where project is located.  IntelliJ should open the project, and allow for editing of code.

### IntelliJ IDEA bug

If you try to compile the solution (Build-Rebuild Project), IntelliJ IDEA will likely complain about root-build and webservices-build test/output directories overlapping.  At this time, I don't know exactly what is causing the issue, or what a proper fix for the issue is, however I have been able to compile properly by removing the webservices-build module from my Project Structure (File-Project Structure).

*Important*: removing the webservices-build module seems to be necessary after each run of gen-idea, otherwise you will not be able to compile the project using IntelliJ.

# Quirks

## IntelliJ IDEA overallapping output/test directories

As mentioned earlier, gen-idea creates a root-build and webservices-build (and in some cases other) modules with overlapping output/test directories.  The webservices-build (and any other *-build modules apart from root-build) will need to be removed in order to get IntelliJ IDEA to compile the project.  These steps will need to be performed after each time `gen-idea` task is run.

## Adding dependencies

Another IntelliJ IDEA bug occurs when adding new dependencies to projects.  IntelliJ does not pick up the changes made to `build.sbt`, which means it does not pick up the newly added dependency.  Not recognizing the new dependency breaks auto-complete and code highlighting.  A workaround is to run the `gen-idea` SBT task every time a new dependency is added, which will regenerate the IntelliJ IDEA project files and reload the project, and in turn recognize the new dependencies.  (see above note about removing *-build modules from project structure after running the `gen-idea` task)
