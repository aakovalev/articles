
= How to shoot yourself in the foot with Lombok and Java 8+

== Strange Error

We were migrating our project to Java 8. I thought we could start from the safe option. We could just compile the project with Java 8 and specify target platform to Java 7. Quick and easy step. I honestly did not expect to have any issues at this stage. So I made a new build and started executing our test suite with Java 7 runtime. 
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
What?! It was weird at the first glance and I was very confused. I looked at the code that wanted _BiFunction_ class and it was _java.uti.Map_. 
My confusion started to grow rapidly. The code was executed under Java 7 run-time. So the code for _java.util.Map_ also had to be from Java 7 but by some reason it wanted classes from Java 8. I reverted JDK version for the build back to Java 7, re-built the project, re-ran the tests - everything became green again…

I took a pause and then decided to look one more time at the stack trace. 
I triangulated after several steps a class which when refered in the code caused this exception to be thrown. The class did not look fancy. It looked pretty much similar to the class I present below: 

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

Again nothing suspicious at the first glance. And nothing should cause _NoClassDefFoundError_. However, class contained _@Delegate_ annotation from https://projectlombok.org[Lombok] library. 

== @Delegate…What wrong can be here? 

What _@Delegate_ annotation does is the following: using byte code generation it inserts into the class methods that correspond to the given interface/class and implements them by delegating to the method invocations at the annotated field…In our example (see class _Foo_ above) annotation _@Delegate_ will add all the methods of _Map<String, String>_ into _Foo_ class. Implementations of these methods will contain invocations of corresponding methods on field _params_, which is a _HashMap<String, String>_ and therefore supports _Map<String,String>_ contract. And here I recalled and realized that there is a difference between _java.util.Map_ interface in Java 7 and Java 8. See the picture below:

[uml]
--
interface Java7.Map<K,V> {
    int size();
    boolean isEmpty()
    boolean containsKey(Object key)
    boolean containsValue(Object value)
    V get(Object key)
    V put(K key, V value)
    V remove(Object key)
    void putAll(Map<? extends K, ? extends V> m)
    void clear()
    Set<K> keySet()
    Collection<V> values()
    Set<Map.Entry<K, V>> entrySet()
    boolean equals(Object o)
    int hashCode()
}

interface Java8.Map<K,V> {
    int size()
    boolean isEmpty()
    boolean containsKey(Object key)
    boolean containsValue(Object value)
    V get(Object key)
    V put(K key, V value)
    V remove(Object key)
    void putAll(Map<? extends K, ? extends V> m)
    void clear()
    Set<K> keySet()
    Collection<V> values()
    Set<Map.Entry<K, V>> entrySet()
    boolean equals(Object o)
    int hashCode()
    V getOrDefault(Object key, V defaultValue) 
    void forEach(BiConsumer<? super K, ? super V> action) 
    void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) 
    V putIfAbsent(K key, V value) 
    boolean remove(Object key, Object value) 
    boolean replace(K key, V oldValue, V newValue) 
    V replace(K key, V value) 
    V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) 
    V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) 
    V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)
    V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction) 
}
--

Java 8 version of _java.util.Map_ interface contains set of new (default) methods that have references to _BiFunction_ class and some other classes from Java 8.
Now everything became clear. What Lombok's _@Delegate_ annotation did when the project was compiled with Java 8 was the following. It honestly generated the code that delegated implementation of _java.util.Map_ interface, to its Java 8 version. And since the code was compiled with Java 8 the generated by Lombok code contained references to the classes and methods of new Java 8 version of Map interface. When the problem class being loaded corresponding methods and types were not in Java 7 run-time. It caused this weird - at first glance - error. 

== Summary

Be careful if you are migrating to newer Java version and your code contains Lombok’s annotation like _@Delegate_. Chances are that you might run into the situation I had. While googling the internet I also found this add-on to the Maven - http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin -
It allows you to detect similar situation while project build. But this is another story and probably topic of the next blog post.

I hope this article will help you to diagnose similar issues and fix them sooner that it took for me.