# @SneakyThrows Annotation

## Intent

Throw checked exceptions without declaring them in the method signature.

## Basic Usage

```java
public class FileReader {
    @SneakyThrows
    public String readFile(String path) {
        return Files.readString(Path.of(path));
        // IOException is thrown without 'throws' declaration
    }
}

// Equivalent to:
public String readFile(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        throw sneakyThrow(e);  // Throws without wrapping
    }
}
```

## Specific Exception Types

```java
public class DataProcessor {
    @SneakyThrows(IOException.class)
    public void processFile(String path) {
        // Only IOException is sneakily thrown
        Files.readAllLines(Path.of(path));
    }

    @SneakyThrows({IOException.class, InterruptedException.class})
    public void asyncProcess(String path) {
        Files.readAllLines(Path.of(path));
        Thread.sleep(1000);
    }
}
```

## Common Use Cases

### Lambda Expressions

```java
public class StreamProcessor {
    @SneakyThrows
    public List<String> readAllFiles(List<Path> paths) {
        return paths.stream()
            .map(this::readContent)  // Would need try-catch without @SneakyThrows
            .collect(Collectors.toList());
    }

    @SneakyThrows
    private String readContent(Path path) {
        return Files.readString(path);
    }
}
```

### Test Methods

```java
@Test
@SneakyThrows
void shouldProcessFile() {
    String content = processor.readFile("test.txt");
    assertThat(content).isNotEmpty();
}
```

## When to Use

✅ **Use `@SneakyThrows` for:**

- Lambdas where try-catch is impossible
- Test methods (exceptions fail the test anyway)
- When you're sure the exception won't occur
- Wrapping checked exceptions you can't handle

❌ **Avoid `@SneakyThrows` for:**

- Production code where exceptions need handling
- API boundaries (callers expect checked exceptions)
- When exception type matters for error handling
- Complex error recovery scenarios

## Comparison with Alternatives

```java
// @SneakyThrows
@SneakyThrows
public String read(Path path) {
    return Files.readString(path);
}

// Manual wrapping
public String read(Path path) {
    try {
        return Files.readString(path);
    } catch (IOException e) {
        throw new RuntimeException(e);  // Wraps exception
    }
}

// Proper handling
public String read(Path path) throws IOException {
    return Files.readString(path);  // Declares exception
}
```

## Pitfalls

| Issue                | Problem                              |
| -------------------- | ------------------------------------ |
| Hidden exceptions    | Callers don't know what to catch     |
| Lost exception type  | Not visible in method signature      |
| Debugging difficulty | Unexpected exception flow            |
| API design           | Violates principle of least surprise |

## Related Annotations

- **@Cleanup**: May throw exceptions on close
