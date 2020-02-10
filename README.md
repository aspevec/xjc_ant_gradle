# xjc_ant_gradle
Repository for simulating bug in XJC implementation

Issue was discovered during upgrade from Java8 to Java11. On project we use Gradle plugins to execute Ant tasks.

Everything worked fine when we called Ant tasks directly but through Gradle plugin it caused the following exception:

```
2020-02-10T15:13:08.149+0100 [DEBUG] [org.gradle.api.internal.project.ant.AntLoggingAdapter] Could not load class (org.apache.tools.ant.taskdefs.Replace) for type replace
....
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] FAILURE: Build failed with an exception.
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * What went wrong:
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] Execution failed for task ':a_generate_new'.
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] > The following error occurred while executing this line:
2020-02-10T15:13:08.175+0100 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]   java.lang.NoClassDefFoundError: org/apache/tools/ant/launch/Launcher
```

We found the problem in XJC Ant task and libraries XJC (2.2.11) and JAXB. After executing XJC it couldn't find Ant classes like Replace any more. 

If we call Replace task before XJC and then again after XJC, it worked. Because ClassLoader had Replace class defined.

Problem was that XJC library and ProtectedTask closes wrong ClassLoader. It closes ant-loader (org.gradle.internal.classloader.VisitableUrlClassLoader) and
this blocks execution of all other Ant tasks which are executed for the first time. Exact scenario as described in the definition of the close method in the URLClassLoader.

```
   Closes this URLClassLoader, so that it can no longer be used to load
   new classes or resources that are defined by this loader.
   Classes and resources defined by any of this loader's parents in the
   delegation hierarchy are still accessible. Also, any classes or resources
   that are already loaded, are still accessible.
```

I believe that it should close the ClassLoader which it creates for it's own execution but somehow it gets the one from Gradle.

We had old group of JAXB and XJC libraries which were used to process .wsdl files (XJC version 2.2.1.1) so we tried with this group of jars for XJC job and it worked perfectly.

You can simulate the case with new libraries this command (this fails):

```
$ ./gradlew --stop
$ ./gradlew a_clean
$ ./gradlew a_generate_new --full-stacktrace --debug
```

Case with old libraries (this works):
```
$ ./gradlew --stop
$ ./gradlew a_clean
$ ./gradlew a_generate_old
```

You can also both of the tasks through Ant and they will both build successfully:

```
$ ant clean
$ ant generate_old
$ ant clean
$ ant generate_new
```

To check jars included in each build please check build.xml and folders lib_new and lib_old.

This is a standard way of including the Ant tasks to Gradle like it's described here: https://docs.gradle.org/current/userguide/ant.html

Since more and more people will upgrade to Java11 and Gradle is widely used, this will cause a lot of problems.
