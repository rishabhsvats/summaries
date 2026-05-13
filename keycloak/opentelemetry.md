  # Keycloak OpenTelemetry Tracing

  > **Comprehensive distributed tracing implementation for Keycloak using OpenTelemetry**

  This document explains how OpenTelemetry has been configured in Keycloak to generate distributed traces across all system components.

  ---

  ## Table of Contents

  - [Overview](#overview)
    - [Architecture Layers](#architecture-layers)
    - [Key Design Principles](#key-design-principles)
  - [Quick Start](#quick-start)
  - [Configuration](#configuration)
    - [Available Options](#available-options)
    - [Example Configurations](#example-configurations)
  - [Architecture Deep Dive](#architecture-deep-dive)
    - [1. Configuration Layer](#1-configuration-layer)
    - [2. Provider SPI Layer](#2-provider-spi-layer)
    - [3. Implementation Layer](#3-implementation-layer)
    - [4. Integration Layer](#4-integration-layer)
  - [Usage Patterns](#usage-patterns)
    - [Basic Usage](#basic-usage)
    - [Advanced Usage](#advanced-usage)
    - [Real-World Examples](#real-world-examples)
  - [Runtime Behavior](#runtime-behavior)
    - [Activation Flow](#activation-flow)
    - [Context Propagation](#context-propagation)
  - [Debugging and Testing](#debugging-and-testing)
  - [Components Coverage](#components-coverage)

  ---

  ## Overview

  OpenTelemetry tracing in Keycloak provides end-to-end observability across authentication flows, cache operations, database queries, and cluster
  communications.

  ### Architecture Layers

  ┌─────────────────────────────────────────────────┐
  │          Configuration Layer                    │
  │        (TracingOptions.java)                    │
  └─────────────────────────────────────────────────┘
                        ↓
  ┌─────────────────────────────────────────────────┐
  │          Provider SPI Layer                     │
  │        (TracingProvider interface)              │
  └─────────────────────────────────────────────────┘
                        ↓
  ┌─────────────────────────────────────────────────┐
  │       Implementation Layer                      │
  │  OTelTracingProvider / NoopTracingProvider      │
  └─────────────────────────────────────────────────┘
                        ↓
  ┌─────────────────────────────────────────────────┐
  │       Integration Layer                         │
  │  REST | JDBC | HTTP Client | Infinispan | JGroups│
  └─────────────────────────────────────────────────┘

  ### Key Design Principles

  | Principle | Description |
  |-----------|-------------|
  | **Feature-gated** | Controlled by `Profile.Feature.OPENTELEMETRY` feature flag |
  | **Provider pattern** | Swappable between active (`OTelTracingProvider`) and no-op (`NoopTracingProvider`) implementations |
  | **Context propagation** | Uses W3C Trace Context for distributed tracing across services |
  | **Lifecycle management** | Automatic span lifecycle with scoped contexts |
  | **Zero overhead** | No performance penalty when disabled via no-op implementation |

  ---

  ## Quick Start

  ### Enable Tracing

  ```bash
  # Build with tracing enabled (required, build-time option)
  ./mvnw clean install -Poperator -DskipTests --tracing-enabled=true

  # Start server with tracing
  java -jar quarkus/server/target/lib/quarkus-run.jar start-dev \
    --tracing-enabled=true \
    --tracing-endpoint=http://localhost:4317 \
    --tracing-protocol=grpc

  Verify Tracing is Active

  # Enable debug logging
  --log-level=org.keycloak.quarkus.runtime.tracing:debug

  You should see log entries like:
  Start span 'ClassName.methodName' (spanId: '1234567890abcdef')
  End span 'ClassName.methodName' (spanId: '1234567890abcdef')

  ---
  Configuration

  Available Options

  All options are prefixed with tracing-*:

  ┌─────────────────────────────┬─────────┬───────────────────────┬────────────┬──────────────────────────────────────────┐
  │           Option            │  Type   │        Default        │ Build-time │               Description                │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-enabled             │ Boolean │ false                 │ ✅         │ Enable OpenTelemetry tracing             │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-endpoint            │ String  │ http://localhost:4317 │ ❌         │ OTLP endpoint for exporting traces       │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-service-name        │ String  │ keycloak              │ ❌         │ Service name in traces                   │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-protocol            │ String  │ grpc                  │ ❌         │ Protocol: grpc or http/protobuf          │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-sampler-type        │ String  │ traceidratio          │ ✅         │ Sampler type (see Quarkus docs)          │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-sampler-ratio       │ Double  │ 1.0                   │ ❌         │ Sampling ratio [0.0, 1.0]                │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-compression         │ Enum    │ none                  │ ❌         │ Compression: none or gzip                │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-resource-attributes │ List    │ -                     │ ❌         │ Resource attributes: key1=val1,key2=val2 │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-jdbc-enabled        │ Boolean │ true                  │ ✅         │ Enable JDBC tracing                      │
  ├─────────────────────────────┼─────────┼───────────────────────┼────────────┼──────────────────────────────────────────┤
  │ tracing-infinispan-enabled  │ Boolean │ true                  │ ❌         │ Enable Infinispan cache tracing          │
  └─────────────────────────────┴─────────┴───────────────────────┴────────────┴──────────────────────────────────────────┘

  ▎ Note: Build-time options require a rebuild when changed. Runtime options can be changed without rebuilding.

  Example Configurations

  ./mvnw clean install --tracing-enabled=true

  java -jar quarkus/server/target/lib/quarkus-run.jar start \
    --tracing-enabled=true \
    --tracing-endpoint=http://jaeger:4317 \
    --tracing-service-name=keycloak-prod \
    --tracing-protocol=grpc \
    --tracing-sampler-ratio=0.1 \
    --tracing-compression=gzip \
    --tracing-resource-attributes=environment=production,cluster=us-east-1
  ./mvnw clean install --tracing-enabled=true

  java -jar quarkus/server/target/lib/quarkus-run.jar start-dev \
    --tracing-enabled=true \
    --tracing-endpoint=http://localhost:4317 \
    --tracing-sampler-ratio=1.0 \
    --log-level=org.keycloak.quarkus.runtime.tracing:debug
  java -jar quarkus/server/target/lib/quarkus-run.jar start \
    --tracing-enabled=true \
    --tracing-endpoint=http://otlp-collector:4318 \
    --tracing-protocol=http/protobuf
  ---
  Architecture Deep Dive

  1. Configuration Layer

  Location: quarkus/config-api/src/main/java/org/keycloak/config/TracingOptions.java

  Defines all tracing configuration options using Keycloak's option builder pattern:

  public static final Option<Boolean> TRACING_ENABLED =
      new OptionBuilder<>("tracing-enabled", Boolean.class)
          .category(OptionCategory.TRACING)
          .description("Enables the OpenTelemetry tracing.")
          .defaultValue(Boolean.FALSE)
          .buildTime(true)
          .build();

  Key Characteristics:
  - Integrates with Quarkus OpenTelemetry extension
  - Build-time vs runtime option separation
  - Type-safe configuration with validation
  - Default values optimized for development

  ---
  2. Provider SPI Layer
  
  Location: server-spi-private/src/main/java/org/keycloak/tracing/TracingProvider.java

  The TracingProvider interface abstracts all tracing operations, making it swappable and testable.

  Core API Methods

  public interface TracingProvider extends Provider {
      // Get current active span
      Span getCurrentSpan();

      // Manual span lifecycle (low-level API)
      Span startSpan(String tracerName, String spanName);
      Span startSpan(Class<?> tracerClass, String spanSuffix);
      void endSpan();

      // Automatic span lifecycle (PREFERRED)
      void trace(Class<?> tracerClass, String spanSuffix, Consumer<Span> execution);
      <T> T trace(Class<?> tracerClass, String spanSuffix, Function<Span, T> execution);

      // Error handling
      void error(Throwable exception);

      // Tracer management
      Tracer getTracer(String name, String scopeVersion);
  }

  Design Patterns

  ┌─────────────────────┬──────────────────────────────┬──────────────────────────────────────────────────┐
  │       Pattern       │            Usage             │                     Example                      │
  ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────┤
  │ Automatic lifecycle │ Preferred for most use cases │ trace(MyClass.class, "operation", span -> {...}) │
  ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────┤
  │ Manual lifecycle    │ Complex control flows        │ startSpan() → try/finally → endSpan()            │
  ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────┤
  │ Class-based naming  │ Consistent span naming       │ Span name: ClassName.operation                   │
  ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────┤
  │ Error recording     │ Exception tracking           │ error(exception) adds full context               │
  └─────────────────────┴──────────────────────────────┴──────────────────────────────────────────────────┘

  ---
  3. Implementation Layer
  
  3.1 Active Implementation: OTelTracingProvider

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProvider.java

  The production implementation wrapping OpenTelemetry SDK:

  public class OTelTracingProvider implements TracingProvider {
      private final OpenTelemetry openTelemetry;  // Quarkus CDI bean
      private final Deque<Scope> scopes;          // Nested span stack

      @Override
      public Span startSpan(SpanBuilder builder) {
          var currentSpan = builder.startSpan();
          scopes.push(currentSpan.makeCurrent());  // Thread-local context
          log.debugf("Start span '%s' (spanId: '%s')", name, spanId);
          return currentSpan;
      }

      @Override
      public void endSpan() {
          var span = getCurrentSpan();
          span.end();
          scopes.pop().close();  // Restore previous context
          log.debugf("End span '%s' (spanId: '%s')", name, spanId);
      }
  }

  Key Features:
  - Scope management: Thread-safe deque for nested spans
  - Context propagation: Automatic parent-child relationships
  - Error recording: OTel Semantic Conventions compliant
  - Debug logging: Detailed span lifecycle events
  - Validation: Detects unclosed spans (memory leak prevention)

  3.2 Factory: OTelTracingProviderFactory

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProviderFactory.java

  Provider lifecycle management and activation logic:

  public class OTelTracingProviderFactory implements TracingProviderFactory {
      private static OpenTelemetry OTEL_SINGLETON;

      @Override
      public TracingProvider create(KeycloakSession session) {
          if (OTEL_SINGLETON == null) {
              // Lazy initialization from Quarkus CDI
              OTEL_SINGLETON = CDI.current().select(OpenTelemetry.class).get();
          }
          return new OTelTracingProvider(OTEL_SINGLETON);
      }

      @Override
      public boolean isSupported(Config.Scope config) {
          return Profile.isFeatureEnabled(Profile.Feature.OPENTELEMETRY)
              && Configuration.isTrue(TracingOptions.TRACING_ENABLED);
      }
  }

  Activation Conditions:
  1. ✅ Profile.Feature.OPENTELEMETRY enabled (enabled by default)
  2. ✅ tracing-enabled=true in configuration 
  3. If both true → OTelTracingProvider
  4. If either false → NoopTracingProvider

  3.3 Fallback: NoopTracingProvider

  Location: server-spi-private/src/main/java/org/keycloak/tracing/NoopTracingProvider.java

  Zero-overhead implementation when tracing is disabled:

  public class NoopTracingProvider implements TracingProvider {
      @Override
      public Span getCurrentSpan() {
          return Span.getInvalid();  // No-op span
      }

      @Override
      public void trace(String tracerName, String spanName, Consumer<Span> execution) {
          execution.accept(getCurrentSpan());  // Execute without tracing
      }

      // All other methods are no-ops
  }

  Benefits:
  - No runtime overhead when tracing disabled
  - Application code unchanged (same API)
  - No null checks needed

  ---
  4. Integration Layer

  4.1 🌐 REST Endpoint Auto-Instrumentation

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/integration/resteasy/KeycloakTracingCustomizer.java

  Automatically creates spans for every JAX-RS resource method invocation:

  public class KeycloakTracingCustomizer implements HandlerChainCustomizer {

      // Executed BEFORE method invocation
      class StartHandler implements ServerRestHandler {
          @Override
          public void handle(ResteasyReactiveRequestContext requestContext) {
              OpenTelemetry otel = CDI.current().select(OpenTelemetry.class).get();
              Tracer tracer = otel.getTracer(className, Version.VERSION);

              SpanBuilder builder = tracer.spanBuilder("ClassName.methodName");
              builder.setParent(Context.current().with(Span.current()));
              builder.setAttribute("code.function", methodName);
              builder.setAttribute("code.namespace", className);

              Span span = builder.startSpan();
              requestContext.setProperty("span", span);
              requestContext.setProperty("scope", span.makeCurrent());
          }
      }

      // Executed AFTER method invocation
      class EndHandler implements ServerRestHandler {
          @Override
          public void handle(ResteasyReactiveRequestContext requestContext) {
              ((Scope) requestContext.getProperty("scope")).close();
              ((Span) requestContext.getProperty("span")).end();
          }
      }
  }

  Span Attributes:
  - code.function → Method name (e.g., token)
  - code.namespace → Fully qualified class name
  - Span name → ClassName.methodName

  Behavior:
  - Hooks into RESTEasy Reactive handler chain
  - BEFORE_METHOD_INVOKE phase → Start span
  - AFTER_METHOD_INVOKE phase → End span
  - Automatically inherits parent context from HTTP request

  ---
  4.2 🗄️  JDBC Tracing
  
  Dependency: io.opentelemetry.instrumentation:opentelemetry-jdbc
  Configuration: tracing-jdbc-enabled=true (default, build-time)

  How it Works:
  - OpenTelemetry JDBC instrumentation wraps JDBC drivers
  - Automatically creates spans for SQL queries
  - No code changes required

  Span Information:
  - Operation: SELECT, INSERT, UPDATE, DELETE
  - Database system, connection string (sanitized)
  - Query duration

  Enable/Disable:
  # Disable JDBC tracing (requires rebuild)
  ./mvnw clean install --tracing-jdbc-enabled=false

  ---
  4.3 🌍 HTTP Client Tracing
  
  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelHttpClientBuilder.java

  Instruments outbound HTTP requests (OIDC, SAML, etc.):

  public class OTelHttpClientBuilder extends HttpClientBuilder {
      private final OTelTracingProvider provider;

      @Override
      protected HttpClientBuilder getApacheHttpClientBuilder() {
          return ApacheHttpClientTelemetry
              .builder(provider.getOpenTelemetry())
              .build()
              .newHttpClientBuilder();
      }
  }

  Capabilities:
  - ✅ Creates spans for all outbound HTTP calls
  - ✅ Propagates trace context via traceparent HTTP header (W3C)
  - ✅ Captures HTTP status codes, URLs, methods
  - ✅ Used for all Keycloak external communications
  
  Trace Propagation:
  GET /protocol/openid-connect/token HTTP/1.1
  Host: external-idp.example.com
  traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01

  ---
  4.4 💾 Infinispan Cache Tracing
  
  Location: model/infinispan/src/main/java/org/keycloak/infinispan/module/factory/OpenTelemetryService.java

  Traces embedded Infinispan cache operations:

  public class OpenTelemetryService implements InfinispanTelemetry {
      private final Tracer tracer;

      @Override
      public <T> InfinispanSpan<T> startTraceRequest(
              String operationName,
              InfinispanSpanAttributes attributes) {

          var builder = tracer.spanBuilder(operationName)
              .setSpanKind(SpanKind.SERVER);

          // Add cache-specific attributes
          attributes.cacheName().ifPresent(
              cacheName -> builder.setAttribute("cache", cacheName));
          builder.setAttribute("category", attributes.category().toString());
          builder.setAttribute("server.address", nodeName);

          return new OpenTelemetrySpan<>(builder.startSpan());
      }
  }

  Configuration: tracing-infinispan-enabled=true (default, runtime)

  Traced Operations:
  - Cache get, put, remove, clear
  - Cache creation/destruction
  - Cluster-wide cache operations

  Span Attributes:
  - cache → Cache name (e.g., sessions, users)
  - category → Operation category
  - server.address → Node name in cluster

  ---
  4.5 🔗 JGroups Cluster Tracing
  
  Location: model/infinispan/src/main/java/org/keycloak/jgroups/protocol/OPEN_TELEMETRY.java

  Traces cluster communication between Keycloak nodes:

  @MBean(description = "Records OpenTelemetry traces of sent and received messages")
  public class OPEN_TELEMETRY extends Protocol {

      // Sending a message to cluster
      @Override
      public Object down(Message msg) {
          Span span = tracer.spanBuilder("JGroups.sendSingleMessage")
              .setAttribute("kc.jgroups.dest", msg.getDest().toString())
              .setAttribute("kc.jgroups.src", msg.getSrc().toString())
              .startSpan();

          try (var scope = span.makeCurrent()) {
              // Inject W3C Trace Context into message header
              TracerHeader hdr = new TracerHeader();
              W3CTraceContextPropagator.getInstance()
                  .inject(Context.current(), hdr, (carrier, key, val) ->
                      hdr.put(key, val));
              msg.putHeader(OPEN_TELEMETRY_ID, hdr);

              return down_prot.down(msg);
          } finally {
              span.end();
          }
      }

      // Receiving a message from cluster
      @Override
      public Object up(Message msg) {
          TracerHeader hdr = msg.getHeader(OPEN_TELEMETRY_ID);

          // Extract parent context from header
          Context extractedContext = otel.getPropagators()
              .getTextMapPropagator()
              .extract(Context.current(), hdr, TEXT_MAP_GETTER);

          Span span = tracer.spanBuilder("JGroups.deliverSingleMessage")
              .setSpanKind(SpanKind.SERVER)
              .setParent(extractedContext)
              .startSpan();

          try (Scope scope = span.makeCurrent()) {
              return up_prot.up(msg);
          } finally {
              span.end();
          }
      }
  }

  Key Features:
  - ✅ Distributed tracing across cluster nodes
  - ✅ W3C Trace Context propagation via JGroups headers
  - ✅ Single message and batched message support
  - ✅ Source/destination node tracking
  - ✅ Enables end-to-end tracing in clustered deployments
  
  JGroups Configuration:
  The protocol must be added to JGroups stack configuration (just above transport layer).

  Span Attributes:
  - kc.jgroups.dest → Destination node
  - kc.jgroups.src → Source node
  - from → Message sender
  - batch-msg → Batch position (for batched messages)

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
          // Automatic span lifecycle - PREFERRED approach
          tracing.trace(UserService.class, "createUser", span -> {
              // Your business logic here
              validateUser(user);
              persistUser(user);
              sendWelcomeEmail(user);

              // Add custom attributes
              span.setAttribute("user.id", user.getId());
              span.setAttribute("user.realm", user.getRealm());
          });
          // Span automatically ended, even on exception
      }
  }

  Resulting Span:
  - Name: UserService.createUser
  - Tracer: org.keycloak.services.UserService
  - Attributes: user.id, user.realm

  ---
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

  ---
  Advanced Usage

  Example 3: Manual Span Management

  Use when you need fine-grained control (try-finally required):

  void complexAuthFlow() {
      Span span = tracing.startSpan(AuthenticationFlow.class, "complexAuth");
      try {
          // Step 1
          validateCredentials();
          span.setAttribute("step.credentials", "valid");

          // Step 2
          checkMFA();
          span.setAttribute("step.mfa", "passed");

          // Step 3
          createSession();
          span.setAttribute("step.session", "created");

      } catch (AuthenticationException e) {
          // Record exception with full stack trace
          tracing.error(e);
          throw e;
      } finally {
          // MUST manually end the span
          tracing.endSpan();
      }
  }

  ---
  Example 4: Enriching Current Span

  void processRequest(HttpRequest request) {
      // Get the current active span (created by REST handler)
      Span currentSpan = tracing.getCurrentSpan();

      // Add custom attributes
      currentSpan.setAttribute("http.user_agent", request.getHeader("User-Agent"));
      currentSpan.setAttribute("client.ip", request.getRemoteAddr());
      currentSpan.setAttribute("request.id", UUID.randomUUID().toString());

      // Continue processing...
  }

  ---
  Example 5: Nested Spans

  void parentOperation() {
      tracing.trace(MyService.class, "parentOp", parentSpan -> {

          // Child span 1
          tracing.trace(MyService.class, "childOp1", childSpan1 -> {
              doWork1();
          });

          // Child span 2
          tracing.trace(MyService.class, "childOp2", childSpan2 -> {
              doWork2();
          });

      });
  }

  Trace Structure:
  MyService.parentOp
  ├── MyService.childOp1
  └── MyService.childOp2

  ---
  Real-World Examples

  Example from Codebase

  Location: services/src/main/java/org/keycloak/services/DefaultKeycloakContext.java:245

  public class DefaultKeycloakContext implements KeycloakContext {
      private KeycloakSession session;

      @Override
      public Span getCurrentTracingSpan() {
          return session.getProvider(TracingProvider.class).getCurrentSpan();
      }
  }

  This allows any component to access the current span via KeycloakContext:

  void someBusinessLogic(KeycloakContext context) {
      Span span = context.getCurrentTracingSpan();
      span.setAttribute("custom.attribute", "value");
  }

  ---
  Runtime Behavior

  Activation Flow

  graph TD
      A[Server Startup] --> B{Build-time: tracing-enabled?}
      B -->|false| C[OpenTelemetry excluded from build]
      B -->|true| D[OpenTelemetry included in build]
      D --> E{Runtime: Feature.OPENTELEMETRY enabled?}
      E -->|false| F[NoopTracingProvider]
      E -->|true| G{Runtime: tracing-enabled=true?}
      G -->|false| F
      G -->|true| H[OTelTracingProvider created]
      H --> I[Quarkus creates OpenTelemetry CDI bean]
      I --> J[OTLP Exporter configured]
      J --> K[TracingProvider available in session]

  Startup Sequence:

  1. Build-time:
    - Quarkus reads tracing-enabled configuration
    - If true: Includes quarkus-opentelemetry extension
    - If false: Extension excluded (smaller artifact)
  2. Runtime initialization:
    - Quarkus creates OpenTelemetry CDI bean
    - Configures OTLP exporter (gRPC or HTTP)
    - OTelTracingProviderFactory.isSupported() checks:
        - ✅ Profile.Feature.OPENTELEMETRY enabled
      - ✅ tracing-enabled=true
    - Creates appropriate provider implementation
  3. Per-request execution:
    - Quarkus HTTP extension creates root span from request
    - KeycloakTracingCustomizer creates REST method spans
    - Application code creates additional spans
    - All spans share same trace ID

  ---
  Context Propagation

  OpenTelemetry context is propagated across different boundaries:

  ┌──────────────────┬─────────────────────────┬───────────────────────────────┐
  │     Boundary     │        Mechanism        │            Format             │
  ├──────────────────┼─────────────────────────┼───────────────────────────────┤
  │ Incoming HTTP    │ Quarkus auto-extraction │ W3C traceparent header        │
  ├──────────────────┼─────────────────────────┼───────────────────────────────┤
  │ Outgoing HTTP    │ OTelHttpClientBuilder   │ W3C traceparent header        │
  ├──────────────────┼─────────────────────────┼───────────────────────────────┤
  │ JGroups messages │ OPEN_TELEMETRY protocol │ TracerHeader with W3C context │
  ├──────────────────┼─────────────────────────┼───────────────────────────────┤
  │ Thread-local     │ Span.makeCurrent()      │ OpenTelemetry Context API     │
  ├──────────────────┼─────────────────────────┼───────────────────────────────┤
  │ Infinispan       │ OpenTelemetryService    │ Parent context inheritance    │
  └──────────────────┴─────────────────────────┴───────────────────────────────┘

  W3C Trace Context Format

  traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
               │   │                                │                  │
               │   └─ Trace ID (128-bit)            │                  └─ Flags
               │                                    └─ Span ID (64-bit)
               └─ Version

  Example: End-to-End Trace

  HTTP Request → Keycloak Node 1
                 │
                 ├─> REST Handler (span: UserResource.getUser)
                 │   │
                 │   ├─> UserService (span: UserService.findUser)
                 │   │   │
                 │   │   ├─> Infinispan Cache (span: cache.get)
                 │   │   │   │
                 │   │   │   └─> JGroups Message (span: JGroups.sendSingleMessage)
                 │   │   │       │
                 │   │   │       └─> Keycloak Node 2 (span: JGroups.deliverSingleMessage)
                 │   │   │
                 │   │   └─> JDBC Query (span: SELECT * FROM USER_ENTITY)
                 │   │
                 │   └─> HTTP Client (span: HTTP GET https://external-idp.com)

  All spans in this trace share the same trace ID, enabling full request correlation.

  ---
  Debugging and Testing

  Enable Debug Logging

  ./mvnw -f testsuite/integration-arquillian/pom.xml clean install \
    -Dtest=MyTest \
    -Dlog-level=org.keycloak.quarkus.runtime.tracing:debug

  Output:
  DEBUG [org.keycloak.quarkus.runtime.tracing.OTelTracingProvider] Start span 'UserService.createUser' (spanId: '1234567890abcdef')
  DEBUG [org.keycloak.quarkus.runtime.tracing.OTelTracingProvider] End span 'UserService.createUser' (spanId: '1234567890abcdef')

  ---
  Span Lifecycle Validation
  
  Detect memory leaks from unclosed spans:

  // In test cleanup or request end
  boolean allEnded = tracing.validateAllSpansEnded();

  if (!allEnded) {
      // Warns: "Some spans during tracing were not ended. 
      //         It may lead to memory leaks and incorrect scoping."
  }

  ---
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

  @Test
  public void testTracingActive() {
      getTestingClient().server(TEST_REALM_NAME).run(session -> {
          TracingProvider provider = session.getProvider(TracingProvider.class);

          // Verify OTel provider is active
          assertThat(provider.getClass().getSimpleName(),
                     is(OTelTracingProvider.class.getSimpleName()));

          // Create test span
          provider.trace(MyTest.class, "testOp", span -> {
              span.setAttribute("test.attribute", "value");
          });

          // Verify all spans closed
          assertThat(provider.validateAllSpansEnded(), is(true));
      });
  }

  ---
  Common Issues

  Checklist:
  1. ✅ tracing-enabled=true set during build
  2. ✅ tracing-endpoint points to correct OTLP collector
  3. ✅ Firewall allows connection to endpoint
  4. ✅ Check logs for export errors
  5. ✅ Verify sampling ratio (tracing-sampler-ratio)

  Debug:
  --log-level=io.opentelemetry:debug
  Warning:
  WARN [OTelTracingProvider] Some spans during tracing were not ended.

  Cause: Span started but never ended (missing endSpan() or exception in trace() block)

  Fix:
  // BAD - span leaked on exception
  tracing.startSpan(...);
  doWork(); // throws exception
  tracing.endSpan(); // never reached!
  
  // GOOD - automatic cleanup
  tracing.trace(..., span -> {
      doWork(); // span ended even on exception
  });
  
  // GOOD - manual with try-finally
  tracing.startSpan(...);
  try {
      doWork();
  } finally {
      tracing.endSpan(); // always called
  }
  Verify activation conditions:
  # Check feature flag
  grep -r "Feature.OPENTELEMETRY" common/src/main/java/org/keycloak/common/Profile.java

  # Check build-time config
  ./mvnw clean install --tracing-enabled=true

  # Check runtime config
  --tracing-enabled=true
  ---
  Components Coverage

  Automatic Instrumentation (Zero Code Changes)

  ┌────────────────┬───────────────────────────┬──────────────────────┬───────────────┐
  │   Component    │        Technology         │      Enabled By      │ Build/Runtime │
  ├────────────────┼───────────────────────────┼──────────────────────┼───────────────┤
  │ HTTP Requests  │ Quarkus OpenTelemetry     │ tracing-enabled      │ Build-time    │
  ├────────────────┼───────────────────────────┼──────────────────────┼───────────────┤
  │ REST Endpoints │ KeycloakTracingCustomizer │ tracing-enabled      │ Build-time    │
  ├────────────────┼───────────────────────────┼──────────────────────┼───────────────┤
  │ JDBC Queries   │ opentelemetry-jdbc        │ tracing-jdbc-enabled │ Build-time    │
  ├────────────────┼───────────────────────────┼──────────────────────┼───────────────┤
  │ HTTP Client    │ OTelHttpClientBuilder     │ tracing-enabled      │ Build-time    │
  └────────────────┴───────────────────────────┴──────────────────────┴───────────────┘

  Manual Instrumentation (Code Required)

  ┌──────────────────┬─────────────────────────┬────────────────────────────┬────────────────────────────┐
  │    Component     │     Implementation      │         Enabled By         │      Runtime Control       │
  ├──────────────────┼─────────────────────────┼────────────────────────────┼────────────────────────────┤
  │ Business Logic   │ TracingProvider API     │ tracing-enabled            │ Always traces when enabled │
  ├──────────────────┼─────────────────────────┼────────────────────────────┼────────────────────────────┤
  │ Infinispan Cache │ OpenTelemetryService    │ tracing-infinispan-enabled │ Runtime toggle             │
  ├──────────────────┼─────────────────────────┼────────────────────────────┼────────────────────────────┤
  │ JGroups Cluster  │ OPEN_TELEMETRY protocol │ JGroups config             │ Protocol-level toggle      │
  └──────────────────┴─────────────────────────┴────────────────────────────┴────────────────────────────┘

  Trace Propagation

  ┌───────────────────────┬───────────────────────┬───────────────────┐
  │   Integration Point   │  Propagation Method   │     Standard      │
  ├───────────────────────┼───────────────────────┼───────────────────┤
  │ HTTP → Keycloak       │ traceparent header    │ W3C Trace Context │
  ├───────────────────────┼───────────────────────┼───────────────────┤
  │ Keycloak → External   │ traceparent header    │ W3C Trace Context │
  ├───────────────────────┼───────────────────────┼───────────────────┤
  │ Node → Node (JGroups) │ TracerHeader          │ W3C Trace Context │
  ├───────────────────────┼───────────────────────┼───────────────────┤
  │ Thread → Thread       │ Context.makeCurrent() │ OpenTelemetry API │
  └───────────────────────┴───────────────────────┴───────────────────┘

  ---
  Summary

  OpenTelemetry integration in Keycloak provides:

  ✅ Comprehensive Coverage: HTTP, REST, JDBC, caches, cluster messaging
  ✅ Distributed Tracing: W3C Trace Context across all boundaries
  ✅ Zero Overhead: No-op provider when disabled
  ✅ Developer-Friendly API: Simple trace() methods, automatic lifecycle
  ✅ Production-Ready: Sampling, compression, configurable exporters
  ✅ Extensible: Provider SPI allows custom implementations
  ✅ Observable: Debug logging, span validation, integration tests

  Key Files:

  server-spi-private/src/main/java/org/keycloak/tracing/
  ├── TracingProvider.java                    # Core SPI
  ├── NoopTracingProvider.java                # Fallback implementation

  quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/
  ├── OTelTracingProvider.java                # Active implementation
  ├── OTelTracingProviderFactory.java         # Lifecycle management
  ├── OTelHttpClientBuilder.java              # HTTP client instrumentation
  └── KeycloakTracingCustomizer.java          # REST auto-instrumentation

  model/infinispan/src/main/java/org/keycloak/
  ├── infinispan/module/factory/
  │   └── OpenTelemetryService.java           # Infinispan integration
  └── jgroups/protocol/
      └── OPEN_TELEMETRY.java                 # JGroups cluster tracing

  quarkus/config-api/src/main/java/org/keycloak/config/
  └── TracingOptions.java                     # Configuration options

  ---
  Contributing

  When adding new tracing:

  1. Use TracingProvider API - Don't directly use OpenTelemetry SDK
  2. Prefer automatic lifecycle - Use trace() methods over startSpan()/endSpan()
  3. Follow naming convention - ClassName.operation for span names
  4. Add meaningful attributes - Use OTel Semantic Conventions
  5. Handle errors - Call tracing.error(exception) in catch blocks
  6. Test both modes - Verify with OTelTracingProvider and NoopTracingProvider
  7. Document configuration - Add new options to TracingOptions.java

