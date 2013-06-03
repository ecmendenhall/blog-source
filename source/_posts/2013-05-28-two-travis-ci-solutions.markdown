---
layout: post
title: "Two Travis CI solutions: Ivy dependencies and headless testing"
date: 2013-05-28 01:06
comments: true
categories: [Java, Travis CI, Today in Yak Shaving]
---
One of last week's projects was setting up [Travis
CI](https://travis-ci.org/), a hosted [continuous
integration](http://martinfowler.com/articles/continuousIntegration.html)
service that's free for open source projects. Even if you haven't
heard about Travis (as I hadn't), you've probably already seen the
service on Github—it's responsible for the green and red build status
icons at the top of many project READMEs.

After every push to a remote repo, Travis builds your project, runs
the test suite, and sends off a notification with the results. Once
it's configured, it's a great way to ensure that even if your project is
unfinished, the published code will work correctly for anyone who
pulls the repo. 

The official [Travis docs](http://about.travis-ci.org/) are close to
comprehensive, but I ran into a few catches getting everything set up (and have an inbox full of
failed build notifications to prove it). Hopefully these solutions
will rescue someone else from a frustrating afternoon.

# Resolving Ivy dependencies
My Tic Tac Toe project uses [Ivy](https://ant.apache.org/ivy/) to
manage build dependencies, and my ant build script includes a [standard
snippet](https://ant.apache.org/ivy/history/latest-milestone/install.html) that downloads Ivy if it's not available locally:

{% codeblock lang:xml %}
<property name="ivy.install.version" value="2.1.0-rc2" />
  <condition property="ivy.home" value="${env.IVY_HOME}">
    <isset property="env.IVY_HOME" />
  </condition>
<property name="ivy.home" value="${user.home}/.ant" />
<property name="ivy.jar.dir" value="${ivy.home}/lib" />
<property name="ivy.jar.file" value="${ivy.jar.dir}/ivy.jar" />

<target name="download-ivy" unless="offline">

  <mkdir dir="${ivy.jar.dir}"/>
  <!-- download Ivy from web site so that it can be used even without --
    -- any special installation -->
  <get src="http://repo2.maven.org/maven2/org/apache/ivy/ivy/${ivy.install.version}/ivy-${ivy.install.version}.jar"
       dest="${ivy.jar.file}"
       usetimestamp="true"/>
</target>

<target name="init-ivy"
        depends="download-ivy">
<!-- try to load ivy here from ivy home, in case the user has not --
  -- already dropped it into ant's lib dir (note that the latter copy --
  -- will always take precedence). We will not fail as long as local --
  -- lib dir exists (it may be empty) and ivy is in at least one of --
  -- ant's lib dir or the local lib dir. -->
  <path id="ivy.lib.path">
    <fileset dir="${ivy.jar.dir}"
             includes="*.jar"/>

  </path>
  <taskdef resource="org/apache/ivy/ant/antlib.xml"
           uri="antlib:org.apache.ivy.ant"
           classpathref="ivy.lib.path"/>
</target>
{% endcodeblock %}

If your build and test targets are set to depend on the `init-ivy`
task, the script will ensure that Ivy resolves itself and the
project's other dependencies correctly. Telling Travis how to build
and test a project is as simple as writing a few lines of YAML. For
example:

{% codeblock lang:yml %}
language: java
install: ant resolve
jdk:
  - oraclejdk7
  - openjdk7
  - openjdk6
{% endcodeblock %}

This tells Travis the project language (which comes with its own
[default settings](http://about.travis-ci.org/docs/user/languages/)\),
points to `ant resolve` as the command necessary to resolve
dependencies, and describes the target Java versions to test against.
But my builds failed with the following mysterious error:

{% codeblock %}
$ ant resolve
Buildfile: /home/travis/build/ecmendenhall/Java-TTT/build.xml

  [mkdir] Created dir: /home/travis/build/ecmendenhall/Java-TTT/lib

check-ivy:
  [echo] Checking for Ivy .jar in local directories.

bootstrap-ivy:
  [echo] Bootstrapping Ivy installation.
  [mkdir] Created dir: /home/travis/.ant/lib
  [get] Getting: http://search.maven.org/remotecontent?filepath=org/apache/ivy/ivy/2.3.0/ivy-2.3.0.jar
  [get] To: /home/travis/.ant/lib/ivy.jar

resolve:
  [echo] Resolving project dependencies.

BUILD FAILED
/home/travis/build/ecmendenhall/Java-TTT/build.xml:52: Problem: failed
to create task or type antlib:org.apache.ivy.ant:retrieve
Cause: The name is undefined.
Action: Check the spelling.
Action: Check that any custom tasks/types have been declared.
Action: Check that any <presetdef>/<macrodef> declarations have taken
        place.

        No types or tasks have been defined in this namespace yet
        This appears to be an antlib declaration.
        Action: Check that the implementing library exists in one of:
            -/usr/share/ant/lib
            -/home/travis/.ant/lib
            -a directory added on the command line with the -lib argument
{% endcodeblock %}

Even though ant was correctly downloading an ivy jarfile, the build
script failed to find it. The answers to my [Stack Overflow question](http://stackoverflow.com/questions/16673978/travis-ci-cant-find-ivy-jarfile)
explained that this was related to the order in which jars are
loaded to the Java classpath. The easiest solution is to run ant
twice, once to download ivy, and again to build and test the
project.

Once the source of this error was cleared up, the solution was simple.
Adding a `before_install` script to my Travis build ensured that ivy
would be downloaded before trying to build the project. This meant
adding just one line to my `.travis.yml`:

{% codeblock lang:yml %}
language: java
before_install: ant init-ivy
install: ant resolve
jdk:
  - oraclejdk7
  - openjdk7
  - openjdk6
{% endcodeblock %}

There are a number of other options to run scripts at different times
in the Travis lifecycle. Check out the rest of the docs
[here](http://about.travis-ci.org/docs/user/build-configuration/#Build-Lifecycle).

# Headless testing with xfvb

My Travis builds worked nicely until I started adding a Swing view to
my Tic Tac Toe project. Although the tests would run and pass locally,
Travis threw a `java.awt.HeadlessException` as soon as it tried to run
the test suite. The [Travis docs](http://about.travis-ci.org/docs/user/gui-and-headless-browsers/) explain using a tool called xvfb (X
Virtual Framebuffer) to simulate a windowing system and run headless
tests, but warn that "you need to tell your testing tool process"
exactly how to use it.

Exactly how to do this wasn't clear, but after fiddling with JVM startup arguments and
trying to shell out from inside ant, I discovered that the solution
was dead simple: just add the recommended arguments to `.travis.yml`,
and ant and JUnit will take care of the rest. Here's my final
`.travis.yml`, including the solutions to both problems.
    
{% codeblock lang:yml %}
language: java
before_install:
  - ant init-ivy
  - "export DISPLAY=:99.0"
  - "sh -e /etc/init.d/xvfb start"
install: ant resolve
jdk:
  - oraclejdk7
  - openjdk7
  - openjdk6
{% endcodeblock %}
