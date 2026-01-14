<overview>
Cloud-native architecture patterns for Azure applications - microservices, serverless, event-driven, CQRS, saga patterns, and modern cloud design principles for 2024-2025.
</overview>

<microservices_architecture>
## Microservices on Azure

**Definition:** Independently deployable services, each owning its data, communicating via APIs or events.

### When to Use Microservices

<decision_tree>
**Use microservices when:**
- Team size > 10 developers (Conway's Law)
- Need independent deployment of components
- Different scaling requirements per component
- Different tech stacks needed per component
- Long-term product (not short-term project)

**Use monolith when:**
- Team < 5 developers
- MVP or prototype
- Unclear domain boundaries
- Need fast development speed
- Limited ops capacity
</decision_tree>

### Azure Services for Microservices

<service_comparison>
| Pattern | Azure Service | When to Use | Cost |
|---------|---------------|-------------|------|
| HTTP microservices | Azure Container Apps | Modern cloud-native, auto-scale to zero | $$ |
| HTTP microservices | App Service | Simple deployment, enterprise ready | $$ |
| Function-based | Azure Functions | Event-driven, sporadic load | $ |
| Orchestrated containers | AKS | Complex orchestration, existing K8s | $$$ |
| Service mesh | AKS + Istio/Linkerd | Advanced traffic management | $$$$ |
</service_comparison>

### Microservices Communication Patterns

<pattern name="Synchronous (HTTP/gRPC)">
**Best for:** Request-response, immediate result needed

```bicep
// API Gateway pattern with Azure API Management
resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: 'apim-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Developer'
    capacity: 1
  }
  properties: {
    publisherEmail: 'admin@company.com'
    publisherName: 'MyCompany'
  }
}

// Backend microservice
resource orderService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-orderapi-prod-eastus'
  location: 'eastus'
  kind: 'app,linux'
  properties: {
    httpsOnly: true
    serverFarmId: appServicePlan.id
  }
}
```

**Pros:**
- Simple to understand
- Immediate response
- Easy debugging

**Cons:**
- Tight coupling
- Cascading failures
- Network latency accumulates
</pattern>

<pattern name="Asynchronous (Message Queue)">
**Best for:** Decoupling, high throughput, eventual consistency

```bicep
// Service Bus for async messaging
resource serviceBus 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Standard'
  }
}

resource orderQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBus
  name: 'orders'
  properties: {
    maxDeliveryCount: 10
    deadLetteringOnMessageExpiration: true
  }
}
```

**Message flow:**
1. Order service publishes to queue
2. Inventory service subscribes
3. Payment service subscribes
4. Each processes independently

**Pros:**
- Decoupled services
- Better resilience
- Natural load leveling

**Cons:**
- Eventual consistency
- Harder debugging
- Duplicate message handling needed
</pattern>

<pattern name="Event-Driven (Pub/Sub)">
**Best for:** Multiple subscribers, domain events, real-time reactions

```bicep
// Event Grid for event-driven architecture
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-12-15-preview' = {
  name: 'evgt-myapp-prod-eastus'
  location: 'eastus'
}

// Event subscription
resource eventSubscription 'Microsoft.EventGrid/eventSubscriptions@2023-12-15-preview' = {
  name: 'order-created-subscription'
  scope: eventGridTopic
  properties: {
    destination: {
      endpointType: 'WebHook'
      properties: {
        endpointUrl: 'https://app-inventory-prod.azurewebsites.net/webhooks/order-created'
      }
    }
    filter: {
      includedEventTypes: [
        'Order.Created'
      ]
    }
  }
}
```

**Event types:**
- Service Bus Topics (messaging patterns)
- Event Grid (lightweight pub/sub)
- Event Hubs (high-throughput streaming)

**When to use each:**
- **Service Bus Topics:** Guaranteed delivery, complex routing, transactions
- **Event Grid:** React to Azure resource changes, custom events, fan-out
- **Event Hubs:** High volume (millions/sec), streaming analytics, replay
</pattern>
</microservices_architecture>

<api_gateway_pattern>
## API Gateway

**Purpose:** Single entry point for all clients, handles cross-cutting concerns.

### Azure API Management

```bicep
resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: 'apim-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Consumption'  // Pay-per-call, auto-scale
  }
  properties: {
    publisherEmail: 'api@company.com'
    publisherName: 'Company'
  }
}

// API definition
resource api 'Microsoft.ApiManagement/service/apis@2023-05-01-preview' = {
  parent: apim
  name: 'orders-api'
  properties: {
    displayName: 'Orders API'
    path: 'orders'
    protocols: ['https']
    subscriptionRequired: true
  }
}

// Backend
resource backend 'Microsoft.ApiManagement/service/backends@2023-05-01-preview' = {
  parent: apim
  name: 'orders-backend'
  properties: {
    url: 'https://app-orders-prod.azurewebsites.net'
    protocol: 'http'
  }
}

// Policy (rate limiting, auth, transformation)
resource apiPolicy 'Microsoft.ApiManagement/service/apis/policies@2023-05-01-preview' = {
  parent: api
  name: 'policy'
  properties: {
    value: '''
      <policies>
        <inbound>
          <rate-limit calls="100" renewal-period="60" />
          <validate-jwt header-name="Authorization">
            <openid-config url="https://login.microsoftonline.com/..." />
          </validate-jwt>
          <set-backend-service backend-id="orders-backend" />
        </inbound>
        <backend>
          <forward-request />
        </backend>
        <outbound>
          <set-header name="X-Powered-By" exists-action="delete" />
        </outbound>
      </policies>
    '''
    format: 'xml'
  }
}
```

**API Gateway responsibilities:**
- Authentication/authorization
- Rate limiting
- Request/response transformation
- Caching
- Monitoring and analytics
- API versioning
- Circuit breaker

**Alternatives to API Management:**
- Azure Application Gateway (L7 load balancer, simpler)
- Azure Front Door (global, CDN, WAF)
- Nginx/Kong on AKS (self-managed)
</api_gateway_pattern>

<cqrs_pattern>
## CQRS (Command Query Responsibility Segregation)

**Concept:** Separate read and write data models.

### When to Use

**Use CQRS when:**
- Complex business logic in writes
- Very different read and write patterns
- High read-to-write ratio (e.g., 1000:1)
- Need different scaling for reads vs writes
- Event sourcing

**Don't use when:**
- Simple CRUD
- Similar read/write patterns
- Small scale

### Implementation on Azure

```bicep
// Write side - Azure SQL Database (normalized, ACID)
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-orders-prod-eastus'
  location: 'eastus'
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: '...' // From Key Vault
  }
}

resource writeDb 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'orders-write'
  location: 'eastus'
  sku: {
    name: 'GP_S_Gen5'
    tier: 'GeneralPurpose'
    capacity: 2
  }
}

// Read side - Cosmos DB (denormalized, optimized for queries)
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: 'cosmos-orders-prod-eastus'
  location: 'eastus'
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Eventual'  // Okay for read side
    }
  }
}

resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2024-05-15' = {
  parent: cosmosAccount
  name: 'orders-read'
  properties: {
    resource: {
      id: 'orders-read'
    }
  }
}
```

**Synchronization pattern:**

```typescript
// Write side (App Service)
async function createOrder(order: Order) {
  // 1. Write to SQL (source of truth)
  await sqlDb.orders.insert(order);

  // 2. Publish event to Service Bus
  await serviceBus.send('order-created', {
    orderId: order.id,
    customerId: order.customerId,
    total: order.total
  });
}

// Read side projector (Azure Function)
async function projectOrderCreated(event: OrderCreatedEvent) {
  // 3. Update read model in Cosmos DB
  const readModel = {
    id: event.orderId,
    customerName: await getCustomerName(event.customerId),
    total: event.total,
    formattedTotal: formatCurrency(event.total),
    status: 'Pending'
  };

  await cosmosDb.orders.upsert(readModel);
}
```

**Benefits:**
- Optimized read queries (denormalized)
- Scale reads independently
- Complex business logic isolated in write side
- Read models can be eventual consistent

**Drawbacks:**
- Complexity
- Eventual consistency between read/write
- Duplicate data
</cqrs_pattern>

<saga_pattern>
## Saga Pattern (Distributed Transactions)

**Problem:** No distributed transactions across microservices. How to maintain data consistency?

**Solution:** Saga - sequence of local transactions coordinated by events or orchestration.

### Choreography-based Saga

**Pattern:** Each service produces events, next service listens and continues.

```bicep
// Service Bus for saga coordination
resource serviceBus 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-saga-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Standard'
  }
}

resource orderTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: serviceBus
  name: 'order-events'
}
```

**Example: E-commerce order saga**

```
1. Order Service: CreateOrder → publishes "OrderCreated"
2. Payment Service: listens → ProcessPayment → publishes "PaymentProcessed" or "PaymentFailed"
3. Inventory Service: listens → ReserveInventory → publishes "InventoryReserved" or "InventoryUnavailable"
4. Shipping Service: listens → CreateShipment → publishes "ShipmentCreated"

If any step fails, compensating transactions:
- PaymentFailed → Order Service: CancelOrder
- InventoryUnavailable → Payment Service: RefundPayment, Order Service: CancelOrder
```

**Pros:**
- Decoupled services
- Natural event-driven architecture

**Cons:**
- Hard to understand flow
- Difficult debugging
- No central visibility

### Orchestration-based Saga

**Pattern:** Central orchestrator coordinates saga steps.

```bicep
// Durable Functions for saga orchestration
resource orchestratorFunc 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-orderorchestrator-prod-eastus'
  location: 'eastus'
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: functionAppPlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20'
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=...'
        }
      ]
    }
  }
}
```

**Durable Functions orchestration:**

```typescript
// Orchestrator
export async function orderSagaOrchestrator(
  context: df.OrchestrationContext
): Promise<string> {
  const order = context.df.getInput();

  try {
    // Step 1: Create order
    const orderId = await context.df.callActivity('createOrder', order);

    // Step 2: Process payment
    const paymentResult = await context.df.callActivity('processPayment', {
      orderId,
      amount: order.total
    });

    if (!paymentResult.success) {
      // Compensate: Cancel order
      await context.df.callActivity('cancelOrder', orderId);
      return 'Payment failed';
    }

    // Step 3: Reserve inventory
    const inventoryResult = await context.df.callActivity('reserveInventory', {
      orderId,
      items: order.items
    });

    if (!inventoryResult.success) {
      // Compensate: Refund payment, cancel order
      await context.df.callActivity('refundPayment', paymentResult.transactionId);
      await context.df.callActivity('cancelOrder', orderId);
      return 'Inventory unavailable';
    }

    // Step 4: Create shipment
    await context.df.callActivity('createShipment', {
      orderId,
      address: order.shippingAddress
    });

    return 'Order completed successfully';

  } catch (error) {
    // Compensate all steps
    // Implementation here...
    throw error;
  }
}
```

**Pros:**
- Clear flow
- Centralized logic
- Easy debugging
- Built-in compensation

**Cons:**
- Orchestrator is single point of failure (mitigated by Durable Functions persistence)
- Orchestrator can become complex
</saga_pattern>

<strangler_fig_pattern>
## Strangler Fig (Legacy Migration)

**Purpose:** Gradually migrate from monolith to microservices without big-bang rewrite.

### Strategy

```
Monolith (running)
  ↓
Route new features to new microservices
  ↓
Gradually migrate existing features
  ↓
Eventually retire monolith
```

### Implementation with Azure

```bicep
// Application Gateway routes traffic
resource appGateway 'Microsoft.Network/applicationGateways@2023-11-01' = {
  name: 'ag-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    sku: {
      name: 'WAF_v2'
      tier: 'WAF_v2'
    }

    // Backend pools
    backendAddressPools: [
      {
        name: 'monolith-pool'
        properties: {
          backendAddresses: [
            { fqdn: 'app-monolith-prod.azurewebsites.net' }
          ]
        }
      }
      {
        name: 'orders-microservice-pool'
        properties: {
          backendAddresses: [
            { fqdn: 'app-orders-prod.azurewebsites.net' }
          ]
        }
      }
      {
        name: 'products-microservice-pool'
        properties: {
          backendAddresses: [
            { fqdn: 'app-products-prod.azurewebsites.net' }
          ]
        }
      }
    ]

    // Routing rules
    urlPathMaps: [
      {
        name: 'path-based-routing'
        properties: {
          defaultBackendAddressPool: { id: 'monolith-pool' }
          defaultBackendHttpSettings: { id: 'http-settings' }
          pathRules: [
            {
              name: 'orders-rule'
              properties: {
                paths: ['/api/orders/*']
                backendAddressPool: { id: 'orders-microservice-pool' }
              }
            }
            {
              name: 'products-rule'
              properties: {
                paths: ['/api/products/*']
                backendAddressPool: { id: 'products-microservice-pool' }
              }
            }
          ]
        }
      }
    ]
  }
}
```

**Migration steps:**

1. **Week 1-2:** Setup Application Gateway, route all traffic to monolith
2. **Week 3-4:** Extract first microservice (e.g., Orders), deploy alongside monolith
3. **Week 5:** Route `/api/orders/*` to new microservice, monitor
4. **Week 6+:** Repeat for other domains
5. **Month 6:** Remove monolith code that's been extracted
6. **Month 12:** Retire monolith entirely

**Key principle:** Always have working system, never "down for migration"
</strangler_fig_pattern>

<circuit_breaker_pattern>
## Circuit Breaker

**Purpose:** Prevent cascading failures when downstream service fails.

### States

```
CLOSED (normal) → calls pass through
  ↓ (threshold failures)
OPEN (tripped) → fail fast, don't call downstream
  ↓ (timeout)
HALF-OPEN (testing) → allow one request
  ↓ (success) → CLOSED
  ↓ (failure) → OPEN
```

### Implementation

**Using Polly (.NET):**

```csharp
// In App Service or Azure Function
var circuitBreakerPolicy = Policy
  .Handle<HttpRequestException>()
  .CircuitBreakerAsync(
    handledEventsAllowedBeforeBreaking: 3,  // Trip after 3 failures
    durationOfBreak: TimeSpan.FromSeconds(30) // Stay open for 30s
  );

var retryPolicy = Policy
  .Handle<HttpRequestException>()
  .WaitAndRetryAsync(3, retryAttempt =>
    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))
  );

// Combine retry + circuit breaker
var policyWrap = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

// Use
var response = await policyWrap.ExecuteAsync(async () => {
  return await httpClient.GetAsync("https://downstream-service/api");
});
```

**Using Azure API Management:**

```xml
<policies>
  <inbound>
    <base />
    <set-backend-service backend-id="orders-backend" />
  </inbound>
  <backend>
    <retry condition="@(context.Response.StatusCode >= 500)"
           count="3"
           interval="1"
           delta="1">
      <forward-request />
    </retry>
  </backend>
  <on-error>
    <return-response>
      <set-status code="503" reason="Service Unavailable" />
      <set-body>Circuit breaker tripped. Try again later.</set-body>
    </return-response>
  </on-error>
</policies>
```

**Benefits:**
- Fast failure (don't wait for timeout)
- Give downstream service time to recover
- Prevent resource exhaustion
</circuit_breaker_pattern>

<backend_for_frontend>
## Backend for Frontend (BFF)

**Pattern:** Separate backend per client type (web, mobile, IoT).

### Why BFF?

**Problem:** Different clients need different data shapes.
- Web: Full data, multiple requests okay
- Mobile: Minimal data, battery/bandwidth conscious
- IoT: Tiny payloads

**Solution:** Dedicated backend aggregates and shapes data per client.

```bicep
// Web BFF
resource webBff 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-web-bff-prod-eastus'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
  }
}

// Mobile BFF
resource mobileBff 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-mobile-bff-prod-eastus'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```

**Example:**

```typescript
// Web BFF - rich response
GET /api/order/123
{
  "orderId": 123,
  "customer": { /* full customer object */ },
  "items": [ /* full product details */ ],
  "shipping": { /* full address */ },
  "timeline": [ /* order history */ ]
}

// Mobile BFF - minimal response
GET /api/order/123
{
  "orderId": 123,
  "customerName": "John Doe",
  "total": "$99.99",
  "status": "Shipped",
  "eta": "2025-01-15"
}
```

**Trade-off:**
- Pros: Optimized per client, no over-fetching
- Cons: Code duplication, more services to maintain

**Alternative:** GraphQL (client specifies what data to return)
</backend_for_frontend>

<serverless_patterns>
## Serverless Patterns

### Event-driven Processing

```bicep
// Storage account triggers function
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stprocessprodeastus'
  location: 'eastus'
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource funcApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-imageprocess-prod-eastus'
  location: 'eastus'
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: consumptionPlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};...'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'node'
        }
      ]
    }
  }
}
```

**Function code:**

```typescript
// Blob trigger - process uploaded images
export async function processImage(
  blob: Buffer,
  context: InvocationContext
): Promise<void> {
  const blobName = context.triggerMetadata.name;

  // Resize image
  const thumbnail = await sharp(blob)
    .resize(200, 200)
    .toBuffer();

  // Upload to thumbnails container
  await blobService.uploadBlob('thumbnails', blobName, thumbnail);
}
```

**Common serverless patterns:**
- **Blob trigger:** Image processing, file validation, ETL
- **Queue trigger:** Background jobs, async workflows
- **HTTP trigger:** APIs, webhooks
- **Timer trigger:** Scheduled tasks, cleanup jobs
- **Event Grid trigger:** React to Azure resource events
</serverless_patterns>

<best_practices>
## Architecture Best Practices

1. **Design for failure**
   - Assume everything fails
   - Use retry, circuit breaker, timeout
   - Implement health checks

2. **Observability first**
   - Distributed tracing (Application Insights)
   - Correlation IDs across services
   - Structured logging

3. **Eventual consistency**
   - Accept that data will be inconsistent temporarily
   - Design compensating transactions
   - Use idempotent operations

4. **Independently deployable**
   - Each service has own CI/CD pipeline
   - Database per service (or schema per service)
   - Versioned APIs

5. **Security in depth**
   - Managed Identity for service-to-service auth
   - Private Endpoints for data services
   - API Gateway for external traffic
   - Secrets in Key Vault

6. **Cost optimization**
   - Use consumption plans for sporadic load (Functions, Container Apps)
   - Auto-scale based on metrics
   - Use reserved instances for steady-state workloads
</best_practices>
