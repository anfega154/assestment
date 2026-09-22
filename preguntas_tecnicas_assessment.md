# Preguntas técnicas — Review de Assessment

Cada respuesta corta está diseñada para unos 30 segundos. La respuesta profunda sirve para una conversación de aproximadamente dos minutos.

---

# Reactor y WebFlux

## 1. ¿Qué es Reactive Streams y qué diferencia hay entre Publisher y Subscriber?

### Respuesta corta — 30 segundos

Reactive Streams es una especificación para procesar secuencias asíncronas con backpressure. El Publisher ofrece elementos. El Subscriber consume señales y controla demanda mediante la Subscription.

### Respuesta profunda — 2 minutos

El contrato define `Publisher`, `Subscriber`, `Subscription` y `Processor`. Al suscribirse, el Subscriber recibe `onSubscribe` y solicita elementos con `request(n)`. El Publisher no debe superar esa demanda. Después envía `onNext` y finalmente una sola señal terminal: `onComplete` o `onError`. Reactor implementa el contrato mediante `Flux` y `Mono`.

### Ejemplo de mi proyecto

`ArchivoRecargaMasivaS3Adapter` adapta el publisher asíncrono de S3 a `Flux<RegistroRecargaMasiva>`.

### Posible repregunta

¿Qué ocurre si un Publisher emite después de `onComplete`?

---

## 2. ¿Qué es backpressure y cómo se representa la demanda con `request(n)`?

### Respuesta corta — 30 segundos

Backpressure permite que el consumidor indique cuántos elementos puede recibir. La demanda viaja por la cadena mediante `request(n)` y cada operador puede transformarla.

### Respuesta profunda — 2 minutos

La Subscription mantiene la demanda pendiente. Un Subscriber puede pedir cantidades acotadas o `Long.MAX_VALUE`. Operadores como `buffer(100)` solicitan suficientes elementos para completar cada colección. `flatMap` maneja demanda, prefetch y concurrencia para publishers internos. Backpressure no sustituye límites de conexiones ni evita que un productor externo ignore el protocolo.

### Ejemplo de mi proyecto

El pipeline agrupa 100 registros y `concatMap` mantiene un lote interno activo, protegiendo R2DBC y el servicio externo.

### Posible repregunta

¿Cómo probarías demanda gradual con StepVerifier?

---

## 3. ¿`Mono` se ejecuta inmediatamente y quién hace `subscribe()`?

### Respuesta corta — 30 segundos

Normalmente `Mono` es lazy: describe trabajo. En un endpoint WebFlux, el framework se suscribe. En el worker programado existe una suscripción explícita porque ese método es un boundary sin caller reactivo.

### Respuesta profunda — 2 minutos

Construir operadores no ejecuta la fuente. `subscribe()` crea una Subscription y activa el recorrido hacia la fuente. Cada suscripción puede repetir efectos si el publisher es cold. `Mono.defer` reconstruye el trabajo por suscripción. Los casos de uso deberían devolver el publisher y dejar la suscripción al framework o al boundary.

### Ejemplo de mi proyecto

`ProcesadorRecargaMasivaProgramado.procesarPendiente()` se suscribe y libera el `AtomicBoolean` con `doFinally`.

### Posible repregunta

¿Cuándo usarías `cache()` o `share()`?

---

## 4. ¿Cuál es la diferencia entre `map`, `flatMap` y `concatMap`?

### Respuesta corta — 30 segundos

`map` transforma un valor en otro valor. `flatMap` transforma a publishers y los combina concurrentemente. `concatMap` también aplana publishers, pero los suscribe uno por uno y conserva el orden.

### Respuesta profunda — 2 minutos

`map` es apropiado para transformación síncrona. `flatMap` sirve para I/O reactivo dependiente del valor y puede limitarse con un parámetro de concurrencia. Las salidas llegan según finalicen. `concatMap` espera la terminación de cada publisher interno. `flatMapSequential` permite concurrencia interna y reordena la entrega.

### Ejemplo de mi proyecto

`concatMap(registros -> procesarLote(...))` evita procesar varios lotes de recargas al mismo tiempo.

### Posible repregunta

¿Cómo aumentarías throughput sin perder totalmente el control?

---

## 5. ¿Qué diferencia hay entre `publishOn` y `subscribeOn`?

### Respuesta corta — 30 segundos

`subscribeOn` influye en la suscripción a la fuente. `publishOn` mueve los operadores posteriores a otro scheduler.

### Respuesta profunda — 2 minutos

`subscribeOn` programa la señal de suscripción y normalmente domina el primer `subscribeOn` efectivo. `publishOn` introduce una frontera asíncrona y encola señales para continuar en un worker del scheduler. Ninguno vuelve no bloqueante una librería síncrona; solo decide dónde corre.

### Ejemplo de mi proyecto

El cliente Cognito síncrono se envuelve con `Mono.fromCallable` y se aísla en `boundedElastic`.

### Posible repregunta

¿Qué costo introduce `publishOn`?

---

## 6. ¿Qué es `boundedElastic` y qué ocurre si bloqueo el event loop?

### Respuesta corta — 30 segundos

`boundedElastic` es un scheduler acotado para trabajo bloqueante inevitable. Bloquear Netty detiene otras conexiones atendidas por el mismo event-loop thread.

### Respuesta profunda — 2 minutos

Netty usa pocos threads para muchas conexiones. Si una llamada JDBC o un SDK síncrono ocupa uno, aumenta la latencia de todas las conexiones asignadas. `boundedElastic` traslada ese trabajo a otro pool, con límites de threads y cola. La mejor solución sigue siendo un cliente no bloqueante.

### Ejemplo de mi proyecto

`CognitoService` adapta operaciones síncronas. El pipeline de S3 usa `S3AsyncClient` y no necesita aislar el SDK.

### Posible repregunta

¿Cómo detectarías bloqueo accidental?

---

## 7. ¿`parallel()` mejora cualquier flujo?

### Respuesta corta — 30 segundos

No. `parallel()` divide en rails y necesita `runOn`. Es útil para CPU independiente. Para I/O asíncrono suele ser más directo usar `flatMap` con concurrencia acotada.

### Respuesta profunda — 2 minutos

El paralelismo usa varios cores; la concurrencia permite tareas solapadas. Crear rails para llamadas I/O no mejora necesariamente throughput y complica orden. Hay que medir CPU, latencia, pool de conexiones y capacidad del downstream.

### Ejemplo de mi proyecto

La recarga masiva usa `concatMap` deliberadamente porque prioridad es orden y protección del servicio externo.

### Posible repregunta

¿En qué operación de este proyecto considerarías paralelismo?

---

## 8. ¿Qué problema causa un `subscribe()` interno?

### Respuesta corta — 30 segundos

Crea una segunda ejecución fuera del control del caller. Se pierden propagación de error, cancelación, contexto y composición transaccional.

### Respuesta profunda — 2 minutos

La cadena principal puede completar antes de la secundaria. Un error secundario no llega al endpoint. Reactor Context puede faltar y las pruebas requieren esperar efectos laterales. La solución es devolver o componer el publisher con `flatMap`, `then` o `thenReturn`.

### Ejemplo de mi proyecto

`ValidarActivacionTarjetaUseCase` guarda en Redis mediante un `subscribe()` interno. Lo compondría como best effort dentro del mismo flujo.

### Posible repregunta

¿Cómo mantendrías disponibilidad si Redis falla sin usar una segunda suscripción?

---

# Spring Boot

## 9. ¿Cómo funciona la inyección de dependencias y qué es el `ApplicationContext`?

### Respuesta corta — 30 segundos

Spring registra definiciones, crea beans y resuelve sus dependencias dentro del `ApplicationContext`. La inyección por constructor hace explícito qué necesita cada clase.

### Respuesta profunda — 2 minutos

Durante el arranque, Spring procesa configuraciones, scanning y auto-configuración. Registra `BeanDefinition`, resuelve candidatos por tipo, nombre, qualifier y primary, crea instancias, ejecuta postprocesadores y aplica proxies cuando corresponde.

### Ejemplo de mi proyecto

`R2dbcConfig` registra factories, templates y transaction managers separados para READ y WRITE.

### Posible repregunta

¿Qué ocurre con una dependencia circular?

---

## 10. ¿Cuál es la diferencia entre `@Bean`, `@Component`, `@Service` y `@Repository`?

### Respuesta corta — 30 segundos

`@Bean` registra el objeto retornado por un método. Los otros son estereotipos detectados por scanning; `@Service` y `@Repository` comunican responsabilidad y Repository puede traducir excepciones.

### Respuesta profunda — 2 minutos

Uso `@Bean` cuando necesito controlar construcción o registrar una clase externa. Los estereotipos registran la clase misma. Todas generan beans, pero expresan roles distintos y pueden activar postprocesamiento específico.

### Ejemplo de mi proyecto

Los `SqsAsyncClient` y `ConnectionFactory` se construyen con `@Bean`; adapters y servicios se detectan como componentes.

### Posible repregunta

¿Por qué preferir constructor injection a field injection?

---

## 11. ¿Cómo resuelve Spring múltiples beans? ¿Qué diferencia `@Primary` de `@Qualifier`?

### Respuesta corta — 30 segundos

`@Primary` define el candidato preferido. `@Qualifier` selecciona un bean específico en el punto de inyección.

### Respuesta profunda — 2 minutos

Spring primero busca candidatos por tipo. El qualifier reduce el conjunto usando metadata o nombre. Si quedan varios, primary ayuda a resolver. Si la ambigüedad continúa, el contexto no arranca.

### Ejemplo de mi proyecto

`writeEntityTemplate` recibe `@Qualifier("writeConnectionFactory")`, mientras WRITE también se marca `@Primary`.

### Posible repregunta

¿Usarías primary en un sistema con lectura y escritura separadas?

---

## 12. ¿Qué scopes existen y cuándo importan?

### Respuesta corta — 30 segundos

Los principales son singleton, prototype, request y session. Singleton es el default y exige evitar estado mutable por petición.

### Respuesta profunda — 2 minutos

Singleton significa una instancia por contexto, no una clase global de JVM. Prototype crea por resolución. Los scopes web se vinculan a request o session. En aplicaciones reactivas no se debe usar ThreadLocal para simular request scope.

### Ejemplo de mi proyecto

Los servicios de Reactor son singleton y los datos por tenant viven en Reactor Context, no en campos mutables.

### Posible repregunta

¿Qué ocurre al inyectar un prototype dentro de un singleton?

---

## 13. ¿Qué proxies usa Spring y cuál es el problema de autoinvocación?

### Respuesta corta — 30 segundos

Spring usa proxies JDK o CGLIB. Una llamada `this.metodo()` dentro del mismo objeto no atraviesa el proxy, por lo que aspectos como transacciones pueden no ejecutarse.

### Respuesta profunda — 2 minutos

El proxy intercepta llamadas externas y aplica advice. JDK proxy implementa interfaces. CGLIB crea una subclase. Métodos privados o finales limitan la interceptación. La solución es separar el boundary en otro bean o usar composición explícita.

### Ejemplo de mi proyecto

Los transaction managers R2DBC se registran por datasource; cualquier `@Transactional` debe seleccionar el manager correcto y devolver el publisher.

### Posible repregunta

¿En qué momento empieza una transacción reactiva?

---

# AWS messaging

## 14. ¿Cuál es la diferencia entre SNS, SQS y EventBridge?

### Respuesta corta — 30 segundos

SNS publica a suscriptores. SQS almacena mensajes para que un consumidor los procese. EventBridge enruta eventos mediante reglas hacia múltiples destinos.

### Respuesta profunda — 2 minutos

SNS ofrece pub/sub push y fan-out simple. SQS ofrece buffer duradero, polling, visibility timeout y ritmo controlado por consumidor. EventBridge agrega buses, reglas y filtrado por contenido. SNS o EventBridge suelen combinarse con una SQS por consumidor.

### Ejemplo de mi proyecto

ASULADO usa SQS con un atributo `topic`; un Pipe lleva resultados al bus SmartPay y EventBridge enruta hacia colas de consumidores.

### Posible repregunta

¿Cuándo usarías SNS en este flujo?

---

## 15. ¿Qué es fan-out y por qué una SQS por consumidor?

### Respuesta corta — 30 segundos

Fan-out distribuye un evento a varios consumidores. Una cola por consumidor separa backlog, disponibilidad, retries y velocidad.

### Respuesta profunda — 2 minutos

Si varios consumidores comparten una sola cola, compiten por mensajes y cada evento llega a uno, no a todos. Para fan-out cada destino recibe una copia en su cola. El productor no conoce las aplicaciones consumidoras.

### Ejemplo de mi proyecto

El resultado del motor de novedades alimenta notificación core y la State Machine mediante reglas hacia colas diferentes.

### Posible repregunta

¿Qué ocurre si un consumidor permanece caído más que la retención de la cola?

---

## 16. ¿Qué ocurre si un consumidor está caído?

### Respuesta corta — 30 segundos

Su cola acumula mensajes hasta que regrese o venza la retención. Los demás consumidores continúan con sus propias colas.

### Respuesta profunda — 2 minutos

Debo monitorear profundidad y edad del mensaje más antiguo. Al recuperarse, el consumidor puede generar un pico y necesita rate limiting. Si la indisponibilidad supera retención, los mensajes se pierden salvo archive o replay externo.

### Ejemplo de mi proyecto

Notificación core y la State Machine tienen destinos separados en el fan-out de SmartPay.

### Posible repregunta

¿Cómo evitarías saturar una dependencia durante la recuperación?

---

## 17. ¿Qué significa at-least-once y cómo manejas duplicados?

### Respuesta corta — 30 segundos

SQS Standard puede entregar el mismo mensaje más de una vez. El consumidor debe ser idempotente mediante una clave estable y una escritura condicional.

### Respuesta profunda — 2 minutos

Registro el event ID o una clave de negocio antes de aplicar el efecto. Si la inserción ya existe, respondo como procesado. La frontera entre deduplicación y efecto requiere transacción local, inbox o una máquina de estados para no marcar procesado antes de completar.

### Ejemplo de mi proyecto

La State Machine consulta idempotencia en DynamoDB antes de procesar la novedad.

### Posible repregunta

¿Por qué el `messageId` de SQS puede no ser suficiente?

---

## 18. ¿Qué es visibility timeout y qué pasa si no haces delete?

### Respuesta corta — 30 segundos

Después de receive, SQS oculta el mensaje durante el visibility timeout. Si no se borra, vuelve a estar disponible y puede procesarse otra vez.

### Respuesta profunda — 2 minutos

El timeout debe cubrir el procesamiento o extenderse. Si expira antes, aparece concurrencia duplicada. El delete usa el receipt handle de esa recepción, no el message ID.

### Ejemplo de mi proyecto

El listener configura 30 segundos y llama `deleteMessage` después de `processor.apply(message)`.

### Posible repregunta

¿Qué harías con procesos cuya duración no se conoce?

---

## 19. ¿Qué es una DLQ y qué significa `maxReceiveCount`?

### Respuesta corta — 30 segundos

La DLQ recibe mensajes que superan cierto número de recepciones fallidas. `maxReceiveCount` define ese umbral en la redrive policy.

### Respuesta profunda — 2 minutos

La DLQ aísla poison messages, pero no los resuelve. Necesita alarmas, análisis de causa, corrección y redrive controlado. La retención de la DLQ debe permitir investigación.

### Ejemplo de mi proyecto

No hay una redrive policy visible en los manifiestos mostrados; sería una configuración de infraestructura a agregar para completar el ciclo operativo.

### Posible repregunta

¿Reprocesarías toda la DLQ automáticamente?

---

## 20. ¿FIFO o Standard y cómo conservas orden?

### Respuesta corta — 30 segundos

Standard prioriza throughput y no garantiza orden estricto. FIFO conserva orden dentro de cada `MessageGroupId` y reduce duplicados con deduplication ID.

### Respuesta profunda — 2 minutos

FIFO serializa por grupo, por lo que diseño grupos por entidad de negocio para mantener paralelismo entre entidades. Aun con FIFO, el consumidor debe ser idempotente ante reintentos y fallos en la aplicación.

### Ejemplo de mi proyecto

Las colas documentadas son Standard y los eventos incluyen topic y traceId; la lógica no debe depender de orden global.

### Posible repregunta

¿Cómo manejarías eventos fuera de orden sobre la misma solicitud?

---

## 21. ¿Cómo enruta EventBridge un evento?

### Respuesta corta — 30 segundos

Compara el evento contra patrones de reglas y entrega una copia a cada target compatible.

### Respuesta profunda — 2 minutos

Las reglas pueden filtrar source, detail-type y campos de detail. Un evento puede coincidir con varias reglas, generando fan-out. Debo versionar el contrato y evitar reglas abiertas que reciban eventos no esperados.

### Ejemplo de mi proyecto

El atributo `topic` viaja desde el adapter SQS. La arquitectura SmartPay lo usa para dirigir resultados a notificación y orquestación.

### Posible repregunta

¿Qué diferencia hay entre una regla y una cola?

---

## 22. ¿Qué failure mode observaste en el listener SQS?

### Respuesta corta — 30 segundos

Si el processor transforma un error en `Mono.empty()`, el listener ve completion y ejecuta delete. Eso evita el retry del broker.

### Respuesta profunda — 2 minutos

El contrato debería distinguir éxito de fallo. Los errores procesables deben propagarse para que no ocurra ACK. Los errores permanentes pueden enviarse a una ruta explícita. Swallowing general elimina la posibilidad de DLQ y oculta pérdida.

### Ejemplo de mi proyecto

`SQSProcessor` y `SQSListener` contienen `onErrorResume(... -> Mono.empty())` en niveles que deben revisarse juntos.

### Posible repregunta

¿Cómo probarías que no se borra al fallar?

---

## 23. ¿Qué es el patrón outbox?

### Respuesta corta — 30 segundos

Guarda el cambio de negocio y el evento en la misma transacción local. Otro proceso publica la outbox y marca el evento como enviado.

### Respuesta profunda — 2 minutos

Evita el dual write DB + broker. La publicación puede repetirse, por eso el consumidor sigue siendo idempotente. El relay puede usar polling o CDC.

### Ejemplo de mi proyecto

Aplicaría outbox al registro de una novedad si su persistencia y publicación deben ocurrir de forma atómica.

### Posible repregunta

¿Outbox garantiza exactly-once end-to-end?

---

# Step Functions

## 24. ¿Cuál es la diferencia entre Standard y Express?

### Respuesta corta — 30 segundos

Standard sirve para procesos duraderos con historial y callbacks. Express optimiza ejecuciones cortas y de alto volumen.

### Respuesta profunda — 2 minutos

Cambian duración, historial, pricing y semántica. Standard permite espera prolongada y task tokens. Express se adapta a procesamiento breve y masivo; el diseño debe tolerar su modelo de entrega.

### Ejemplo de mi proyecto

La State Machine de novedades usa callbacks de hasta 900 segundos, apropiados para Standard.

### Posible repregunta

¿Qué costo controlarías en Standard?

---

## 25. ¿Qué hacen `Task` y `Choice`?

### Respuesta corta — 30 segundos

Task ejecuta una integración. Choice evalúa condiciones y selecciona la siguiente ruta.

### Respuesta profunda — 2 minutos

Task puede invocar Lambda, AWS SDK, HTTP o patrones optimizados. Choice no produce efectos externos; evalúa el input. Debe incluir `Default` cuando ningún caso coincide.

### Ejemplo de mi proyecto

`ValidateNovedadType` elige flujo y `UpdatePaymentPlan` publica a SQS con task token.

### Posible repregunta

¿Qué pasa si ningún Choice coincide y no existe Default?

---

## 26. ¿Qué diferencia hay entre `Map` y `Parallel`?

### Respuesta corta — 30 segundos

Map aplica un subflujo a elementos de una colección. Parallel ejecuta ramas diferentes sobre el mismo input.

### Respuesta profunda — 2 minutos

Map controla concurrencia e incluso modo Distributed para grandes colecciones. Parallel espera todas las ramas y combina resultados. Ambos requieren diseñar idempotencia por rama.

### Ejemplo de mi proyecto

El grafo de liquidación contiene recorridos por elementos y ramas, mientras la slide se concentra en el callback de novedades.

### Posible repregunta

¿Cómo limitarías concurrencia en Map?

---

## 27. ¿Cómo funcionan `Retry` y `Catch`?

### Respuesta corta — 30 segundos

Retry repite el estado para errores configurados. Al agotarse o no coincidir, Catch guarda el error y dirige a otro estado.

### Respuesta profunda — 2 minutos

Retry define `IntervalSeconds`, `BackoffRate`, `MaxAttempts` y opcional jitter según soporte. Debe limitarse a errores transitorios. Catch usa `ResultPath` para no perder el input original.

### Ejemplo de mi proyecto

`UpdatePaymentPlan` reintenta `States.TaskFailed` tres veces y dirige cualquier error a `PersistSingleError`.

### Posible repregunta

¿Qué riesgo tiene reintentar una Task no idempotente?

---

## 28. ¿Qué hacen `ResultPath` y `OutputPath`?

### Respuesta corta — 30 segundos

`ResultPath` decide dónde combinar el resultado con el input. `OutputPath` filtra lo que sale hacia el siguiente estado.

### Respuesta profunda — 2 minutos

Si `ResultPath` es `$`, reemplaza el input. Si es una ruta, agrega o reemplaza ese campo. `null` descarta el resultado. OutputPath se evalúa al final y puede eliminar información necesaria.

### Ejemplo de mi proyecto

El callback guarda salida en `$.updatePaymentPlanResult`; Catch guarda error en `$.error`.

### Posible repregunta

¿Qué diferencia hay con `Parameters`?

---

## 29. ¿Qué es el callback task token?

### Respuesta corta — 30 segundos

Step Functions genera un token, lo envía al worker y suspende la Task hasta recibir success, failure o timeout.

### Respuesta profunda — 2 minutos

El token identifica una Task específica y debe protegerse. El worker llama `SendTaskSuccess` con output o `SendTaskFailure`. El timeout evita ejecuciones suspendidas indefinidamente.

### Ejemplo de mi proyecto

`UpdatePaymentPlan` envía `$$.Task.Token` por SQS a liquidación y espera hasta 900 segundos.

### Posible repregunta

¿Qué haces con un callback que llega después del timeout?

---

# Docker y Compose

## 30. ¿Qué diferencia hay entre image y container?

### Respuesta corta — 30 segundos

La imagen es el artefacto inmutable por capas. El contenedor es un proceso que ejecuta esa imagen con configuración y una capa escribible.

### Respuesta profunda — 2 minutos

La imagen se identifica por digest. El runtime combina capas, namespaces, cgroups, red y mounts. El contenedor no es una VM y comparte el kernel del host.

### Ejemplo de mi proyecto

Compose construye `franquicias-app:latest` y ejecuta un contenedor para la aplicación y otro para Mongo.

### Posible repregunta

¿Qué datos sobreviven al reemplazar un contenedor?

---

## 31. ¿Qué son layers y cómo mejoran el build cache?

### Respuesta corta — 30 segundos

Cada instrucción genera una capa reutilizable. Si una instrucción cambia, Docker reconstruye esa capa y las siguientes.

### Respuesta profunda — 2 minutos

Copiar primero archivos de dependencias permite reutilizar descarga de Gradle cuando solo cambia código. Un contexto grande o `COPY . .` temprano invalida el cache.

### Ejemplo de mi proyecto

El Dockerfile copia archivos Gradle antes de módulos y ejecuta `bootJar` en la etapa builder.

### Posible repregunta

¿Cómo usarías BuildKit cache mounts con Gradle?

---

## 32. ¿Por qué multi-stage y JRE en runtime?

### Respuesta corta — 30 segundos

La primera etapa compila con JDK. La final copia solo el JAR a una imagen JRE, reduciendo tamaño y herramientas disponibles.

### Respuesta profunda — 2 minutos

Separar build y runtime mejora reproducibilidad y superficie de ataque. Aun se deben escanear ambas imágenes y fijar versiones o digests.

### Ejemplo de mi proyecto

Gradle JDK 21 Alpine produce el JAR; Temurin JRE 21 Alpine lo ejecuta.

### Posible repregunta

¿Alpine siempre produce la imagen más segura?

---

## 33. ¿Qué diferencia hay entre ENTRYPOINT y CMD y por qué no root?

### Respuesta corta — 30 segundos

ENTRYPOINT define el ejecutable principal y CMD sus argumentos por defecto. Un usuario no root limita privilegios si el proceso se compromete.

### Respuesta profunda — 2 minutos

La forma exec propaga señales directamente. `sh -c` expande variables, pero debería usar `exec` para que Java sea PID 1. Non-root se complementa con filesystem de solo lectura y capabilities mínimas.

### Ejemplo de mi proyecto

La imagen crea `appuser`, cambia con `USER appuser` y ejecuta el JAR mediante ENTRYPOINT.

### Posible repregunta

¿Cómo manejarías señales SIGTERM y graceful shutdown?

---

## 34. ¿Qué hace Docker Compose?

### Respuesta corta — 30 segundos

Describe varios contenedores, redes, volúmenes, variables, puertos y dependencias para un entorno reproducible.

### Respuesta profunda — 2 minutos

Compose crea una red donde los servicios se resuelven por nombre. Los volúmenes conservan datos. Healthchecks y `depends_on` ayudan al arranque, pero no reemplazan retries en runtime.

### Ejemplo de mi proyecto

La aplicación usa `mongodb://mongo:27017` y espera el healthcheck de Mongo.

### Posible repregunta

¿Usarías Compose como orquestador de producción?

---

# Kubernetes

## 35. ¿Qué son Pod, Deployment y Service?

### Respuesta corta — 30 segundos

Pod ejecuta contenedores. Deployment administra réplicas y rollouts. Service ofrece una dirección estable y balancea hacia Pods listos.

### Respuesta profunda — 2 minutos

Deployment crea ReplicaSets declarativos. El Service selecciona Pods por labels. Los Pods son reemplazables; el estado debe externalizarse o usar workloads apropiados.

### Ejemplo de mi proyecto

ASULADO define un Deployment con labels, una réplica y puerto 8080.

### Posible repregunta

¿Qué pasa si cambian los labels del Pod?

---

## 36. ¿Qué diferencia hay entre ConfigMap y Secret?

### Respuesta corta — 30 segundos

ConfigMap guarda configuración no sensible. Secret guarda datos sensibles, pero base64 no es cifrado.

### Respuesta profunda — 2 minutos

Debo habilitar encryption at rest, RBAC mínimo y preferir un gestor externo cuando corresponda. Montar un Secret como archivo permite rotación distinta a variables de entorno.

### Ejemplo de mi proyecto

El Deployment monta `application.yml` desde un Secret como volumen.

### Posible repregunta

¿Cómo rotarías una credencial sin reconstruir la imagen?

---

## 37. ¿Qué diferencia hay entre requests y limits?

### Respuesta corta — 30 segundos

Requests influyen en scheduling y reserva. Limits establecen el máximo que el contenedor puede usar.

### Respuesta profunda — 2 minutos

CPU por encima del limit se throttling. Memoria por encima puede causar OOMKill. Requests incorrectos reducen densidad o generan contención. Se ajustan con métricas reales.

### Ejemplo de mi proyecto

El Deployment parametriza CPU y memoria tanto para requests como limits.

### Posible repregunta

¿Por qué evitar un limit de CPU demasiado bajo en Java?

---

## 38. ¿Qué diferencia hay entre liveness y readiness?

### Respuesta corta — 30 segundos

Readiness decide si recibe tráfico. Liveness decide si Kubernetes debe reiniciar el contenedor.

### Respuesta profunda — 2 minutos

Una readiness fallida mantiene el proceso vivo y lo retira del Service. Una liveness fallida provoca restart. Una dependencia externa no debería tumbar liveness y causar cascadas.

### Ejemplo de mi proyecto

ASULADO usa probes TCP al puerto 8080 con delays y thresholds parametrizados.

### Posible repregunta

¿Qué agrega una startup probe?

---

## 39. ¿Cómo funciona HPA?

### Respuesta corta — 30 segundos

HPA compara métricas observadas con el objetivo y ajusta el número de réplicas dentro de mínimos y máximos.

### Respuesta profunda — 2 minutos

Para CPU basada en utilization necesita requests definidos. El escalado tiene retardos y no sustituye capacidad de downstream. Para colas puede ser mejor escalar por longitud o edad usando métricas externas.

### Ejemplo de mi proyecto

El HPA de novedades escala de una a dos réplicas al 80% de CPU.

### Posible repregunta

¿Qué problema crea escalar consumidores SQS sin idempotencia?

---

# Redis

## 40. ¿Qué es cache-aside?

### Respuesta corta — 30 segundos

La aplicación consulta caché; ante miss lee la base y guarda el resultado con TTL.

### Respuesta profunda — 2 minutos

La aplicación controla la estrategia. Redis no sabe cargar la BD. Debo definir comportamiento ante error, TTL, invalidación y concurrencia de misses.

### Ejemplo de mi proyecto

`ValidarActivacionTarjetaUseCase` consulta Redis, usa R2DBC ante miss y guarda la tarjeta.

### Posible repregunta

¿Qué diferencia hay con read-through?

---

## 41. ¿Cómo relacionas TTL e invalidación?

### Respuesta corta — 30 segundos

TTL limita la vida máxima. Invalidación elimina la clave cuando conozco un cambio y reduce stale data.

### Respuesta profunda — 2 minutos

Solo TTL acepta una ventana de inconsistencia. Solo invalidación puede dejar claves eternas si se pierde el evento. Combinar ambos limita el daño. Jitter evita expiraciones masivas simultáneas.

### Ejemplo de mi proyecto

Tarjetas usan TTL de cinco días y un método `invalidarTarjeta` ejecuta delete por número.

### Posible repregunta

¿Invalidas antes o después del commit de base de datos?

---

## 42. ¿Qué diferencia hay entre caché local y distribuida?

### Respuesta corta — 30 segundos

Local vive dentro de la instancia y es más rápida. Redis es compartida entre réplicas, pero agrega red y dependencia externa.

### Respuesta profunda — 2 minutos

Una caché local puede tener valores distintos en cada Pod. La distribuida facilita coherencia relativa y capacidad compartida. También requiere timeouts, pool, observabilidad y fallback.

### Ejemplo de mi proyecto

`ReactiveRedisTemplate` comparte tarjetas entre instancias del servicio.

### Posible repregunta

¿Combinarías L1 local y L2 Redis?

---

## 43. ¿Qué es cache stampede y cómo lo evitas?

### Respuesta corta — 30 segundos

Muchos requests ven el mismo miss y consultan simultáneamente la fuente. Se mitiga con single-flight, lock por clave, TTL con jitter o refresh-ahead.

### Respuesta profunda — 2 minutos

El lock necesita expiración y propietario. Stale-while-revalidate puede servir datos anteriores mientras uno renueva. La técnica depende de cuánto stale data tolere el negocio.

### Ejemplo de mi proyecto

Varios intentos de activación sobre la misma tarjeta podrían llegar juntos después de expirar la clave.

### Posible repregunta

¿Cómo evitarías que el lock se vuelva otro punto de falla?

---

## 44. ¿Qué estructuras de Redis conoces y cuándo usarías cada una?

### Respuesta corta — 30 segundos

Strings para valores simples, Hashes para campos, Lists para extremos ordenados, Sets para unicidad, Sorted Sets para ranking y Streams para logs consumibles.

### Respuesta profunda — 2 minutos

La estructura debe responder al patrón de acceso. Guardar siempre JSON en String obliga a leer y escribir el objeto completo. Hash permite campos; Sorted Set resuelve top-N; Set resuelve membresía; Stream mantiene IDs y consumer groups.

### Ejemplo de mi proyecto

El código mostrado usa Strings mediante `opsForValue`; la lista de tipos de tarjeta se serializa como un único valor.

### Posible repregunta

¿Usarías una List Redis como reemplazo de SQS?

---

## 45. ¿Qué son eviction policies y cómo responde la aplicación si Redis falla?

### Respuesta corta — 30 segundos

La política decide qué claves expulsar al llegar a maxmemory. Ante fallo de Redis, el servicio puede tratarlo como miss y usar la base, con protección para no saturarla.

### Respuesta profunda — 2 minutos

`allkeys-lru`, `allkeys-lfu`, `volatile-ttl` y `noeviction` cambian el comportamiento. El fallback debe incluir timeout, circuit breaker o rate limiting. De lo contrario, una caída de caché produce una tormenta sobre la BD.

### Ejemplo de mi proyecto

La lectura Redis usa `onErrorResume` a vacío, por lo que R2DBC actúa como fallback.

### Posible repregunta

¿Qué métrica indica que la caché dejó de aportar valor?

---

# Seguridad y multitenancy

## 46. ¿Qué diferencia hay entre decodificar y validar un JWT?

### Respuesta corta — 30 segundos

Decodificar solo lee header y payload. Validar comprueba firma, expiración, issuer y restricciones antes de confiar en los claims.

### Respuesta profunda — 2 minutos

Un atacante puede construir cualquier payload y codificarlo en Base64. El Resource Server obtiene claves públicas por JWKS, selecciona la clave por `kid` y verifica la firma. Después valida `exp`, `nbf`, issuer y audience según la configuración.

### Ejemplo de mi proyecto

`TenantJwtWebFilter` usa `ReactiveJwtDecoder.decode(token)` antes de extraer `custom:tenant_id` y `custom:tenant_profile`.

### Posible repregunta

¿Qué haces durante la rotación de claves JWKS?

---

## 47. ¿Por qué Reactor Context y no ThreadLocal para el tenant?

### Respuesta corta — 30 segundos

ThreadLocal pertenece al thread. Reactor Context pertenece a la suscripción y sigue el pipeline aunque cambie de thread.

### Respuesta profunda — 2 minutos

Netty multiplexa muchas solicitudes. Una cadena puede cruzar schedulers. Con ThreadLocal podría perder el tenant o, peor, leer uno dejado por otra ejecución. `contextWrite` y `deferContextual` conservan el contexto lógico.

### Ejemplo de mi proyecto

El filtro escribe `TenantContext` y `TenantConnectionFactoryWrapper` lo lee al crear la conexión R2DBC.

### Posible repregunta

¿En qué dirección ve el contexto un operador respecto a `contextWrite`?

---

## 48. ¿Cómo evitas fuga de datos entre schemas cuando existe un pool?

### Respuesta corta — 30 segundos

Configuro el `search_path` al adquirir la conexión, sanitizo el schema y lo restablezco antes de devolver la conexión al pool.

### Respuesta profunda — 2 minutos

Una conexión conserva estado de sesión. Si el schema queda configurado, el siguiente borrower puede consultar el tenant anterior. El wrapper devuelve una conexión que intercepta close, ejecuta el reset y después cierra el delegate.

### Ejemplo de mi proyecto

`SearchPathResettingConnection.close()` ejecuta `SET search_path` al schema por defecto.

### Posible repregunta

¿Qué prueba concurrente escribirías para verificar aislamiento?

---

# Arquitectura limpia

## 49. ¿Qué significa que el dominio no dependa de infraestructura?

### Respuesta corta — 30 segundos

Los casos de uso dependen de interfaces definidas hacia adentro. Mongo, HTTP o AWS implementan esas interfaces desde adapters externos.

### Respuesta profunda — 2 minutos

Las dependencias de compilación apuntan al dominio. Los modelos de negocio no importan tipos de Spring Data ni SDKs. La composición ocurre en el módulo de aplicación mediante DI.

### Ejemplo de mi proyecto

`FranchiseUseCase` depende de `FranchiseRepository`; `MongoRepositoryAdapter` implementa ese contrato.

### Posible repregunta

¿Dónde ubicas los DTO HTTP?

---

## 50. ¿Cuándo una interfaz aporta valor y cuándo solo agrega ruido?

### Respuesta corta — 30 segundos

Aporta valor en una frontera, dependencia externa o política sustituible. Agrega ruido cuando replica cada método sin separar responsabilidades.

### Respuesta profunda — 2 minutos

Un puerto debe expresar una capacidad que el dominio necesita. No se crea por cumplir una plantilla. Debe evitar filtrar detalles tecnológicos y mantener cohesión.

### Ejemplo de mi proyecto

`FranchiseRepository` aísla persistencia. Un helper puramente interno del caso de uso no necesita un gateway.

### Posible repregunta

¿Cómo evitarías un puerto demasiado grande?

---

## 51. ¿Dónde debe vivir timeout, retry y circuit breaker?

### Respuesta corta — 30 segundos

Cuando atienden fallos de una tecnología, viven en el adapter. Cuando forman parte de la regla del proceso, los coordina el caso de uso u orquestador.

### Respuesta profunda — 2 minutos

Un timeout de Mongo es infraestructura. Un retry de una operación de negocio no idempotente necesita conocimiento del caso. La ubicación debe preservar semántica y evitar políticas duplicadas en capas.

### Ejemplo de mi proyecto

`MongoResilienceExecutor` envuelve operaciones del adapter sin contaminar `FranchiseUseCase`.

### Posible repregunta

¿Qué errores no reintentarías?

---

# Testing

## 52. ¿Qué verifica StepVerifier que no muestra un `block()`?

### Respuesta corta — 30 segundos

Verifica la secuencia de señales, demanda, error, completion, cancelación y tiempo virtual sin convertir el publisher en una llamada síncrona.

### Respuesta profunda — 2 minutos

`block()` retorna o lanza, pero oculta orden de señales y backpressure. StepVerifier actúa como Subscriber controlado y permite pedir elementos por etapas.

### Ejemplo de mi proyecto

Las pruebas de recarga verifican completion normal, error fatal y efectos sobre gateways.

### Posible repregunta

¿Cómo pruebas que no se emitió ningún elemento?

---

## 53. ¿Cómo pruebas continuidad después de un error individual?

### Respuesta corta — 30 segundos

Configuro la primera llamada para fallar y la segunda para responder. Después verifico completion y los dos resultados persistidos en orden.

### Respuesta profunda — 2 minutos

La prueba debe diferenciar un error convertido en dato del error terminal del publisher. También comprueba que se invocó el siguiente registro y que el resumen final contiene un éxito y un fallo.

### Ejemplo de mi proyecto

`debeContinuarCuandoFallaUnaRecarga` devuelve timeout y luego éxito; espera `RECARGA_ERROR` y `RECARGA_EXITOSA`.

### Posible repregunta

¿Qué operador permite esa continuidad en la implementación?

---

## 54. ¿Cómo probarías retry y backoff sin esperar tiempo real?

### Respuesta corta — 30 segundos

Usaría `StepVerifier.withVirtualTime`, avanzaría el reloj virtual y verificaría número de intentos y señal final.

### Respuesta profunda — 2 minutos

El publisher debe construirse dentro del supplier de `withVirtualTime` para que capture el scheduler virtual. Después se usa `thenAwait` y verificaciones del mock.

### Ejemplo de mi proyecto

Aplicaría esta técnica a un adapter con `Retry.backoff`, no al Retry declarativo de Step Functions, que se valida sobre ASL y pruebas de integración.

### Posible repregunta

¿Qué error excluirías del retry?

---

## 55. ¿Qué diferencia hay entre una prueba de use case y una de integración?

### Respuesta corta — 30 segundos

La prueba del use case aísla puertos y reglas. La de integración verifica wiring, serialización, configuración y tecnologías reales o emuladas.

### Respuesta profunda — 2 minutos

Los mocks permiten explorar ramas rápidamente, pero no detectan propiedades mal nombradas, codecs incompatibles o drivers mal conectados. Necesito una pirámide que combine ambas.

### Ejemplo de mi proyecto

El use case comprueba lotes y errores con mocks. Los módulos también contienen pruebas de entry points y adapters.

### Posible repregunta

¿Qué parte del fan-out probarías con LocalStack?

## 56. ¿Qué significa ACID y dónde aparece en tu implementación?

### Respuesta corta — 30 segundos

ACID significa Atomicidad, Consistencia, Aislamiento y Durabilidad. En ASULADO, `TransactionalOperator` delimita la unidad R2DBC; constraints e índices protegen invariantes; PostgreSQL aplica el aislamiento configurado y confirma cambios durables al hacer commit.

### Respuesta profunda — 2 minutos

Atomicidad revierte toda la unidad cuando la cadena emite error. Consistencia combina reglas de negocio con constraints. Aislamiento define qué observan operaciones concurrentes. Durabilidad protege un commit exitoso. El framework delimita la transacción, pero el motor entrega las garantías y solo para recursos participantes.

### Ejemplo de mi proyecto

`DeductionSettlementChangeUseCase` llama `ReactiveTransactionPort`; `ReactiveTransactionAdapter` envuelve el `Mono` con `transactionalOperator.transactional(operation)`.

### Posible repregunta

¿Una llamada SQS dentro de esa cadena también hace rollback?

---

## 57. ¿Por qué usar `TransactionalOperator` en código reactivo?

### Respuesta corta — 30 segundos

Porque una transacción reactiva se asocia al contexto de la suscripción, no a un `ThreadLocal`. `TransactionalOperator` mantiene la frontera a través del publisher y confirma o revierte según su señal terminal.

### Respuesta profunda — 2 minutos

En Reactor una ejecución puede cambiar de hilo. El operador transaccional obtiene y vincula la conexión en el contexto reactivo. Todos los repositorios participantes deben ejecutarse dentro de la misma cadena. Un `subscribe()` interno crea otra suscripción y puede escapar de la transacción.

### Ejemplo de mi proyecto

`ReactiveTransactionAdapter` usa `Mono.defer` y delega a `TransactionalOperator`; el dominio solo conoce `ReactiveTransactionPort`.

### Posible repregunta

¿Qué problema evita `Mono.defer` en ese adaptador?

---

## 58. ¿Cuándo hace commit o rollback una transacción reactiva?

### Respuesta corta — 30 segundos

Normalmente confirma cuando el publisher termina correctamente y revierte cuando termina con error. Si convierto el error en completion antes de la frontera, el operador puede interpretar que debe confirmar.

### Respuesta profunda — 2 minutos

La transacción observa señales reactivas. Por eso el lugar de `onErrorResume` importa. También debo considerar cancelación y no mantener transacciones abiertas durante I/O externo lento. Las pruebas unitarias que solo mockean el operador no demuestran rollback real; necesito una integración contra la base.

### Ejemplo de mi proyecto

El caso de cambio de deducciones encierra toda la actualización dentro de `executeWithinTransaction` y aplica retry fuera de esa unidad.

### Posible repregunta

¿Por qué colocarías el retry fuera de la transacción?

---

## 59. ¿Qué recursos quedan fuera de una transacción R2DBC?

### Respuesta corta — 30 segundos

SQS, EventBridge, DynamoDB y APIs HTTP no participan automáticamente. La transacción cubre el recurso administrado por el `ReactiveTransactionManager`.

### Respuesta profunda — 2 minutos

Intentar una transacción distribuida añade coordinación y acoplamiento. Para base más mensajería prefiero transactional outbox: cambio de negocio y evento en la misma transacción local, publicación posterior reintentable y consumidor idempotente.

### Ejemplo de mi proyecto

ASULADO combina PostgreSQL y mensajería. La frontera R2DBC protege escrituras relacionales, pero el evento necesita una estrategia explícita de consistencia.

### Posible repregunta

¿Cómo evitas publicar dos veces desde la outbox?

---

## 60. ¿Qué niveles de aislamiento conoces y qué anomalías controlan?

### Respuesta corta — 30 segundos

`READ COMMITTED` evita dirty reads; `REPEATABLE READ` mantiene lecturas estables y reduce phantoms según el motor; `SERIALIZABLE` busca equivalencia con ejecución secuencial, pero puede abortar transacciones y exigir retry.

### Respuesta profunda — 2 minutos

El aislamiento controla dirty reads, non-repeatable reads, phantoms y conflictos de escritura. No usaría siempre el nivel máximo: mayor aislamiento puede aumentar locks, abortos y latencia. Elijo según la invariante y pruebo concurrencia real.

### Ejemplo de mi proyecto

Para impedir dos preliquidaciones activas no dependería solo del aislamiento; el índice único parcial convierte la invariante en una garantía de base.

### Posible repregunta

¿Cómo manejarías una serialization failure?

---

## 61. ¿Cómo decides crear un índice y cuál es su costo?

### Respuesta corta — 30 segundos

Parto de una consulta y su plan. Un índice puede reducir filas examinadas, pero ocupa espacio y encarece insert, update y delete. Valido el cambio con `EXPLAIN ANALYZE` y métricas representativas.

### Respuesta profunda — 2 minutos

Reviso filtros, joins, orden, selectividad y orden de columnas. Un índice compuesto no sirve igual para cualquier prefijo. Un índice parcial reduce tamaño cuando el predicado coincide con el acceso. Un índice unique además protege consistencia.

### Ejemplo de mi proyecto

Liquibase crea un índice único parcial sobre participante, período y número de pago solo cuando el estado no es `CANCELADO`, además de índices de consulta por participante y período.

### Posible repregunta

¿Por qué el optimizador podría ignorar un índice?

---

## 62. ¿Cuándo usarías un procedimiento almacenado?

### Respuesta corta — 30 segundos

Cuando una operación intensiva sobre datos se beneficia de ejecutarse cerca del motor, requiere una unidad transaccional controlada o la comparten varios consumidores. Evaluaría el costo de acoplamiento y versionado.

### Respuesta profunda — 2 minutos

Puede reducir viajes de red y aprovechar el optimizador, pero desplaza lógica a otro runtime, dificulta portabilidad y exige observabilidad, pruebas y migraciones propias. Mantendría reglas de dominio cambiantes en la aplicación y usaría funciones para trabajo claramente orientado a datos.

### Ejemplo de mi proyecto

Kire contiene funciones PostgreSQL para reportes y operaciones masivas; las trataría como artefactos versionados y mediría sus planes antes de optimizarlas.

### Posible repregunta

¿Cómo probarías y desplegarías una función sin bloquear producción?

---

## 63. ¿Cómo se representan CRUD y actualización condicional en DynamoDB?

### Respuesta corta — 30 segundos

`PutItem` crea o reemplaza, `GetItem` lee por clave, `UpdateItem` modifica atributos y `DeleteItem` elimina. Las condition expressions evitan sobrescrituras o implementan control optimista.

### Respuesta profunda — 2 minutos

`PutItem` reemplaza el ítem completo con la misma clave si no hay condición. `UpdateItem` permite cambios parciales y contadores atómicos. Para creación idempotente usaría `attribute_not_exists(pk)`. El CRUD se diseña alrededor de claves y patrones de acceso, no de joins.

### Ejemplo de mi proyecto

`PreLiquidationParametersDynamoAdapter` usa `PutItem` y `GetItem`; `PreLiquidationDynamoRepositoryAdapter` usa `Query` para todos los ítems de un participante.

### Posible repregunta

¿Cómo implementarías optimistic locking?

---

## 64. ¿Qué diferencia hay entre partition key y sort key?

### Respuesta corta — 30 segundos

La partition key determina dónde se distribuye el ítem. La sort key ordena y distingue varios ítems dentro de la misma partition key, permitiendo consultas por rango y prefijo.

### Respuesta profunda — 2 minutos

Una clave simple tiene solo PK; una compuesta tiene PK y SK. Debo distribuir tráfico para evitar hot partitions. La SK puede codificar jerarquías o tiempo, pero el formato debe responder a consultas concretas y evitar colecciones de ítems descontroladas.

### Ejemplo de mi proyecto

El adapter consulta `id_participante = :pk`. DynamoDB dirige `Query` a ese grupo y el código pagina con `LastEvaluatedKey`.

### Posible repregunta

¿Qué ocurre si todos los eventos usan la misma partition key?

---

## 65. ¿Qué diferencia hay entre GSI y LSI?

### Respuesta corta — 30 segundos

Un GSI puede usar otra partition key y otra sort key, se administra después y solo ofrece lectura eventual. Un LSI conserva la partition key, cambia la sort key, se define al crear la tabla y permite lectura fuerte.

### Respuesta profunda — 2 minutos

El GSI soporta patrones de acceso globales y tiene capacidad y almacenamiento propios, pero su propagación es asíncrona. El LSI consulta otra ordenación dentro de la misma partition key y comparte restricciones de esa colección. No creo un índice “por si acaso”; parto del patrón de acceso.

### Ejemplo de mi proyecto

El código revisado consulta la tabla base y no demuestra un GSI o LSI configurado. Los usaría solo si aparece una consulta que la clave actual no resuelve eficientemente.

### Posible repregunta

¿Por qué `ConsistentRead(true)` no funciona sobre un GSI?

---

## 66. ¿Qué diferencia hay entre `Query` y `Scan` en DynamoDB?

### Respuesta corta — 30 segundos

`Query` usa una partition key y opcionalmente condiciones sobre la sort key. `Scan` examina todos los ítems antes de filtrar, por lo que suele consumir más capacidad y crecer peor.

### Respuesta profunda — 2 minutos

Ambas operaciones paginan hasta 1 MB por respuesta. Un filter expression no reduce la capacidad consumida por los ítems leídos. Para rutas frecuentes diseño una clave o índice que permita `Query`; reservo `Scan` para operaciones controladas o administrativas.

### Ejemplo de mi proyecto

`PreLiquidationDynamoRepositoryAdapter` construye un `QueryRequest` por `id_participante` y usa `expand` para continuar mientras exista `LastEvaluatedKey`.

### Posible repregunta

¿Cómo impondrías límites de demanda al paginar reactivamente?

---

## 67. ¿Cómo relacionas CAP, BASE y `ConsistentRead` en DynamoDB?

### Respuesta corta — 30 segundos

CAP importa cuando hay partición: no puedo garantizar a la vez consistencia lineal y disponibilidad para todas las solicitudes. BASE acepta estado temporal y convergencia eventual. DynamoDB permite elegir consistencia por lectura; `ConsistentRead(true)` pide el dato confirmado más reciente en tabla o LSI, no en GSI.

### Respuesta profunda — 2 minutos

No etiquetaría todo DynamoDB simplemente como AP. Una lectura eventual reduce costo y tolera datos temporalmente antiguos; una fuerte exige una garantía mayor y consume más capacidad. GSI replica asíncronamente y siempre es eventual. La decisión depende de cuánto stale data tolera el caso de uso y qué respuesta espero durante fallos.

### Ejemplo de mi proyecto

El adapter de parámetros usa `GetItem` con `consistentRead(true)`, mientras el `Query` de preliquidaciones usa la consistencia por defecto. Son decisiones diferentes por operación.

### Posible repregunta

¿Qué regla de negocio justificaría pagar lectura fuerte?

