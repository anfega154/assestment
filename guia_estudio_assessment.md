# Guía de estudio — Review de Assessment técnico

**Reunión:** jueves 24 de septiembre de 2026  
**Prioridad:** P0 = brecha directa; P1 = implementación propia que debe defenderse; P2 = profundización; P3 = complemento.  
**Objetivo:** llegar a nivel 4–5 en los conceptos centrales: definir, explicar internamente, conectar con la implementación, justificar decisiones y discutir límites.

---

# 1. Spring Boot, IoC y Beans — P0

## Definición

IoC significa que la aplicación no crea manualmente todas sus dependencias: el contenedor de Spring construye los objetos, resuelve sus relaciones y administra su ciclo de vida. Un bean es un objeto registrado en ese contenedor.

## Cómo funciona

1. Spring crea el `ApplicationContext`.
2. Descubre configuración (`@Configuration`, auto-configuración, component scanning).
3. Registra definiciones de beans.
4. Resuelve dependencias, preferentemente por constructor.
5. Crea singletons, aplica post-processors/proxies y ejecuta callbacks de inicialización.
6. Al apagar, invoca callbacks de destrucción de beans administrados.

`BeanFactory` es el contenedor mínimo. `ApplicationContext` lo amplía con eventos, perfiles, internacionalización, recursos y auto-configuración.

## Ejemplo

```java
@Bean
@Primary
R2dbcEntityTemplate writeEntityTemplate(
    @Qualifier("writeConnectionFactory") ConnectionFactory factory) {
  return new R2dbcEntityTemplate(factory);
}
```

## Dónde lo utilicé

- `gift-card-back-auth/.../R2dbcConfig.java`: beans de lectura/escritura, `@Qualifier` y `@Primary`.
- `gift-card-back-auth/.../CognitoService.java`: `@Service` con inyección por constructor.
- `Franquicias/.../UseCasesConfig.java`: component scan selectivo de clases `*UseCase`.

## Por qué lo utilicé

Para desacoplar creación y uso, escoger correctamente entre dos `ConnectionFactory`, centralizar configuración y permitir sustituir adapters en pruebas.

## Alternativas

- Construcción manual con `new`: explícita, pero escala mal y acopla wiring.
- CDI/Jakarta o Guice: contenedores alternativos.
- Una sola conexión R2DBC: más simple, pero sin separación lectura/escritura.

## Trade-offs

- Ventaja: bajo acoplamiento, configuración central y testabilidad.
- Costo: errores de wiring aparecen al iniciar; demasiada magia dificulta diagnóstico.
- `@Primary` define preferencia global; `@Qualifier` hace explícita la selección local.

## Error común

Usar field injection, crear objetos con `new` fuera de configuración y esperar que Spring les inyecte dependencias, o declarar dos beans del mismo tipo sin criterio de selección.

## Pregunta típica de assessment

**¿Qué diferencia existe entre `@Bean` y `@Component`?**  
`@Component` marca una clase descubierta por scanning. `@Bean` registra el objeto retornado por un método de configuración y sirve especialmente para SDKs o clases que no controlo.

## Pregunta avanzada

**¿Qué ocurre si existen dos `ConnectionFactory` y no uso `@Qualifier` ni `@Primary`?**  
Spring no puede decidir y normalmente lanza `NoUniqueBeanDefinitionException`. `@Primary` da un default; `@Qualifier` selecciona por intención.

---

# 2. Seguridad: Cognito, JWT y contexto tenant — P0/P1

## Definición

Cognito autentica usuarios y emite tokens. Un JWT contiene header, payload y firma. Los claims describen identidad, audiencia, emisor, expiración y atributos como tenant.

## Cómo funciona

1. El backend invoca `ADMIN_USER_PASSWORD_AUTH` con `SECRET_HASH`.
2. Cognito devuelve tokens o un challenge como `NEW_PASSWORD_REQUIRED`.
3. El flujo extrae `tenant_id/profile` y crea un `TenantContext` Reactor.
4. La consulta del usuario ocurre bajo el schema tenant y valida estado activo.
5. El refresh token solicita nuevos tokens sin reenviar contraseña.

Validar JWT implica verificar firma con JWKS, algoritmo permitido, `iss`, `aud/client_id`, `exp` y otros constraints. Base64-decodificar el payload **no** valida el token.

## Ejemplo

```java
return cognitoGateway.login(user, password)
  .flatMap(this::procesarRespuestaCognito)
  .contextWrite(tenantContext::writeToContext);
```

## Dónde lo utilicé

- `IniciarSesionUseCase.java`: login, usuario activo, contexto tenant.
- `CognitoService.java`: password auth, challenge, refresh y global sign-out.
- `DecodificadorTokenService.java`: extracción de claims del token recién emitido.
- Librería compartida `TenantJwtWebFilter`: validación JWKS para requests entrantes (autoría de equipo).

## Por qué lo utilicé

Para delegar credenciales a Cognito, propagar tenant sin headers manipulables y seleccionar el schema correcto en un flujo reactivo.

## Alternativas

- Spring Authorization Server/Keycloak.
- API key para integraciones máquina a máquina.
- OAuth2 client credentials para servicios.
- mTLS para identidad de transporte.

## Trade-offs

Cognito reduce operación de identidad, pero introduce dependencia del proveedor y complejidad de claims/challenges. El contexto Reactor preserva datos por suscripción, pero se pierde si se sale del pipeline o se usa `ThreadLocal`.

## Error común

Registrar tokens/contraseñas, aceptar algoritmo del header sin restricciones, no validar expiración, usar un claim tenant sin verificar firma o confundir refresh token con un segundo mecanismo de autenticación.

## Pregunta típica

**¿Qué diferencia hay entre autenticación y autorización?**  
Autenticación prueba quién eres; autorización decide qué puedes hacer. Cognito autentica, y roles/claims/políticas controlan autorización.

## Pregunta avanzada

**¿La implementación demuestra dos mecanismos de autenticación?**  
Demuestra varios flujos Cognito (password, challenge, refresh) y consumo JWT. No conviene afirmar dos mecanismos independientes sin acordar la definición del evaluador.

---

# 3. Reactor, Reactive Streams y WebFlux — P0

## Definición

Reactive Streams es un contrato para flujo asíncrono con backpressure: `Publisher`, `Subscriber`, `Subscription` y `Processor`. Reactor lo implementa con `Mono` (0..1) y `Flux` (0..N). WebFlux integra ese modelo con HTTP.

## Cómo funciona

El pipeline es una descripción lazy. Al suscribirse, la demanda viaja upstream y las señales (`onNext`, `onError`, `onComplete`) viajan downstream. Netty usa pocos event-loop threads; por eso una operación bloqueante puede frenar muchas solicitudes.

## Ejemplo

```java
archivoPort.leer(key)
  .buffer(100)
  .concatMap(batch -> procesarLote(batch))
  .then(Mono.defer(this::resumir));
```

## Dónde lo utilicé

- `EjecucionRecargaMasivaUseCase`.
- `ArchivoRecargaMasivaS3Adapter`.
- `Franquicias Handler/ProductUseCase`.
- Tests con `StepVerifier`.

## Por qué lo utilicé

Para componer S3, R2DBC y APIs externas sin reservar un hilo por espera, y para procesar datos por demanda.

## Alternativas

- Spring MVC + virtual threads: modelo imperativo simple con alta concurrencia de I/O.
- `CompletableFuture`: async sin el vocabulario completo de streams/backpressure.
- Batch clásico: apropiado para jobs con checkpoints y ventanas.

## Trade-offs

WebFlux rinde mejor cuando toda la cadena es no bloqueante y hay alta espera I/O. Añade complejidad de debugging/contexto y no acelera automáticamente tareas CPU-bound.

## Error común

Llamar `block()` dentro del event loop, usar JDBC o SDK síncrono directamente, hacer `subscribe()` en una capa intermedia o suponer que “reactivo = paralelo”.

## Pregunta típica

**¿Quién implementa Reactive Streams?**  
Reactive Streams define interfaces/especificación. Project Reactor es una implementación; Spring WebFlux la integra.

## Pregunta avanzada

**¿Cómo funciona un `Mono` internamente?**  
Es un `Publisher` especializado 0..1. Cada operador envuelve al publisher anterior; al suscribirse se construye una cadena de subscribers y la demanda se propaga upstream. El valor/error/completion fluye downstream.

---

# 4. Concurrencia, paralelismo y schedulers — P0

## Definición

- **Concurrencia:** varias tareas progresan solapadas.
- **Paralelismo:** varias tareas se ejecutan literalmente al mismo tiempo en múltiples núcleos.
- **Proceso:** espacio de memoria aislado.
- **Hilo:** unidad de ejecución dentro del proceso.

## Cómo funciona

El event loop multiplexa eventos. `subscribeOn` afecta dónde se realiza la suscripción/upstream; `publishOn` cambia el contexto de ejecución a partir de ese punto. `boundedElastic` está destinado a trabajo bloqueante tolerado; `parallel` a trabajo CPU-bound corto.

## Ejemplo

```java
Mono.fromCallable(() -> cognitoClient.adminInitiateAuth(request))
    .subscribeOn(Schedulers.boundedElastic());
```

## Dónde lo utilicé

- `CognitoService`: aislamiento de SDK síncrono.
- `ProcesadorRecargaMasivaProgramado`: `AtomicBoolean` y `doFinally`.
- `EjecucionRecargaMasivaUseCase`: procesamiento secuencial controlado.

## Por qué lo utilicé

Para no bloquear el event loop y evitar ejecuciones solapadas del worker local.

## Alternativas

- SDK AWS async.
- Virtual threads con Spring MVC.
- `flatMap(fn, concurrency)` si el downstream es idempotente y tolera concurrencia.
- Lock distribuido/claim atómico para varias réplicas.

## Trade-offs

`boundedElastic` protege Netty pero no elimina bloqueo. `concatMap` reduce throughput potencial a cambio de orden y presión controlada. Un `AtomicBoolean` no coordina varios pods.

## Error común

Usar `parallel()` sin necesidad, introducir trabajo bloqueante en `map/flatMap`, aumentar concurrencia sin límites o asumir orden con `flatMap`.

## Pregunta típica

**¿Diferencia entre `flatMap` y `concatMap`?**  
`flatMap` mezcla resultados y permite concurrencia; `concatMap` espera cada publisher y conserva orden.

## Pregunta avanzada

**¿Cuándo usarías `boundedElastic`?**  
Solo como frontera temporal para una API inevitablemente bloqueante. Definiría timeout y capacidad, observaría saturación y migraría a un cliente async si es posible.

---

# 5. Procesamiento masivo, streaming y backpressure — P0/P1

## Definición

Procesar masivamente no significa cargar todo en memoria. Significa consumir incrementalmente, limitar elementos en vuelo y persistir progreso/fallos.

## Cómo funciona

S3 entrega `ByteBuffer` asíncronos. El adapter conserva solo bytes de la línea incompleta, emite registros y el caso de uso agrupa 100. `concatMap` procesa un lote por vez. Cada resultado se persiste; al final se resume el estado.

## Ejemplo

```java
Flux.from(responsePublisher)
  .concatMapIterable(this::extraerLineas)
  .map(this::deserializarRegistro);
```

## Dónde lo utilicé

`ArchivoRecargaMasivaS3Adapter` + `EjecucionRecargaMasivaUseCase`.

## Por qué lo utilicé

Para acotar memoria, reducir consultas N+1 mediante lotes y continuar frente a errores individuales.

## Alternativas

- Spring Batch con chunk/checkpoint.
- SQS por registro.
- AWS Batch/Glue para escala offline.
- Concurrencia limitada con `flatMap`.

## Trade-offs

La lectura es streaming, pero la subida inicial serializa una lista completa a `String`. `buffer(100)` acota lotes, aunque cada lote consulta y procesa secuencialmente. El resultado parcial mejora continuidad, pero exige semántica clara de reintento.

## Error común

Hacer `collectList()` sobre todo el archivo, reintentar el lote completo sin idempotencia o perder el registro de qué elementos ya fueron exitosos.

## Pregunta típica

**¿Qué pasa si falla el registro 500 de 10.000?**  
El diseño materializa el fallo técnico como resultado individual, lo persiste y continúa. El estado final puede ser PARCIAL. Un retry selectivo debe usar una clave idempotente.

## Pregunta avanzada

**¿Cómo evitarías saturar Redeban?**  
Concurrencia explícita baja, rate limiter, timeouts, circuit breaker, backoff con jitter, métricas de latencia/429/5xx y capacidad acordada con el proveedor.

---

# 6. R2DBC, transacciones y multitenancy — P1

## Definición

R2DBC ofrece acceso relacional no bloqueante mediante `Publisher`, a diferencia de JDBC, que bloquea el hilo durante I/O.

## Cómo funciona

Una `ConnectionFactory` crea conexiones reactivas. El tenant viaja en Reactor Context; el wrapper aplica `search_path` al adquirir conexión y lo restablece al devolverla. `R2dbcTransactionManager` enlaza la transacción al contexto reactivo.

## Ejemplo

```java
TenantContext.fromContext()
  .flatMap(ctx -> Mono.from(delegate.create())
    .flatMap(conn -> applySearchPath(conn, ctx.resolveSchemaName())));
```

## Dónde lo utilicé

- `gift-card-back-auth/R2dbcConfig`.
- Integraciones multitenant en Kire.

## Por qué lo utilicé

Para conservar cadena no bloqueante y aislar datos por schema.

## Alternativas

- Base por tenant.
- Columna `tenant_id` con row-level security.
- JDBC + virtual threads.

## Trade-offs

R2DBC tiene menos ecosistema que JPA, manejo transaccional distinto y exige que el contexto no se pierda. `search_path` requiere sanitización y reset para evitar fuga entre conexiones del pool.

## Error común

Construir schema con texto no validado, olvidar reset, iniciar transacción fuera del pipeline o mezclar JDBC en el event loop.

## Pregunta típica

**¿Cómo manejarías una transacción reactiva?**  
Con `TransactionalOperator` o `@Transactional` sobre un método que devuelve `Publisher`, sin `subscribe()` interno y manteniendo todas las operaciones en la misma cadena.

## Pregunta avanzada

**¿Una transacción cubre Redeban + PostgreSQL?**  
No. Es una transacción distribuida. Se requieren idempotencia, estados, outbox/saga o compensaciones; no ACID global.

---

# 7. Redis y estrategias de caché — P0 (brecha abierta)

## Definición

La caché guarda datos más cerca del consumidor para reducir latencia/carga. Redis es un almacén distribuido en memoria con strings, hashes, lists, sets, sorted sets, streams y otras estructuras.

## Cómo funciona

- **Cache-aside:** aplicación lee cache; si miss, lee fuente y llena cache.
- **Read-through:** la capa de cache carga la fuente.
- **Write-through:** escritura sincroniza cache y fuente antes de responder.
- **Write-behind:** cache responde y persiste después; mayor riesgo de pérdida.
- **Refresh-ahead:** renueva antes de expirar.

## Ejemplo encontrado

`UsuarioVentaMasivaRedisService` usa `opsForValue`, key por venta y TTL de 15 días. `RedisCacheService` invalida por `SCAN` y `delete`.

## Dónde lo utilicé

**EVIDENCIA INSUFICIENTE de autoría propia.** El código existe en Kire, pero la implementación principal pertenece a otros autores.

## Por qué se utilizaría

Para evitar consultas repetidas, compartir datos entre réplicas y conservar información efímera.

## Alternativas

Caffeine local, Memcached, DynamoDB DAX, materialized views o ninguna cache si la fuente cumple SLO.

## Trade-offs

Cache local es rápida pero inconsistente entre instancias; distribuida agrega red/operación. TTL limita stale data, no garantiza coherencia. Invalidación por patrón puede ser costosa.

## Error común

Sin TTL, keys sin namespace, `KEYS` en producción, cache stampede, datos sensibles sin controles o tratar cache como fuente de verdad.

## Pregunta típica

**¿Cómo evitarías cache stampede?**  
Single-flight/lock por key, TTL con jitter, stale-while-revalidate, precarga y límites de concurrencia.

## Pregunta avanzada

**¿Qué estructura usarías para ranking?**  
Sorted Set: miembro + score, consultas de rango eficientes. Para objeto con campos parciales usaría Hash; para unicidad, Set.

---

# 8. Docker, Compose, Kubernetes y ECS — P0

## Definición

Un contenedor empaqueta proceso, runtime y filesystem aislado. Docker construye/ejecuta imágenes; Compose coordina servicios locales; Kubernetes/ECS orquestan despliegue, red, salud y escala.

## Cómo funciona

El Dockerfile de Franquicias compila en una etapa JDK y copia solo el JAR a un JRE. Ejecuta como usuario no root y limita heap con `MaxRAMPercentage`. Compose conecta app y Mongo en una red y usa volumen/healthchecks.

## Ejemplo

```dockerfile
FROM gradle:8.14.3-jdk21-alpine AS builder
RUN ./gradlew clean bootJar
FROM eclipse-temurin:21-jre-alpine
COPY --from=builder ... /app/app.jar
USER appuser
```

## Dónde lo utilicé

- `Franquicias/deployment/Dockerfile`.
- `Franquicias/deployment/docker-compose.yml`.
- ASULADO `deployment/k8s` como experiencia de equipo.

## Por qué lo utilicé

Para builds reproducibles, entorno local completo y despliegue ECS consistente.

## Alternativas

Buildpacks/Jib, Distroless, Kubernetes, Lambda o App Runner.

## Trade-offs

Alpine es pequeño, pero usa musl y puede presentar incompatibilidades. Distroless reduce superficie, pero complica debugging. Multi-stage mejora tamaño y separación; no reemplaza escaneo de CVEs.

## Error común

Ejecutar como root, copiar el repositorio completo, tags `latest` en producción, secretos en `ENV`, no definir healthcheck o construir ARM64 y desplegar en runtime AMD64.

## Pregunta típica

**¿Docker vs Kubernetes?**  
Docker construye/ejecuta contenedores; Kubernetes coordina muchas réplicas, despliegues, configuración, red, salud y escalado.

## Pregunta avanzada

**¿Por qué no Distroless aquí?**  
Se eligió Alpine JRE por simplicidad y herramientas mínimas. Distroless sería una mejora de hardening si se resuelve observabilidad/debug y compatibilidad.

---

# 9. Step Functions, idempotencia y resiliencia — P1

## Definición

Step Functions expresa una orquestación durable como máquina de estados. `Task` ejecuta trabajo; `Choice` decide; `Pass` transforma; `Fail/Succeed` terminan; `Map/Parallel` procesan colecciones/ramas; `Wait` pausa.

## Cómo funciona

La state machine normaliza el evento, valida estado, consulta idempotencia, adquiere lock, enruta por tipo, invoca Lambda/HTTP/SQS y libera lock. `Retry` maneja errores transitorios; `Catch` dirige fallos a persistencia/limpieza.

`InputPath` selecciona entrada, `Parameters` construye input, `ResultSelector` transforma resultado, `ResultPath` lo combina y `OutputPath` filtra salida.

## Ejemplo

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::http:invoke",
  "Retry": [{"IntervalSeconds": 2, "BackoffRate": 2, "MaxAttempts": 3}],
  "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "PersistSingleError"}]
}
```

## Dónde lo utilicé

`state-machine-novedades/statemachine/novedades.asl.json` y commits `c00c265`, `f665417`, `2c72a17`, `a534c14`, `892ad67`.

## Por qué lo utilicé

Para hacer visible/durable un proceso multietapa, mantener retries/errores fuera de Lambdas y auditar decisiones.

## Alternativas

Orquestador dentro de un servicio, Temporal/Camunda, coreografía por eventos o una Lambda monolítica.

## Trade-offs

Gana observabilidad y estado durable; añade costo por transición, límites de payload/ejecución y acoplamiento AWS. Express sirve alto volumen/corta duración; Standard ofrece historial durable y ejecución larga.

## Error común

Retry a errores 4xx, no liberar locks, payloads crecientes, no usar idempotency key o asumir exactly-once.

## Pregunta típica

**¿Por qué Step Functions y no una Lambda?**  
Porque hay decisiones, esperas/callbacks, varios servicios y manejo de fallos. Una Lambda larga concentraría estado, retries y observabilidad.

## Pregunta avanzada

**¿Cómo evitas retry storms?**  
Clasificación de errores, backoff exponencial con jitter, máximo de intentos, circuit breaker/rate limits downstream, DLQ y alarmas.

---

# 10. Clean/Hexagonal, SOLID y patrones — P1

## Definición

Clean/Hexagonal coloca reglas de negocio en el centro. Puertos expresan lo que el dominio necesita; adapters implementan HTTP, DB o AWS. Las dependencias apuntan hacia adentro.

## Cómo funciona

`Handler` transforma HTTP a dominio y llama un input port. El use case depende de repositorios/gateways. Mongo implementa esos puertos. Spring wiring conecta las piezas.

## Ejemplo

```text
HTTP Handler → ProductInputPort → ProductUseCase → ProductRepository ← Mongo Adapter
```

## Dónde lo utilicé

Franquicias y Kire: módulos `domain/model`, `domain/usecase`, `infrastructure`, `applications`.

## Por qué lo utilicé

Para probar reglas sin Spring/Mongo y cambiar tecnología sin reescribir dominio.

## Alternativas

Arquitectura por capas tradicional, vertical slices o modular monolith. Hexagonal y Clean comparten inversión; difieren más en énfasis/vocabulario que en objetivo.

## Trade-offs

Más interfaces/mapeos y complejidad inicial; beneficio cuando hay integraciones, evolución y pruebas. Para un CRUD trivial puede ser sobrearquitectura.

## Error común

Poner DTO HTTP o anotaciones de persistencia en dominio, crear gateways “por cada método” sin abstracción útil o afirmar que carpetas por sí solas garantizan arquitectura.

## Pregunta típica

**¿Dónde aplicas DIP?**  
`ProductUseCase` depende de `ProductRepository` del dominio; el adapter Mongo depende e implementa ese contrato.

## Pregunta avanzada

**¿Qué patrón aparece en validaciones de recarga masiva?**  
Strategy: cada regla implementa un contrato; el contexto compone estrategias. Agrega reglas sin modificar el orquestador, favoreciendo OCP.

---

# 11. Testing reactivo y TDD — P1

## Definición

`StepVerifier` se suscribe al publisher y verifica secuencia de señales, valores, error y completion. Mockito aísla puertos; WebTestClient prueba endpoints reactivos.

## Cómo funciona

La prueba configura publishers controlados, crea el flujo y declara expectativas. Nada sucede antes de la suscripción de `StepVerifier`.

## Ejemplo

```java
StepVerifier.create(useCase.procesarSiguiente())
  .expectError(IllegalStateException.class)
  .verify();
```

## Dónde lo utilicé

- `EjecucionRecargaMasivaUseCaseTest`: lotes, continuidad, errores y estado parcial.
- `IniciarSesionUseCaseTest`: challenge, usuario inactivo y fallos Cognito.
- Franquicias: use cases, handlers, router y adapters.
- Lambda datos bancarios: 39 pruebas pytest.

## Por qué lo utilicé

Para probar semántica temporal/lazy y señales, no solo el valor final.

## Alternativas

Testcontainers para integración real, WireMock/MockWebServer, LocalStack, contract tests y pruebas de carga.

## Trade-offs

Mocks son rápidos, pero pueden validar una fantasía. Tests de integración dan confianza a mayor costo. TDD es disciplina de diseño, no sinónimo de cobertura.

## Error común

Llamar `block()` en pruebas como única verificación, olvidar `verifyComplete`, no probar errores/cancelación o afirmar TDD solo porque existen tests.

## Pregunta típica

**¿Qué valida StepVerifier?**  
Las señales del publisher en orden: emisiones, error, completion y, con tiempo virtual, comportamiento temporal.

## Pregunta avanzada

**¿Cómo probarías backpressure?**  
Con demanda inicial 0, `thenRequest(n)`, expectativas por bloques y verificación de que el publisher no emite sin demanda.

---

# 12. AWS, Terraform y observabilidad — P1/P2

## Definición

Terraform declara infraestructura deseada; el provider calcula cambios. En Franquicias, API Gateway es entrada pública, VPC Link conecta al ALB interno y ECS Fargate ejecuta el contenedor en subred privada.

## Cómo funciona

Security groups encadenan solo el tráfico necesario. Secrets Manager inyecta `MONGO_URI`. ECR escanea imágenes. CloudWatch recibe logs de ECS/API Gateway y S3 guarda access logs del ALB. El state remoto se protege con versionado/encryption y lock DynamoDB.

## Dónde lo utilicé

`Franquicias/deployment/terraform` y README operativo.

## Por qué lo utilicé

Para reproducibilidad, revisión por diff, aislamiento de red, secretos fuera de imagen y troubleshooting.

## Alternativas

CloudFormation/SAM/CDK, App Runner, EKS o Lambda.

## Trade-offs

NAT Gateway y ALB tienen costo fijo; API Gateway agrega capa/costo. ECS es más simple que EKS, pero menos portable. `MUTABLE` tags facilitan desarrollo, aunque producción debería preferir tags/digests inmutables.

## Error común

State local sin locking, secretos en tfvars, SG abiertos, tareas con IP pública, logs sin retención o despliegues con `latest`.

## Pregunta típica

**¿IaaS, PaaS y SaaS?**  
IaaS entrega infraestructura base; PaaS administra más runtime/plataforma; SaaS entrega aplicación. ECS Fargate es cómputo administrado cercano a PaaS/CaaS; S3/Secrets Manager son servicios administrados.

## Pregunta avanzada

**¿Qué costo cuestionarías?**  
NAT por hora/datos, ALB, API Gateway por request, logs y Fargate. Para poco tráfico consideraría App Runner/Lambda o endpoints VPC para reducir NAT según necesidades.

---

# 13. Protocolos e integraciones — P2

## Definición

- REST: request/response sobre HTTP, stateless.
- WebSocket: canal persistente bidireccional.
- WebHook: callback HTTP iniciado por el productor.
- AMQP/MQTT/STOMP/gRPC/GraphQL/RSocket: modelos distintos; no afirmar implementación sin código.

## Dónde lo utilicé

REST es amplio en Java/Python. El assessment previo acredita WebSocket. `smartpay-arquitectura.html` documenta API Gateway WebSocket + DynamoDB connection IDs, pero debe usarse como apoyo y no como nueva autoría demostrada.

## Trade-offs

WebSocket sirve tiempo real, pero requiere gestionar conexiones, escalado, heartbeats y cleanup. WebHook es más simple entre servidores, pero necesita firma, retry e idempotencia. REST es universal, aunque no ofrece push nativo.

## Pregunta avanzada

**¿Cuándo no usarías WebSocket?**  
Cuando las actualizaciones son esporádicas, un WebHook/SSE/polling cumple, o la infraestructura no necesita conexión bidireccional.

---

# Plan de estudio: 21–24 septiembre de 2026

## Lunes 21 — construir la defensa (3–4 h)

1. **P0 (60 min):** Spring IoC/beans; explicar `R2dbcConfig` sin notas.
2. **P0 (75 min):** Reactor/concurrencia; dibujar request → event loop → boundedElastic.
3. **P1 (60 min):** recarga masiva; recorrer S3 → buffer → proceso → resultado parcial.
4. **P1 (30 min):** ejecutar/leer las pruebas y memorizar solo las cifras verificadas.
5. **Salida:** respuestas de 90 segundos para Spring y concurrencia.

## Martes 22 — brechas restantes (3–4 h)

1. **P0 (60 min):** Dockerfile, capas, multi-stage, usuario no root.
2. **P0 (45 min):** Compose vs Kubernetes vs ECS.
3. **P0 (90 min):** Redis: 5 estrategias, local/distribuida, 5 estructuras, stampede.
4. **P1 (45 min):** Terraform Franquicias: flujo de red y seguridad.
5. **Salida:** reconocer caché como pendiente y proponer experimento concreto.

## Miércoles 23 — profundidad y simulación (4 h)

1. **P1 (75 min):** Step Functions; explicar 5 estados propios y Retry/Catch.
2. **P1 (60 min):** JWT/Cognito/R2DBC multitenant.
3. **P2 (45 min):** trade-offs WebFlux vs MVC/virtual threads; Step Functions vs coreografía.
4. **P0 (2 × 30 min):** dos simulaciones cronometradas.
5. **Salida:** lista de preguntas que aún producen dudas; resolver solo las P0/P1.

## Jueves 24 — consolidar (45–60 min)

1. **15 min:** cheat sheet.
2. **15 min:** rutas/commits de las cinco evidencias.
3. **15 min:** riesgos y “qué no afirmar”.
4. **10 min:** apertura y cierre.
5. No estudiar temas nuevos.

---

# Banco de preguntas

## 20 preguntas generales

1. **¿Cuál fue tu principal evolución?** Pasar de conceptos a decisiones defendibles con código, pruebas y límites conocidos.
2. **¿Qué brechas trabajaste?** Spring/Beans, concurrencia, contenedores; caché sigue parcial.
3. **¿Cuál evidencia es más fuerte?** Recarga masiva porque integra Reactor, S3, lotes, fallos parciales y pruebas.
4. **¿Cómo verificas autoría?** Historial Git por archivo/commit, no solo presencia en el repo.
5. **¿Qué resultado puedes medir?** Validaciones locales: 162 + 71 pruebas Java y 39 Python; no métricas productivas.
6. **¿Qué aprendiste de un error?** Que una solución reactiva puede bloquear si usa SDK síncrono sin frontera.
7. **¿Qué harías diferente?** Cliente async, concurrencia medida, idempotencia reforzada y métricas.
8. **¿Cómo decides una tecnología?** Problema/SLO, constraints, operación, equipo, costo y reversibilidad.
9. **¿Cómo comunicas riesgo?** Probabilidad, impacto, detección y mitigación con evidencia.
10. **¿Qué no está consolidado?** Caché avanzada con autoría propia.
11. **¿Cómo orientas a otro desarrollador?** Contrato, ejemplo mínimo, guardrails, pruebas y revisión.
12. **¿Qué evidencia muestra autonomía?** Terraform completo, flujo masivo y ampliaciones de state machine.
13. **¿Cómo evitas un inventario de tecnologías?** Caso → problema → decisión → implementación → resultado → aprendizaje.
14. **¿Qué trade-off te parece más importante?** Orden/control con `concatMap` frente a throughput con `flatMap`.
15. **¿Qué decisión revertirías?** Evaluaría reemplazar cliente Cognito síncrono por async.
16. **¿Qué deuda aceptaste?** Upload de CSV aún materializa contenido; lock local no cubre réplicas.
17. **¿Cómo sabes que una arquitectura funciona?** Tests, reglas de dependencia, observabilidad y facilidad de cambio.
18. **¿Qué evidencia es compartida?** Librería tenant y manifiestos Kubernetes.
19. **¿Qué significa nivel 5?** Diseñar, anticipar fallos, justificar trade-offs y orientar; no solo implementar.
20. **¿Por qué no pides directamente un 5?** La valoración debe emerger de la evidencia y del criterio demostrado.

## 20 preguntas técnicas

1. **`map` vs `flatMap`:** transformación síncrona vs composición de publisher.
2. **`flatMap` vs `concatMap`:** concurrencia/orden no garantizado vs secuencia ordenada.
3. **`defer`:** crea publisher/valor por suscripción; evita ejecución eager.
4. **`switchIfEmpty`:** ruta alternativa cuando completa sin valor, no cuando hay error.
5. **`onErrorResume` vs `onErrorMap`:** recupera con publisher alterno vs transforma el error.
6. **`materialize`:** convierte señales en objetos `Signal` para tratarlas como datos.
7. **Cold vs hot:** cold reinicia por suscriptor; hot comparte emisión independiente.
8. **Backpressure:** subscriber controla demanda mediante `request(n)`.
9. **`buffer` vs `window`:** lista materializada vs sub-flujos.
10. **`zip` vs `merge`:** combina posiciones/resultados vs intercala emisiones.
11. **`subscribeOn` vs `publishOn`:** ubicación de suscripción/upstream vs downstream desde el operador.
12. **R2DBC vs JDBC:** protocolo/reactividad no bloqueante vs API bloqueante.
13. **Pool reactivo:** reutiliza conexiones; sigue necesitando límites/timeouts.
14. **Transacción reactiva:** contexto de suscripción, no `ThreadLocal` clásico.
15. **Idempotencia:** repetir produce el mismo efecto observable.
16. **Exponential backoff:** espera creciente entre reintentos.
17. **Jitter:** aleatoriza espera para evitar sincronización masiva.
18. **Circuit breaker:** corta llamadas cuando fallos superan umbral.
19. **Timeout:** límite de espera; no cancela necesariamente trabajo remoto ya iniciado.
20. **Optimistic locking:** detecta conflicto por versión; requiere estrategia de retry/merge.

## 10 preguntas de arquitectura

1. **Clean vs Hexagonal:** mismo objetivo de inversión; Clean enfatiza capas, Hexagonal puertos/adapters.
2. **¿Qué capa conoce a cuál?** Infra conoce dominio; dominio no conoce infraestructura.
3. **¿Por qué gateway en dominio?** Expresa necesidad de negocio sin tecnología.
4. **¿Cuándo evitar Clean Architecture?** CRUD pequeño con baja evolución y costo de ceremonias mayor al beneficio.
5. **¿Saga?** Coordinación de transacciones locales con compensaciones.
6. **Orquestación vs coreografía:** coordinador explícito vs reacción distribuida a eventos.
7. **¿Outbox?** Persistir cambio y evento en la misma transacción local.
8. **¿Cómo versionar contratos?** Compatibilidad hacia atrás, consumer tests y transición gradual.
9. **¿Qué mejora cohesión?** Módulos por capacidad y reglas cerca de sus datos.
10. **¿Cómo pruebas arquitectura?** ArchUnit, boundaries de build, revisión y pruebas contractuales.

## 10 preguntas WebFlux

1. **¿Por qué WebFlux y no MVC?** Cadena I/O reactiva y alta espera; no por moda.
2. **¿Qué pasa si bloqueas event loop?** Se atascan muchas requests que comparten pocos hilos.
3. **¿Qué es event loop?** Hilo que despacha eventos/callbacks sin esperar I/O.
4. **¿Qué es un Publisher?** Fuente que emite según demanda.
5. **¿Qué es Subscription?** Vínculo que permite pedir/cancelar.
6. **¿Cuándo `boundedElastic`?** Frontera para bloqueo inevitable.
7. **¿`subscribe()` manual?** Solo en bordes controlados; framework normalmente se suscribe.
8. **¿Cómo pruebas error signal?** `expectError`/`expectErrorSatisfies`.
9. **¿Cómo controlas concurrencia?** `flatMap(fn, n)`, rate limiter y capacidad downstream.
10. **¿Cómo depuras?** logs con traceId, checkpoints puntuales, métricas y Reactor debug en no producción.

## 10 preguntas AWS

1. **Standard vs Express Step Functions:** durabilidad/larga ejecución vs alto volumen/corta duración.
2. **Task token:** pausa hasta callback correlacionado.
3. **DynamoDB para idempotencia:** conditional write por clave.
4. **SQS Standard:** al menos una vez y orden no garantizado.
5. **DLQ:** aísla mensajes agotados para análisis/reproceso.
6. **API Gateway VPC Link:** conecta entrada gestionada con recurso privado.
7. **Por qué ALB interno:** reduce superficie pública.
8. **Rol de ejecución vs task role ECS:** pull/logs/secrets de plataforma vs permisos de la app.
9. **CloudWatch qué observar:** latencia, error rate, throttling, saturación y logs correlacionados.
10. **Costo Step Functions:** transiciones/ejecuciones; reducir pasos solo si no sacrifica claridad/operación.

## 10 preguntas sobre decisiones

1. **¿Por qué `concatMap`?** Orden y presión controlada sobre Redeban.
2. **¿Por qué lote 100?** Constante explícita y prueba; el valor óptimo requiere medición.
3. **¿Por qué S3 intermedio?** desacopla request de procesamiento y permite recuperación/auditoría.
4. **¿Por qué estado PARCIAL?** representa verdad operacional cuando algunos registros sí se aplicaron.
5. **¿Por qué Cognito?** identidad gestionada y flujos estándar.
6. **¿Por qué dos conexiones R2DBC?** separar lectura/escritura y permitir políticas distintas.
7. **¿Por qué multi-stage?** runtime menor y sin toolchain de build.
8. **¿Por qué API Gateway + ALB?** capacidades de borde con compute privado.
9. **¿Por qué Step Functions?** estado/retry/decisión visibles y durables.
10. **¿Por qué admitir brecha de caché?** honestidad técnica; el dominio incluye saber cuándo falta evidencia.

## Preguntas específicas del código

1. **¿Por qué `materialize()` en recarga?** Convierte éxito/error de Redeban en señal inspeccionable para persistir un resultado por registro y continuar.
2. **¿Qué pasa con una línea CSV dividida entre buffers?** `ByteArrayOutputStream lineaPendiente` conserva bytes hasta recibir salto de línea.
3. **¿Qué limita memoria?** Streaming S3 y `buffer(100)`; la subida original aún recibe una lista completa.
4. **¿Qué evita dos workers en la misma JVM?** `AtomicBoolean.compareAndSet`.
5. **¿Qué evita dos pods?** Debe hacerlo `reclamarSiguiente()` con operación atómica; el `AtomicBoolean` no alcanza.
6. **¿Por qué `then(Mono.defer(...))`?** Ejecuta el resumen solo tras completion y crea la operación al momento correcto.
7. **¿Por qué `@Primary` en escritura?** Define la dependencia por defecto cuando Spring inyecta por tipo.
8. **¿Qué riesgo tiene el token decoder?** No valida firma; solo es seguro en ese punto porque token proviene directamente de Cognito. Requests externos requieren decoder JWKS.
9. **¿Qué hace `ResultPath`?** Inserta el resultado de un Task en una ubicación sin perder necesariamente la entrada.
10. **¿Qué decisión agregó `CheckNextStep`?** Saltar actualización del plan cuando la cuenta no cambió y continuar por la rama necesaria.

---

# CHEAT SHEET — 15 MINUTOS ANTES DEL ASSESSMENT

## Mensaje central

> Convertí brechas de Spring, concurrencia y contenedores en implementaciones reales. Puedo explicar cómo funcionan, por qué tomé las decisiones, dónde fallan y cómo las mejoraría. Caché sigue siendo un frente consciente de consolidación.

## Cuatro brechas

| Brecha | Evidencia | Estado defendible |
|---|---|---|
| Spring Boot | Auth Cognito + R2DBC `@Bean/@Qualifier/@Primary` | Fuerte |
| Concurrencia | S3 async + Reactor por lotes + boundedElastic | Fuerte |
| Contenedores | Docker multi-stage + Compose; exposición K8s | Fuerte/compartida |
| Caché | Redis existe, autoría insuficiente | Pendiente |

## Cinco evidencias

1. Recarga masiva: `buffer(100)` + `concatMap` + estados parciales.
2. Auth: Cognito + contexto tenant + R2DBC read/write.
3. Franquicias: Docker/Compose + Clean + circuit breaker.
4. Terraform: API GW → VPC Link → ALB interno → ECS privado.
5. Step Functions: 52 estados, Lambda/HTTP/SQS/Dynamo, Retry/Catch/locks.

## Diferencias que no puedo confundir

- Concurrente ≠ paralelo.
- Async ≠ non-blocking.
- Decodificar JWT ≠ validar JWT.
- `flatMap` ≠ `concatMap`.
- `subscribeOn` ≠ `publishOn`.
- Docker ≠ Kubernetes.
- Cache local ≠ distribuida.
- Retry ≠ resiliencia completa.
- At-least-once ≠ exactly-once.
- Clean Architecture ≠ solo carpetas.

## Frases clave

- “`boundedElastic` aísla bloqueo; no convierte el SDK en no bloqueante.”
- “Elegí `concatMap` para preservar orden y proteger el downstream; mediría antes de aumentar concurrencia.”
- “El resultado parcial es una decisión de negocio, no un error técnico escondido.”
- “La idempotencia es una propiedad diseñada con claves y estados, no una garantía automática de AWS.”
- “No tengo evidencia suficiente para afirmar cierre completo de caché.”

## Riesgos y mejoras

- Upload CSV materializa string → multipart streaming.
- Lock local → claim atómico/lock distribuido.
- Cliente Cognito sync → async.
- Redis sin autoría propia → laboratorio con cache-aside + Hash/ZSet + stampede tests.
- Tags ECR mutables → digest/tag inmutable.
- Métricas productivas ausentes → OpenTelemetry + dashboards SLO.

## Evidencia cuantitativa verificable

- 162 pruebas usecase Kire: 0 fallos.
- 71 pruebas Franquicias: 0 fallos.
- 39 pruebas Lambda Python: 0 fallos.
- Terraform, Compose y SAM validados.
- 52 estados / 95 transiciones ASL válidas.

## Cierre

> No presento perfección ni una lista de tecnologías. Presento decisiones reales, límites conocidos y capacidad para diseñar la siguiente mejora.
