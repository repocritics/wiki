# alibaba/nacos

An open-source platform for dynamic service discovery, configuration management, and service management aimed at cloud-native and microservice systems.

## What it is

Nacos is a server-and-client platform that lets services register themselves, discover one another, and pull configuration that can change at runtime without redeploying. It targets teams building microservices on stacks like Dubbo, Spring Cloud, or Kubernetes who need a central place to manage service instances, health, and configuration. Recent modules also position it as a registry for AI-related artifacts such as MCP servers and agents.

## Key features

- Service registration and discovery over DNS or HTTP interfaces, with real-time health checks to route traffic away from unhealthy instances.
- Dynamic configuration management that pushes configuration changes across environments without application redeployment.
- Weighted routing and DNS-based service resolution for mid-tier load balancing and flow control.
- A service dashboard for managing service metadata, configuration, health, and metrics.
- Documented integrations for Dubbo, Spring Cloud, and Kubernetes services.
- Newer modules for an AI/MCP registry and agent/skill registration (per repo topics and module layout).

## Tech stack

- Java, built with Maven (requires Maven 3.2.5+); the server compiles for Java 17, client artifacts for Java 1.8.
- Current development version `3.3.0-SNAPSHOT` on the `develop` branch.
- Spring Boot dependencies 4.0.6.
- gRPC-java 1.78.0 and protobuf-java 3.25.5 for RPC.
- JRaft 1.4.0 for the consensus/replication layer.
- Pluggable storage via MySQL connector 8.2.0, PostgreSQL 42.7.11, and embedded Derby 10.14.2.0, with HikariCP 3.4.2 pooling.
- Kubernetes Java client 22.0.0 and an MCP dependency at 0.18.2.

## When to reach for it

- Centralizing service discovery and health checking across a microservice fleet.
- Managing runtime configuration for many services and environments from one place.
- Running on the Spring Cloud Alibaba or Dubbo ecosystem where Nacos is a first-class integration.
- Needing DNS-based discovery and weighted routing inside a data center.

## When *not* to reach for it

- Small deployments or single services where a config file or environment variables are enough.
- Teams already standardized on a different discovery and configuration stack with no reason to migrate.
- Non-JVM environments where running and operating a Java server plus its database is undesirable operational overhead.
- Use cases needing only a plain key-value store rather than service-aware discovery and health.

## Maturity signal

With over 33,000 stars, an Apache-2.0 license, a history reaching back to 2018, and commits pushed within the last day, Nacos reads as actively maintained and widely adopted. The move to Java 17, Spring Boot 4, and new AI/MCP registry modules indicates ongoing investment rather than maintenance-only status. It is backed by Alibaba and has a large contributor and user base.

## Alternatives

- Consul — use instead when you want a language-agnostic discovery and key-value store with a strong multi-datacenter and service-mesh story.
- etcd — use instead when you need a strongly consistent key-value store as a building block rather than a full service-management platform.
- Netflix Eureka — use instead for a lighter, discovery-only component in an existing Spring Cloud Netflix stack.
- Apollo — use instead when your need is centered on configuration management and you do not require service discovery.

## Notes

The repository has grown well beyond classic service discovery: module names and repo topics (a2a-registry, ai-registry, mcp-registry, skills) show it is being extended into a registry for MCP servers and AI agents, which is unusual for a project that started as a Dubbo and Spring Cloud service registry.

## Tags

java, service-discovery, configuration-management, microservices, distributed-systems, spring-cloud, kubernetes, devops, grpc, mcp
