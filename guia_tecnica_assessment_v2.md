# Guía técnica de estudio — Assessment v2

Esta guía contiene solamente los temas que aparecen en `review_assessment_v2.html`. El objetivo es dominar el funcionamiento interno y responder usando el código mostrado.

## Orden de estudio

1. WebFlux, Reactor y Reactive Streams.
2. Spring DI, beans y proxies.
3. JWT, Reactor Context y multitenancy R2DBC.
4. SQS, EventBridge y fan-out.
5. Step Functions.
6. Docker, Compose y Kubernetes.
7. Redis y cache-aside.
8. Arquitectura limpia.
9. Bases de datos: ACID, TCL, índices, DynamoDB, CAP y BASE.
10. Testing reactivo.

---

# 1. WebFlux, Reactor y Reactive Streams

## Qué es

Spring WebFlux es el stack web reactivo de Spring. Puede operar sobre Netty y representa el trabajo asíncrono con tipos de Reactor.

- `Mono<T>`: publisher que emite cero o un valor y luego termina.
- `Flux<T>`: publisher que emite cero o muchos valores y luego termina.
- Ambos pueden terminar con `onComplete` o `onError`. Después de una señal terminal no existen más señales.

## Cómo funciona

La cadena declarada no comienza a producir datos cuando se construye. Los operadores forman un grafo. La ejecución comienza cuando un Subscriber se suscribe. En WebFlux, el framework realiza esa suscripción al procesar la petición o respuesta.

Reactive Streams define:

- `Publisher`: acepta Subscribers.
- `Subscriber`: recibe `onSubscribe`, `onNext`, `onError` y `onComplete`.
- `Subscription`: representa la relación y expone `request(n)` y `cancel()`.
- `Processor`: combina Subscriber y Publisher.

## Dónde lo usé

```text
gift-card-back-bulk-authorizations
├── ArchivoRecargaMasivaS3Adapter.java
├── EjecucionRecargaMasivaUseCase.java
└── ProcesadorRecargaMasivaProgramado.java
```

El adaptador S3 convierte el `ResponsePublisher<ByteBuffer>` del SDK asíncrono en un `Flux<RegistroRecargaMasiva>`. El caso de uso agrupa registros y procesa los lotes en orden.

## Código

```java
return archivoRecargaMasivaPort.leer(lote.llaveArchivo())
    .buffer(TAMANO_LOTE_PROCESAMIENTO)
    .concatMap(registros -> procesarLote(lote, registros))
    .then(Mono.defer(() -> recargaPort.resumir(lote.id(), lote.tenantId())))
    .flatMap(resumen -> finalizar(lote, resumen))
    .onErrorResume(error -> finalizarDespuesDeError(lote).then(Mono.error(error)));
```

## Qué ocurre internamente

1. WebFlux o el worker ejecuta `subscribe()`.
2. La suscripción recorre la cadena hacia la fuente S3.
3. El `ResponsePublisher` emite buffers cuando existe demanda.
4. El parser conserva bytes incompletos entre buffers y emite líneas.
5. `buffer(100)` acumula hasta cien registros.
6. `concatMap` suscribe un lote y espera su terminación antes del siguiente.
7. Al completar el `Flux`, `then` ejecuta el `Mono` de resumen.
8. Un error fatal cambia la ruta hacia `finalizarDespuesDeError` y luego vuelve a emitir el error original.

## Backpressure

Backpressure es el mecanismo por el cual el Subscriber controla cuántos elementos acepta mediante `request(n)`. No significa limitar threads. Significa regular demanda a través de la cadena.

Un operador puede transformar esa demanda. `buffer(100)` necesita acumular cien valores para emitir una lista. `flatMap` puede tener varios publishers internos activos y aplica prefetch. `concatMap` mantiene uno activo a la vez.

## `map` vs `flatMap`

```java
Flux<String> nombres = usuarios.map(Usuario::getNombre);
```

`map` recibe `Usuario` y devuelve `String`.

```java
Flux<Usuario> usuarios = ids.flatMap(repository::findById);
```

`flatMap` recibe un valor y devuelve un publisher. Después combina las emisiones internas.

Si se usa `map(repository::findById)`, el resultado sería `Flux<Mono<Usuario>>`, no los usuarios.

## `flatMap` vs `concatMap`

| Operador | Publishers internos | Orden | Uso típico |
|---|---:|---|---|
| `flatMap` | varios | no garantizado | I/O independiente y mayor throughput |
| `flatMap(fn, n)` | hasta `n` | no garantizado | concurrencia acotada |
| `concatMap` | uno | garantizado | orden o protección de downstream |
| `flatMapSequential` | varios | reordena salida | concurrencia con orden de entrega |

## `publishOn` vs `subscribeOn`

`subscribeOn(scheduler)` afecta el lugar donde ocurre la suscripción a la fuente. Su posición visual suele importar menos que el primer `subscribeOn` efectivo.

`publishOn(scheduler)` cambia el contexto de ejecución para los operadores que aparecen después.

Ninguno crea paralelismo por sí solo si la fuente o las operaciones no permiten trabajo concurrente.

## `boundedElastic`

`Schedulers.boundedElastic()` mantiene un pool acotado diseñado para trabajo bloqueante que no puede eliminarse. Evita bloquear el event loop, pero agrega threads y no convierte la librería síncrona en no bloqueante.

En Kire, el cliente Cognito síncrono se adapta con `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.

## `parallel`

`parallel()` divide un flujo en rails, pero necesita `runOn(scheduler)` para ejecutar rails en threads del scheduler. Después se usa `sequential()` para volver a un `Flux` normal. Resulta más apropiado para CPU que para llamadas I/O ya asíncronas.

## Trade-offs

- WebFlux escala bien con mucha concurrencia I/O y operaciones no bloqueantes.
- La composición y depuración requieren conocer señales, demanda y schedulers.
- Un `block()`, JDBC o SDK síncrono en el event loop reduce la capacidad de todas las conexiones que comparten ese hilo.
- `concatMap` protege orden y downstream, pero sacrifica throughput.

## Failure modes

- `subscribe()` dentro de un caso de uso crea una segunda cadena sin control del caller.
- `onErrorResume(error -> Mono.empty())` puede convertir un fallo en éxito aparente.
- Un `flatMap` sin límite puede saturar conexiones o servicios externos.
- `collectList()` sobre un stream grande retiene todos los datos.
- `AtomicBoolean` no sincroniza varias réplicas.

## Preguntas típicas

**¿Mono ejecuta inmediatamente?**  
No. Normalmente es lazy. Describe el trabajo hasta que ocurre una suscripción.

**¿Quién realiza subscribe?**  
WebFlux en un endpoint, o código explícito en un boundary como el worker programado. Un caso de uso reutilizable no debería suscribirse internamente.

**¿Qué ocurre si bloqueo el event loop?**  
Ese thread deja de atender otras conexiones. La latencia crece y la aplicación pierde su ventaja de concurrencia I/O.

## Preguntas Senior

**¿Cómo limitarías veinte llamadas externas en paralelo?**  
`flatMap(this::call, 20)` si cada operación es independiente. También limitaría conexiones y agregaría timeouts, retry selectivo y métricas.

**¿Cómo demostrarías backpressure en una prueba?**  
Con `StepVerifier.create(publisher, 0)`, `thenRequest(n)` y aserciones que demuestren que no se emite sin demanda.

---

# 2. Spring DI, beans y proxies

## Qué es

Dependency Injection separa la construcción de objetos de su uso. Spring administra los objetos registrados como beans dentro del `ApplicationContext`.

## Cómo funciona

1. Spring descubre `@Configuration`, `@Component`, `@Service` y `@Repository`.
2. Registra definiciones de beans.
3. Resuelve dependencias por tipo, nombre y qualifiers.
4. Construye instancias y aplica postprocesadores.
5. Puede envolver el bean en un proxy para transacciones, caché, seguridad o aspectos.

## Dónde lo usé

`gift-card-back-auth/infrastructure/driven-adapter/persistence/.../R2dbcConfig.java` configura conexiones separadas de lectura y escritura.

## Código

```java
@Bean
@Primary
public R2dbcEntityTemplate writeEntityTemplate(
        @Qualifier("writeConnectionFactory") ConnectionFactory connectionFactory) {
    return new R2dbcEntityTemplate(connectionFactory);
}

@Bean
public R2dbcEntityTemplate readEntityTemplate(
        @Qualifier("readConnectionFactory") ConnectionFactory connectionFactory) {
    return new R2dbcEntityTemplate(connectionFactory);
}
```

## `@Bean` vs estereotipos

- `@Bean`: el método de configuración construye y retorna el objeto. Adecuado para SDKs y clases externas.
- `@Component`: registro general detectado por scanning.
- `@Service`: componente que comunica intención de servicio o lógica de aplicación.
- `@Repository`: componente de persistencia. También habilita traducción de excepciones para tecnologías compatibles.

## `@Primary` vs `@Qualifier`

- `@Primary` marca el candidato preferido cuando existen varios del mismo tipo.
- `@Qualifier` selecciona explícitamente un candidato.
- Si ambos existen, el qualifier del injection point tiene prioridad para esa dependencia.

## Scopes

- `singleton`: una instancia por `ApplicationContext`.
- `prototype`: una instancia por solicitud al contenedor.
- `request`: una por petición HTTP.
- `session`: una por sesión web.
- `application`: una por `ServletContext`.

En WebFlux, los scopes web requieren cuidado porque el modelo no depende de un thread por request.

## Proxies

Spring suele usar proxies JDK cuando existe una interfaz y CGLIB para subclases. El proxy intercepta la llamada externa y aplica comportamiento como `@Transactional`.

La autoinvocación `this.metodoTransaccional()` no cruza el proxy. Métodos `final` o privados también limitan la interceptación basada en subclases.

## Trade-offs

- Centraliza configuración y facilita pruebas.
- Un contexto grande aumenta tiempo de arranque y puede ocultar dependencias si se usa field injection.
- Inyección por constructor hace dependencias obligatorias y permite objetos inmutables.

## Failure modes

- Varios beans sin qualifier ni primary producen `NoUniqueBeanDefinitionException`.
- Circular dependencies suelen indicar responsabilidades mezcladas.
- Un repositorio READ conectado a WRITE o al revés modifica carga y consistencia.
- Configurar un bean reactivo con un driver bloqueante rompe el modelo de ejecución.

## Preguntas típicas

**¿Qué es `ApplicationContext`?**  
El contenedor que almacena definiciones, crea beans, resuelve dependencias, publica eventos y aplica postprocesadores.

**¿Cómo se crea un bean?**  
Mediante scanning de componentes, métodos `@Bean`, auto-configuración, imports o registro programático.

## Preguntas Senior

**¿Cómo condicionas un bean por configuración?**  
Con condiciones de Spring Boot como `@ConditionalOnProperty`, manteniendo una interfaz estable para el consumidor.

**¿Qué riesgo tiene `@Primary`?**  
Puede ocultar una decisión que debería ser explícita. En componentes críticos READ/WRITE prefiero qualifiers en los puntos de inyección.

---

# 3. JWT, Reactor Context y multitenancy R2DBC

## Qué es

El JWT transporta claims firmados. El flujo multitenant extrae claims confiables, los propaga con Reactor Context y selecciona el schema de PostgreSQL al crear la conexión.

## Dónde lo usé

```text
gift-card-back-jwt-filter-library
├── tenant-web-filter/TenantJwtWebFilter.java
├── tenant-context/TenantContext.java
└── tenant-r2dbc/TenantConnectionFactoryWrapper.java
```

## Autenticación, autorización y JWT

- Autenticación: comprobar la identidad.
- Autorización: comprobar si esa identidad puede ejecutar una acción.
- ID Token: contiene información de identidad para el cliente.
- Access Token: autoriza acceso a recursos.
- Refresh Token: obtiene nuevos tokens sin repetir credenciales.

Un JWT incluye header, payload y firma. Leer payload no prueba autenticidad.

## Firma y JWKS

El issuer publica claves públicas mediante JWKS. `ReactiveJwtDecoder` selecciona la clave por `kid`, verifica la firma y valida restricciones configuradas como issuer y expiración.

Nunca se deben colocar secretos, contraseñas o información sensible innecesaria dentro del JWT. El payload puede leerse sin conocer la clave.

## Reactor Context

```java
return jwtDecoder.decode(token)
    .map(this::extractTenantContext)
    .flatMap(tenant -> chain.filter(exchange)
        .contextWrite(tenant::writeToContext));
```

`contextWrite` asocia el tenant a la suscripción. `TenantContext.fromContext()` usa `deferContextual` para leerlo cuando se solicita una conexión.

## Selección de schema

```java
return TenantContext.fromContext()
    .flatMap(tenant -> tenant.resolveSchemaName()
        .map(schema -> connectionMono.flatMap(conn -> applySearchPath(conn, schema)))
        .orElse(connectionMono));
```

El schema se normaliza y valida contra una expresión permitida. Después el wrapper ejecuta `SET search_path`. Al cerrar, restaura el schema por defecto antes de devolver la conexión al pool.

## Por qué `ThreadLocal` no sirve

`ThreadLocal` funciona cuando toda la petición permanece en el mismo thread. Reactor puede cambiar de thread y multiplexa múltiples suscripciones en pocos event loops. Reactor Context se une a la Subscription y no al thread.

## Trade-offs

- Schema por tenant ofrece separación lógica fuerte y consultas simples.
- Muchos schemas complican migraciones, monitoreo y pooling.
- Row-level multitenancy escala con menos objetos de base de datos, pero exige filtros confiables en todas las consultas.

## Failure modes

- Confiar en claims decodificados sin validar firma.
- Aceptar un nombre de schema sin sanitización.
- No restablecer `search_path` al devolver la conexión al pool.
- Usar contexto vacío en un endpoint que no tiene otra barrera de seguridad.
- Perder el Context al crear una suscripción independiente.

## Preguntas típicas

**¿Por qué no enviar tenant en un header libre?**  
El cliente podría manipularlo. El tenant debe derivarse de una identidad validada o de un canal confiable.

**¿Qué diferencia hay entre Access Token y Refresh Token?**  
El primero autoriza solicitudes y suele durar poco. El segundo solo se usa con el servidor de identidad para obtener nuevos tokens y debe protegerse más.

## Preguntas Senior

**¿Cómo evitas fuga entre tenants con pooling?**  
Configuro el schema al adquirir la conexión, lo restablezco al cerrarla, sanitizo identificadores y agrego pruebas concurrentes que alternen tenants sobre el mismo pool.

---

# 4. AWS messaging: SQS, EventBridge y fan-out

## Qué es fan-out

Un evento publicado una vez se distribuye a múltiples consumidores. Cada consumidor puede procesar a su ritmo y fallar sin bloquear a los demás.

## Implementación mostrada

```text
ms-novedades
  ↓ publica body + MessageAttribute topic
q-novedades / salida de dominio
  ↓ EventBridge Pipe
bus-smartpay
  ├─ rule por topic → SQS notificación core
  ├─ rule por topic → SQS State Machine novedades
  └─ rule por topic manual → SQS notificación front
```

El adapter Java agrega `topic`, `traceId` y metadata a `SendMessageRequest.messageAttributes`.

## SNS vs SQS vs EventBridge

### SNS

Pub/Sub. Un publish se entrega a múltiples suscriptores. Adecuado cuando el routing es simple y los destinos soportados encajan con SNS. SNS no sustituye una cola duradera por consumidor.

### SQS

Cola. Proporciona buffer, desacoplamiento temporal, long polling, visibility timeout y retención. El consumidor controla su tasa.

### EventBridge

Bus de eventos. Evalúa reglas sobre el evento y lo envía a targets. Permite routing por atributos, múltiples fuentes y destinos, archive/replay según configuración.

## Por qué una SQS por consumidor

- Backlog independiente.
- Visibility timeout independiente.
- Redrive policy independiente.
- Throughput independiente.
- Un consumidor caído no impide que los demás borren sus propios mensajes.

## Standard vs FIFO

| Característica | Standard | FIFO |
|---|---|---|
| Throughput | alto | limitado por cuotas y grupos |
| Orden | best effort | por `MessageGroupId` |
| Entrega | at-least-once | deduplicación dentro de ventana, aun requiere idempotencia defensiva |
| Duplicados | posibles | reducidos mediante deduplication ID |

## Visibility timeout

Tras `ReceiveMessage`, SQS oculta temporalmente el mensaje. El consumidor debe borrarlo al completar. Si no lo borra, reaparece. El timeout debe superar el procesamiento o renovarse con `ChangeMessageVisibility`.

## At-least-once

El mismo mensaje puede recibirse más de una vez debido a reentrega, timeout o condiciones distribuidas. Un consumidor idempotente produce el mismo efecto final ante repeticiones.

Patrones:

- Tabla inbox con `messageId` y escritura condicional.
- Clave de idempotencia de negocio, no solo ID generado por transporte.
- Estado de proceso con versión.
- Outbox para publicar después de la transacción local.

## DLQ

Una redrive policy define `deadLetterTargetArn` y `maxReceiveCount`. Después de varios receives fallidos, SQS mueve el mensaje a la DLQ. La DLQ necesita alarmas, inspección y proceso controlado de redrive.

No se encontró una redrive policy en los manifiestos usados en la presentación. Se debe hablar de ella como configuración necesaria para la operación, no como implementación confirmada.

## Código del listener

```java
return getMessages()
    .flatMap(message -> processor.apply(message)
        .then(confirm(message))
        .onErrorResume(e -> Mono.empty()));
```

El orden es correcto si el error llega hasta `onErrorResume`: `confirm` solo se concatena después del processor. Sin embargo, si `processor.apply()` internamente convierte el error en `Mono.empty()`, `then(confirm)` borra el mensaje.

## Trade-offs

- Desacoplamiento y absorción de picos.
- Consistencia eventual.
- Duplicados y reordenamiento.
- Más componentes operativos, métricas y trazabilidad.

## Failure modes

- Visibility timeout menor que la duración del trabajo.
- Error convertido en complete y ACK prematuro.
- Reintentos sin idempotencia.
- DLQ sin alarma o sin proceso de redrive.
- Reglas EventBridge demasiado amplias.
- Payload incompatible entre productor y consumidor.

## Preguntas típicas

**¿Qué ocurre si B falla?**  
Su cola conserva mensajes mientras A y C continúan con sus propias colas.

**¿Qué ocurre si no hago delete?**  
El mensaje vuelve a ser visible después del timeout.

**¿Cómo conservar orden?**  
FIFO y `MessageGroupId` para el conjunto que necesita orden. También se puede incluir una secuencia de negocio y rechazar versiones antiguas.

## Preguntas Senior

**¿Qué métrica vigilarías?**  
`ApproximateAgeOfOldestMessage`, profundidad de cola, receives, deletes, errores de consumidor, DLQ visible messages y latencia end-to-end por traceId.

---

# 5. AWS Step Functions

## Qué es

Servicio de orquestación que representa un proceso como una máquina de estados en Amazon States Language.

## Implementación mostrada

`state-machine-novedades/statemachine/novedades.asl.json` contiene Tasks, Choices, Retry, Catch, idempotencia, locks y dos integraciones `sqs:sendMessage.waitForTaskToken`.

## Estados principales

- `Task`: ejecuta una integración.
- `Choice`: selecciona una rama.
- `Pass`: transforma o agrega datos sin trabajo externo.
- `Map`: ejecuta un subflujo para elementos de una colección.
- `Parallel`: ejecuta ramas distintas y espera todas.
- `Succeed` / `Fail`: terminación explícita.

## Retry y Catch

`Retry` se evalúa antes de `Catch`. Los reintentos conservan el mismo estado con espera y backoff. Después de agotar intentos, `Catch` agrega el error mediante `ResultPath` y redirige.

Solo deben reintentarse errores transitorios. Un error de validación no mejora con backoff.

## Paths

- `InputPath`: filtra el input del estado.
- `Parameters`: construye el input del recurso.
- `ResultSelector`: transforma el resultado del recurso.
- `ResultPath`: decide dónde combinar el resultado.
- `OutputPath`: filtra la salida al siguiente estado.

## Callback con task token

```json
"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
"MessageBody": {
  "taskToken.$": "$$.Task.Token"
},
"TimeoutSeconds": 900
```

Step Functions publica el token y suspende el estado. El worker completa mediante `SendTaskSuccess` o `SendTaskFailure`.

## Standard vs Express

- Standard: historial durable, ejecuciones largas, callback patterns y semántica apropiada para procesos de negocio.
- Express: alto volumen, corta duración y menor costo por ejecución; historial y semántica diferentes.

El callback con task token se asocia normalmente a Standard.

## Trade-offs

- Estados, retries y errores visibles.
- Recuperación sin mantener una Lambda activa.
- Costo por transición.
- Acoplamiento con AWS y límites de payload.
- Cambios de contrato deben coordinar ASL y workers.

## Failure modes

- Callback nunca llega y consume todo el timeout.
- Token expuesto en logs.
- Retry sobre efectos no idempotentes.
- `ResultPath` sobrescribe datos que otro estado necesita.
- Ejecución duplicada sin control de idempotencia.

## Preguntas Senior

**¿Cómo reintentas una Task que pudo ejecutar el efecto antes de fallar?**  
El worker debe usar una clave idempotente y recuperar el resultado existente. El retry de Step Functions no garantiza exactly-once.

---

# 6. Docker, Compose y Kubernetes

## Image vs container

La imagen es un filesystem inmutable organizado por capas más metadata. El contenedor es un proceso aislado que ejecuta esa imagen con una capa escribible.

## Multi-stage build

```dockerfile
FROM gradle:8.14.3-jdk21-alpine AS builder
RUN ./gradlew clean bootJar

FROM eclipse-temurin:21-jre-alpine
COPY --from=builder .../app.jar /app/app.jar
USER appuser
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

La etapa final excluye Gradle, fuentes y JDK. Esto reduce tamaño y superficie de ataque.

## Layers

Cada instrucción produce una capa reutilizable. Conviene copiar primero archivos de dependencias y luego código que cambia con frecuencia para mejorar cache del build. `COPY` demasiado amplio invalida capas innecesariamente.

## ENTRYPOINT vs CMD

- `ENTRYPOINT`: ejecutable principal.
- `CMD`: argumentos por defecto o comando si no existe entrypoint.
- Forma exec: evita shell y entrega señales directamente.
- `sh -c`: permite expandir variables, pero el shell queda como intermediario. Se debe usar `exec java ...` si se quiere reemplazar el shell y propagar señales correctamente.

## Usuario no root

Reduce el impacto si el proceso se compromete. Kubernetes refuerza esto con `runAsNonRoot`, `allowPrivilegeEscalation: false`, capabilities drop y filesystem de solo lectura.

## Docker Compose

Define aplicación, Mongo, red implícita, variables, puertos, volumen y healthchecks. `depends_on` con `service_healthy` ordena el arranque, pero la aplicación aun necesita retry ante pérdida posterior de Mongo.

## Kubernetes

- Pod: unidad mínima de ejecución.
- Deployment: administra ReplicaSets, actualizaciones y réplicas.
- Service: nombre estable y balanceo hacia Pods.
- ConfigMap: configuración no sensible.
- Secret: datos sensibles, aunque base64 no equivale a cifrado.
- Requests: recursos reservados para scheduling.
- Limits: techo de consumo.
- Readiness: aptitud para recibir tráfico.
- Liveness: capacidad de seguir vivo; falla provoca restart.
- HPA: ajusta réplicas según métricas.

## Trade-offs

- Portabilidad y runtime reproducible.
- Más complejidad operativa y de observabilidad.
- Límites de CPU pueden producir throttling.
- Limits de memoria pueden causar OOMKill.

## Failure modes

- Ejecutar como root.
- Incluir secretos en la imagen.
- Usar `latest` sin digest en despliegues reproducibles.
- Liveness dependiente de servicios externos y restart loop.
- Request demasiado bajo o limit demasiado pequeño.
- Imagen Alpine incompatible con una librería nativa.

## Preguntas Senior

**¿Por qué una imagen pequeña no garantiza seguridad?**  
Reduce superficie, pero siguen importando CVEs, permisos, secretos, provenance, firma, SBOM y configuración del runtime.

---

# 7. Redis y cache-aside

## Qué es

Redis es un almacén en memoria con estructuras de datos y operaciones atómicas. En la presentación aparece como caché distribuida reactiva.

## Cache-aside

1. La aplicación consulta Redis.
2. Hit: usa el valor.
3. Miss: consulta la base de datos.
4. Guarda el resultado con TTL.
5. Retorna el valor.

La aplicación mantiene la responsabilidad de cargar e invalidar.

## Código real

`ValidarActivacionTarjetaUseCase` implementa lectura Redis, fallback R2DBC y escritura de caché. `TarjetaRedisService` aplica TTL de cinco días e invalidación por `delete`.

## TTL e invalidación

TTL limita la antigüedad máxima, pero durante ese intervalo puede existir stale data. La invalidación explícita reduce esa ventana cuando la aplicación conoce el cambio.

Estrategias relacionadas:

- Read-through: el proveedor de caché carga el dato ante miss.
- Write-through: la escritura actualiza caché y almacenamiento en la misma ruta síncrona.
- Write-behind: la caché acepta la escritura y persiste después; mejora latencia y aumenta riesgo de pérdida.
- Refresh-ahead: renueva antes de expirar según uso o umbral.

La presentación demuestra cache-aside. Las demás estrategias se explican como comparación.

## Local vs distribuida

- Local: latencia mínima, sin red, pero copia por instancia e invalidación compleja.
- Distribuida: estado compartido entre réplicas, mayor latencia y dependencia de red.

## Estructuras Redis

- String: valor simple u objeto serializado.
- Hash: campos de un objeto modificables individualmente.
- List: secuencia ordenada con operaciones en extremos.
- Set: miembros únicos sin orden.
- Sorted Set: miembros únicos con score y ranking.
- Stream: log append-only con consumer groups.

El código presentado usa `opsForValue`, equivalente a Strings.

## Cache stampede

Muchos requests observan el mismo miss y consultan la BD simultáneamente. Opciones:

- Lock por clave con expiración.
- Single-flight dentro de instancia.
- TTL con jitter.
- Refresh-ahead.
- Stale-while-revalidate si el negocio lo tolera.

## Eviction policies

Cuando Redis llega al límite de memoria puede rechazar escrituras o expulsar claves según políticas como `allkeys-lru`, `allkeys-lfu`, `volatile-ttl` o `noeviction`.

## Failure modes

- TTL demasiado largo produce datos obsoletos.
- TTL demasiado corto reduce hit ratio y aumenta carga.
- Invalidación por patrón con `SCAN` puede ser costosa.
- Errores de Redis convertidos en miss pueden causar una tormenta a la BD.
- `subscribe()` interno separa la escritura de la cadena.
- Objetos grandes elevan memoria y latencia de serialización.

## Preguntas Senior

**¿Cómo evitarías stale data en una tarjeta modificada?**  
Invalidación después de confirmar la transacción. Para evitar inconsistencias entre DB y Redis, se puede usar outbox o eventos de invalidación.

**¿Usarías Redis como source of truth?**  
No para este caso. La base de datos conserva el estado autoritativo; Redis acelera lecturas y puede perder claves por expiración o eviction.

---

# 8. Arquitectura limpia y puertos

## Qué es

Organiza dependencias para que el dominio no dependa de frameworks ni tecnologías externas.

## Implementación mostrada

```text
HTTP Handler
   ↓
FranchiseInputPort
   ↓
FranchiseUseCase
   ↓
FranchiseRepository
   ↑
MongoRepositoryAdapter
```

El adapter HTTP conoce WebFlux y DTOs. El caso de uso conoce modelos y puertos. El adapter Mongo conoce Spring Data, entidades y resiliencia.

## Principios aplicados

- Dependency Inversion: el dominio define la interfaz requerida.
- Single Responsibility: entrada, regla y persistencia tienen motivos distintos de cambio.
- Interface Segregation: los puertos deben representar capacidades cohesivas.
- Open/Closed: un nuevo adapter puede implementar el mismo contrato.

## Trade-offs

- Pruebas del caso de uso sin infraestructura.
- Sustitución de adapters.
- Más código de traducción.
- Riesgo de crear interfaces sin valor arquitectónico.

## Failure modes

- DTO HTTP dentro del modelo de dominio.
- Anotaciones de Mongo o JPA en entidades de negocio.
- Puerto que expone tipos de AWS o Spring.
- Caso de uso que llama directamente a un SDK.
- Mapper que contiene reglas de negocio.

## Preguntas Senior

**¿Dónde ubicas una transacción?**  
En el boundary de aplicación que coordina las operaciones que deben ser atómicas, usando una abstracción que no filtre detalles innecesarios al dominio.

---

# 9. Bases de datos: ACID, TCL, CAP y BASE

## Implementación relacional mostrada

ASULADO define un puerto para que el dominio solicite una unidad transaccional sin depender de Spring:

```java
public interface ReactiveTransactionPort {
    <T> Mono<T> executeWithinTransaction(Mono<T> operation);
}
```

El adaptador usa el mecanismo reactivo del framework:

```java
public <T> Mono<T> executeWithinTransaction(Mono<T> operation) {
    return Mono.defer(() -> transactionalOperator.transactional(operation));
}
```

Un caso de uso coloca dentro de esa frontera la lectura, actualización y recálculo relacionados:

```java
return reactiveTransactionPort.executeWithinTransaction(
        refreshAppliedDeductions(participantPaymentId, period, metadata));
```

`TransactionalOperator` enlaza la transacción al contexto reactivo. La operación debe conservar una sola cadena; un `subscribe()` interno crea otra suscripción que queda fuera de esa frontera.

## TCL y ciclo de la transacción

TCL incluye `BEGIN`, `COMMIT`, `ROLLBACK` y, según el motor, `SAVEPOINT`.

En el flujo reactivo normal:

1. La suscripción abre o participa en una transacción.
2. Las operaciones obtienen la conexión ligada al contexto.
3. `onComplete` permite commit.
4. `onError` provoca rollback.
5. Cancelación también debe considerarse una terminación que no confirma trabajo incompleto.

Capturar un error con `onErrorResume` y convertirlo en completion puede cambiar el resultado: desde la perspectiva del operador ya no existe una señal de error que obligue rollback.

## ACID

### Atomicidad

La unidad completa confirma o revierte. No significa que todo el sistema distribuido sea atómico. PostgreSQL no revierte automáticamente SQS, DynamoDB ni una API externa.

### Consistencia

El commit debe conservar reglas como tipos, `NOT NULL`, claves foráneas, `CHECK`, unicidad e invariantes de negocio. En ASULADO un índice único parcial impide más de un registro activo para participante, período y número de pago:

```sql
CREATE UNIQUE INDEX UQ_TLIQ_PRELIQ_PARTICIPANTE_PERIODO_PAGO_ACTIVA
ON TLIQ_PRELIQ_PARTICIPANTE
  (LIQ_CD_ID_PARTICIPANTE, LIQ_DS_PERIODO_LIQUIDACION, LIQ_NM_NUMERO_PAGO)
WHERE LIQ_DS_ESTADO <> 'CANCELADO';
```

La regla vive en la base y protege incluso si dos instancias compiten.

### Aislamiento

Anomalías principales:

- Dirty read: leer cambios no confirmados.
- Non-repeatable read: la misma fila cambia entre dos lecturas.
- Phantom read: una segunda consulta encuentra filas nuevas que cumplen el predicado.
- Lost update: una escritura pisa otra sin detectar el conflicto.

Niveles comunes:

- `READ COMMITTED`: evita dirty reads; cada sentencia puede ver nuevos commits.
- `REPEATABLE READ`: mantiene lecturas estables dentro de la transacción y reduce más anomalías.
- `SERIALIZABLE`: busca un resultado equivalente a ejecutar transacciones en serie; puede abortar y requerir retry.

Mayor aislamiento reduce anomalías, pero aumenta esperas, abortos o contención. La elección depende de la invariante, no de usar siempre el máximo nivel.

### Durabilidad

Después de un commit confirmado, el motor conserva el cambio mediante mecanismos como WAL y almacenamiento persistente. Durabilidad no sustituye backups, réplica ni pruebas de restauración.

## Índices

Un índice reduce el trabajo de búsqueda cuando coincide con filtros, joins u ordenamientos. Sus costos:

- ocupa almacenamiento;
- cada insert, update o delete debe mantenerlo;
- demasiados índices empeoran escrituras;
- el orden de columnas importa en índices compuestos;
- baja selectividad puede hacerlo poco útil.

Antes de crear uno: partir de una consulta real, revisar cardinalidad y validar con `EXPLAIN ANALYZE`. Un índice único también implementa consistencia, no solo rendimiento.

## Procedimientos y funciones almacenadas

Son apropiados cuando una operación intensiva sobre datos se beneficia de ejecutarse cerca del motor, debe reutilizarse desde varios consumidores o requiere una unidad transaccional muy controlada. Sus riesgos incluyen acoplamiento al motor, versionado más difícil, observabilidad separada y lógica duplicada con la aplicación.

En Kire existen funciones PostgreSQL para reportes y operaciones masivas. La presentación usa esa evidencia como contexto de optimización, pero la diapositiva principal se apoya en el flujo transaccional e índice de ASULADO.

## DynamoDB: CRUD real

El adaptador usa:

- `PutItem` para crear o reemplazar un ítem.
- `GetItem` para obtener un ítem por su clave completa.
- `Query` para obtener varios ítems con la misma partition key.
- `LastEvaluatedKey` y `ExclusiveStartKey` para paginar.

`UpdateItem` permite cambios parciales y expresiones condicionales. `DeleteItem` elimina por clave. `PutItem` reemplaza el ítem completo si la clave ya existe, salvo que una condition expression lo impida.

## Partition key y sort key

- Partition key: el valor se procesa para decidir la partición física. Debe distribuir carga.
- Sort key: ordena ítems dentro de la misma partition key y permite condiciones de rango.
- Clave simple: solo partition key.
- Clave compuesta: partition key más sort key.

El diseño empieza por patrones de acceso. En el código, `id_participante = :pk` permite `Query` sin recorrer toda la tabla. Una clave con muy pocos valores o tráfico concentrado puede crear una partición caliente.

## GSI y LSI

### GSI

Puede usar otra partition key y otra sort key. Permite un patrón de acceso que la clave primaria no resuelve. Se crea o elimina después de crear la tabla. DynamoDB propaga sus cambios de forma asíncrona; sus lecturas son eventualmente consistentes.

### LSI

Mantiene la misma partition key y define una sort key alternativa. Debe declararse al crear la tabla. Comparte el grupo lógico de la partition key y permite lectura eventual o fuertemente consistente.

No hay GSI o LSI mostrado en la infraestructura revisada. Debes explicarlos como opciones de diseño y justificar cuál patrón de acceso requeriría cada uno.

## Query frente a Scan

`Query` exige una partition key y puede filtrar por sort key. Lee un conjunto dirigido. `Scan` examina toda la tabla o índice antes de filtrar; consume capacidad por los ítems evaluados. Debe evitarse en rutas frecuentes y tablas grandes.

DynamoDB pagina resultados. Si la respuesta trae `LastEvaluatedKey`, se continúa con `ExclusiveStartKey`. El adaptador real usa `expand` para consumir todas las páginas sin recursión manual bloqueante.

## Consistencia de lectura

Las lecturas son eventualmente consistentes por defecto. Tabla base y LSI admiten `ConsistentRead(true)`; un GSI no admite lectura fuerte. Una lectura fuerte cuesta más capacidad y puede aumentar latencia.

El adaptador de parámetros usa `consistentRead(true)` porque quiere el valor confirmado más reciente. Esa elección debe responder a una necesidad del negocio, no aplicarse por defecto.

## CAP

CAP afirma que, cuando existe una partición de red, un sistema distribuido no puede garantizar simultáneamente consistencia lineal y disponibilidad para todas las solicitudes.

- C: cada lectura observa el valor más reciente bajo el modelo exigido.
- A: cada solicitud recibe una respuesta no errónea.
- P: el sistema sigue operando aunque se pierdan mensajes entre nodos.

No conviene etiquetar toda DynamoDB simplemente como “AP”. La consistencia se decide por operación y configuración. Una lectura eventual favorece disponibilidad y menor costo; una lectura fuerte exige una garantía mayor y puede fallar si el servicio no puede satisfacerla.

## BASE

- Basically Available: el sistema intenta continuar disponible.
- Soft State: el estado puede cambiar por propagación asíncrona incluso sin una nueva solicitud del usuario.
- Eventual Consistency: si cesan las escrituras, las réplicas convergen.

BASE no significa ausencia de reglas. Se combina con claves, condition expressions, idempotencia y procesos de reconciliación.

## Transacciones distribuidas y outbox

Una transacción R2DBC no abarca PostgreSQL y SQS. Para publicar un evento de forma confiable:

1. Guardar el cambio de negocio y una fila outbox en la misma transacción.
2. Un publicador lee la outbox y envía el evento.
3. Marca la fila como enviada o registra el intento.
4. El consumidor procesa idempotentemente porque pueden existir duplicados.

## Failure modes

- Ejecutar una escritura fuera del publisher transaccional mediante `subscribe()`.
- Atrapar el error antes del boundary y confirmar estado parcial sin intención.
- Mantener una transacción abierta mientras se espera una API lenta.
- Crear índices sin medir consultas ni costo de escritura.
- Usar `Scan` en una ruta de alto tráfico.
- Elegir una partition key que concentra carga.
- Esperar lectura fuerte desde un GSI.
- Confundir consistencia ACID local con consistencia del flujo distribuido.

## Preguntas Senior

**¿Qué garantiza realmente `TransactionalOperator`?**  
Delimita una transacción reactiva sobre recursos administrados por el transaction manager. No extiende atomicidad a SDKs externos.

**¿Cómo evitas duplicados sin una transacción distribuida?**  
Constraint o clave idempotente en el estado local, outbox para publicación y consumidor idempotente.

**¿Cuándo usarías GSI?**  
Cuando un patrón de acceso necesita otra partition key. Debo aceptar propagación eventual, costo adicional de escritura y capacidad del índice.

**¿Qué demostraría que un índice sirve?**  
El plan con `EXPLAIN ANALYZE`, selectividad, filas examinadas, tiempo, buffers y medición antes/después bajo carga representativa.

## Fuentes oficiales para repaso

- Spring Framework: Transaction Management, `https://docs.spring.io/spring-framework/reference/data-access/transaction.html`.
- AWS: DynamoDB read consistency, `https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html`.
- AWS: DynamoDB transactions, `https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html`.

---

# 10. Testing reactivo

## Qué es StepVerifier

Subscriber de prueba para declarar y verificar señales de un publisher.

## Operaciones principales

- `expectNext`: verifica valores.
- `expectNextMatches`: predicado sobre valor.
- `expectComplete`: declara completion esperado.
- `expectError`: declara tipo de error.
- `thenRequest(n)`: controla demanda.
- `thenCancel`: verifica cancelación.
- `withVirtualTime`: prueba delays y retries sin esperar tiempo real.

## Pruebas mostradas

1. 201 registros producen tres consultas de lotes y 201 llamadas externas.
2. Un timeout individual produce resultado fallido y el siguiente registro continúa.
3. Un error de S3 después de cien éxitos conserva estado parcial.

## Qué valida cada capa

- Use case: reglas, composición y efectos sobre puertos.
- Adapter: serialización, SDK y mapeo de errores.
- Entry point: contrato HTTP y validación.
- Integración: wiring real entre componentes y configuración.

## TDD

- Red: escribir un comportamiento observable que falla.
- Green: implementación mínima que cumple.
- Refactor: mejorar diseño sin cambiar comportamiento.

TDD no consiste en escribir pruebas después. La prueba guía el contrato y el diseño.

## Failure modes

- Probar solo `verifyComplete()` sin verificar efectos.
- Usar mocks para todo y no probar serialización real.
- `Thread.sleep` en pruebas reactivas.
- `block()` que oculta problemas de señales.
- No probar cancelación, timeout, retry o demanda.

## Preguntas Senior

**¿Cómo pruebas un retry con backoff?**  
`StepVerifier.withVirtualTime`, avanzar tiempo virtual y verificar número de invocaciones y señal terminal.

**¿Cómo pruebas que no se emite sin demanda?**  
Crear StepVerifier con demanda inicial cero, verificar ausencia de eventos y llamar `thenRequest` por etapas.

---

# Repaso de 15 minutos

1. Explicar el pipeline `leer → buffer → concatMap → resumir` sin mirar notas.
2. Decir `map`, `flatMap`, `concatMap`, `subscribeOn`, `publishOn` en una frase cada uno.
3. Dibujar READ/WRITE beans y explicar `@Qualifier` frente a `@Primary`.
4. Dibujar JWT → Reactor Context → R2DBC → schema.
5. Dibujar SQS → Pipe → EventBridge → SQS por consumidor.
6. Explicar visibility timeout, delete, duplicados e idempotencia.
7. Recitar el flujo de task token y qué hacen Retry y Catch.
8. Explicar multi-stage, non-root, readiness y liveness.
9. Dibujar cache-aside y mencionar TTL, invalidación y stampede.
10. Explicar ACID usando el `TransactionalOperator` y el índice único parcial.
11. Comparar PK/SK, GSI/LSI, Query/Scan y lectura eventual/fuerte en DynamoDB.
12. Relacionar CAP y BASE con una decisión concreta de consistencia.
13. Describir las dos pruebas principales y las señales verificadas.
