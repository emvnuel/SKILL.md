# Template Method Pattern - Jakarta EE Cookbook

## Intent

Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

## Jakarta EE Implementation

### 1. Abstract Template Class

```java
public abstract class ReportGenerator {

    // Template method - final to prevent override
    public final Report generate(ReportRequest request) {
        validateRequest(request);
        Data data = fetchData(request);
        Data processedData = processData(data);
        String content = formatContent(processedData);  // Abstract hook
        Report report = createReport(content, request);
        postProcess(report);  // Optional hook
        return report;
    }

    // Common implementation
    protected void validateRequest(ReportRequest request) {
        if (request.getStartDate().isAfter(request.getEndDate())) {
            throw new IllegalArgumentException("Invalid date range");
        }
    }

    protected Data fetchData(ReportRequest request) {
        return dataService.fetch(request.getStartDate(), request.getEndDate());
    }

    protected Data processData(Data data) {
        return data;  // Default: no processing
    }

    // Abstract hook - subclasses must implement
    protected abstract String formatContent(Data data);

    protected Report createReport(String content, ReportRequest request) {
        return new Report(content, request.getTitle());
    }

    // Optional hook - subclasses can override
    protected void postProcess(Report report) {
        // Default: no post-processing
    }
}
```

### 2. Concrete Implementations

```java
@ApplicationScoped
public class PdfReportGenerator extends ReportGenerator {

    @Inject
    PdfService pdfService;

    @Override
    protected String formatContent(Data data) {
        return pdfService.render(data);
    }

    @Override
    protected void postProcess(Report report) {
        report.setContentType("application/pdf");
        report.setExtension("pdf");
    }
}

@ApplicationScoped
public class ExcelReportGenerator extends ReportGenerator {

    @Inject
    ExcelService excelService;

    @Override
    protected String formatContent(Data data) {
        return excelService.render(data);
    }

    @Override
    protected Data processData(Data data) {
        // Excel-specific data transformation
        return data.pivot();
    }

    @Override
    protected void postProcess(Report report) {
        report.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        report.setExtension("xlsx");
    }
}

@ApplicationScoped
public class CsvReportGenerator extends ReportGenerator {

    @Override
    protected String formatContent(Data data) {
        StringBuilder csv = new StringBuilder();
        // Simple CSV formatting
        for (Row row : data.getRows()) {
            csv.append(String.join(",", row.getValues())).append("\n");
        }
        return csv.toString();
    }
}
```

### 3. Data Processing Template

```java
public abstract class DataProcessor<T, R> {

    public final R process(T input) {
        validate(input);
        T prepared = prepare(input);
        R result = doProcess(prepared);
        return finalize(result);
    }

    protected void validate(T input) {
        // Default validation - can be overridden
    }

    protected T prepare(T input) {
        return input;  // Default: no preparation
    }

    protected abstract R doProcess(T input);

    protected R finalize(R result) {
        return result;  // Default: no finalization
    }
}

@ApplicationScoped
public class OrderDataProcessor extends DataProcessor<RawOrderData, ProcessedOrder> {

    @Override
    protected void validate(RawOrderData input) {
        if (input.getItems().isEmpty()) {
            throw new ValidationException("Order must have items");
        }
    }

    @Override
    protected RawOrderData prepare(RawOrderData input) {
        // Sanitize and normalize data
        return input.normalize();
    }

    @Override
    protected ProcessedOrder doProcess(RawOrderData input) {
        // Core processing logic
        return new ProcessedOrder(input);
    }
}
```

## When to Use

✅ **Use Template Method when:**

- Multiple classes share the same algorithm structure
- You want to control extension points in subclasses
- Common behavior should be localized in one class

❌ **Avoid Template Method when:**

- Algorithm varies significantly between implementations
- You need runtime algorithm switching (use Strategy instead)
