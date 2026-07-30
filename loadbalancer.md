# Load Balancing Fundamentals

This is independent of AWS.

### Why load balancing?

* scalability
* availability
* fault tolerance
* horizontal scaling
* zero downtime

---

### OSI layers

Understand

```
L2
L3
L4
L7
```

especially

* TCP
* UDP
* HTTP
* HTTPS

---

### Types

* L4
* <img width="1024" height="768" alt="image" src="https://github.com/user-attachments/assets/3c8d734a-33ef-46db-95c2-10e23eba7fc9" />
pros: simple and faster. only one tcp connection. just liek a NAT or router.
cons: no smart load balancing algo that uses data and since can't read the data-packet can't change the data like headers. also can't route to different micro services based on the url path (one of the features of api gw)
* L7
* <img width="1024" height="768" alt="image" src="https://github.com/user-attachments/assets/587e3695-e99e-4189-a86e-4ed2f6a2f58c" />

* Reverse Proxy
* Forward Proxy

---

### Algorithms

* Round Robin
* Weighted Round Robin
* Least Connections
* Least Response
* Random
* IP Hash
* Consistent Hashing
* Maglev Hashing (Google)
* Rendezvous Hashing

---

### Session affinity

* Sticky cookies
* Source IP
* Cookie based
* Hash based

---

### Health Checks

* passive
* active
* readiness
* liveness

---

### Failure Handling

* fail open
* fail closed
* circuit breaker
* retries
* connection draining
* graceful shutdown

---

### TLS

* TLS termination
* TLS passthrough
* end-to-end TLS
* mutual TLS

---

### Scaling

How does LB scale?

* horizontal
* regional
* multi-AZ
* anycast
* DNS

---

### HA

* active-active
* active-passive
* quorum
* split brain
* VRRP
* keepalived

---

# Phase 3 — AWS Load Balancers

Understand

### ELB family

* ALB
* NLB
* GWLB
* Classic ELB

---

### Components

* Listener
* Listener Rules
* Target Group
* Target Types

---

### Features

* Cross-zone balancing
* Idle timeout
* Deregistration delay
* Sticky sessions
* Health checks
* SSL certificate
* SNI
* WAF integration

---

### Internal vs Internet-facing

---

### Cost model

---

# Phase 4 — Kubernetes Networking

This is huge.

Pods are not directly reachable.

Understand why.

---

### Pod IP

---

### ClusterIP

---

### NodePort

---

### LoadBalancer Service

---

### ExternalName

---

### Headless Service

---

### kube-proxy

* iptables
* IPVS
* eBPF (Cilium)

---

### Endpoint / EndpointSlice

---

### DNS

CoreDNS

---

### Ingress

* path routing
* host routing
* TLS

---

### Ingress Controllers

* AWS Load Balancer Controller
* NGINX
* Traefik
* HAProxy

---

### Gateway API

The successor to Ingress.

---

# Phase 5 — Service Mesh

* Istio
* Linkerd
* Envoy
* Sidecars

Traffic

```
Pod

↓

Envoy

↓

Envoy

↓

Pod
```

---

### Topics

* mTLS
* retries
* traffic splitting
* canary
* circuit breaking
* rate limiting

---

# Phase 6 — API Gateway

Difference from LB.

* JWT
* OAuth
* API Keys
* Usage Plans
* Throttling
* Caching

---

# Phase 7 — Edge Networking

* CDN
* CloudFront
* Global Accelerator
* Route53
* Geo Routing
* Latency Routing
* Weighted Routing
* Failover Routing

---

# Phase 8 — Production Patterns

These are the things interviewers love.

* Blue/Green
* Canary
* Rolling Update
* Shadow Traffic
* A/B Testing
* Weighted Routing
* Multi-region
* DR
* Active-active
* Active-passive

---

## My recommendation for the learning order

I would slightly reorder Phase 4 so that Kubernetes concepts build naturally on one another:

1. **Load Balancer Fundamentals** (vendor-neutral)
2. **AWS ELB family (ALB/NLB/GWLB)**
3. **Kubernetes Networking Basics** (Pod IP, CNI, kube-proxy, Services)
4. **Ingress & AWS Load Balancer Controller**
5. **Gateway API**
6. **Service Mesh (Envoy/Istio/Linkerd)**
7. **API Gateway**
8. **Global traffic management** (Route 53, CloudFront, Global Accelerator)

The nice thing is that every topic answers the next question. For example:

```
Pods have IPs
        │
        ▼
Why can't clients use Pod IPs?
        │
        ▼
Services (ClusterIP, NodePort, LoadBalancer)
        │
        ▼
How do I expose HTTP nicely?
        │
        ▼
Ingress
        │
        ▼
Who creates the ALB?
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
How is traffic managed inside the cluster?
        │
        ▼
kube-proxy / CNI / EndpointSlices
```
