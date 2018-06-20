This document serves as migration guide for code previously targeted to the Nashorn engine.
See the [JavaInterop.md](JavaInterop.md) for an overview of supported Java interoperability features.

## Migration guide
Both Nashorn and Graal JavaScript support a similar set of syntax and semantics for Java interoperability.
The most important differences relevant for migration are listed here.

### Launcher name `js`
When shipped with GraalVM, Graal JavaScript comes with a binary launcher named `js`.
Note that, depending on the build setup, GraalVM might still ship Nashorn and its `jjs` launcher.

### Fully qualified names
Graal Javascript requires the use of `Java.type(typename)`.
It does not support accessing classes just by their fully qualified class name.
`Java.type` brings more clarity and avoids the accidental use of Java classes in JavaScript code.
Patterns like this:

   var bd = new java.math.BigDecimal('10');

need to be rewritten to:

   var BigDecimal = Java.type('java.math.BigDecimal');
   var bd = new BigDecimal('10');

### Overloading Java functions and argument conversion
Graal JavaScript does not allow lossy conversions of arguments when calling Java methods.
This could lead to severe bugs with numeric values that are hard to detect.

Graal JavaScript will always select the overloaded method with the narrowest possible argument types that can be converted to without loss.
If no such overloaded method is available, Graal JavaScript throws a `TypeError` instead of lossy conversion.
In general, this affects which overloaded method is executed.

### Java Strings have a `length` property, not a `length` function.
Graal JavaScript does not tread the length property of a String specially.
I only provides one canonical way of accessing a String length: by reading the `length` property.

    myJavaString.length;

Nashorn allows to both access `length` as a property and a function.
Existing function calls `length()` need to be rewritten to a property access.

### `ScriptObjectMirror` to represent JavaScript objects in Java
Graal JavaScript does not provide objects of the class `ScriptObjectMirror`.
Instead, JavaScript objects are exposed to Java code as instances of `com.oracle.truffle.api.interop.java.TruffleMap`.
This class implements Java's `Map` interface.

Code referencing `ScriptObjectMirror` instances can be rewritten by changing the type to `Map`.

### Multithreading
Graal JavaScript supports multithreading by creating several `Context` objects from Java code.
Multiple JavaScript engines can be created from a Java application, and can be safely executed in parallel on multiple threads.

   Context polyglot = Context.create();
   Value array = polyglot.eval("js", "[1,2,42,4]");

Graal JavaScript does not allow the creation of threads from JavaScript applications with access to the current `Context`.
This could lead to unmanagable synchronization problems like data races in a language that is not prepared for multithreading.

   new Thread(function() {
      print('printed from another thread'); //FAILS on Graal JavaScript
   }).start();

JavaScript code can create and start threads with `Runnable`s implemented in Java.
The child thread may not access the `Context` of the parent thread or of any other polyglot thread.
In case of violations, an `IllegalStateException` will be thrown.
A child thread may create a new `Context` instance, though.

   new Thread(aJavaRunnable).start(); //allowed on Graal JavaScript

### JavaImporter
The `JavaImporter` feature is currently not supported by Graal JavaScript.

### Java.* methods
Several methods provided by Nashorn on the `Java` global object are currently not supported by Graal JavaScript.
This applies to `Java.extend`, `Java.super`, `Java.isJavaMethod`, `Java.isJavaFunction`, `Java.isScriptFunction`, `Java.isScriptObject`, and `Java.asJSONCompatible`.
We are evaluating their use in real-world applications and might add them in the future, by default or behind a flag.

## Additional aspects to consider

### Console output of Java classes and Java objects
Graal JavaScript provides a `print` builtin function compatible with Nashorn.

Note that Graal JavaScript also provides a `console.log` function.
This is an alias for `print` in pure JavaScript mode, but uses an implementation provided by Node.js when running in Node mode.
Behavior around Java objects differs for `console.log` in Node mode as Node.js does not implement special treatment for such objects.
