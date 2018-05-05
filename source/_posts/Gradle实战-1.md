---
title: Gradle实战-1
date: 2016-02-16 23:47:58 +0800
categories: Programming
tags: Gradle
---
#### 1. 使用Gradle创建Java项目目录结构
创建目录`LearnGradle`，并执行
```bash
$ gradle init
:wrapper
:init

BUILD SUCCESSFUL

Total time: 0.707 secs
```
<!-- more -->

#### 2. Gradle中的task
在`build.gradle`中添加

```bash
/*hellow gradle*/
task helloGradle {
    doLast{
        println 'Hello Gradle'
    }
}
```
在项目根目录下运行

```bash
$ gradle helloGradle
:helloGradle
Hello Gradle

BUILD SUCCESSFUL

Total time: 0.667 secs
```

#### 3. 高级特性示例
```groovy
/*task chain sample*/
task startChain {
    doLast {
        chainMessage()
    }
}

//隐含对Ant任务的使用
def chainMessage() {
    ant.echo(message: 'Repeat from here...')
}

//循环定义任务，使用隐式变量it
3.times {
    task "repeatTasks$it" {
        doLast {
            println "do repeatTask$it"
        }
    }
}

//定义任务依赖
repeatTasks0.dependsOn startChain
repeatTasks1.dependsOn repeatTasks0
repeatTasks2.dependsOn repeatTasks1

task taskChainSample (dependsOn: repeatTasks2)
```

在项目根目录下运行

```bash
$ gradle taskChainSample
:startChain
[ant:echo] Repeat from here...
:repeatTasks0
do repeatTasktask ':repeatTasks0'
:repeatTasks1
do repeatTasktask ':repeatTasks1'
:repeatTasks2
do repeatTasktask ':repeatTasks2'
:taskChainSample

BUILD SUCCESSFUL

Total time: 0.752 secs
```

#### 4. Gradle Help
```bash
$ gradle --help

USAGE: gradle [option...] [task...]

-?, -h, --help          Shows this help message.
-a, --no-rebuild        Do not rebuild project dependencies.
-b, --build-file        Specifies the build file.
-c, --settings-file     Specifies the settings file.
--configure-on-demand   Only relevant projects are configured in this build run. This means faster build for large multi-project builds. [incubating]
--console               Specifies which type of console output to generate. Values are 'plain', 'auto' (default) or 'rich'.
--continue              Continues task execution after a task failure.
-D, --system-prop       Set system property of the JVM (e.g. -Dmyprop=myvalue).
-d, --debug             Log in debug mode (includes normal stacktrace).
--daemon                Uses the Gradle Daemon to run the build. Starts the Daemon if not running.
--foreground            Starts the Gradle Daemon in the foreground. [incubating]
-g, --gradle-user-home  Specifies the gradle user home directory.
--gui                   Launches the Gradle GUI.
-I, --init-script       Specifies an initialization script.
-i, --info              Set log level to info.
--include-build         Includes the specified build in the composite. [incubating]
-m, --dry-run           Runs the builds with all task actions disabled.
--max-workers           Configure the number of concurrent workers Gradle is allowed to use. [incubating]
--no-daemon             Do not use the Gradle Daemon to run the build.
--offline               The build should operate without accessing network resources.
-P, --project-prop      Set project property for the build script (e.g. -Pmyprop=myvalue).
-p, --project-dir       Specifies the start directory for Gradle. Defaults to current directory.
--parallel              Build projects in parallel. Gradle will attempt to determine the optimal number of executor threads to use. [incubating]
--profile               Profiles build execution time and generates a report in the <build_dir>/reports/profile directory.
--project-cache-dir     Specifies the project-specific cache directory. Defaults to .gradle in the root project directory.
-q, --quiet             Log errors only.
--recompile-scripts     Force build script recompiling.
--refresh-dependencies  Refresh the state of dependencies.
--rerun-tasks           Ignore previously cached task results.
-S, --full-stacktrace   Print out the full (very verbose) stacktrace for all exceptions.
-s, --stacktrace        Print out the stacktrace for all exceptions.
--status                Shows status of running and recently stopped Gradle Daemon(s).
--stop                  Stops the Gradle Daemon if it is running.
-t, --continuous        Enables continuous build. Gradle does not exit and will re-execute tasks when task file inputs change. [incubating]
-u, --no-search-upward  Don't search in parent folders for a settings.gradle file.
-v, --version           Print version info.
-x, --exclude-task      Specify a task to be excluded from execution.
```

#### 5. 列出项目所有tasks
```bash
$ gradle -q tasks --all

------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------

Build tasks
-----------
assemble - Assembles the outputs of this project.
build - Assembles and tests this project.
buildDependents - Assembles and tests this project and all projects that depend on it.
buildNeeded - Assembles and tests this project and all projects it depends on.
classes - Assembles main classes.
clean - Deletes the build directory.
jar - Assembles a jar archive containing the main classes.
testClasses - Assembles test classes.

Build Setup tasks
-----------------
init - Initializes a new Gradle build. [incubating]
wrapper - Generates Gradle wrapper files. [incubating]

Documentation tasks
-------------------
javadoc - Generates Javadoc API documentation for the main source code.

Help tasks
----------
buildEnvironment - Displays all buildscript dependencies declared in root project 'LearnGradle'.
components - Displays the components produced by root project 'LearnGradle'. [incubating]
dependencies - Displays all dependencies declared in root project 'LearnGradle'.
dependencyInsight - Displays the insight into a specific dependency in root project 'LearnGradle'.
dependentComponents - Displays the dependent components of components in root project 'LearnGradle'. [incubating]
help - Displays a help message.
model - Displays the configuration model of root project 'LearnGradle'. [incubating]
projects - Displays the sub-projects of root project 'LearnGradle'.
properties - Displays the properties of root project 'LearnGradle'.
tasks - Displays the tasks runnable from root project 'LearnGradle'.

IDE tasks
---------
cleanEclipse - Cleans all Eclipse files.
eclipse - Generates all Eclipse files.

Verification tasks
------------------
check - Runs all checks.
test - Runs the unit tests.

Other tasks
-----------
cleanEclipseClasspath
cleanEclipseJdt
cleanEclipseProject
compileJava - Compiles main Java source.
compileTestJava - Compiles test Java source.
eclipseClasspath - Generates the Eclipse classpath file.
eclipseJdt - Generates the Eclipse JDT settings file.
eclipseProject - Generates the Eclipse project file.
helloGradle
processResources - Processes main resources.
processTestResources - Processes test resources.
repeatTasks0
repeatTasks1
repeatTasks2
startChain
taskChainSample

Rules
-----
Pattern: clean<TaskName>: Cleans the output files of a task.
Pattern: build<ConfigurationName>: Assembles the artifacts of a configuration.
Pattern: upload<ConfigurationName>: Assembles and uploads the artifacts belonging to a configuration.
```

#### 6. 查看某个task的详细信息
```bash
$ gradle -q help --task build
Detailed task information for build

Path
     :build

Type
     Task (org.gradle.api.Task)

Description
     Assembles and tests this project.

Group
     build
```

#### 7. 列出项目所有properties
```bash
$ gradle -q properties

------------------------------------------------------------
Root project
------------------------------------------------------------

allprojects: [root project 'LearnGradle']
ant: org.gradle.api.internal.project.DefaultAntBuilder@1f8f3840
antBuilderFactory: org.gradle.api.internal.project.DefaultAntBuilderFactory@1fbffe05
archivesBaseName: LearnGradle
artifacts: org.gradle.api.internal.artifacts.dsl.DefaultArtifactHandler_Decorated@4c645666
asDynamicObject: DynamicObject for root project 'LearnGradle'
assemble: task ':assemble'
attributesSchema: org.gradle.api.internal.attributes.DefaultAttributesSchema_Decorated@2170586e
baseClassLoaderScope: org.gradle.api.internal.initialization.DefaultClassLoaderScope@475b3938
buildDependents: task ':buildDependents'
buildDir: /Users/xyq/workspace/LearnGradle/build
buildFile: /Users/xyq/workspace/LearnGradle/build.gradle
buildNeeded: task ':buildNeeded'
buildScriptSource: org.gradle.groovy.scripts.UriScriptSource@2774b800
buildscript: org.gradle.api.internal.initialization.DefaultScriptHandler@494d1dee
check: task ':check'
childProjects: {}
class: class org.gradle.api.internal.project.DefaultProject_Decorated
classLoaderScope: org.gradle.api.internal.initialization.DefaultClassLoaderScope@6add8f81
classes: task ':classes'
cleanEclipse: task ':cleanEclipse'
cleanEclipseClasspath: task ':cleanEclipseClasspath'
cleanEclipseJdt: task ':cleanEclipseJdt'
cleanEclipseProject: task ':cleanEclipseProject'
compileJava: task ':compileJava'
compileTestJava: task ':compileTestJava'
components: [org.gradle.api.internal.java.JavaLibrary@7451e91b]
configurationActions: org.gradle.configuration.project.DefaultProjectConfigurationActionContainer@76e7eb29
configurations: [configuration ':archives', configuration ':compile', configuration ':compileClasspath', configuration ':compileOnly', configuration ':default', configuration ':runtime', configuration ':testCompile', configuration ':testCompileClasspath', configuration ':testCompileOnly', configuration ':testRuntime']
convention: org.gradle.api.internal.plugins.DefaultConvention@57642a76
defaultArtifacts: org.gradle.api.internal.plugins.DefaultArtifactPublicationSet_Decorated@5143069c
defaultTasks: []
deferredProjectConfiguration: org.gradle.api.internal.project.DeferredProjectConfiguration@2aa1349e
dependencies: org.gradle.api.internal.artifacts.dsl.dependencies.DefaultDependencyHandler_Decorated@63562c40
dependencyCacheDir: /Users/xyq/workspace/LearnGradle/build/dependency-cache
dependencyCacheDirName: dependency-cache
depth: 0
description: null
displayName: root project 'LearnGradle'
distsDir: /Users/xyq/workspace/LearnGradle/build/distributions
distsDirName: distributions
docsDir: /Users/xyq/workspace/LearnGradle/build/docs
docsDirName: docs
eclipse: org.gradle.plugins.ide.eclipse.model.EclipseModel_Decorated@15178dc6
eclipseClasspath: task ':eclipseClasspath'
eclipseJdt: task ':eclipseJdt'
eclipseProject: task ':eclipseProject'
ext: org.gradle.api.internal.plugins.DefaultExtraPropertiesExtension@5ca91e1e
extensions: org.gradle.api.internal.plugins.DefaultConvention@57642a76
fileOperations: org.gradle.api.internal.file.DefaultFileOperations@7f8fc4d1
fileResolver: org.gradle.api.internal.file.BaseDirFileResolver@2d69c34c
gradle: build 'LearnGradle'
group:
helloGradle: task ':helloGradle'
identityPath: :
inheritedScope: org.gradle.api.internal.ExtensibleDynamicObject$InheritedDynamicObject@4b9029a
jar: task ':jar'
javadoc: task ':javadoc'
libsDir: /Users/xyq/workspace/LearnGradle/build/libs
libsDirName: libs
logger: org.gradle.internal.logging.slf4j.OutputEventListenerBackedLogger@5dd205df
logging: org.gradle.internal.logging.services.DefaultLoggingManager@1074a5fd
modelRegistry: org.gradle.model.internal.registry.DefaultModelRegistry@115c426e
modelSchemaStore: org.gradle.model.internal.manage.schema.extract.DefaultModelSchemaStore@2dfbc134
module: org.gradle.api.internal.artifacts.ProjectBackedModule@44f77247
name: LearnGradle
org.gradle.eclipse.postprocess.applied: true
parent: null
parentIdentifier: null
path: :
pluginManager: org.gradle.api.internal.plugins.DefaultPluginManager_Decorated@724c80aa
plugins: [org.gradle.api.plugins.HelpTasksPlugin@42a20621, org.gradle.language.base.plugins.LifecycleBasePlugin@482e65be, org.gradle.api.plugins.BasePlugin@4590f8ff, org.gradle.api.plugins.ReportingBasePlugin@33edb140, org.gradle.platform.base.plugins.ComponentBasePlugin@2d705ad3, org.gradle.language.base.plugins.LanguageBasePlugin@73c9898c, org.gradle.platform.base.plugins.BinaryBasePlugin@338c4dea, org.gradle.api.plugins.JavaBasePlugin@3a1924a5, org.gradle.api.plugins.JavaPlugin@759dc13f, org.gradle.plugins.ide.eclipse.EclipsePlugin@7673da29]
processOperations: org.gradle.api.internal.file.DefaultFileOperations@7f8fc4d1
processResources: task ':processResources'
processTestResources: task ':processTestResources'
project: root project 'LearnGradle'
projectDir: /Users/xyq/workspace/LearnGradle
projectEvaluationBroadcaster: ProjectEvaluationListener broadcast
projectEvaluator: org.gradle.configuration.project.LifecycleProjectEvaluator@3eade1ab
projectPath: :
projectRegistry: org.gradle.api.internal.project.DefaultProjectRegistry@b43e0ce
properties: {...}
repeatTasks0: task ':repeatTasks0'
repeatTasks1: task ':repeatTasks1'
repeatTasks2: task ':repeatTasks2'
reporting: org.gradle.api.reporting.ReportingExtension_Decorated@787f9cfa
reportsDir: /Users/xyq/workspace/LearnGradle/build/reports
repositories: [org.gradle.api.internal.artifacts.repositories.DefaultMavenArtifactRepository_Decorated@6ea00a8f]
resources: org.gradle.api.internal.resources.DefaultResourceHandler@3fd10e49
rootDir: /Users/xyq/workspace/LearnGradle
rootProject: root project 'LearnGradle'
scriptHandlerFactory: org.gradle.api.internal.initialization.DefaultScriptHandlerFactory@2b80dced
scriptPluginFactory: org.gradle.configuration.ScriptPluginFactorySelector@2f9c0d2b
serviceRegistryFactory: org.gradle.internal.service.scopes.ProjectScopeServices$4@41869e25
services: ProjectScopeServices
sourceCompatibility: 1.8
sourceSets: [source set 'main', source set 'test']
standardOutputCapture: org.gradle.internal.logging.services.DefaultLoggingManager@1074a5fd
startChain: task ':startChain'
state: project state 'EXECUTED'
status: integration
subprojects: []
targetCompatibility: 1.8
taskChainSample: task ':taskChainSample'
tasks: [task ':assemble', task ':buildDependents', task ':buildNeeded', task ':check', task ':classes', task ':cleanEclipse', task ':cleanEclipseClasspath', task ':cleanEclipseJdt', task ':cleanEclipseProject', task ':compileJava', task ':compileTestJava', task ':eclipse', task ':eclipseClasspath', task ':eclipseJdt', task ':eclipseProject', task ':helloGradle', task ':jar', task ':javadoc', task ':processResources', task ':processTestResources', task ':properties', task ':repeatTasks0', task ':repeatTasks1', task ':repeatTasks2', task ':startChain', task ':taskChainSample', task ':test', task ':testClasses']
test: task ':test'
testClasses: task ':testClasses'
testReportDir: /Users/xyq/workspace/LearnGradle/build/reports/tests
testReportDirName: tests
testResultsDir: /Users/xyq/workspace/LearnGradle/build/test-results
testResultsDirName: test-results
version: unspecified
```

#### 8. Gradle守护进程
- 开启守护进程

```bash
$ gradle taskChainSample --daemon
:startChain
[ant:echo] Repeat from here...
:repeatTasks0
do repeatTasktask ':repeatTasks0'
:repeatTasks1
do repeatTasktask ':repeatTasks1'
:repeatTasks2
do repeatTasktask ':repeatTasks2'
:taskChainSample

BUILD SUCCESSFUL

Total time: 0.735 secs
```
后续触发的`gradle <task> --daemon`会重用gradle守护进程
使用`gradle <task> --no-daemon`选择不重用守护进程

- 查看守护进程

```bash
$ ps | grep gradle
69403 ttys002    0:00.00 grep gradle
```

- 关闭守护进程

```bash
$ gradle --stop
Stopping Daemon(s)
1 Daemon stopped
```
守护进程会在3小时候自动过期
