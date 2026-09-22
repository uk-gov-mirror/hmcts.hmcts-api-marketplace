# Consumer: Production access

Covers both public-data and protected APIs — they share the same sign-up and application
registration steps, and diverge only at the point of subscribing to a specific API.

```mermaid
sequenceDiagram
    actor Dev as External service owner
    participant Portal as Developer Portal (prod)
    participant Mgmt as Management Plane
    participant PM as Product Manager
    participant Entra as Entra External ID / Entra ID
    participant Gateway as API Gateway
    participant Backend as Backend service

    Dev->>Portal: Sign up (name, email)
    Portal-->>Dev: Account created instantly (Entra External ID, B2B to Entra ID)
    Dev->>Portal: Register client application
    Portal->>Mgmt: Request application registration
    Mgmt->>Entra: Create Client ID + Client Secret
    Entra-->>Mgmt: Client ID + Client Secret
    Mgmt-->>Portal: Client ID + Client Secret
    Portal-->>Dev: Shown once - save it yourself

    alt Public data API
        Note over Dev,Backend: No further approval needed
    else Protected API
        Dev->>Portal: Subscribe to a protected API
        Portal->>PM: Route request for review (email)
        PM->>PM: Review, contact requester off-system if needed
        PM-->>Dev: Data Sharing Agreement
        Dev-->>PM: Accept agreement
        PM->>Mgmt: Approve subscription
        Mgmt-->>Dev: Subscription Key
    end

    rect rgb(245, 246, 250)
    Note over Dev,Backend: At runtime
    Dev->>Entra: Request token (Client ID + Secret)
    Entra-->>Dev: Session token
    Dev->>Gateway: API call + token (+ Subscription Key if protected)
    Gateway->>Entra: Validate token
    Entra-->>Gateway: Valid
    Gateway->>Gateway: Validate Subscription Key is active (if protected)
    Gateway->>Backend: Forward request
    Backend-->>Dev: Response
    end
```
