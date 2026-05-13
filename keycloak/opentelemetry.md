  # Keycloak OpenTelemetry Tracing

  Comprehensive distributed tracing implementation for Keycloak using OpenTelemetry.

  This document explains how OpenTelemetry has been configured in Keycloak to generate distributed traces across all system components.

  ---

  ## Table of Contents

  - [Overview](#overview)
  - [Quick Start](#quick-start)
  - [Configuration](#configuration)
  - [Architecture Deep Dive](#architecture-deep-dive)
  - [Usage Patterns](#usage-patterns)
  - [Runtime Behavior](#runtime-behavior)
  - [Debugging and Testing](#debugging-and-testing)
  - [Components Coverage](#components-coverage)

  ---

  ## Overview

  OpenTelemetry tracing in Keycloak provides end-to-end observability across authentication flows, cache operations, database queries, and cluster
  communications.

  ### Architecture Layers

  The tracing system is organized into five distinct layers:

  1. **Configuration Layer** - Quarkus-based configuration options that control tracing behavior
  2. **Provider SPI Layer** - Service Provider Interface (TracingProvider) that abstracts tracing operations
  3. **Implementation Layer** - OTelTracingProvider that wraps OpenTelemetry SDK
  4. **Integration Layer** - Specialized integrations for HTTP, JDBC, Infinispan, and JGroups
  5. **Auto-instrumentation Layer** - REST endpoint instrumentation via Quarkus extensions

  ### Key Design Principles

  - **Feature-gated**: Controlled by Profile.Feature.OPENTELEMETRY feature flag
  - **Provider pattern**: Swappable between active (OTelTracingProvider) and no-op (NoopTracingProvider) implementations
  - **Context propagation**: Uses W3C Trace Context for distributed tracing across services
  - **Lifecycle management**: Automatic span lifecycle with scoped contexts
  - **Zero overhead**: No performance penalty when disabled via no-op implementation

  ---

  ## Quick Start

  ### Enable Tracing

  Build with tracing enabled (required, build-time option):

  ```bash
  ./mvnw clean install -Poperator -DskipTests --tracing-enabled=true

  Start server with tracing:

  java -jar quarkus/server/target/lib/quarkus-run.jar start-dev \
    --tracing-enabled=true \
    --tracing-endpoint=http://localhost:4317 \
    --tracing-protocol=grpc

  Verify Tracing is Active

  Enable debug logging:

  --log-level=org.keycloak.quarkus.runtime.tracing:debug

  You should see log entries like:

  Start span 'ClassName.methodName' (spanId: '1234567890abcdef')
  End span 'ClassName.methodName' (spanId: '1234567890abcdef')
  ```bash
  
  ---
  Configuration

  Available Options

  All options are prefixed with tracing-*:

  Core Configuration:

  - tracing-enabled (Boolean, default: false, build-time)
    - Enable OpenTelemetry tracing
  - tracing-endpoint (String, default: http://localhost:4317)
    - OTLP endpoint for exporting traces
  - tracing-service-name (String, default: keycloak)
    - Service name in traces
  - tracing-protocol (String, default: grpc)
    - Protocol: grpc or http/protobuf

  Sampling Configuration:

  - tracing-sampler-type (String, default: traceidratio, build-time)
    - Sampler type (see Quarkus docs)
  - tracing-sampler-ratio (Double, default: 1.0)
    - Sampling ratio between 0.0 and 1.0

  Advanced Options:

  - tracing-compression (Enum, default: none)
    - Compression: none or gzip
  - tracing-resource-attributes (List, default: none)
    - Resource attributes in format: key1=val1,key2=val2

  Sub-system Tracing:

  - tracing-jdbc-enabled (Boolean, default: true, build-time)
    - Enable JDBC tracing
  - tracing-infinispan-enabled (Boolean, default: true)
    - Enable Infinispan cache tracing

  Note: Build-time options (marked in bold) require a rebuild when changed. Other options can be changed at runtime.

  Example Configurations

  Production Configuration (with Jaeger):

  ./mvnw clean install --tracing-enabled=true

  java -jar quarkus/server/target/lib/quarkus-run.jar start \
    --tracing-enabled=true \
    --tracing-endpoint=http://jaeger:4317 \
    --tracing-service-name=keycloak-prod \
    --tracing-protocol=grpc \
    --tracing-sampler-ratio=0.1 \
    --tracing-compression=gzip \
    --tracing-resource-attributes=environment=production,cluster=us-east-1

  Development Configuration (100% sampling):

  ./mvnw clean install --tracing-enabled=true

  java -jar quarkus/server/target/lib/quarkus-run.jar start-dev \
    --tracing-enabled=true \
    --tracing-endpoint=http://localhost:4317 \
    --tracing-sampler-ratio=1.0 \
    --log-level=org.keycloak.quarkus.runtime.tracing:debug

  ---
  Architecture Deep Dive

  1. Configuration Layer

  Location: quarkus/config-api/src/main/java/org/keycloak/config/TracingOptions.java

  Defines all tracing configuration options using Keycloak's option builder pattern. Each option specifies:
  - Category (TRACING)
  - Description
  - Default value
  - Whether it's a build-time or runtime option

  Key characteristics:
  - Integrates with Quarkus OpenTelemetry extension
  - Build-time vs runtime option separation
  - Type-safe configuration with validation
  - Default values optimized for development

  ---
  2. Provider SPI Layer

  Location: server-spi-private/src/main/java/org/keycloak/tracing/TracingProvider.java

  The TracingProvider interface abstracts all tracing operations, making it swappable and testable.

  Core API Methods:

  - Span getCurrentSpan() - Get current active span
  - Span startSpan(String tracerName, String spanName) - Manual span lifecycle (low-level API)
  - Span startSpan(Class<?> tracerClass, String spanSuffix) - Class-based span creation
  - void endSpan() - End the current span
  - void trace(Class<?> tracerClass, String spanSuffix, Consumer<Span> execution) - Automatic lifecycle (PREFERRED)
  - <T> T trace(Class<?> tracerClass, String spanSuffix, Function<Span, T> execution) - Automatic lifecycle with return value
  - void error(Throwable exception) - Record exception
  - Tracer getTracer(String name, String scopeVersion) - Get tracer instance

  Design Patterns:

  - Automatic lifecycle: Preferred for most use cases - trace(MyClass.class, "operation", span -> {...})
  - Manual lifecycle: For complex control flows - startSpan() → try/finally → endSpan()
  - Class-based naming: Consistent span naming - Span name: ClassName.operation
  - Error recording: Exception tracking - error(exception) adds full context

  ---
  3. Implementation Layer

  3.1 Active Implementation: OTelTracingProvider

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProvider.java

  The production implementation wrapping OpenTelemetry SDK with these key features:

  - Scope management: Maintains a ConcurrentLinkedDeque of Scope for nested spans
  - Context propagation: Uses Span.makeCurrent() to propagate context
  - Error recording: Follows OTel Semantic Conventions for exceptions
  - Debug logging: Logs span start/end with span names and IDs
  - Validation: Detects unclosed spans (memory leak prevention)

  The provider maintains a stack of scopes to handle nested spans correctly. When a span is started, its made current in the thread-local context. When ended,
   the scope is closed and the previous context is restored.

  3.2 Factory: OTelTracingProviderFactory

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProviderFactory.java

  Provider lifecycle management and activation logic. The factory:

  1. Checks if OPENTELEMETRY feature is enabled
  2. Checks if tracing-enabled=true in configuration
  3. If both conditions are met, creates OTelTracingProvider
  4. Otherwise, returns NoopTracingProvider
  5. Retrieves OpenTelemetry instance from Quarkus CDI container

  3.3 Fallback: NoopTracingProvider

  Location: server-spi-private/src/main/java/org/keycloak/tracing/NoopTracingProvider.java

  Zero-overhead implementation when tracing is disabled. Returns invalid spans for all operations and executes traced code blocks without instrumentation.
  Ensures no performance penalty when tracing is disabled.

  ---
  4. Integration Layer

  4.1 REST Endpoint Auto-Instrumentation

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/integration/resteasy/KeycloakTracingCustomizer.java

  Automatically creates spans for every JAX-RS resource method invocation by hooking into RESTEasy Reactives handler chain.

  How it works:

  - StartHandler: Executed BEFORE method invocation
    - Retrieves OpenTelemetry from CDI
    - Creates span with name: ClassName.methodName
    - Adds attributes: code.function (method name) and code.namespace (class name)
    - Stores span and scope in request context
  - EndHandler: Executed AFTER method invocation
    - Closes the scope
    - Ends the span

  The customizer automatically inherits parent context from HTTP headers (Quarkus handles this automatically).

  4.2 JDBC Tracing

  Dependency: io.opentelemetry.instrumentation:opentelemetry-jdbc
  Configuration: tracing-jdbc-enabled=true (default, build-time)

  OpenTelemetry JDBC instrumentation wraps JDBC drivers and automatically creates spans for SQL queries. No code changes required.

  Span Information:
  - Operation: SELECT, INSERT, UPDATE, DELETE
  - Database system, connection string (sanitized)
  - Query duration

  4.3 HTTP Client Tracing

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelHttpClientBuilder.java

  Instruments outbound HTTP requests (OIDC, SAML, etc.) by wrapping Apache HttpClient with OpenTelemetry instrumentation.

  Capabilities:
  - Creates spans for all outbound HTTP calls
  - Propagates trace context via traceparent HTTP header (W3C)
  - Captures HTTP status codes, URLs, methods
  - Used for all Keycloak external communications

  4.4 Infinispan Cache Tracing

  Location: model/infinispan/src/main/java/org/keycloak/infinispan/module/factory/OpenTelemetryService.java

  Traces embedded Infinispan cache operations including:
  - Cache get, put, remove, clear
  - Cache creation/destruction
  - Cluster-wide cache operations
  
  Configuration: tracing-infinispan-enabled=true (default, runtime)

  Span Attributes:
  - cache - Cache name (e.g., sessions, users)
  - category - Operation category
  - server.address - Node name in cluster

  4.5 JGroups Cluster Tracing

  Location: model/infinispan/src/main/java/org/keycloak/jgroups/protocol/OPEN_TELEMETRY.java

  Traces cluster communication between Keycloak nodes. This is a JGroups protocol that must be configured in the JGroups stack (just above transport).

  Key Features:
  - Distributed tracing across cluster nodes
  - W3C Trace Context propagation via JGroups headers
  - Single message and batched message support
  - Source/destination node tracking
  - Enables end-to-end tracing in clustered deployments

  How it works:

  When sending a message:
  1. Creates a span named "JGroups.sendSingleMessage"
  2. Adds source and destination attributes
  3. Injects W3C Trace Context into message header
  4. Sends the message

  When receiving a message:
  1. Extracts parent context from message header
  2. Creates a child span named "JGroups.deliverSingleMessage"
  3. Sets span kind to SERVER
  4. Processes the message with proper context

  ---
  Usage Patterns

  Basic Usage

  Example 1: Simple Tracing Block (Preferred)

  class UserService {
      private final TracingProvider tracing;

      UserService(KeycloakSession session) {
          this.tracing = session.getProvider(TracingProvider.class);
      }

      void createUser(UserModel user) {
          tracing.trace(UserService.class, "createUser", span -> {
              validateUser(user);
              persistUser(user);
              sendWelcomeEmail(user);

              span.setAttribute("user.id", user.getId());
              span.setAttribute("user.realm", user.getRealm());
          });
      }
  }

  Resulting span name: UserService.createUser

  Example 2: Returning Values

  class TokenService {
      private final TracingProvider tracing;

      String generateToken(ClientModel client) {
          return tracing.trace(TokenService.class, "generateToken", span -> {
              String token = createJWT(client);

              span.setAttribute("token.type", "access");
              span.setAttribute("token.length", token.length());
              span.setAttribute("client.id", client.getClientId());

              return token;
          });
      }
  }

  Advanced Usage

  Example 3: Manual Span Management

  Use when you need fine-grained control (try-finally required):

  void complexAuthFlow() {
      Span span = tracing.startSpan(AuthenticationFlow.class, "complexAuth");
      try {
          validateCredentials();
          span.setAttribute("step.credentials", "valid");

          checkMFA();
          span.setAttribute("step.mfa", "passed");

          createSession();
          span.setAttribute("step.session", "created");

      } catch (AuthenticationException e) {
          tracing.error(e);
          throw e;
      } finally {
          tracing.endSpan();
      }
  }

  Example 4: Enriching Current Span

  void processRequest(HttpRequest request) {
      Span currentSpan = tracing.getCurrentSpan();

      currentSpan.setAttribute("http.user_agent", request.getHeader("User-Agent"));
      currentSpan.setAttribute("client.ip", request.getRemoteAddr());
      currentSpan.setAttribute("request.id", UUID.randomUUID().toString());
  }

  Example 5: Nested Spans

  void parentOperation() {
      tracing.trace(MyService.class, "parentOp", parentSpan -> {

          tracing.trace(MyService.class, "childOp1", childSpan1 -> {
              doWork1();
          });

          tracing.trace(MyService.class, "childOp2", childSpan2 -> {
              doWork2();
          });

      });
  }

  Trace structure:
  MyService.parentOp
  ├── MyService.childOp1
  └── MyService.childOp2

  Real-World Example from Codebase

  Location: services/src/main/java/org/keycloak/services/DefaultKeycloakContext.java:245

  public class DefaultKeycloakContext implements KeycloakContext {
      private KeycloakSession session;

      @Override
      public Span getCurrentTracingSpan() {
          return session.getProvider(TracingProvider.class).getCurrentSpan();
      }
  }

  This allows any component to access the current span via KeycloakContext.

  ---
  Runtime Behavior

  Activation Flow

  Startup Sequence:

  1. Build-time:
    - Quarkus reads tracing-enabled configuration
    - If true: Includes quarkus-opentelemetry extension
    - If false: Extension excluded (smaller artifact)
  2. Runtime initialization:
    - Quarkus creates OpenTelemetry CDI bean
    - Configures OTLP exporter (gRPC or HTTP)
    - OTelTracingProviderFactory checks activation conditions
    - Creates appropriate provider implementation
  3. Per-request execution:
    - Quarkus HTTP extension creates root span from request
    - KeycloakTracingCustomizer creates REST method spans
    - Application code creates additional spans
    - All spans share same trace ID

  Context Propagation

  OpenTelemetry context is propagated across different boundaries:

  Incoming HTTP
  - Mechanism: Quarkus auto-extraction
  - Format: W3C traceparent header

  Outgoing HTTP
  - Mechanism: OTelHttpClientBuilder
  - Format: W3C traceparent header

  JGroups messages
  - Mechanism: OPEN_TELEMETRY protocol
  - Format: TracerHeader with W3C context

  Thread-local
  - Mechanism: Span.makeCurrent()
  - Format: OpenTelemetry Context API

  Infinispan
  - Mechanism: OpenTelemetryService
  - Format: Parent context inheritance

  W3C Trace Context Format

  traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01

  Format breakdown:
  - Version: 00
  - Trace ID: 0af7651916cd43dd8448eb211c80319c (128-bit)
  - Span ID: b7ad6b7169203331 (64-bit)
  - Flags: 01
  
  Example End-to-End Trace

  HTTP Request → Keycloak Node 1
    → REST Handler (UserResource.getUser)
      → UserService (UserService.findUser)
        → Infinispan Cache (cache.get)
          → JGroups Message (JGroups.sendSingleMessage)
            → Keycloak Node 2 (JGroups.deliverSingleMessage)
        → JDBC Query (SELECT * FROM USER_ENTITY)
      → HTTP Client (HTTP GET https://external-idp.com)

  All spans share the same trace ID for full request correlation.

  ---
  Debugging and Testing

  Enable Debug Logging

  ./mvnw -f testsuite/integration-arquillian/pom.xml clean install \
    -Dtest=MyTest \
    -Dlog-level=org.keycloak.quarkus.runtime.tracing:debug

  Output example:

  DEBUG [org.keycloak.quarkus.runtime.tracing.OTelTracingProvider]
    Start span 'UserService.createUser' (spanId: '1234567890abcdef')
  DEBUG [org.keycloak.quarkus.runtime.tracing.OTelTracingProvider]
    End span 'UserService.createUser' (spanId: '1234567890abcdef')

  Span Lifecycle Validation

  Detect memory leaks from unclosed spans:

  boolean allEnded = tracing.validateAllSpansEnded();

  if (!allEnded) {
      // Warns about unclosed spans
  }

  Integration Tests

  Location: testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/tracing/OTelTracingProviderTest.java

  Enable tracing in tests:

  @Before
  public void startContainer() {
      AbstractQuarkusDeployableContainer container = ...;
      container.setAdditionalBuildArgs(List.of(
          "--tracing-enabled=true",
          "--log-level=org.keycloak.quarkus.runtime.tracing:debug"
      ));
      controller.start(containerQualifier);
  }

  Common Issues

  Issue: Spans not appearing in trace backend

  Checklist:
  1. tracing-enabled=true set during build
  2. tracing-endpoint points to correct OTLP collector
  3. Firewall allows connection to endpoint
  4. Check logs for export errors
  5. Verify sampling ratio
  
  Debug with: --log-level=io.opentelemetry:debug

  Issue: Memory leak warnings

  Cause: Span started but never ended

  Fix: Use automatic lifecycle with trace() methods or ensure endSpan() is in finally block.

  Issue: NoopTracingProvider instead of OTelTracingProvider

  Verify:
  1. Feature flag enabled: Profile.Feature.OPENTELEMETRY
  2. Build-time config: ./mvnw clean install --tracing-enabled=true
  3. Runtime config: --tracing-enabled=true

  ---
  Components Coverage

  Automatic Instrumentation (Zero Code Changes)

  HTTP Requests
  - Technology: Quarkus OpenTelemetry
  - Enabled By: tracing-enabled
  - Type: Build-time

  REST Endpoints
  - Technology: KeycloakTracingCustomizer
  - Enabled By: tracing-enabled
  - Type: Build-time

  JDBC Queries
  - Technology: opentelemetry-jdbc
  - Enabled By: tracing-jdbc-enabled
  - Type: Build-time

  HTTP Client
  - Technology: OTelHttpClientBuilder
  - Enabled By: tracing-enabled
  - Type: Build-time

  Manual Instrumentation (Code Required)

  Business Logic
  - Implementation: TracingProvider API
  - Enabled By: tracing-enabled
  - Control: Always traces when enabled
  
  Infinispan Cache
  - Implementation: OpenTelemetryService
  - Enabled By: tracing-infinispan-enabled
  - Control: Runtime toggle

  JGroups Cluster
  - Implementation: OPEN_TELEMETRY protocol
  - Enabled By: JGroups config
  - Control: Protocol-level
  
  Trace Propagation

  HTTP to Keycloak
  - Propagation Method: traceparent header
  - Standard: W3C Trace Context

  Keycloak to External
  - Propagation Method: traceparent header
  - Standard: W3C Trace Context

  Node to Node (JGroups)
  - Propagation Method: TracerHeader
  - Standard: W3C Trace Context
  
  Thread to Thread
  - Propagation Method: Context.makeCurrent()
  - Standard: OpenTelemetry API

  ---
  Summary

  OpenTelemetry integration in Keycloak provides:

  - Comprehensive Coverage: HTTP, REST, JDBC, caches, cluster messaging
  - Distributed Tracing: W3C Trace Context across all boundaries
  - Zero Overhead: No-op provider when disabled
  - Developer-Friendly API: Simple trace() methods, automatic lifecycle
  - Production-Ready: Sampling, compression, configurable exporters
  - Extensible: Provider SPI allows custom implementations
  - Observable: Debug logging, span validation, integration tests

  Key Files Reference

  Core SPI Layer:
  server-spi-private/src/main/java/org/keycloak/tracing/
  ├── TracingProvider.java
  └── NoopTracingProvider.java
  
  Quarkus Runtime Implementation:
  quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/
  ├── OTelTracingProvider.java
  ├── OTelTracingProviderFactory.java
  ├── OTelHttpClientBuilder.java
  └── KeycloakTracingCustomizer.java

  Infinispan & JGroups Integration:
  model/infinispan/src/main/java/org/keycloak/
  ├── infinispan/module/factory/OpenTelemetryService.java
  └── jgroups/protocol/OPEN_TELEMETRY.java
  
  Configuration:
  quarkus/config-api/src/main/java/org/keycloak/config/
  └── TracingOptions.java

  Tests:
  testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/tracing/
  └── OTelTracingProviderTest.java
