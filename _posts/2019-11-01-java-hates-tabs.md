---
layout: post
title: Java hates TABs!
---

Today I Learned one more thing for "tabs versus spaces" debate: Java hates tabs.

Turns out Java compiler counts each tab symbol as 8 spaces while parsing the source code. And when it reports errors it uses the 8-space version of column number.

You don't get this normally when using `javac` from the command line, but all IDEs call Java compiler programmatically to access diagnostic information. So here is a sample code that compiles "Hello world" program containing a bit more tabs than usual

```java
import javax.tools.*;
import javax.tools.JavaCompiler.CompilationTask;
import java.net.URI;
import java.util.Collections;

public class Javac {
  static class MemorySource extends SimpleJavaFileObject {
    final String code;
    MemorySource(String name, String code) {
      super(URI.create("memory:///" + name + Kind.SOURCE.extension), Kind.SOURCE);
      this.code = code;
    }
    @Override
    public CharSequence getCharContent(boolean ignoreEncodingErrors) {
      return code;
    }
  }

  public static void main(String[] args) {
    JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
    DiagnosticCollector<JavaFileObject> diagnostics = new DiagnosticCollector<>();

    MemorySource file = new MemorySource("Hello",
                      "public class Hello {\n" +
                      "\t\t\tpublic static void main(String[] args) {\n" +
                      "\t\t\t\t\t\tSyste.out.println(\"Hello world\");\n " +
                      "\t\t\t}\n" +
                      "}\n");
    CompilationTask task = compiler.getTask(null, null, diagnostics, null, null, Collections.singletonList(file));
    task.call();
    for (Diagnostic diagnostic : diagnostics.getDiagnostics()) {
      System.out.println(diagnostic.getKind() + " on line:" + diagnostic.getLineNumber() + " column:" + diagnostic.getColumnNumber() + " - " + diagnostic.getMessage(null));
      int n = 1;
      for (String line : file.code.split("\n")) {
        System.out.println("Line " + (n++) + " length " + line.length());
      }
    }
  }
}
```

And the output it produces says that error is around column 54 while the longest line of code is just 43 characters long.

```
ERROR on line:3 column:54 - package Syste does not exist
Line 1 length 20
Line 2 length 43
Line 3 length 39
Line 4 length 5
Line 5 length 1
```

One more point in favor of spaces!
