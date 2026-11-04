# Kubernetes Gateway API 

## What is Gateway API?

**Gateway API** is a Kubernetes project that provides a standardized and extensible way to manage traffic entering a Kubernetes cluster.

It is a more flexible alternative to the traditional Kubernetes Ingress API.

Both Ingress and Gateway API help route external traffic to Kubernetes Services.

![gateway Output](images/gateway.png)

## 1. How Ingress works

Ingress defines HTTP/HTTPS routing rules, such as routing traffic based on hostname or URL path.

```text
Internet
   |
   v
Ingress Controller (NGINX)
   |
   v
Ingress Resource
   |
   |--- /       → frontend-service
   |
   |--- /api    → backend-service
   |
   |--- /admin  → admin-service
```

Ingress works well for basic traffic routing.

However, advanced features such as canary deployments, traffic splitting, and custom policies often require controller-specific annotations.

This can make migration between different Ingress controllers more difficult.

## 2. What is Gateway API?

Gateway API provides a more structured way to configure Kubernetes traffic management.

Instead of putting everything into one Ingress resource, it separates configuration into different resources.

### Main Gateway API resources

| Resource | Purpose |
|---|---|
| GatewayClass | Defines which Gateway controller manages the Gateway |
| Gateway | Defines the traffic entry point, listeners, ports, and TLS |
| HTTPRoute | Defines HTTP routing rules and backend Services |

## 3. How Gateway API works

```text
Internet
   |
   v
GatewayClass
   |
   v
Gateway Controller
   |
   v
Gateway
   |
   v
HTTPRoute
   |
   |--- /       → frontend-service
   |
   |--- /api    → backend-service
   |
   |--- /admin  → admin-service
```


### Simple explanation

- **GatewayClass:** Which traffic-management implementation should manage the Gateway?
- **Gateway:** Where does traffic enter the cluster?
- **HTTPRoute:** Where should each HTTP request go?
- **Gateway Controller:** The software that implements the configuration and routes traffic.

Gateway API is the overall API framework. GatewayClass, Gateway, and HTTPRoute are resources within that framework.

## 4. Real-world example

Imagine a company running multiple applications in Kubernetes.

The platform team manages the shared Gateway:

- Public IP address
- HTTPS/TLS certificates
- Listeners and ports
- Shared traffic entry point

Application teams manage their own HTTPRoutes:

- `/` → frontend
- `/api` → backend
- `/admin` → admin application

This separates infrastructure management from application routing.

## 5. Ingress vs Gateway API

| Feature | Ingress | Gateway API |
|---|---|---|
| Basic HTTP routing | Yes | Yes |
| Host/path routing | Yes | Yes |
| TLS support | Yes | Yes |
| Advanced traffic management | Often controller-specific | More standardized |
| Traffic splitting | Often controller-specific | Supported by HTTPRoute |
| Role separation | Limited | Better support |
| Portability | Can depend on annotations | Standard resources improve portability |

## 6. Does Gateway API replace Ingress?

No. Ingress is still supported and useful.

Use Ingress when:
- The application needs basic HTTP/HTTPS routing.
- The existing controller meets your requirements.
- You want a simple configuration.

Consider Gateway API when:
- You need advanced traffic management.
- Multiple teams share a traffic entry point.
- You want clearer separation between platform and application configuration.
- You want more standardized routing configuration across implementations.

## 7. Important concept

Gateway API is **not a Gateway Controller**.

It defines Kubernetes resources. A compatible controller is still required to implement them.

Examples include:
- Envoy Gateway
- Istio
- Supported cloud Gateway API implementations

## Interview answer

"Gateway API is a Kubernetes traffic-management API that improves on Ingress by providing a more extensible and standardized model. It separates infrastructure configuration from application routing using GatewayClass, Gateway, and HTTPRoute. This enables better role separation, advanced routing, and improved portability across Gateway implementations. Ingress is still suitable for simpler applications."

## Summary

**Ingress:** A resource that defines basic HTTP/HTTPS routing.

**Gateway API:** A broader framework that separates the traffic entry point from routing rules and provides more flexible traffic management.

Remember:

- GatewayClass = which controller
- Gateway = traffic entry point
- HTTPRoute = routing rules
- Controller = makes it work
