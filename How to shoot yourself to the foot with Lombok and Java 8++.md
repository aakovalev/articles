
= How to shoot yourself in the foot with Lombok and Java 8+

== Strange Error

We were migrating our project to Java 8. I thought we could start from the safe option. We could just compile the project with Java 8 and specify target platform to Java 7. Quick and easy step. I honestly did not expect to have any issues at this stage. So we made a new build and started executing our test suite with Java 7 runtime. 
Tests were passing one by one but then…Boom! We got a crash. 
The stack trace looked like this:
[source]
Exception in thread "main" java.lang.NoClassDefFoundError: java/util/function/BiFunction
        at java.lang.Class.getDeclaredMethods0(Native Method)
        at java.lang.Class.privateGetDeclaredMethods(Class.java:2625)
        at java.lang.Class.getMethod0(Class.java:2866)
        at java.lang.Class.getMethod(Class.java:1676)
        at sun.launcher.LauncherHelper.getMainMethod(LauncherHelper.java:494)
        at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:486)
Caused by: java.lang.ClassNotFoundException: java.util.function.BiFunction
        at java.net.URLClassLoader$1.run(URLClassLoader.java:359)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:348)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:347)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:315)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 6 more

By some reason our code started to require _BiFunction_ class - which is the part of Java 8.
What?! It was weird at the first glance and I was very confused. I looked at the code that wanted _BiFunction_ class and it was java.uti.map. 
My confusion started to grow rapidly. The code was executed under Java 7 runtime. So the code for java.util.map also had to be from Java 7 but by some reason it wanted classes from Java 8. I reverted JDK version for the build back to Java7, re-built the project, re-ran the tests - everything became green again…

I took a pause and then decided to look one more time at the stack trace. 
I triangulated after several steps a class which when reffered in the code caused this exception to be thrown. The class did not look fancy. It looked pretty much similar to the class I presents below: 

[source,java]
import java.util.HashMap;
import java.util.Map; 
import lombok.experimental.Delegate;
public class Foo implements Map<String, String> {
   @Delegate
   private final Map<String, String> params = new HashMap<>();
   public Foo(String... params) {
      // implementation
   }
   // other methods skipped for brevity
}


Again nothing suspicious at the first glance. And nothing should cause NoClassDefFoundError. However, class contained @Delegate annotation from Lombok library. 

== @Delegate…What wrong can be here? 

What @Delegate annotation does is that it inserts methods into the class that correspond to the given interface/class and implements them by delegating to the method invocations on the annotated field…And here I recalled and realized that there is a difference between java.util.Map interface in Java 7 and Java 8. 
Now everything became clear. What Lombok's @Delegate annotation did when the project was compiled with Java 8 was the following. It honestly generated the code that delegated implementation of java.util.Map interface to the class field (specify class field). Since the code was compiled with Java 8 the generated code contained references to the classes and methods of new Java8 version of Map interface. And when the problem class was executed with Java 7 run-time the corresponding methods & types were not in Java 7 runtime. When the class being loaded the class loader tried to load corresponding types and classes from Java 8 and did not find them - since it was Java 7 runtime. It caused this weird - at first glance - error. 

== Summary

Be careful if you are migrating to newer Java version and your code contains Lombok’s annotation like @Delegate. Chances are that you might run into the situation I had. While googling the internet I also found this add-on to the Maven - http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin -
It allows you to detect similar situation while project build. But this is another story and probably topic of the next blog post.

I hope this article will help you to diagnose similar issues and fix them sooner that it took for me.