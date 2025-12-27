# Visitor Pattern - Jakarta EE Cookbook

## Intent

Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.

## Jakarta EE Implementation

### 1. Element Interface

```java
public interface DocumentElement {
    void accept(DocumentVisitor visitor);
}
```

### 2. Concrete Elements

```java
public class Paragraph implements DocumentElement {
    private final String text;

    public Paragraph(String text) {
        this.text = text;
    }

    public String getText() { return text; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

public class Image implements DocumentElement {
    private final String url;
    private final String altText;

    public Image(String url, String altText) {
        this.url = url;
        this.altText = altText;
    }

    public String getUrl() { return url; }
    public String getAltText() { return altText; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

public class Table implements DocumentElement {
    private final List<List<String>> rows;

    public Table(List<List<String>> rows) {
        this.rows = rows;
    }

    public List<List<String>> getRows() { return rows; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}
```

### 3. Visitor Interface

```java
public interface DocumentVisitor {
    void visit(Paragraph paragraph);
    void visit(Image image);
    void visit(Table table);
}
```

### 4. Concrete Visitors

```java
@ApplicationScoped
public class HtmlExportVisitor implements DocumentVisitor {

    private final StringBuilder html = new StringBuilder();

    @Override
    public void visit(Paragraph paragraph) {
        html.append("<p>").append(escapeHtml(paragraph.getText())).append("</p>\n");
    }

    @Override
    public void visit(Image image) {
        html.append("<img src=\"").append(image.getUrl())
            .append("\" alt=\"").append(image.getAltText()).append("\" />\n");
    }

    @Override
    public void visit(Table table) {
        html.append("<table>\n");
        for (List<String> row : table.getRows()) {
            html.append("  <tr>\n");
            for (String cell : row) {
                html.append("    <td>").append(escapeHtml(cell)).append("</td>\n");
            }
            html.append("  </tr>\n");
        }
        html.append("</table>\n");
    }

    public String getHtml() { return html.toString(); }

    public void reset() { html.setLength(0); }
}

@ApplicationScoped
public class MarkdownExportVisitor implements DocumentVisitor {

    private final StringBuilder markdown = new StringBuilder();

    @Override
    public void visit(Paragraph paragraph) {
        markdown.append(paragraph.getText()).append("\n\n");
    }

    @Override
    public void visit(Image image) {
        markdown.append("![").append(image.getAltText())
                .append("](").append(image.getUrl()).append(")\n\n");
    }

    @Override
    public void visit(Table table) {
        List<List<String>> rows = table.getRows();
        if (rows.isEmpty()) return;

        // Header
        markdown.append("| ").append(String.join(" | ", rows.get(0))).append(" |\n");
        markdown.append("| ").append("--- | ".repeat(rows.get(0).size())).append("\n");

        // Body
        for (int i = 1; i < rows.size(); i++) {
            markdown.append("| ").append(String.join(" | ", rows.get(i))).append(" |\n");
        }
        markdown.append("\n");
    }

    public String getMarkdown() { return markdown.toString(); }
}

@ApplicationScoped
public class WordCountVisitor implements DocumentVisitor {

    private int wordCount = 0;

    @Override
    public void visit(Paragraph paragraph) {
        wordCount += paragraph.getText().split("\\s+").length;
    }

    @Override
    public void visit(Image image) {
        // Images don't contribute to word count
    }

    @Override
    public void visit(Table table) {
        for (List<String> row : table.getRows()) {
            for (String cell : row) {
                wordCount += cell.split("\\s+").length;
            }
        }
    }

    public int getWordCount() { return wordCount; }
}
```

### 5. Document Service

```java
@ApplicationScoped
public class DocumentService {

    @Inject
    HtmlExportVisitor htmlVisitor;

    @Inject
    MarkdownExportVisitor markdownVisitor;

    @Inject
    WordCountVisitor wordCountVisitor;

    public String exportToHtml(List<DocumentElement> elements) {
        htmlVisitor.reset();
        elements.forEach(e -> e.accept(htmlVisitor));
        return htmlVisitor.getHtml();
    }

    public String exportToMarkdown(List<DocumentElement> elements) {
        markdownVisitor.reset();
        elements.forEach(e -> e.accept(markdownVisitor));
        return markdownVisitor.getMarkdown();
    }

    public int countWords(List<DocumentElement> elements) {
        elements.forEach(e -> e.accept(wordCountVisitor));
        return wordCountVisitor.getWordCount();
    }
}
```

### 6. Usage

```java
@Path("/documents")
@ApplicationScoped
public class DocumentResource {

    @Inject
    DocumentService documentService;

    @GET
    @Path("/{id}/export")
    @Produces(MediaType.TEXT_HTML)
    public String exportHtml(@PathParam("id") Long id) {
        List<DocumentElement> elements = loadDocument(id);
        return documentService.exportToHtml(elements);
    }

    @GET
    @Path("/{id}/stats")
    public DocumentStats getStats(@PathParam("id") Long id) {
        List<DocumentElement> elements = loadDocument(id);
        return new DocumentStats(documentService.countWords(elements));
    }
}
```

## When to Use

✅ **Use Visitor when:**

- You need many distinct operations on an object structure
- Element classes rarely change but operations frequently added
- Operations need to work across unrelated classes

❌ **Avoid Visitor when:**

- Element hierarchy changes frequently (adding new elements requires updating all visitors)
- Operations are simple and few
