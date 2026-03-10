---
layout: post
title: You're up and running!
---

# Running LiveKit on EKS: Designing a Production-Shaped Environment for Exploration

Most real-time products eventually run into the same moment: the point where prototypes stop being enough. Local demos work beautifully, but the moment multiple users connect across networks, NAT traversal enters the conversation, latency becomes visible, and reliability suddenly matters. That’s where systems like **LiveKit** begin to shine.

This post walks through how I designed a **cloud-native LiveKit deployment on AWS**, primarily for **production exploration rather than production usage**. The goal wasn’t to support millions of calls, but to build something **production-shaped**: secure, observable, scalable, and realistic enough that operational characteristics resemble what a real deployment would require. The reference guide from the LiveKit documentation is an excellent starting point:
[https://docs.livekit.io/transport/self-hosting/kubernetes/](https://docs.livekit.io/transport/self-hosting/kubernetes/)

Instead of treating deployment as a checklist, I approached it like a system design problem: what does it take to run a real-time media platform responsibly on the cloud?

---

## Framing the Problem

WebRTC infrastructure has a deceptively simple surface area. At first glance, deploying LiveKit can be as straightforward as running a container and exposing a port. In practice, reliable media delivery requires solving a handful of hard infrastructure problems: NAT traversal, low-latency networking, distributed coordination, TLS everywhere, and predictable scaling under load.

For experimentation and platform exploration, the goal was not maximum scale but **operational fidelity**. I wanted an environment where WebRTC signaling, TURN relay, clustering, and networking behaved exactly as they would in production. That meant introducing components like **Redis for distributed state**, **TURN servers for relay**, **TLS termination**, and **load balancing**, even if the actual traffic levels remained small.

The result is a system deployed on **Amazon EKS**, combining Kubernetes orchestration with AWS networking primitives.

---

## The Architecture at a Glance

At a high level, the system follows a layered architecture separating **networking, compute, storage, and application concerns**. Each layer exists for a specific operational reason, and each decision reflects a trade-off between simplicity and realism.

The core components are:

* **Amazon EKS** running the LiveKit server
* **Application Load Balancer (ALB)** handling HTTPS and WebSocket connections
* **CoTURN** running on a dedicated EC2 instance for media relay
* **ElastiCache Redis** providing distributed state for LiveKit clustering
* **Route53** managing domain-based addressing
* **AWS Certificate Manager (ACM)** providing TLS certificates
* **A dedicated VPC** isolating all infrastructure resources

This mirrors how many real-time systems evolve: a central signaling service surrounded by specialized networking infrastructure that ensures media flows reliably between clients.

---

## Designing the Network Layer

Real-time communication is heavily shaped by network topology. Unlike traditional APIs, WebRTC requires both **control traffic and media traffic** to move efficiently through unpredictable client networks.

The deployment runs inside a custom **VPC**, spanning two availability zones for basic resilience. Public subnets are used across the board. In a typical enterprise architecture this might feel unconventional, but for WebRTC infrastructure it simplifies routing considerably. TURN servers and media ports must be reachable from the public internet, and avoiding NAT gateways eliminates unnecessary cost and complexity.

Security instead relies on **tight security groups**, restricting access to only the ports required for signaling, media, and TURN relay. For LiveKit, this includes the HTTP API port, TCP fallback ports, and a UDP range used for media transmission.

This network layout prioritizes **predictability over abstraction**. When debugging real-time connectivity issues, simpler network paths often make the difference between minutes and hours of troubleshooting.

---

## Compute: Kubernetes for the Application, EC2 for the Network

The compute layer is intentionally split into two distinct environments.

LiveKit itself runs on **Amazon EKS**, which provides the managed Kubernetes control plane. Kubernetes is a natural fit here because LiveKit is stateless and benefits from container orchestration, rolling updates, and horizontal scaling. Even in a small exploratory setup, running the application in Kubernetes keeps the deployment aligned with real production practices.

The TURN server, however, runs separately on a **network-optimized EC2 instance**. TURN traffic can be extremely bandwidth-intensive, and isolating it from the application layer avoids contention between signaling workloads and media relay traffic.

This separation also provides operational flexibility. LiveKit pods can scale independently of TURN capacity, which is particularly useful when testing workloads that involve large relay volumes.

---

## Distributed State with Redis

While a single LiveKit instance can function without external state, clustering requires coordination between nodes. For that, Redis becomes essential.

The system uses **Amazon ElastiCache Redis** as the shared backend for room state, participant tracking, and inter-node signaling. Redis also enables LiveKit to scale horizontally if more pods are added in the future.

Even though the current deployment runs with a single replica, including Redis from the start ensures that scaling later does not require architectural changes. It’s a classic engineering principle: design the system so that the first scale step is incremental, not disruptive.

---

## The Application Layer

Inside the cluster, LiveKit runs as a Kubernetes deployment with resource limits configured to reflect realistic usage patterns. The service is exposed using a **NodePort**, which allows the AWS load balancer to route traffic to Kubernetes nodes without tightly coupling infrastructure layers.

This design intentionally keeps the **load balancing logic outside the cluster**. The AWS Load Balancer Controller watches Kubernetes ingress resources and automatically provisions ALBs when needed.

Operationally, this is one of the most powerful integrations in the AWS ecosystem. Infrastructure remains defined declaratively in Kubernetes, while AWS handles the lifecycle of the underlying load balancer.

---

## TURN: The Unsung Hero of WebRTC

If WebRTC worked perfectly across all networks, TURN servers wouldn’t exist. Unfortunately, corporate firewalls, symmetric NATs, and restricted mobile networks make direct peer-to-peer connections unreliable.

This is where **CoTURN** enters the architecture.

The TURN server provides a relay path for media when direct connectivity fails. It listens on UDP and TCP ports, and also supports **TURN over TLS** for restrictive environments where only HTTPS-like traffic is allowed.

Separating TURN from LiveKit is a deliberate decision. Media relay can become extremely bandwidth-heavy, and isolating it prevents signaling services from being affected during relay spikes.

---

## Domain and TLS Strategy

One subtle but important design decision is **domain-based addressing everywhere**.

LiveKit endpoints, TURN servers, and APIs are exposed through DNS names managed by **Route53**. TLS certificates are issued by **AWS Certificate Manager** and attached to the appropriate components.

This might seem trivial, but it avoids a common deployment pitfall: attempting to run WebRTC infrastructure using raw IP addresses. Modern browsers expect valid TLS certificates tied to domain names, and certificate validation becomes unreliable otherwise.

Using DNS also provides flexibility. Infrastructure can change underneath the system without requiring client updates.

---

## Security Considerations

Security is often overlooked in exploratory environments, but it’s easiest to implement when the architecture is still small.

The deployment applies several baseline practices:

* **TLS everywhere** for signaling and TURN connections
* **IMDSv2** on EC2 nodes to protect metadata access
* **IAM roles** with least-privilege permissions
* **Security groups** restricting access to only required ports

Authentication for LiveKit APIs relies on **JWT tokens generated with API keys and secrets**, allowing clients to authenticate securely when creating rooms or joining sessions.

Even though this environment is exploratory, these patterns mirror how production deployments typically operate.

---

## Understanding the Traffic Flow

Real-time communication systems often appear complex because multiple network flows coexist.

The typical connection lifecycle begins with a client establishing a **secure WebSocket connection** to the LiveKit API through the Application Load Balancer. This channel carries signaling information, including ICE candidates and session negotiation.

Once peers attempt to establish media connections, WebRTC tries direct peer-to-peer communication first. If NAT traversal fails, the TURN server acts as a relay, forwarding encrypted media packets between participants.

Meanwhile, LiveKit nodes coordinate session state through Redis, ensuring that room membership and participant events remain synchronized across instances.

---

## Reliability and Scaling Perspectives

Although the current deployment runs with a single LiveKit pod and node, the architecture is designed to grow.

Scaling the system primarily involves increasing three independent dimensions:

* **LiveKit pods** for signaling capacity
* **EKS nodes** for compute resources
* **TURN servers** for media relay bandwidth

Redis already enables distributed coordination, and the load balancer automatically distributes traffic across nodes. This means scaling becomes mostly an operational adjustment rather than a structural redesign.

Multi-region deployments can also be achieved by replicating the architecture across AWS regions and using DNS-based routing.

---

## Observability and Operations

Any real-time system eventually becomes an operations challenge. Visibility into system health is critical.

Metrics from EKS, ALB, and EC2 flow into **CloudWatch**, providing insight into resource usage and connection patterns. Logs from LiveKit and CoTURN help diagnose connectivity issues, which are common in WebRTC environments.

Health checks at the load balancer and Kubernetes levels ensure that unhealthy nodes are automatically removed from rotation. Even in a small cluster, these guardrails improve resilience during experimentation.

---

## Final Thoughts

Deploying LiveKit on Kubernetes is not particularly difficult. Designing an environment that behaves like a production system is where the real work happens.

This architecture balances **simplicity, realism, and operational clarity**. It introduces the components necessary for reliable real-time communication while keeping the system understandable and easy to evolve.

For engineers exploring WebRTC infrastructure, building a **production-shaped environment early** pays dividends. It forces decisions about networking, security, and observability long before they become urgent problems.

And perhaps most importantly, it transforms experimentation into learning that translates directly into production systems.

---

**Further Reading**

LiveKit Self-Hosting Guide
[https://docs.livekit.io/transport/self-hosting/kubernetes/](https://docs.livekit.io/transport/self-hosting/kubernetes/)

LiveKit Documentation
[https://docs.livekit.io/](https://docs.livekit.io/)

AWS EKS Documentation
[https://docs.aws.amazon.com/eks/](https://docs.aws.amazon.com/eks/)
