---
name: include-swagger
description: "Add Swagger UI and OpenAPI to classic Spring MVC WAR applications. Use when configuring springdoc with XML contexts, web.xml, Spring 6, and Jakarta."
metadata:
  author: John D'Souza
---

# Skill: Add Swagger UI / OpenAPI to a Classic Spring Framework (non-Boot) WAR

## When to use this skill
Use this when adding **springdoc-openapi + Swagger UI** to a **Spring Framework MVC** web
application that:

- Is packaged as a **WAR** (deployed to Tomcat 10.1+ / Jakarta EE 9+), **not Spring Boot**.
- Is configured with **XML application contexts** (e.g. `application-config.xml`,
  `dispatcher-config.xml`) and `web.xml`, using `<mvc:annotation-driven/>`.
- Uses **`@RestController`** classes for its REST endpoints.
- Runs on **Spring 6.x** and the **Jakarta** namespace (`jakarta.*`, not `javax.*`).

Springdoc 2.x is built around Spring Boot auto-configuration, so in a classic WAR the required
beans must be registered **by hand**. This skill documents that wiring and every pitfall hit
along the way.

---

## Version compatibility (critical)
Match versions to your Spring line. For **Spring 6.1.x**:

| Dependency | Version | Notes |
|---|---|---|
| `org.springdoc:springdoc-openapi-starter-webmvc-api` | `2.6.0` | Spec generation only (`/v3/api-docs`). The `-api` artifact, NOT `-ui`. |
| `org.webjars:swagger-ui` | `5.17.14` | Static Swagger UI assets served as a WebJar. |
| `jakarta.xml.bind:jakarta.xml.bind-api` | `4.0.2` | Must be JAXB 4 (Jakarta). JAXB 2.3.x uses `javax.xml.bind` and breaks springdoc. |
| `org.glassfish.jaxb:jaxb-runtime` | `4.0.5` | Must match the Jakarta JAXB 4 API. |
| `org.ehcache:ehcache` | `3.10.8` + `<classifier>jakarta</classifier>` | The default Ehcache classifier uses `javax.xml.bind`; the `jakarta` classifier uses JAXB 4. |

> Rule of thumb: once any library forces JAXB onto the Jakarta namespace, **everything** that
> touches JAXB (Ehcache, Jackson XML) must be on JAXB 4 too, or you get
> `NoClassDefFoundError: javax/xml/bind/...`.

---

## Step-by-step

### 1. Add dependencies (`pom.xml`)
```xml
<!-- properties -->
<springdoc.version>2.6.0</springdoc.version>
<swagger-ui.version>5.17.14</swagger-ui.version>
<bind-api.version>4.0.2</bind-api.version>
<jaxb-runtime.version>4.0.5</jaxb-runtime.version>

<!-- dependencies -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-api</artifactId>
    <version>${springdoc.version}</version>
</dependency>
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>swagger-ui</artifactId>
    <version>${swagger-ui.version}</version>
</dependency>
```
Also add `<classifier>jakarta</classifier>` to the `ehcache` dependency, and bump
`jakarta.xml.bind-api` / `jaxb-runtime` to the versions above.

### 2. Create the OpenAPI `@Configuration` class
Springdoc has no auto-config here, so import its core config classes explicitly.

```java
@Configuration
@PropertySource("classpath:springdoc.properties")
@Import({ SpringDocConfiguration.class,
          SpringDocConfigProperties.class,
          SpringDocWebMvcConfiguration.class })
public class OpenApiConfig {
    @Bean
    public OpenAPI appOpenAPI(@Value("${app.version}") String version) {
        return new OpenAPI().info(new Info()
                .title("My Service")
                .description("...")
                .version(version))
                // Relative server URL, resolved by Swagger UI against the spec URL
                // (<context-root>/swagger/api-docs) -> <context-root>/. Required behind proxies.
                .servers(List.of(new Server().url("../")));
    }
}
```
Without `.servers(...)`, springdoc guesses the server URL from the incoming request. Behind a
reverse proxy that rewrites the path (e.g. public `/uat/imt/<app>/`), the guess is wrong and
"Try it out" calls go to the bare host. The relative `../` works on localhost and behind proxies.
If the spec path is not exactly one level below the context root, adjust the `../` to match.

### 3. `springdoc.properties`
```ini
springdoc.api-docs.path=/swagger/api-docs
app.version=${project.version}
```
Loaded via `@PropertySource` in `OpenApiConfig`. The `api-docs.path` moves the spec off the
default `/v3/api-docs`. `app.version` is filled by Maven filtering (see step 6).

### 4. Split the Spring contexts (the big one)
A classic WAR usually loads the **same** XML in both the root context (`ContextLoaderListener`)
and the dispatcher context (`DispatcherServlet`). That causes springdoc to initialize in the
root context, where `mvcConversionService` does not exist. Split them:

- **`application-config.xml` (root context):** data sources, cache, services/DAOs.
  Exclude web components from its scan:
  ```xml
  <context:component-scan base-package="com.example">
      <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
      <context:exclude-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice" />
      <context:exclude-filter type="annotation" expression="org.springframework.context.annotation.Configuration" />
  </context:component-scan>
  ```
- **`dispatcher-config.xml` (web context):** controllers + `@Configuration` + MVC only:
  ```xml
  <context:component-scan base-package="com.example" use-default-filters="false">
      <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
      <context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice" />
      <context:include-filter type="annotation" expression="org.springframework.context.annotation.Configuration" />
  </context:component-scan>

  <mvc:resources mapping="/resources/**" location="/resources/" />
  <mvc:resources mapping="/swagger/**" location="/resources/swagger/" />
  <mvc:resources mapping="/webjars/swagger-ui/**"
                 location="classpath:/META-INF/resources/webjars/swagger-ui/5.17.14/" />

  <!-- Named explicitly: springdoc looks up a bean called 'mvcConversionService' -->
  <bean id="mvcConversionService"
        class="org.springframework.format.support.FormattingConversionServiceFactoryBean" />
  <mvc:annotation-driven conversion-service="mvcConversionService" />
  ```
- **`web.xml`:** point `DispatcherServlet`'s `contextConfigLocation` at `dispatcher-config.xml`;
  keep `application-config.xml` for the `ContextLoaderListener`.

### 5. Swagger UI page
Place `index.html` under `src/main/webapp/resources/swagger/`. Point it at the spec and WebJar
assets with paths relative to `/swagger/`:
- spec URL: `api-docs`
- assets: `../webjars/swagger-ui/...`

### 6. Pull the version from the pom (build-time filtering)
`src/main/resources` is NOT filtered by default. Declaring `<resources>` replaces the default,
so you need two blocks:
```xml
<build>
  <finalName>...</finalName>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>
      <includes><include>springdoc.properties</include></includes>
    </resource>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>false</filtering>
      <excludes><exclude>springdoc.properties</exclude></excludes>
    </resource>
  </resources>
  <plugins><!-- existing plugins --></plugins>
</build>
```
Prefer **build-time filtering** over reading the WAR manifest's `Implementation-Version`:
it resolves at package time, works identically on every Tomcat, and can't return null in an
exploded deployment. Verify after build: `target/classes/springdoc.properties` shows a real
version, not `${project.version}`.

### 7. Hide endpoints you don't want documented
Add `@Hidden` (`io.swagger.v3.oas.annotations.Hidden`) to a controller class or method.
Note: `@Hidden` only removes it from the OpenAPI doc; the endpoint still works and still shows
in Spring's handler mappings.

### 8. Annotate for readable docs
- `@Tag(name=..., description=...)` on each controller class.
- `@Operation(summary=..., description=...)` on each endpoint.
- (Optional next step) `@Parameter` descriptions on request params.

---

## Pitfalls & fixes (in the order they typically appear)

| Symptom | Cause | Fix |
|---|---|---|
| `NoSuchBeanDefinitionException: No bean named 'mvcConversionService'` from **`ContextLoader`** (root) | Same XML loaded in both contexts; springdoc initializes in root where no MVC conversion service exists | Split contexts (step 4). Ensure root scan excludes `@Configuration`. |
| Same error but from **`DispatcherServlet`** (`FrameworkServlet`) | XML `<mvc:annotation-driven>` registers its conversion service under a generated name, not `mvcConversionService` | Declare a `mvcConversionService` bean and wire it via `conversion-service=` (step 4). |
| Old error persists after a fix | **Stale build** — `target/.../WEB-INF/classes` still has old XML | `mvn clean package`; in Eclipse also Server → Clean + Clean Tomcat Work Directory; delete stale exploded dirs. |
| `Handler dispatch failed: NoClassDefFoundError: jakarta/xml/bind/annotation/XmlRootElement` when hitting `/swagger/api-docs` | JAXB API pinned to 2.3.x (`javax` namespace) | Bump `jakarta.xml.bind-api` to 4.0.2 and `jaxb-runtime` to 4.0.5. |
| `BeanCreationException` on `ehcache` → `NoClassDefFoundError: javax/xml/bind/ValidationEventHandler` | Default Ehcache classifier uses `javax.xml.bind` while the rest is on JAXB 4 | Add `<classifier>jakarta</classifier>` to the ehcache dependency. |
| `Failed to load API definition` in Swagger UI, but UI chrome loads | The spec request (`/swagger/api-docs`) returns 404/500 | Open the spec URL directly; fix based on its status (see below). |
| `/swagger/api-docs` 404 | springdoc endpoint not registered | Confirm `OpenApiConfig` is scanned by the **dispatcher** context; confirm the `@Import` list. |
| `/swagger/api-docs` 500 | springdoc fails building spec | Read the Tomcat stack trace; usually a JAXB/Jackson version clash or an unresolvable controller param type. |
| `${project.version}` shows literally at runtime | Resource filtering didn't run (e.g. Eclipse publish) | Use Maven build; verify `target/classes/springdoc.properties`. |
| Behind a reverse proxy (e.g. `http://host/uat/imt/<app>/`), the spec's server URL is just `http://host/` and "Try it out" calls fail | springdoc builds the server URL from the request it sees, which lacks the proxy's path prefix | Set a relative server in the `OpenAPI` bean: `.servers(List.of(new Server().url("../")))` (step 2). |
| Flood of `DEBUG ModelConverterContextImpl.resolve` (and other `io.swagger`) lines | No `io` logger configured; falls through to root which defaults to DEBUG in log4j 1.x | Add an `io` logger at WARN, or set an explicit `<level>` on `<root>`. |
| Flood of `DEBUG ModelConverterContextImpl.resolve` (and other `io.swagger.v3.core.*`) lines | springdoc/swagger-core classes live under `io.swagger.*`, uncovered by any `com`/`org`/`net` logger, so they fall through to root (DEBUG in log4j 1.x) | Add an explicit `io` logger at WARN in `log4j.xml`: `<logger name="io"><level value="WARN" /></logger>` |

### Diagnosing a double context load
Seeing `Mappings.detectHandlerMethods` **twice** is usually normal: Spring logs one block
**per controller bean**, and springdoc's own `OpenApiWebMvcResource` is itself a controller, so
it gets its own block. To actually confirm there's no double load, grep the log for exactly one
of each:
- `Root WebApplicationContext: initialization completed`
- `FrameworkServlet 'dispatcher': initialization completed`

If other once-per-startup lines (data source init, cache init) appear twice, the **app is
deployed twice** (e.g. an Eclipse-published copy plus a WAR/`<Context>` in `server.xml`).

---

## Resulting URLs
Assuming context path `/<app>` and `springdoc.api-docs.path=/swagger/api-docs`:
- Swagger UI: `http://<host>:<port>/<app>/swagger/index.html`
- Spec (JSON): `http://<host>:<port>/<app>/swagger/api-docs`
- Spec (YAML): `http://<host>:<port>/<app>/swagger/api-docs.yaml`

---

## Side effects to keep in mind
- **Controllers move to the child (dispatcher) context.** `@Cacheable`/`@Transactional` placed
  **directly on a controller method** no longer apply unless the relevant
  `cache:annotation-driven` / `tx` config is also in that context. Keep such concerns on
  services in the root context.
- **The swagger-ui WebJar version is duplicated** into the `mvc:resources` path. Change both the
  pom property and the XML path together (or filter `dispatcher-config.xml` too).
- **Test contexts:** if a test XML loads the combined config, exclude `@Configuration` there too
  so tests don't drag in springdoc.

---

## Security / production
It is common practice to disable or restrict Swagger UI in production — it advertises endpoints
and lets anyone invoke them (heavier risk when endpoints run expensive work or mutate state).
Options:
- `springdoc.api-docs.enabled=false` (add to `springdoc.properties`, ideally per-profile),
- restrict `/swagger/**` and `/swagger/api-docs*` at the web server / network layer,
- require authentication.
If the service is internal-only and already behind network controls, leaving it on is a common
accepted tradeoff.

---

## Quick checklist for a new project
- [ ] Add springdoc `-api` + `swagger-ui` WebJar; align JAXB to 4.x; ehcache `jakarta` classifier.
- [ ] Create `OpenApiConfig` with explicit `@Import` of springdoc core config.
- [ ] Add `springdoc.properties` (`api-docs.path`, `app.version=${project.version}`).
- [ ] Split root vs dispatcher contexts; exclude web components from root, include only them in dispatcher.
- [ ] Declare a named `mvcConversionService` and wire it into `<mvc:annotation-driven>`.
- [ ] Add `mvc:resources` for `/swagger/**` and `/webjars/swagger-ui/<version>/**`.
- [ ] Add the Swagger UI `index.html` with correct relative paths.
- [ ] Enable Maven filtering for `springdoc.properties` only (two `<resource>` blocks).
- [ ] `@Tag`/`@Operation` on controllers; `@Hidden` where needed.
- [ ] Clean build + clean republish; verify `/swagger/api-docs` returns JSON.
- [ ] Add an `io` logger at WARN in `log4j.xml` to silence `ModelConverterContextImpl`/swagger-core DEBUG noise.
- [ ] Decide production exposure policy.
