Docker Test Runner Gradle Plugin
================================
[![Circle CI](https://circleci.com/gh/palantir/gradle-docker-test-runner.svg?style=svg)](https://circleci.com/gh/palantir/gradle-docker-test-runner)

This plugin provides a simple way to test Gradle projects in multiple
different environments specified in Dockerfiles as part of a build. Given a set
of Dockerfiles that contain the desired environments, this plugin adds tasks
for running the project's tests and coverage tasks in those containers. The
test and coverage output for each environment is persisted separately, and
it is possible to combine the coverage from the tests run in all of the
environments using [gradle-jacoco-coverage](https://github.com/palantir/gradle-jacoco-coverage).

This plugin makes it trivial to test existing projects against multiple
different environments in a lightweight manner. The ability to easily run tests
in different environments locally makes the development process faster and
allows environment verification to be part of the core build. It also allows
code coverage to properly account for code that will only execute in certain
environments.

An example use case of this plugin is testing a project against multiple
different versions or vendors of a JDK.

Usage
-----
Apply the plugin using standard gradle convention:

```groovy
plugins {
    id 'com.palantir.docker-test-runner'
}
```

Because this plugin adds test and coverage tasks, it requires the 'java' and
'jacoco' plugins to already be applied.

Configure the plugin using the 'dockerTestRunner' block. The simplest way to
configure the plugin is to specify a `FileCollection` of Dockerfiles that
should be used as the test environments:

```groovy
dockerTestRunner {
    dockerFiles fileTree(project.rootDir) {
        include '**/Dockerfile'
    }
}
```

The `dockerTestRunner` block offers the following options:

* `dockerFiles`: adds the specified Dockerfiles to use as test environments.
* `dockerRunner`: adds the provided runner to use as a test environment. This
 method takes a map with the following properties:
  * `dockerFile`: the `File` for the test environment being added.
  * (optional) `name` the name of the test environment. If it is not
  provided, then a default name is generated based on the path of the file.
  * (optional) `testConfig` configuration object for this runner that contains
  `runArgs`, `testConfig and `jacocoConfig`. The runner-specific configuration
  is applied in addition to the top-level configuration. See description of
  top-level options for explanation of these options.
* (optional) `runArgs` is a [`Multimap`](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Multimap.html)
   that can be provided with custom arguments that should be provided to the
   Docker `run` commmand that is executed by the task. The keys are the flags
   and the values are the values for the flag.
* (optional) `testConfig` is a closure that can be used to configure the
 [`Test`](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.testing.Test.html)
 task run in the container. The closure's delegate is set to be the Test task,
 so Test properties such as `classpath` and `reports` can be configured using
 this closure.
* (optional) `jacocoConfig` is a closure that can be used to configure the
 [`Jacoco`](https://docs.gradle.org/current/dsl/org.gradle.testing.jacoco.tasks.JacocoReport.html)
 task run in the container. The closure's delegate is set to be the
 JacocoReport task, so Jacoco properties such as `executionData` and `
 `sourceDirectories` can be configured using this closure.
* (optional) `createGradleCacheVolumeImage` specifies the image that should be
 used by the `createGradleCacheVolume` task. If it is null, then the image of
 the first specified dockerRunner will be used.

Configuration Examples
----------------------

Add all `Dockerfile` files as test environments and configure tests to only
run tests in a specific directory:

```groovy
dockerTestRunner {
    dockerFiles fileTree(project.rootDir) {
        include '**/Dockerfile'
    }

    testConfig = {
        include('**/docker/*Tests*')
    }
}
```

Add 2 separate named test environments with custom test configurations and
global coverage configuration:

```groovy
dockerTestRunner {
    dockerRunner name: 'openjdk-7',
                 dockerFile: file('dockerFiles/openjdk-7.dockerfile'),
                 testConfig: {
                    include('**/jkd7/*Tests*')
                 }

    dockerRunner name: 'openjdk-8',
                 dockerFile: file('dockerFiles/openjdk-8.dockerfile'),
                 testConfig: {
                     include('**/jkd8/*Tests*')
                 }

    jacocoConfig = {
        classDirectories = project.files(classDirectories.files.collect {
            project.fileTree(dir: it, exclude: '**/Immutable*.class')
        })
    }
}
```

Tasks
-----
Each environment has a name that is either specified as the runner's name or
derived from the parent directory and name of its Dockerfile.

Each environment has its own task group that contains 3 tasks:
* `buildDockerTestRunner-{name}`
* `runJacocoTestReportDockerTestRunner-{name}` (if the Jacoco plugin is applied
 to the project)
* `runTestDockerTestRunner-{name}`

If there is at least one environment, then the following bulk tasks that run
the tasks for all of the environments are added:
* `buildDockerTestRunner`
* `createGradleCacheVolume`
* `jacocoTestReportDockerTestRunner` (if the Jacoco plugin is applied to the
 project)
* `removeGradleCacheVolume`
* `testDockerTestRunner`

Build
-----
The build tasks run `docker build` to create the images for the environments.
The build tasks are invoked automatically as needed, but can be called directly
to build the images for caching purposes.

Test
----
The test tasks use `docker run` to run a Docker container of the image for the
specified environment and then runs a Gradle test task within that container.
The test task that is run within the container tests all of the classes in the
project's Gradle test output classes directory using the test runtime path.

The default destination for the XML and HTML reports will be the same
destination used by the standard Gradle test task with the name appended
to it. For example, if the regular test tasks writes its output to
'build/reports/tests', then this test task will write its output to
'build/reports/tests-{name}'. The raw Jacoco execution output is written to
"${project.getBuildDir()}/jacoco/${name}.exec". This task can be modified or
configured using the `testConfig` closure.

Jacoco Test Report
------------------
The Jacoco test report tasks will be present only if the `jacoco` plugin is
applied to the project. These tasks are analogous to the test tasks, but
generate a coverage report rather than runningtests. The default output
location of the coverage reports is the same location used by the standard
Gradle Jacoco reports task, but the name of the directory within the reports
directory will be the test environment name. This task can be modified or
configured using the `jacocoConfig` closure.

Create Gradle Cache Volume
--------------------------
This tasks creates the Gradle cache Docker data volume and copies the content
of the current Gradle user home directory into the data volume. This data
volume is shared by all subprojects in a given Gradle project. The image that
is used to perform the copy operation can be specified using the
`createGradleCacheVolumeImage` property (by default, the image file of the
first specified dockerRunner will be used). The image being used must already
be built/present -- if a custom image is being used, ensure that it is built
using the appropriate `build` task before invoking this task. This task
typically only needs to be run in CI environments.

In order to increase performance, the Docker test containers use a Docker
[data volume](https://docs.docker.com/engine/userguide/containers/dockervolumes/)
as the Gradle user home (the Gradle cache). The data volume is created by a
container if it does not already exist. This means that, the first time a test
container runs, it may need to download the Gradle dependencies into the cache.
In local environments, incurring this one-time cost typically is not an issue
because the data volume will persist. However, in CI environments this can be
costly. In those scenarios, this task can be run before the other test tasks
are run to copy the local Gradle cache into the Gradle cache Docker volume.

Note that this task copies the entire contents of the Gradle user home
directory, so if that directory is large (as it can be in local environments)
this task may take a long time. This task should generally not be used if the
local Gradle user home cache is large.

Remove Gradle Cache Volume
--------------------------
Removes the Gradle cache Docker data volume using the `docker volume rm`
command. Note that the `docker volume` command only exists in Docker 1.9.0 and
later. This task will fail if the `docker volume` command is not available
locally or if the volume could not be removed because it is in use by a
container.

Combining Coverage
------------------
The coverage outputs of multiple different test environments can be combined
using [gradle-jacoco-coverage](https://github.com/palantir/gradle-jacoco-coverage)
without any further modifications. Running that plugin's `jacocoFullReport`
task will create a report that includes the coverage outputs of the tests
run in the Docker environments.

In addition to creating these scripts, this plugin will merge the entire
contents of `${projectDir}/service` and `${projectDir}/var` into the package.

License
-------
This plugin is made available under the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0).
