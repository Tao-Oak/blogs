## I. Setting up a project
* Command Line
* Eclipse
* Android Studio

## II. Resolve native entry
### i. RegisterNatives
### ii. Resolving Native Method Names
* Java type signature
* native method names


## III. JNI Types and Data Structures

## IV. Native Method Arguments
jclass -> static method    
jobject -> instance method

## V. JNIEXPORT & JNICALL
### i. Symbol Visibility
### ii. Call Convertion

## VI. Calling Java Method from Native Side
你好

```shell
javac -d ./bin ./src/com/example/caotao/jniexample/TestJNI.java
java -cp ./bin -Djava.library.path=./jni com.example.caotao.jniexample.TestJNI

javah -classpath ./src -d ./jni com.example.caotao.jniexample.TestJNI

Exception in thread "main" java.lang.UnsatisfiedLinkError: no helloworld in java.library.path
	at java.lang.ClassLoader.loadLibrary(ClassLoader.java:1867)
	at java.lang.Runtime.loadLibrary0(Runtime.java:870)
	at java.lang.System.loadLibrary(System.java:1122)
	at com.example.caotao.jniexample.TestJNI.<clinit>(TestJNI.java:7)
```

