---
blog:
  publish: true
  slug: forward-proxy-and-reverse-proxy
  title: Forward Proxy and Reverse Proxy
  description: A practical explanation of proxies, forward proxies, reverse proxies, VPNs, load balancers, and ingress controllers.
  date: 2026-06-27
  tags:
    - networking
    - proxies
    - architecture
---

# Forward Proxy and Reverse Proxy

You use proxies more often than it looks.

When a company filters employee web traffic, there is probably a proxy involved. When a website handles millions of users without exposing every backend server to the internet, there is probably a proxy involved. When Kubernetes sends traffic from one public hostname to the right internal service, there is probably a proxy involved there too.

The word "proxy" sounds abstract, but the idea is simple:

> A proxy is something that talks to one side of a connection on behalf of the other side.

Instead of this:

```text
Client -> Server
```

you get this:

```text
Client -> Proxy -> Server
```

That middle position is powerful. Once traffic passes through a proxy, the proxy can route it, filter it, log it, cache it, authenticate it, terminate TLS, or hide details about one side from the other.

The two most common patterns are:

- A forward proxy represents clients.
- A reverse proxy represents servers.

The confusing part is that both are "just proxies." The difference is which side they are helping.

## Forward Proxy

Imagine you are inside a company network and your laptop wants to open a website.

Your laptop does not go directly to the website. Instead, it sends the request to a company-controlled proxy.

```text
Laptop -> Forward Proxy -> Website
```

![Forward proxy traffic flow](https://raw.githubusercontent.com/adi-param/networking-wiki/main/docs/architectures/assets/forward-proxy.svg)

The website sees the proxy making the request. The proxy knows the original laptop, applies company policy, and then decides whether to allow the traffic.

That is a forward proxy.

It sits near the client side and helps clients reach something outside.

## Why Use a Forward Proxy?

### Enterprise Web Filtering

Companies often want a central place to control outbound web access.

```text
Employee Laptop -> Corporate Proxy -> Internet
```

The proxy can block malicious domains, enforce acceptable-use policies, require authentication, and log which users accessed which sites.

Without the proxy, every laptop would make its own direct outbound connections, and the company would have less control over what leaves the network.

### Controlled Egress for Servers

In stricter environments, servers are not allowed to reach the internet directly.

```text
Private Server -> Egress Proxy -> External API
```

This is useful when internal services need limited outbound access, such as downloading packages, calling a SaaS API, or checking for updates.

Instead of giving every server broad internet access, the network allows outbound traffic through a controlled proxy point.

### Source IP or Location Masking

A forward proxy can also make the destination see the proxy's IP address instead of the original client's IP address.

```text
User -> Proxy -> Website
```

From the website's point of view, the request came from the proxy.

That does not mean the user is magically anonymous. The proxy still knows who connected to it, and depending on the protocol, configuration, and logging policy, it may know quite a lot about the traffic.

## Reverse Proxy

Now flip the direction.

You open a website in your browser. You think you are talking to the application, but the first system that receives your request may not be the application server at all.

```text
Browser -> Reverse Proxy -> Application Server
```

![Reverse proxy traffic flow](https://raw.githubusercontent.com/adi-param/networking-wiki/main/docs/architectures/assets/reverse-proxy.svg)

The reverse proxy is standing in front of the servers.

To the browser, the reverse proxy looks like the service. Behind the proxy, there may be one backend server, ten backend servers, multiple services, or an entire Kubernetes cluster.

That is a reverse proxy.

It sits near the server side and helps clients reach the right backend service.

## Why Use a Reverse Proxy?

### Web Application Front Door

A public application often exposes a reverse proxy instead of exposing backend servers directly.

```text
Browser -> Reverse Proxy -> Web App
```

The reverse proxy can handle TLS, route requests, add security headers, compress responses, enforce limits, and keep the backend layout private.

This gives the service a clean public entry point while backend servers remain internal and replaceable.

### Kubernetes Ingress Controller

An ingress controller such as NGINX is one of the clearest reverse proxy examples.

```text
Client -> NGINX Ingress Controller -> Kubernetes Service -> Pod
```

The client connects to the ingress endpoint. NGINX looks at the request and routes it based on rules such as hostname and path.

For example:

```text
app.example.com/api  -> API service
app.example.com/web  -> Web service
```

The client does not need to know which pod is serving the request. It only knows the public endpoint. The ingress controller hides the internal service layout and routes traffic to the right place.

### API Gateway

An API gateway is also commonly a reverse proxy with extra application features.

```text
Client -> API Gateway -> Service A
                    -> Service B
                    -> Service C
```

It may authenticate requests, enforce rate limits, validate tokens, collect metrics, and route each request to the correct backend service.

The backend services do not all need to expose themselves directly to the outside world. The gateway becomes the controlled entry point.

### CDN or Edge Proxy

A CDN is a reverse proxy placed close to users.

```text
User -> CDN Edge -> Origin Server
```

The CDN can cache content, absorb traffic spikes, reduce latency, and protect the origin server from direct exposure.

The user talks to the CDN edge. The CDN talks to the origin when it needs fresh content.

## What About Load Balancers?

This is where terminology gets sloppy.

People often say "load balancer" and "reverse proxy" as if they are the same thing. Sometimes they overlap, but they are not identical.

A layer 4 network load balancer can distribute TCP or UDP connections without understanding the application protocol.

```text
Client -> Network Load Balancer -> Server 1
                              -> Server 2
```

It may only care about IP addresses, ports, and connection flows. That kind of load balancer is not necessarily acting as an application-aware reverse proxy.

A layer 7 application load balancer is much closer to a reverse proxy.

```text
Client -> Application Load Balancer -> /api -> API service
                                  -> /web -> Web service
```

It understands HTTP, terminates client connections, inspects requests, routes by hostname or path, and opens separate connections to backend services.

So the better rule is:

> Some load balancers are reverse proxies, especially layer 7 application load balancers. But not every load balancer is a reverse proxy.

## Forward Proxy vs Reverse Proxy

| Question | Forward Proxy | Reverse Proxy |
| --- | --- | --- |
| Represents | Clients | Servers |
| Sits near | Client side | Server side |
| Typical traffic direction | Outbound from a private network | Inbound to a service |
| Common user | Corporate users, private servers, applications needing controlled egress | Websites, APIs, Kubernetes services, internal applications |
| Common purpose | Filtering, egress control, auditing, source masking | TLS termination, routing, ingress, API gateway behavior, origin protection |
| Other side sees | Proxy as the requester | Proxy as the service endpoint |

## FAQ

### If a proxy hides my source IP or location, how is it different from a VPN?

A proxy usually works for a specific application or protocol.

For example, your browser may send web traffic through an HTTP proxy. In that case, the websites you visit see the proxy as the source. Other traffic from your machine may still use the normal network path.

A VPN usually changes the network path for the device or network. It creates an encrypted tunnel to a VPN endpoint, and traffic is routed through that tunnel depending on the VPN configuration.

In short:

- A proxy usually handles selected application traffic.
- A VPN usually handles broader device or network traffic.
- Both can make the destination see a different source IP.
- Neither automatically guarantees anonymity.

The proxy or VPN provider can still see metadata, and in some cases traffic content, depending on encryption and configuration.

### Is every proxy used for privacy?

No.

Privacy is only one use case. Many proxies are used for control, routing, performance, security, and operational simplicity.

A corporate forward proxy may be about policy and auditing. A reverse proxy in front of an application may be about TLS termination and routing. A CDN may be about caching and latency.

The common idea is not privacy. The common idea is controlled intermediation.

### Is an ingress controller a reverse proxy?

Usually, yes.

An ingress controller receives inbound client traffic and routes it to internal services. If it terminates HTTP or HTTPS and makes routing decisions using hostnames or paths, it is behaving like a reverse proxy.

NGINX Ingress is a good example.

## Simple Rule

Ask who the proxy is helping.

If it helps clients reach outside services, it is usually a forward proxy.

```text
Client -> Forward Proxy -> Internet
```

If it helps outside clients reach your service, it is usually a reverse proxy.

```text
Client -> Reverse Proxy -> Your Service
```

Same idea. Different side of the conversation.

## Related Topics

- HTTP `CONNECT`
- TLS termination
- Load balancing
- NAT
- API gateways
- CDN edge networks
