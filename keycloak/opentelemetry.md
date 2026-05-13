  ---
  High-Level Overview
  
  OpenTelemetry tracing in Keycloak is built on a layered architecture with these main components:

  Architecture Layers:

  1. Configuration Layer - Quarkus-based configuration options that control tracing behavior
  2. Provider SPI Layer - Service Provider Interface (TracingProvider) that abstracts tracing operations
  3. Implementation Layer - OTelTracingProvider that wraps OpenTelemetry SDK
  4. Integration Layer - Specialized integrations for HTTP, JDBC, Infinispan, and JGroups
  5. Auto-instrumentation Layer - REST endpoint instrumentation via Quarkus extensions

  Key Design Principles:

  - Feature-gated: Controlled by Profile.Feature.OPENTELEMETRY feature flag
  - Provider pattern: Swappable between OTelTracingProvider (active) and NoopTracingProvider (disabled)
  - Context propagation: Uses W3C Trace Context for distributed tracing across services
  - Lifecycle management: Automatic span lifecycle with scoped contexts
  - Zero-overhead when disabled: No-op implementation ensures no performance penalty

  ---
  Detailed Implementation

  1. Configuration Layer (TracingOptions.java)

  Location: quarkus/config-api/src/main/java/org/keycloak/config/TracingOptions.java:25-95

  Configuration Properties:

  // Enable/disable tracing (build-time option)
  tracing-enabled = false (default)

  // OTLP endpoint for exporting traces
  tracing-endpoint = "http://localhost:4317" (default)

  // Service identification
  tracing-service-name = "keycloak" (default)

  // Protocol for trace export
  tracing-protocol = "grpc" (default, also supports "http/protobuf")

  // Sampling configuration
  tracing-sampler-type = "traceidratio" (default)
  tracing-sampler-ratio = 1.0 (100% sampling by default)

  // Compression
  tracing-compression = "none" (default, supports "gzip")

  // Resource attributes for telemetry
  tracing-resource-attributes = key1=val1,key2=val2

  // Sub-system tracing toggles
  tracing-jdbc-enabled = true (default, build-time)
  tracing-infinispan-enabled = true (default, runtime)

  Key Characteristics:
  - Some options are build-time (require rebuild): tracing-enabled, tracing-jdbc-enabled, tracing-sampler-type
  - Others are runtime configurable: endpoint, protocol, compression, service name
  - Integrates with Quarkus OpenTelemetry extension configuration

  ---
  2. Provider SPI Layer (TracingProvider.java)
  
  Location: server-spi-private/src/main/java/org/keycloak/tracing/TracingProvider.java:31-265

  Core Interface Methods:

  // Get current span from context
  Span getCurrentSpan()

  // Manual span lifecycle (low-level API)
  Span startSpan(String tracerName, String spanName)
  Span startSpan(Class<?> tracerClass, String spanSuffix)
  Span startSpan(SpanBuilder builder)
  void endSpan()

  // Automatic span lifecycle (preferred API)
  void trace(String tracerName, String spanName, Consumer<Span> execution)
  <T> T trace(String tracerName, String spanName, Function<Span, T> execution)
  void trace(Class<?> tracerClass, String spanSuffix, Consumer<Span> execution)
  <T> T trace(Class<?> tracerClass, String spanSuffix, Function<Span, T> execution)

  // Error handling
  void error(Throwable exception)

  // Tracer management
  Tracer getTracer(String name, String scopeVersion)

  // Validation (for testing)
  boolean validateAllSpansEnded()

  Design Pattern:
  - Preferred approach: Use trace() methods that handle span lifecycle automatically
  - Advanced use: Use startSpan()/endSpan() for complex control flow
  - Naming convention: Span names formatted as ClassName.operation
  - Scope version: Defaults to Keycloak version (e.g., "26.4.6")

  ---
  3. Implementation Layer
  
  3.1 OTelTracingProvider (OTelTracingProvider.java)

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProvider.java:41-162

  Core Implementation:

  public class OTelTracingProvider implements TracingProvider {
      private final OpenTelemetry openTelemetry;  // Quarkus-provided singleton
      private final Deque<Scope> scopes;          // Stack of active scopes

      public Span startSpan(SpanBuilder builder) {
          var currentSpan = builder.startSpan();
          scopes.push(currentSpan.makeCurrent());  // Make span current in context
          return currentSpan;
      }

      public void endSpan() {
          var span = getCurrentSpan();
          span.end();                    // End the span
          var scope = scopes.pop();
          scope.close();                 // Close the scope
      }

      public void error(Throwable exception) {
          var span = getCurrentSpan();
          var exceptionAttributes = Attributes.builder()
              .put(ExceptionAttributes.EXCEPTION_ESCAPED, true)
              .put(ExceptionAttributes.EXCEPTION_MESSAGE, exception.getMessage())
              .put(ExceptionAttributes.EXCEPTION_TYPE, exception.getClass().getCanonicalName())
              .put(ExceptionAttributes.EXCEPTION_STACKTRACE, stackTrace)
              .build();
          span.recordException(exception, exceptionAttributes);
          span.setStatus(StatusCode.ERROR, exception.getMessage());
      }
  }

  Key Features:
  - Scope management: Maintains a ConcurrentLinkedDeque<Scope> for nested spans
  - Automatic error recording: Follows OTel Semantic Conventions for exceptions
  - Debug logging: Logs span start/end with span names and IDs
  - Context propagation: Uses Span.makeCurrent() to propagate context
  
  3.2 OTelTracingProviderFactory (OTelTracingProviderFactory.java)

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelTracingProviderFactory.java:31-77

  Lifecycle Management:

  public class OTelTracingProviderFactory implements TracingProviderFactory {
      private static OpenTelemetry OTEL_SINGLETON;

      @Override
      public TracingProvider create(KeycloakSession session) {
          if (OTEL_SINGLETON == null) {
              // Lazy initialization from Quarkus CDI context
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

  Activation Logic:
  1. Checks OPENTELEMETRY feature is enabled
  2. Checks tracing-enabled=true configuration
  3. If either is false, falls back to NoopTracingProvider
  4. Retrieves OpenTelemetry instance from Quarkus CDI container

  3.3 NoopTracingProvider (Fallback)

  Location: server-spi-private/src/main/java/org/keycloak/tracing/NoopTracingProvider.java:31-91

  Zero-Overhead Fallback:
  - Returns Span.getInvalid() for all span operations
  - Empty implementations for lifecycle methods
  - Executes traced code blocks without instrumentation
  - Ensures no performance penalty when tracing is disabled

  ---
  4. Integration Layer
  
  4.1 REST Endpoint Auto-Instrumentation (KeycloakTracingCustomizer.java)

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/integration/resteasy/KeycloakTracingCustomizer.java:38-100

  How it Works:

  This customizer hooks into RESTEasy Reactive's handler chain to automatically create spans for every REST endpoint method invocation.

  public class KeycloakTracingCustomizer implements HandlerChainCustomizer {

      // Before method execution
      class StartHandler implements ServerRestHandler {
          public void handle(ResteasyReactiveRequestContext requestContext) {
              OpenTelemetry otel = CDI.current().select(OpenTelemetry.class).get();
              Tracer tracer = otel.getTracer(className, Version.VERSION);

              SpanBuilder spanBuilder = tracer.spanBuilder("ClassName.methodName");
              spanBuilder.setParent(Context.current().with(Span.current()));
              spanBuilder.setAttribute("code.function", methodName);
              spanBuilder.setAttribute("code.namespace", className);

              Span span = spanBuilder.startSpan();
              requestContext.setProperty("span", span);
              requestContext.setProperty("scope", span.makeCurrent());
          }
      }

      // After method execution
      class EndHandler implements ServerRestHandler {
          public void handle(ResteasyReactiveRequestContext requestContext) {
              Scope scope = requestContext.getProperty("scope");
              scope.close();
              Span span = requestContext.getProperty("span");
              span.end();
          }
      }
  }

  Characteristics:
  - Automatically instruments all JAX-RS resource methods
  - Span names: ClassSimpleName.methodName
  - Adds code function and namespace attributes
  - Integrates with parent spans from HTTP headers (Quarkus does this automatically)

  4.2 JDBC Tracing

  Enabled via dependency: io.opentelemetry.instrumentation:opentelemetry-jdbc

  Location: quarkus/runtime/pom.xml (dependency)

  Configuration:
  - Controlled by tracing-jdbc-enabled=true (default)
  - Auto-instruments JDBC calls via OpenTelemetry JDBC instrumentation
  - No code changes required - works via JDBC driver wrapping

  4.3 HTTP Client Tracing (OTelHttpClientBuilder.java)

  Location: quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/tracing/OTelHttpClientBuilder.java:26-38

  Apache HttpClient Instrumentation:

  public class OTelHttpClientBuilder extends HttpClientBuilder {
      @Override
      protected HttpClientBuilder getApacheHttpClientBuilder() {
          return ApacheHttpClientTelemetry
              .builder(provider.getOpenTelemetry())
              .build()
              .newHttpClientBuilder();
      }
  }

  What it Does:
  - Wraps Apache HttpClient with OpenTelemetry instrumentation
  - Automatically creates spans for outbound HTTP requests
  - Propagates trace context via HTTP headers (W3C Trace Context)
  - Used for all Keycloak outbound HTTP calls (OIDC, SAML, etc.)

  4.4 Infinispan Tracing (OpenTelemetryService.java)

  Location: model/infinispan/src/main/java/org/keycloak/infinispan/module/factory/OpenTelemetryService.java:15-74

  Infinispan Cache Operations Tracing:

  public class OpenTelemetryService implements InfinispanTelemetry {
      private final Tracer tracer;

      public OpenTelemetryService(OpenTelemetry openTelemetry) {
          this.tracer = openTelemetry.getTracer(
              "org.infinispan.server.tracing", "1.0.0");
      }

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

  Features:
  - Traces Infinispan cache operations (get, put, remove, etc.)
  - Adds cache name, category, and server address attributes
  - Controlled by tracing-infinispan-enabled=true
  - Integrates with embedded Infinispan caches
  
  4.5 JGroups Tracing (OPEN_TELEMETRY.java protocol)

  Location: model/infinispan/src/main/java/org/keycloak/jgroups/protocol/OPEN_TELEMETRY.java:53-194

  Distributed Cluster Message Tracing:

  @MBean(description = "Records OpenTelemetry traces of sent and received messages")
  public class OPEN_TELEMETRY extends Protocol {

      // When sending a message
      public Object down(Message msg) {
          SpanBuilder spanBuilder = tracer.spanBuilder("JGroups.sendSingleMessage");
          spanBuilder.setAttribute("kc.jgroups.dest", msg.getDest().toString());
          spanBuilder.setAttribute("kc.jgroups.src", msg.getSrc().toString());

          Span span = spanBuilder.startSpan();
          try (var scope = span.makeCurrent()) {
              // Inject trace context into message header
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

      // When receiving a message
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
              span.setAttribute("from", msg.src().toString());
              return up_prot.up(msg);
          } finally {
              span.end();
          }
      }
  }

  Capabilities:
  - JGroups protocol that must be configured in JGroups stack (just above transport)
  - Traces cluster messaging between Keycloak nodes
  - Context propagation: Uses W3C Trace Context in JGroups message headers
  - Handles both single messages and batched messages
  - Spans show source/destination nodes
  - Enables distributed tracing across cluster nodes

  ---
  5. Usage Patterns
  
  Example 1: Simple Tracing Block

  class MyService {
      private final TracingProvider tracing;

      MyService(KeycloakSession session) {
          tracing = session.getProvider(TracingProvider.class);
      }

      void processRequest() {
          // Automatic span lifecycle - PREFERRED
          tracing.trace(MyService.class, "processRequest", span -> {
              // Your code here
              doWork();
          });
          // Span automatically ended, even on exception
      }
  }

  Example 2: Returning Values

  String fetchData() {
      return tracing.trace(MyService.class, "fetchData", span -> {
          String result = queryDatabase();
          span.setAttribute("result.size", result.length());
          return result;
      });
  }

  Example 3: Manual Span Management

  void complexOperation() {
      Span span = tracing.startSpan(MyService.class, "complexOp");
      try {
          doStep1();
          span.setAttribute("step1", "completed");
          doStep2();
          span.setAttribute("step2", "completed");
      } catch (Exception e) {
          tracing.error(e);  // Records exception with stack trace
          throw e;
      } finally {
          tracing.endSpan();  // Must manually end
      }
  }

  Example 4: Accessing Current Span

  void enrichCurrentSpan() {
      Span currentSpan = tracing.getCurrentSpan();
      currentSpan.setAttribute("user.id", userId);
      currentSpan.setAttribute("realm", realmName);
  }

  Real Usage Example:

  Location: services/src/main/java/org/keycloak/services/DefaultKeycloakContext.java:245

  public Span getCurrentTracingSpan() {
      return session.getProvider(TracingProvider.class).getCurrentSpan();
  }

  ---
  6. Activation and Runtime Behavior

  Startup Flow:

  1. Build-time: Quarkus reads tracing-enabled configuration
    - If true: Includes OpenTelemetry extension in build
    - If false: OpenTelemetry excluded from build
  2. Runtime initialization:
    - Quarkus creates OpenTelemetry CDI bean with configured exporters
    - OTelTracingProviderFactory checks:
        - Profile.Feature.OPENTELEMETRY enabled? (enabled by default)
      - tracing-enabled=true?
    - If both true: Creates OTelTracingProvider
    - If either false: Creates NoopTracingProvider
  3. Per-request:
    - Quarkus HTTP extension creates root span from incoming request
    - KeycloakTracingCustomizer creates child spans for REST methods
    - Application code creates additional child spans via TracingProvider
    - All spans share same trace ID (distributed tracing)

  Context Propagation:

  - Incoming HTTP: Quarkus automatically extracts W3C traceparent header
  - Outgoing HTTP: OTelHttpClientBuilder injects traceparent header
  - JGroups messages: OPEN_TELEMETRY protocol propagates via TracerHeader
  - Thread boundaries: Spans made current in thread-local Context

  ---
  7. Observability and Debugging

  Debug Logging:

  ./mvnw clean install -Dtest=MyTest \
    -Dlog-level=org.keycloak.quarkus.runtime.tracing:debug

  Shows:
  Start span 'MyClass.myOperation' (spanId: '1234567890abcdef')
  End span 'MyClass.myOperation' (spanId: '1234567890abcdef')

  Span Validation:

  The validateAllSpansEnded() method checks for memory leaks:
  boolean allEnded = tracing.validateAllSpansEnded();
  // If false, warns: "Some spans were not ended. May lead to memory leaks."

  Testing:

  Location: testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/tracing/OTelTracingProviderTest.java

  Enable tracing for tests:
  --tracing-enabled=true

  ---
  Summary

  OpenTelemetry in Keycloak provides comprehensive distributed tracing through:

  1. Unified SPI: TracingProvider abstracts tracing across all components
  2. Automatic instrumentation: REST endpoints, JDBC, HTTP clients traced automatically
  3. Manual instrumentation: Explicit spans for business logic
  4. Distributed tracing: Context propagation via W3C Trace Context (HTTP, JGroups)
  5. Multi-layer coverage: Application → Infinispan → JGroups → Database
  6. Zero overhead when disabled: No-op provider ensures no performance impact
  7. Production-ready: Sampling, compression, configurable exporters

  The implementation is clean, extensible, and follows OpenTelemetry best practices.
