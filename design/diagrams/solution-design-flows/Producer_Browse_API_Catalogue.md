# Producer: Browse the catalogue before publishing

```mermaid
sequenceDiagram
    actor Producer as HMCTS Business Product team
    participant Portal as Developer Portal
    participant Mgmt as Management Plane

    Producer->>Portal: Browse the API catalogue
    Producer->>Mgmt: Check for an existing API covering this need
    alt A suitable API already exists
        Mgmt-->>Producer: Reuse it - no new API needed
    else No suitable API exists
        Mgmt-->>Producer: Proceed to define a new API
    end
```
