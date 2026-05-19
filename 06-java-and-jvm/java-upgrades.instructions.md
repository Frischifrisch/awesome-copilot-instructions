---
description: 'Java upgrade guides: 11→17, 17→21, 21→25'
applyTo: '*'
---


# Java 11 to Java 17 Upgrade Guide

## Project Context

This guide provides comprehensive GitHub Copilot instructions for upgrading Java projects from JDK 11 to JDK 17, covering major language features, API changes, and migration patterns based on 47 JEPs integrated between these versions.

## Language Features and API Changes

### JEP 395: Records (Java 16)

**Migration Pattern**: Convert data classes to records

```java
// Old: Traditional data class
public class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String name() { return name; }
    public int age() { return age; }

    @Override
    public boolean equals(Object obj) { /* boilerplate */ }
    @Override
    public int hashCode() { /* boilerplate */ }
    @Override
    public String toString() { /* boilerplate */ }
}

// New: Record (Java 16+)
public record Person(String name, int age) {
    // Compact constructor for validation
    public Person {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }

    // Custom methods can be added
    public boolean isAdult() {
        return age >= 18;
    }
}
```

### JEP 409: Sealed Classes (Java 17)

**Migration Pattern**: Use sealed classes for restricted inheritance

```java
// New: Sealed class hierarchy
public sealed class Shape
    permits Circle, Rectangle, Triangle {

    public abstract double area();
}

public final class Circle extends Shape {
    private final double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public final class Rectangle extends Shape {
    private final double width, height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;
    }
}

public non-sealed class Triangle extends Shape {
    // Non-sealed allows further inheritance
    private final double base, height;

    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }

    @Override
    public double area() {
        return 0.5 * base * height;
    }
}
```

### JEP 394: Pattern Matching for instanceof (Java 16)

**Migration Pattern**: Simplify instanceof checks

```java
// Old: Traditional instanceof with casting
public String processObject(Object obj) {
    if (obj instanceof String) {
        String str = (String) obj;
        return str.toUpperCase();
    } else if (obj instanceof Integer) {
        Integer num = (Integer) obj;
        return "Number: " + num;
    } else if (obj instanceof List<?>) {
        List<?> list = (List<?>) obj;
        return "List with " + list.size() + " elements";
    }
    return "Unknown type";
}

// New: Pattern matching for instanceof (Java 16+)
public String processObject(Object obj) {
    if (obj instanceof String str) {
        return str.toUpperCase();
    } else if (obj instanceof Integer num) {
        return "Number: " + num;
    } else if (obj instanceof List<?> list) {
        return "List with " + list.size() + " elements";
    }
    return "Unknown type";
}

// Works great with sealed classes
public String describeShape(Shape shape) {
    if (shape instanceof Circle circle) {
        return "Circle with radius " + circle.radius();
    } else if (shape instanceof Rectangle rect) {
        return "Rectangle " + rect.width() + "x" + rect.height();
    } else if (shape instanceof Triangle triangle) {
        return "Triangle with base " + triangle.base();
    }
    return "Unknown shape";
}
```

### JEP 361: Switch Expressions (Java 14)

**Migration Pattern**: Convert switch statements to expressions

```java
// Old: Traditional switch statement
public String getDayType(DayOfWeek day) {
    String result;
    switch (day) {
        case MONDAY:
        case TUESDAY:
        case WEDNESDAY:
        case THURSDAY:
        case FRIDAY:
            result = "Workday";
            break;
        case SATURDAY:
        case SUNDAY:
            result = "Weekend";
            break;
        default:
            throw new IllegalArgumentException("Unknown day: " + day);
    }
    return result;
}

// New: Switch expression (Java 14+)
public String getDayType(DayOfWeek day) {
    return switch (day) {
        case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Workday";
        case SATURDAY, SUNDAY -> "Weekend";
    };
}

// With yield for complex logic
public int calculateScore(Grade grade) {
    return switch (grade) {
        case A -> 100;
        case B -> 85;
        case C -> 70;
        case D -> {
            System.out.println("Consider improvement");
            yield 55;
        }
        case F -> {
            System.out.println("Needs retake");
            yield 0;
        }
    };
}
```

### JEP 406: Pattern Matching for switch (Preview in Java 17)

**Migration Pattern**: Enhanced switch with patterns (Preview feature)

```java
// Requires --enable-preview flag
public String formatValue(Object obj) {
    return switch (obj) {
        case String s -> "String: " + s;
        case Integer i -> "Integer: " + i;
        case null -> "null value";
        case default -> "Unknown: " + obj.getClass().getSimpleName();
    };
}

// With guarded patterns
public String categorizeNumber(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "Negative integer";
        case Integer i when i == 0 -> "Zero";
        case Integer i when i > 0 -> "Positive integer";
        case Double d when d.isNaN() -> "Not a number";
        case Number n -> "Other number: " + n;
        case null -> "null";
        case default -> "Not a number";
    };
}
```

### JEP 378: Text Blocks (Java 15)

**Migration Pattern**: Use text blocks for multi-line strings

```java
// Old: Concatenated strings
String html = "<html>\n" +
              "  <body>\n" +
              "    <h1>Hello World</h1>\n" +
              "    <p>Welcome to Java 17!</p>\n" +
              "  </body>\n" +
              "</html>";

String sql = "SELECT p.id, p.name, p.email, " +
             "       a.street, a.city, a.state " +
             "FROM person p " +
             "JOIN address a ON p.address_id = a.id " +
             "WHERE p.active = true " +
             "ORDER BY p.name";

// New: Text blocks (Java 15+)
String html = """
              <html>
                <body>
                  <h1>Hello World</h1>
                  <p>Welcome to Java 17!</p>
                </body>
              </html>
              """;

String sql = """
             SELECT p.id, p.name, p.email,
                    a.street, a.city, a.state
             FROM person p
             JOIN address a ON p.address_id = a.id
             WHERE p.active = true
             ORDER BY p.name
             """;

// With string interpolation methods
String json = """
              {
                "name": "%s",
                "age": %d,
                "city": "%s"
              }
              """.formatted(name, age, city);
```

### JEP 358: Helpful NullPointerExceptions (Java 14)

**Migration Guidance**: Better NPE debugging (enabled by default in Java 17)

```java
// Old NPE message: "Exception in thread 'main' java.lang.NullPointerException"
// New NPE message shows exactly what was null:
// "Cannot invoke 'String.length()' because the return value of 'Person.getName()' is null"

public class PersonProcessor {
    public void processPersons(List<Person> persons) {
        // This will show exactly which person.getName() returned null
        persons.stream()
            .mapToInt(person -> person.getName().length())  // Clear NPE if getName() returns null
            .sum();
    }

    // Better error messages help with complex expressions
    public void complexExample(Map<String, List<Person>> groups) {
        // NPE will show exactly which part of the chain is null
        int totalNameLength = groups.get("admins")
                                  .get(0)
                                  .getName()
                                  .length();
    }
}
```

### JEP 371: Hidden Classes (Java 15)

**Migration Pattern**: Use for framework and proxy generation

```java
// For frameworks creating dynamic proxies
public class DynamicProxyExample {
    public static <T> T createProxy(Class<T> interfaceClass, InvocationHandler handler) {
        // Hidden classes provide better encapsulation for dynamically generated classes
        MethodHandles.Lookup lookup = MethodHandles.lookup();

        // Framework code would use hidden classes for better isolation
        // This is typically handled by frameworks, not application code
        return interfaceClass.cast(
            Proxy.newProxyInstance(
                interfaceClass.getClassLoader(),
                new Class<?>[]{interfaceClass},
                handler
            )
        );
    }
}
```

### JEP 334: JVM Constants API (Java 12)

**Migration Pattern**: Use for compile-time constants

```java
import java.lang.constant.*;

// For advanced metaprogramming and tooling
public class ConstantExample {
    // Use dynamic constants for computed values
    public static final DynamicConstantDesc<String> COMPUTED_CONSTANT =
        DynamicConstantDesc.of(
            ConstantDescs.BSM_INVOKE,
            "computeValue",
            ConstantDescs.CD_String
        );

    // Primarily used by compiler and framework developers
    public static String computeValue() {
        return "Computed at runtime, cached as constant";
    }
}
```

### JEP 415: Context-Specific Deserialization Filters (Java 17)

**Migration Pattern**: Enhanced security for object deserialization

```java
import java.io.*;

public class SecureDeserialization {
    // Set up deserialization filters for security
    public static void setupSerializationFilters() {
        // Global filter
        ObjectInputFilter globalFilter = ObjectInputFilter.Config.createFilter(
            "java.base/*;java.util.*;!*"
        );
        ObjectInputFilter.Config.setSerialFilter(globalFilter);
    }

    public <T> T deserializeSecurely(byte[] data, Class<T> expectedType) throws IOException, ClassNotFoundException {
        try (ByteArrayInputStream bis = new ByteArrayInputStream(data);
             ObjectInputStream ois = new ObjectInputStream(bis)) {

            // Context-specific filter
            ObjectInputFilter contextFilter = ObjectInputFilter.Config.createFilter(
                expectedType.getName() + ";java.lang.*;!*"
            );
            ois.setObjectInputFilter(contextFilter);

            return expectedType.cast(ois.readObject());
        }
    }
}
```

### JEP 356: Enhanced Pseudo-Random Number Generators (Java 17)

**Migration Pattern**: Use new random generator interfaces

```java
import java.util.random.*;

// Old: Limited Random class
Random oldRandom = new Random();
int oldValue = oldRandom.nextInt(100);

// New: Enhanced random generators (Java 17+)
RandomGenerator generator = RandomGeneratorFactory
    .of("Xoshiro256PlusPlus")
    .create(System.nanoTime());

RandomGenerator.SplittableGenerator splittableGenerator =
    RandomGeneratorFactory.of("L64X128MixRandom").create();

// Better for parallel processing
splittableGenerator.splits(4)
    .parallel()
    .mapToInt(rng -> rng.nextInt(1000))
    .forEach(System.out::println);

// Streamable random values
generator.ints(10, 1, 101)
    .forEach(System.out::println);
```

## I/O and Networking Improvements

### JEP 380: Unix-Domain Socket Channels (Java 16)

**Migration Pattern**: Use Unix domain sockets for local IPC

```java
import java.net.UnixDomainSocketAddress;
import java.nio.channels.*;

// Old: TCP sockets for local communication
// ServerSocketChannel server = ServerSocketChannel.open();
// server.bind(new InetSocketAddress("localhost", 8080));

// New: Unix domain sockets (Java 16+)
public class UnixSocketExample {
    public void createUnixDomainServer() throws IOException {
        Path socketPath = Path.of("/tmp/my-app.socket");
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);

        try (ServerSocketChannel server = ServerSocketChannel.open(StandardProtocolFamily.UNIX)) {
            server.bind(address);

            while (true) {
                try (SocketChannel client = server.accept()) {
                    // Handle client connection
                    handleClient(client);
                }
            }
        }
    }

    public void connectToUnixSocket() throws IOException {
        Path socketPath = Path.of("/tmp/my-app.socket");
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketPath);

        try (SocketChannel client = SocketChannel.open(address)) {
            // Communicate with server
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            client.read(buffer);
        }
    }

    private void handleClient(SocketChannel client) throws IOException {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int bytesRead = client.read(buffer);
        // Process client data
    }
}
```

### JEP 352: Non-Volatile Mapped Byte Buffers (Java 14)

**Migration Pattern**: Use for persistent memory operations

```java
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.StandardOpenOption;

public class PersistentMemoryExample {
    public void usePersistentMemory() throws IOException {
        Path nvmFile = Path.of("/mnt/pmem/data.bin");

        try (FileChannel channel = FileChannel.open(nvmFile,
                StandardOpenOption.READ,
                StandardOpenOption.WRITE,
                StandardOpenOption.CREATE)) {

            // Map as persistent memory
            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_WRITE, 0, 1024,
                ExtendedMapMode.READ_WRITE_SYNC
            );

            // Write data that persists across crashes
            buffer.putLong(0, System.currentTimeMillis());
            buffer.putInt(8, 12345);

            // Force write to persistent storage
            buffer.force();
        }
    }
}
```

## Build System Configuration

### Maven Configuration

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <maven.compiler.release>17</maven.compiler.release>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <release>17</release>
                <!-- Enable preview features if using JEP 406 -->
                <compilerArgs>
                    <arg>--enable-preview</arg>
                </compilerArgs>
            </configuration>
        </plugin>

        <!-- For running tests with preview features -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <argLine>--enable-preview</argLine>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Gradle Configuration

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

tasks.withType<JavaCompile> {
    options.release.set(17)
    // Enable preview features if needed
    options.compilerArgs.addAll(listOf("--enable-preview"))
}

tasks.withType<Test> {
    useJUnitPlatform()
    // Enable preview features for tests
    jvmArgs("--enable-preview")
}
```

## Deprecations and Removals

### JEP 411: Deprecate the Security Manager for Removal

**Migration Pattern**: Remove Security Manager dependencies

```java
// Old: Using Security Manager
SecurityManager sm = System.getSecurityManager();
if (sm != null) {
    sm.checkPermission(new RuntimePermission("shutdownHooks"));
}

// New: Alternative security approaches
// Use application-level security, containers, or process isolation
// Most applications don't need Security Manager functionality
```

### JEP 398: Deprecate the Applet API for Removal

**Migration Pattern**: Migrate from Applets to modern web technologies

```java
// Old: Java Applet (deprecated)
public class MyApplet extends Applet {
    @Override
    public void start() {
        // Applet code
    }
}

// New: Modern alternatives
// 1. Convert to standalone Java application
public class MyApplication extends JFrame {
    public MyApplication() {
        setTitle("My Application");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        // Application code
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            new MyApplication().setVisible(true);
        });
    }
}

// 2. Use Java Web Start alternative (jlink)
// 3. Convert to web application using modern frameworks
```

### JEP 372: Remove the Nashorn JavaScript Engine

**Migration Pattern**: Use alternative JavaScript engines

```java
// Old: Nashorn (removed in Java 17)
// ScriptEngine engine = new ScriptEngineManager().getEngineByName("nashorn");

// New: Alternative approaches
// 1. Use GraalVM JavaScript engine
ScriptEngine engine = new ScriptEngineManager().getEngineByName("graal.js");

// 2. Use external JavaScript execution
ProcessBuilder pb = new ProcessBuilder("node", "script.js");
Process process = pb.start();

// 3. Use web-based approach or embedded browser
```

## JVM and Performance Improvements

### JEP 377: ZGC - A Scalable Low-Latency Garbage Collector (Java 15)

**Migration Pattern**: Enable ZGC for low-latency applications

```bash
# Enable ZGC
-XX:+UseZGC
-XX:+UnlockExperimentalVMOptions  # Not needed in Java 17

# Monitor ZGC performance
-XX:+LogVMOutput
-XX:LogFile=gc.log
```

### JEP 379: Shenandoah - A Low-Pause-Time Garbage Collector (Java 15)

**Migration Pattern**: Enable Shenandoah for consistent latency

```bash
# Enable Shenandoah
-XX:+UseShenandoahGC
-XX:+UnlockExperimentalVMOptions  # Not needed in Java 17

# Shenandoah tuning
-XX:ShenandoahGCHeuristics=adaptive
```

### JEP 341: Default CDS Archives (Java 12) & JEP 350: Dynamic CDS Archives (Java 13)

**Migration Pattern**: Improved startup performance

```bash
# CDS is enabled by default, but you can create custom archives
# Create custom CDS archive
java -XX:DumpLoadedClassList=classes.lst -cp myapp.jar com.example.Main
java -Xshare:dump -XX:SharedClassListFile=classes.lst -XX:SharedArchiveFile=myapp.jsa -cp myapp.jar

# Use custom CDS archive
java -XX:SharedArchiveFile=myapp.jsa -cp myapp.jar com.example.Main
```

## Testing and Migration Strategy

### Phase 1: Foundation (Weeks 1-2)

1. **Update build system**

   - Modify Maven/Gradle configuration for Java 17
   - Update CI/CD pipelines
   - Verify dependency compatibility

2. **Address removals and deprecations**
   - Remove Nashorn JavaScript engine usage
   - Replace deprecated Applet APIs
   - Update Security Manager usage

### Phase 2: Language Features (Weeks 3-4)

1. **Implement Records**

   - Convert data classes to records
   - Add validation in compact constructors
   - Test serialization compatibility

2. **Add Pattern Matching**
   - Convert instanceof chains
   - Implement type-safe casting patterns

### Phase 3: Advanced Features (Weeks 5-6)

1. **Switch Expressions**

   - Convert switch statements to expressions
   - Use new arrow syntax
   - Implement complex yield logic

2. **Text Blocks**
   - Replace concatenated multi-line strings
   - Update SQL and HTML generation
   - Use formatting methods

### Phase 4: Sealed Classes (Weeks 7-8)

1. **Design sealed hierarchies**

   - Identify inheritance restrictions
   - Implement sealed class patterns
   - Combine with pattern matching

2. **Testing and validation**
   - Comprehensive test coverage
   - Performance benchmarking
   - Compatibility verification

## Performance Considerations

### Records vs Traditional Classes

- Records are more memory efficient
- Faster creation and equality checks
- Automatic serialization support
- Consider for data transfer objects

### Pattern Matching Performance

- Eliminates redundant type checks
- Reduces casting overhead
- Better JVM optimization opportunities
- Use with sealed classes for exhaustiveness

### Switch Expressions Optimization

- More efficient bytecode generation
- Better constant folding
- Improved branch prediction
- Use for complex conditional logic

## Best Practices

1. **Use Records for Data Classes**

   - Immutable data containers
   - API data transfer objects
   - Configuration objects

2. **Apply Pattern Matching Strategically**

   - Replace instanceof chains
   - Use with sealed classes
   - Combine with switch expressions

3. **Adopt Text Blocks for Multi-line Content**

   - SQL queries
   - JSON templates
   - HTML content
   - Configuration files

4. **Design with Sealed Classes**

   - Domain modeling
   - State machines
   - Algebraic data types
   - API evolution control

5. **Leverage Enhanced Random Generators**
   - Parallel processing scenarios
   - High-quality random numbers
   - Statistical applications
   - Gaming and simulation

This comprehensive guide enables GitHub Copilot to provide contextually appropriate suggestions when upgrading Java 11 projects to Java 17, focusing on language enhancements, API improvements, and modern Java development practices.

---


# Java 17 to Java 21 Upgrade Guide

These instructions help GitHub Copilot assist developers in upgrading Java projects from JDK 17 to JDK 21, focusing on new language features, API changes, and best practices.

## Major Language Features in JDK 18-21

### Pattern Matching for switch (JEP 441 - Standard in 21)

**Enhanced switch Expressions and Statements**

When working with switch constructs:
- Suggest converting traditional switch to pattern matching where appropriate
- Use pattern matching for type checking and destructuring
- Example upgrade patterns:
```java
// Old approach (Java 17)
public String processObject(Object obj) {
    if (obj instanceof String) {
        String s = (String) obj;
        return s.toUpperCase();
    } else if (obj instanceof Integer) {
        Integer i = (Integer) obj;
        return i.toString();
    }
    return "unknown";
}

// New approach (Java 21)
public String processObject(Object obj) {
    return switch (obj) {
        case String s -> s.toUpperCase();
        case Integer i -> i.toString();
        case null -> "null";
        default -> "unknown";
    };
}
```

- Support guarded patterns:
```java
switch (obj) {
    case String s when s.length() > 10 -> "Long string: " + s;
    case String s -> "Short string: " + s;
    case Integer i when i > 100 -> "Large number: " + i;
    case Integer i -> "Small number: " + i;
    default -> "Other";
}
```

### Record Patterns (JEP 440 - Standard in 21)

**Destructuring Records in Pattern Matching**

When working with records:
- Suggest using record patterns for destructuring
- Combine with switch expressions for powerful data processing
- Example usage:
```java
public record Point(int x, int y) {}
public record ColoredPoint(Point point, Color color) {}

// Destructuring in switch
public String describe(Object obj) {
    return switch (obj) {
        case Point(var x, var y) -> "Point at (" + x + ", " + y + ")";
        case ColoredPoint(Point(var x, var y), var color) -> 
            "Colored point at (" + x + ", " + y + ") in " + color;
        default -> "Unknown shape";
    };
}
```

- Use in complex pattern matching:
```java
// Nested record patterns
switch (shape) {
    case Rectangle(ColoredPoint(Point(var x1, var y1), var c1), 
                   ColoredPoint(Point(var x2, var y2), var c2)) 
        when c1 == c2 -> "Monochrome rectangle";
    case Rectangle r -> "Multi-colored rectangle";
}
```

### Virtual Threads (JEP 444 - Standard in 21)

**Lightweight Concurrency**

When working with concurrency:
- Suggest Virtual Threads for high-throughput, concurrent applications
- Use `Thread.ofVirtual()` for creating virtual threads
- Example migration patterns:
```java
// Old platform thread approach
ExecutorService executor = Executors.newFixedThreadPool(100);
executor.submit(() -> {
    // blocking I/O operation
    httpClient.send(request);
});

// New virtual thread approach
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // blocking I/O operation - now scales to millions
        httpClient.send(request);
    });
}
```

- Use structured concurrency patterns:
```java
// Structured concurrency (Preview)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> fetchUser(userId));
    Future<String> order = scope.fork(() -> fetchOrder(orderId));
    
    scope.join();           // Join all subtasks
    scope.throwIfFailed();  // Propagate errors
    
    return processResults(user.resultNow(), order.resultNow());
}
```

### String Templates (JEP 430 - Preview in 21)

**Safe String Interpolation**

When working with string formatting:
- Suggest String Templates for safe string interpolation (preview feature)
- Enable preview features with `--enable-preview`
- Example usage:
```java
// Traditional concatenation
String message = "Hello, " + name + "! You have " + count + " messages.";

// String Templates (Preview)
String message = STR."Hello, \{name}! You have \{count} messages.";

// Safe HTML generation
String html = HTML."<p>User: \{username}</p>";

// Safe SQL queries  
PreparedStatement stmt = SQL."SELECT * FROM users WHERE id = \{userId}";
```

### Sequenced Collections (JEP 431 - Standard in 21)

**Enhanced Collection Interfaces**

When working with collections:
- Use new `SequencedCollection`, `SequencedSet`, `SequencedMap` interfaces
- Access first/last elements uniformly across collection types
- Example usage:
```java
// New methods available on Lists, Deques, LinkedHashSet, etc.
List<String> list = List.of("first", "middle", "last");
String first = list.getFirst();  // "first"
String last = list.getLast();    // "last"
List<String> reversed = list.reversed(); // ["last", "middle", "first"]

// Works with any SequencedCollection
SequencedSet<String> set = new LinkedHashSet<>();
set.addFirst("start");
set.addLast("end");
String firstElement = set.getFirst();
```

### Unnamed Patterns and Variables (JEP 443 - Preview in 21)

**Simplified Pattern Matching**

When working with pattern matching:
- Use unnamed patterns `_` for values you don't need
- Simplify switch expressions and record patterns
- Example usage:
```java
// Ignore unused variables
switch (ball) {
    case RedBall(_) -> "Red ball";     // Don't care about size
    case BlueBall(var size) -> "Blue ball size " + size;
}

// Ignore parts of records
switch (point) {
    case Point(var x, _) -> "X coordinate: " + x; // Ignore Y
    case ColoredPoint(Point(_, var y), _) -> "Y coordinate: " + y;
}

// Exception handling with unnamed variables
try {
    riskyOperation();
} catch (IOException | SQLException _) {
    // Don't need exception details
    handleError();
}
```

### Scoped Values (JEP 446 - Preview in 21)

**Improved Context Propagation**

When working with thread-local data:
- Consider Scoped Values as a modern alternative to ThreadLocal
- Better performance and clearer semantics for virtual threads
- Example usage:
```java
// Define scoped value
private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();

// Set and use scoped value
ScopedValue.where(USER_ID, "user123")
    .run(() -> {
        processRequest(); // Can access USER_ID.get() anywhere in call chain
    });

// In nested method
public void processRequest() {
    String userId = USER_ID.get(); // "user123"
    // Process with user context
}
```

## API Enhancements and New Features

### UTF-8 by Default (JEP 400 - Standard in 18)

When working with file I/O:
- UTF-8 is now the default charset on all platforms
- Remove explicit charset specifications where UTF-8 was intended
- Example simplification:
```java
// Old explicit UTF-8 specification
Files.readString(path, StandardCharsets.UTF_8);
Files.writeString(path, content, StandardCharsets.UTF_8);

// New default behavior (Java 18+)
Files.readString(path);  // Uses UTF-8 by default
Files.writeString(path, content);  // Uses UTF-8 by default
```

### Simple Web Server (JEP 408 - Standard in 18)

When needing basic HTTP server:
- Use built-in `jwebserver` command or `com.sun.net.httpserver` enhancements
- Great for testing and development
- Example usage:
```java
// Command line
$ jwebserver -p 8080 -d /path/to/files

// Programmatic usage
HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
server.createContext("/", new SimpleFileHandler(Path.of("/tmp")));
server.start();
```

### Internet-Address Resolution SPI (JEP 418 - Standard in 19)

When working with custom DNS resolution:
- Implement `InetAddressResolverProvider` for custom address resolution
- Useful for service discovery and testing scenarios

### Key Encapsulation Mechanism API (JEP 452 - Standard in 21)

When working with post-quantum cryptography:
- Use KEM API for key encapsulation mechanisms
- Example usage:
```java
KeyPairGenerator kpg = KeyPairGenerator.getInstance("ML-KEM");
KeyPair kp = kpg.generateKeyPair();

KEM kem = KEM.getInstance("ML-KEM");
KEM.Encapsulator encapsulator = kem.newEncapsulator(kp.getPublic());
KEM.Encapsulated encapsulated = encapsulator.encapsulate();
```

## Deprecations and Warnings

### Finalization Deprecation (JEP 421 - Deprecated in 18)

When encountering `finalize()` methods:
- Remove finalize methods and use alternatives
- Suggest Cleaner API or try-with-resources
- Example migration:
```java
// Deprecated finalize approach
@Override
protected void finalize() throws Throwable {
    cleanup();
}

// Modern approach with Cleaner
private static final Cleaner CLEANER = Cleaner.create();

public MyResource() {
    cleaner.register(this, new CleanupTask(nativeResource));
}

private static class CleanupTask implements Runnable {
    private final long nativeResource;
    
    CleanupTask(long nativeResource) {
        this.nativeResource = nativeResource;
    }
    
    public void run() {
        cleanup(nativeResource);
    }
}
```

### Dynamic Agent Loading (JEP 451 - Warnings in 21)

When working with agents or instrumentation:
- Add `-XX:+EnableDynamicAgentLoading` to suppress warnings if needed
- Consider loading agents at startup instead of dynamically
- Update tooling to use startup agent loading

## Build Configuration Updates

### Preview Features

For projects using preview features:
- Add `--enable-preview` to compiler and runtime
- Maven configuration:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>21</release>
        <compilerArgs>
            <arg>--enable-preview</arg>
        </compilerArgs>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>--enable-preview</argLine>
    </configuration>
</plugin>
```

- Gradle configuration:
```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

tasks.withType<JavaCompile> {
    options.compilerArgs.add("--enable-preview")
}

tasks.withType<Test> {
    jvmArgs("--enable-preview")
}
```

### Virtual Threads Configuration

For applications using Virtual Threads:
- No special JVM flags required (standard feature in 21)
- Consider these system properties for debugging:
```bash
-Djdk.virtualThreadScheduler.parallelism=N  # Set carrier thread count
-Djdk.virtualThreadScheduler.maxPoolSize=N  # Set max pool size
```

## Runtime and GC Improvements

### Generational ZGC (JEP 439 - Available in 21)

When configuring garbage collection:
- Try Generational ZGC for better performance
- Enable with: `-XX:+UseZGC -XX:+ZGenerational`
- Monitor allocation patterns and GC behavior

## Migration Strategy

### Step-by-Step Upgrade Process

1. **Update Build Tools**: Ensure Maven/Gradle supports JDK 21
2. **Language Feature Adoption**: 
   - Start with pattern matching for switch (standard)
   - Add record patterns where beneficial
   - Consider Virtual Threads for I/O heavy applications
3. **Preview Features**: Enable only if needed for specific use cases
4. **Testing**: Comprehensive testing especially for concurrency changes
5. **Performance**: Benchmark with new GC options

### Code Review Checklist

When reviewing code for Java 21 upgrade:
- [ ] Convert appropriate instanceof chains to switch expressions
- [ ] Use record patterns for data destructuring
- [ ] Replace ThreadLocal with ScopedValues where appropriate
- [ ] Consider Virtual Threads for high-concurrency scenarios
- [ ] Remove explicit UTF-8 charset specifications
- [ ] Replace finalize() methods with Cleaner or try-with-resources
- [ ] Use SequencedCollection methods for first/last access patterns
- [ ] Add preview flags only for preview features in use

### Common Migration Patterns

1. **Switch Enhancement**:
   ```java
   // From instanceof chains to switch expressions
   if (obj instanceof String s) return processString(s);
   else if (obj instanceof Integer i) return processInt(i);
   // becomes:
   return switch (obj) {
       case String s -> processString(s);
       case Integer i -> processInt(i);
       default -> processDefault(obj);
   };
   ```

2. **Virtual Thread Adoption**:
   ```java
   // From platform threads to virtual threads
   Executors.newFixedThreadPool(200)
   // becomes:
   Executors.newVirtualThreadPerTaskExecutor()
   ```

3. **Record Pattern Usage**:
   ```java
   // From manual destructuring to record patterns
   if (point instanceof Point p) {
       int x = p.x();
       int y = p.y();
   }
   // becomes:
   if (point instanceof Point(var x, var y)) {
       // use x and y directly
   }
   ```

## Performance Considerations

- Virtual Threads excel with blocking I/O but may not benefit CPU-intensive tasks
- Generational ZGC can reduce GC overhead for most applications
- Pattern matching in switch is generally more efficient than instanceof chains
- SequencedCollection methods provide O(1) access to first/last elements
- Scoped Values have lower overhead than ThreadLocal for virtual threads

## Testing Recommendations

- Test Virtual Thread applications under high concurrency
- Verify pattern matching covers all expected cases
- Performance test with Generational ZGC vs other collectors
- Validate UTF-8 default behavior across different platforms
- Test preview features thoroughly before production use

Remember to enable preview features only when specifically needed and test thoroughly in staging environments before deploying to production.

---


# Java 21 to Java 25 Upgrade Guide

These instructions help GitHub Copilot assist developers in upgrading Java projects from JDK 21 to JDK 25, focusing on new language features, API changes, and best practices.

## Language Features and API Changes in JDK 22-25

### Pattern Matching Enhancements (JEP 455/488 - Preview in 23)

**Primitive Types in Patterns, instanceof, and switch**

When working with pattern matching:
- Suggest using primitive type patterns in switch expressions and instanceof checks
- Example upgrade from traditional switch:
```java
// Old approach (Java 21)
switch (x.getStatus()) {
    case 0 -> "okay";
    case 1 -> "warning"; 
    case 2 -> "error";
    default -> "unknown status: " + x.getStatus();
}

// New approach (Java 25 Preview)
switch (x.getStatus()) {
    case 0 -> "okay";
    case 1 -> "warning";
    case 2 -> "error"; 
    case int i -> "unknown status: " + i;
}
```

- Enable preview features with `--enable-preview` flag
- Suggest guard patterns for more complex conditions:
```java
switch (x.getYearlyFlights()) {
    case 0 -> ...;
    case int i when i >= 100 -> issueGoldCard();
    case int i -> ... // handle 1-99 range
}
```

### Class-File API (JEP 466/484 - Second Preview in 23, Standard in 25)

**Replacing ASM with Standard API**

When detecting bytecode manipulation or class file processing:
- Suggest migrating from ASM library to the standard Class-File API
- Use `java.lang.classfile` package instead of `org.objectweb.asm`
- Example migration pattern:
```java
// Old ASM approach
ClassReader reader = new ClassReader(classBytes);
ClassWriter writer = new ClassWriter(reader, 0);
// ... ASM manipulation

// New Class-File API approach
ClassModel classModel = ClassFile.of().parse(classBytes);
byte[] newBytes = ClassFile.of().transform(classModel, 
    ClassTransform.transformingMethods(methodTransform));
```

### Markdown Documentation Comments (JEP 467 - Standard in 23)

**JavaDoc Modernization**

When working with JavaDoc comments:
- Suggest converting HTML-heavy JavaDoc to Markdown syntax
- Use `///` for Markdown documentation comments
- Example conversion:
```java
// Old HTML JavaDoc
/**
 * Returns the <b>absolute</b> value of an {@code int} value.
 * <p>
 * If the argument is not negative, return the argument.
 * If the argument is negative, return the negation of the argument.
 * 
 * @param a the argument whose absolute value is to be determined
 * @return the absolute value of the argument
 */

// New Markdown JavaDoc  
/// Returns the **absolute** value of an `int` value.
///
/// If the argument is not negative, return the argument.
/// If the argument is negative, return the negation of the argument.
/// 
/// @param a the argument whose absolute value is to be determined
/// @return the absolute value of the argument
```

### Derived Record Creation (JEP 468 - Preview in 23)

**Record Enhancement**

When working with records:
- Suggest using `with` expressions for creating derived records
- Enable preview features for derived record creation
- Example pattern:
```java
// Instead of manual record copying
public record Person(String name, int age, String email) {
    public Person withAge(int newAge) {
        return new Person(name, newAge, email);
    }
}

// Use derived record creation (Preview)
Person updated = person with { age = 30; };
```

### Stream Gatherers (JEP 473/485 - Second Preview in 23, Standard in 25)

**Enhanced Stream Processing**

When working with complex stream operations:
- Suggest using `Stream.gather()` for custom intermediate operations
- Import `java.util.stream.Gatherers` for built-in gatherers
- Example usage:
```java
// Custom windowing operations
List<List<String>> windows = stream
    .gather(Gatherers.windowSliding(3))
    .toList();

// Custom filtering with state
List<Integer> filtered = numbers.stream()
    .gather(Gatherers.fold(0, (state, element) -> {
        // Custom stateful logic
        return state + element > threshold ? element : null;
    }))
    .filter(Objects::nonNull)
    .toList();
```

## Migration Warnings and Deprecations

### sun.misc.Unsafe Memory Access Methods (JEP 471 - Deprecated in 23)

When detecting `sun.misc.Unsafe` usage:
- Warn about deprecated memory-access methods
- Suggest migration to standard alternatives:
```java
// Deprecated: sun.misc.Unsafe memory access
Unsafe unsafe = Unsafe.getUnsafe();
unsafe.getInt(object, offset);

// Preferred: VarHandle API
VarHandle vh = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "fieldName", int.class);
int value = (int) vh.get(object);

// Or for off-heap: Foreign Function & Memory API
MemorySegment segment = MemorySegment.ofArray(new int[10]);
int value = segment.get(ValueLayout.JAVA_INT, offset);
```

### JNI Usage Warnings (JEP 472 - Warnings in 24)

When detecting JNI usage:
- Warn about upcoming restrictions on JNI usage
- Suggest adding `--enable-native-access` flag for applications using JNI
- Recommend migration to Foreign Function & Memory API where possible
- Add module-info.java entries for native access:
```java
module com.example.app {
    requires jdk.unsupported; // for remaining JNI usage
}
```

## Garbage Collection Updates

### ZGC Generational Mode (JEP 474 - Default in 23)

When configuring garbage collection:
- Default ZGC now uses generational mode
- Update JVM flags if explicitly using non-generational ZGC:
```bash
# Explicit non-generational mode (will show deprecation warning)
-XX:+UseZGC -XX:-ZGenerational

# Default generational mode
-XX:+UseZGC
```

### G1 Improvements (JEP 475 - Implemented in 24)

When using G1GC:
- No code changes required - internal JVM optimization
- May see improved compilation performance with C2 compiler

## Vector API (JEP 469 - Eighth Incubator in 25)

When working with numerical computations:
- Suggest Vector API for SIMD operations (still incubating)
- Add `--add-modules jdk.incubator.vector`
- Example usage:
```java
import jdk.incubator.vector.*;

// Traditional scalar computation
for (int i = 0; i < a.length; i++) {
    c[i] = a[i] + b[i];
}

// Vectorized computation
var species = IntVector.SPECIES_PREFERRED;
for (int i = 0; i < a.length; i += species.length()) {
    var va = IntVector.fromArray(species, a, i);
    var vb = IntVector.fromArray(species, b, i);
    var vc = va.add(vb);
    vc.intoArray(c, i);
}
```

## Compilation and Build Configuration

### Preview Features

For projects using preview features:
- Add `--enable-preview` to compiler arguments
- Add `--enable-preview` to runtime arguments
- Maven configuration:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>25</release>
        <compilerArgs>
            <arg>--enable-preview</arg>
        </compilerArgs>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>--enable-preview</argLine>
    </configuration>
</plugin>
```

- Gradle configuration:
```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

tasks.withType<JavaCompile> {
    options.compilerArgs.add("--enable-preview")
}

tasks.withType<Test> {
    jvmArgs("--enable-preview")
}
```

## Migration Strategy

### Step-by-Step Upgrade Process

1. **Update Build Tools**: Ensure Maven/Gradle supports JDK 25
2. **Update Dependencies**: Check for JDK 25 compatibility
3. **Handle Warnings**: Address deprecation warnings from JEPs 471/472
4. **Enable Preview Features**: If using pattern matching or other preview features
5. **Test Thoroughly**: Especially for applications using JNI or sun.misc.Unsafe
6. **Performance Testing**: Verify GC behavior with new ZGC defaults

### Code Review Checklist

When reviewing code for Java 25 upgrade:
- [ ] Replace ASM usage with Class-File API
- [ ] Convert complex HTML JavaDoc to Markdown
- [ ] Use primitive patterns in switch expressions where applicable
- [ ] Replace sun.misc.Unsafe with VarHandle or FFM API
- [ ] Add native-access permissions for JNI usage
- [ ] Use Stream gatherers for complex stream operations
- [ ] Update build configuration for preview features

### Testing Considerations

- Test with `--enable-preview` flag for preview features
- Verify JNI applications work with native access warnings
- Performance test with new ZGC generational mode
- Validate JavaDoc generation with Markdown comments

## Common Pitfalls

1. **Preview Feature Dependencies**: Don't use preview features in library code without clear documentation
2. **Native Access**: Applications using JNI directly or indirectly may need `--enable-native-access` configuration
3. **Unsafe Migration**: Don't delay migrating from sun.misc.Unsafe - deprecation warnings indicate future removal
4. **Pattern Matching Scope**: Primitive patterns work with all primitive types, not just int
5. **Record Enhancement**: Derived record creation requires preview flag in Java 23

## Performance Considerations

- ZGC generational mode may improve performance for most workloads
- Class-File API reduces ASM-related overhead
- Stream gatherers provide better performance for complex stream operations
- G1GC improvements reduce JIT compilation overhead

Remember to test thoroughly in staging environments before deploying Java 25 upgrades to production systems.