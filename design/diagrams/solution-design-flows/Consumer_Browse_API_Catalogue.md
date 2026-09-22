# Consumer: Browse the API catalogue

```mermaid
sequenceDiagram
    actor Dev as External developer
    participant Portal as Developer Portal

    Dev->>Portal: Browse the API catalogue
    Dev->>Portal: Read OpenAPI specs and docs
    Note over Dev,Portal: No sign-up required - anyone can read the catalogue
```
