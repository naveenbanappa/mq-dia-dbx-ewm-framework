```mermaid
graph TD;
    A[Start] --> B(Process Data);
    B --> C{Decision};
    C -->|Yes| D[End];
    C -->|No| B;
```