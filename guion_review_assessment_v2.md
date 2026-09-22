# Guion técnico — Review de Assessment v2

**Duración objetivo:** 20–30 minutos.  
**Regla de exposición:** describir el problema, recorrer el flujo, señalar la decisión y cerrar con el trade-off. No leer el código línea por línea.

---

# Slide 1 — Review de Assessment

## Qué mostrar

Portada simple con nombre y propósito.

## Qué decir

“Voy a mostrar implementaciones concretas relacionadas con las áreas del assessment. En cada caso voy a explicar el problema, el flujo, el código y la decisión técnica.”

## Explicación técnica

No introducir tecnologías todavía. Aclarar que las rutas de los archivos aparecen al pie de cada slide para poder ubicar el código.

## Preguntas que me pueden hacer

¿Cuánto dura la presentación?

## Respuesta

“Entre veinte y treinta minutos. Reservé la mayor parte para Reactor, Spring y la arquitectura de mensajería.”

## Pregunta difícil

¿La presentación cubre todo tu trabajo?

## Respuesta profunda

“No. Seleccioné casos donde una implementación permite examinar varias competencias a la vez y donde puedo recorrer el flujo completo desde la entrada hasta persistencia o infraestructura.”

---

# Slide 2 — Assessment anterior

## Qué mostrar

La tabla compacta con las siete competencias, el nivel anterior y el foco señalado.

## Qué decir

“El assessment ubicó Frameworks, Concurrencia, Contenedores y Caché en nivel 3. Protocolos, Pruebas y Cloud estaban en nivel 4. Las siguientes slides conectan esos focos con implementaciones de Kire, ASULADO, SmartPay y Franquicias.”

## Explicación técnica

Explicar que Spring requiere comprender el contenedor de beans, no solo usar anotaciones. Concurrencia requiere controlar demanda, orden y operaciones bloqueantes. Contenedores incluye la construcción de la imagen y la operación en Kubernetes. Caché exige estrategia, expiración e invalidación.

## Preguntas que me pueden hacer

¿Qué diferencia esperas demostrar entre un nivel 3 y un nivel 4?

## Respuesta

“Que puedo ubicar el concepto en código real, explicar cómo funciona, identificar sus límites y justificar la decisión frente a alternativas.”

## Pregunta difícil

¿Vas a argumentar nivel 5?

## Respuesta profunda

“La evidencia presentada permite discutir implementación frecuente y profundidad técnica. La valoración final depende del alcance que el modelo de competencias exija para ese nivel.”

---

# Slide 3 — WebFlux: procesamiento masivo

## Qué mostrar

El flujo S3 → `Flux` → `buffer(100)` → `concatMap` → persistencia/API → estado final, junto al fragmento de `EjecucionRecargaMasivaUseCase`.

## Qué decir

“El archivo se obtiene mediante `S3AsyncClient` como un publisher de `ByteBuffer`. El adaptador conserva los bytes de una línea que cruza dos buffers y emite registros de manera incremental. El caso de uso agrupa cien registros, procesa un lote por vez y calcula un resumen al terminar. Un fallo individual se convierte en un resultado técnico persistible, mientras un fallo del flujo termina el lote conservando el progreso.”

## Explicación técnica

- `leer()` devuelve `Flux<RegistroRecargaMasiva>` porque puede emitir cero, uno o muchos registros.
- `buffer(100)` cambia la forma del flujo a `Flux<List<RegistroRecargaMasiva>>`.
- `concatMap` espera que el publisher interno actual termine antes de suscribir el siguiente lote.
- `then(Mono.defer(...))` descarta los elementos previos y ejecuta el resumen solamente después de completar el procesamiento.
- `materialize()` convierte las señales `onNext` y `onError` de la recarga externa en un objeto `Signal`; el caso de uso persiste éxito o error por registro sin cancelar el resto.
- `onErrorResume` exterior atiende errores fatales como la pérdida del stream S3.

## Preguntas que me pueden hacer

¿Por qué usar `Flux` y no `List`?

## Respuesta

“Una lista exige tener todos los elementos antes de comenzar. El `Flux` permite procesarlos conforme llegan y propaga demanda hacia el publisher.”

## Pregunta difícil

¿El sistema nunca carga el archivo completo en memoria?

## Respuesta profunda

“La lectura y el parseo sí son incrementales. Cada lote mantiene hasta cien registros más los datos consultados para ese lote. La ruta de escritura inicial del adaptador todavía serializa una lista completa antes de subirla, por lo que la afirmación de streaming aplica a la lectura y al procesamiento.”

---

# Slide 4 — Reactor: concurrencia y backpressure

## Qué mostrar

La comparación visual de `flatMap` y `concatMap`, el parser incremental y los failure modes.

## Qué decir

“Reactor no convierte automáticamente una secuencia en paralela. `flatMap` puede mantener varios publishers internos activos y emitir según terminan. `concatMap` limita esa concurrencia a uno y conserva orden. En este caso elegí `concatMap` porque el servicio externo y la persistencia debían recibir carga controlada.”

## Explicación técnica

Reactive Streams define cuatro actores principales: `Publisher`, `Subscriber`, `Subscription` y `Processor`. Tras `subscribe`, el Subscriber recibe una `Subscription` y expresa demanda con `request(n)`. Los operadores traducen esa demanda aguas arriba.

`map` transforma un valor de forma síncrona y devuelve un valor. `flatMap` espera una función que devuelve otro publisher y luego combina publishers. `concatMap` también aplana publishers, pero los suscribe secuencialmente.

`publishOn` cambia el scheduler para los operadores posteriores. `subscribeOn` influye en la suscripción a la fuente. `boundedElastic` sirve para aislar trabajo bloqueante inevitable, pero no lo vuelve no bloqueante.

## Preguntas que me pueden hacer

¿Qué controla exactamente `buffer(100)`?

## Respuesta

“Controla el tamaño de cada colección emitida. No establece por sí solo el número de lotes procesados en paralelo; esa parte la determina `concatMap` o el parámetro de concurrencia de `flatMap`.”

## Pregunta difícil

¿`AtomicBoolean` evita dos ejecuciones en producción?

## Respuesta profunda

“Evita solapamiento dentro de una única JVM. Si el despliegue tiene varias réplicas, cada instancia conserva su propio booleano. La garantía distribuida debe venir de `reclamarSiguiente()` mediante un update condicional, un lock distribuido o un mecanismo de leasing.”

---

# Slide 5 — Spring: DI y configuración

## Qué mostrar

Los dos `ConnectionFactory`, los `R2dbcEntityTemplate` y el código con `@Bean`, `@Primary` y `@Qualifier`.

## Qué decir

“El problema aparece porque READ y WRITE implementan el mismo tipo. Spring no puede decidir por tipo cuál inyectar. Cada factory recibe un nombre de bean. `@Qualifier` selecciona el bean exacto en el punto de inyección y `@Primary` define el candidato por defecto cuando no hay qualifier.”

## Explicación técnica

Al arrancar, Spring escanea configuraciones y componentes, crea `BeanDefinition`, resuelve dependencias y construye el grafo dentro del `ApplicationContext`. Un método `@Bean` permite registrar objetos cuya construcción controlo explícitamente, como clientes SDK o factories. `@Component` registra una clase detectada por escaneo.

Los beans singleton se crean normalmente al arrancar. Spring también soporta `prototype`, `request`, `session` y otros scopes web. Funciones como `@Transactional` o `@Cacheable` suelen aplicarse mediante proxies. La autoinvocación dentro de la misma instancia puede saltarse el proxy.

## Preguntas que me pueden hacer

¿Qué pasa si retiro `@Qualifier`?

## Respuesta

“Spring encuentra dos beans compatibles. Si uno está marcado `@Primary`, lo selecciona. Si no existe un primario o hay más de uno, el contexto falla por ambigüedad.”

## Pregunta difícil

¿Por qué no crear manualmente el template donde se utiliza?

## Respuesta profunda

“Eso acoplaría el consumidor a credenciales y configuración. El contenedor centraliza el ciclo de vida, permite reemplazar dependencias en pruebas y mantiene explícito el grafo de objetos.”

---

# Slide 6 — Multitenancy reactivo

## Qué mostrar

El flujo JWT → `ReactiveJwtDecoder` → `TenantContext` → `ConnectionFactory` → `SET search_path`.

## Qué decir

“El filtro recibe el Bearer token y usa `ReactiveJwtDecoder`, que valida la firma mediante JWKS. Extrae `tenant_id` y `tenant_profile`, crea un `TenantContext` y lo inserta con `contextWrite`. Cuando R2DBC solicita una conexión, el wrapper lee ese contexto, sanitiza el schema y ejecuta `SET search_path`.”

## Explicación técnica

`ThreadLocal` asocia datos al hilo. En Reactor, una misma suscripción puede continuar en diferentes threads por cambios de scheduler o drivers asíncronos. Reactor Context pertenece a la suscripción y se propaga siguiendo el pipeline.

`contextWrite` modifica el contexto que los operadores aguas arriba observan durante la suscripción. `deferContextual` espera hasta la suscripción para leerlo.

El wrapper restablece el `search_path` antes de devolver la conexión al pool. Sin ese reset, una conexión reutilizada podría consultar el schema de otro tenant.

## Preguntas que me pueden hacer

¿Cuál es la diferencia entre decodificar y validar un JWT?

## Respuesta

“Decodificar Base64 solo permite leer header y claims. Validar verifica la firma, issuer, expiración y demás restricciones antes de confiar en esos claims.”

## Pregunta difícil

¿Qué riesgo existe al usar contexto vacío cuando el token es inválido?

## Respuesta profunda

“El filtro de tenant opera en modo compatible, pero no debe ser el mecanismo que decide acceso. Spring Security debe rechazar el token en la cadena de autenticación. Si un endpoint protegido dependiera solo del tenant vacío, podría terminar usando el schema por defecto.”

---

# Slide 7 — Arquitectura limpia

## Qué mostrar

El recorrido HTTP Handler → Input Port → Use Case → Repository Port → Mongo Adapter.

## Qué decir

“El dominio define `FranchiseInputPort` y `FranchiseRepository`. El caso de uso depende de esa interfaz, no de Spring Data ni de Mongo. El adapter implementa el contrato y agrega timeout y circuit breaker mediante el ejecutor de resiliencia.”

## Explicación técnica

La regla de dependencias apunta hacia el dominio. El entry point traduce HTTP a un modelo de dominio. El caso de uso coordina reglas. El gateway define lo que necesita. El adapter traduce ese contrato a una tecnología.

La arquitectura no elimina acoplamiento; lo orienta hacia abstracciones estables. El precio consiste en mappers, interfaces y composición adicional.

## Preguntas que me pueden hacer

¿Cada clase necesita una interfaz?

## Respuesta

“No. Creo un puerto cuando existe una frontera arquitectónica o una dependencia que el dominio no debe conocer. Una interfaz sin sustitución o sin frontera útil agrega ruido.”

## Pregunta difícil

¿Dónde debe vivir la lógica de retry?

## Respuesta profunda

“Si el retry responde a fallos técnicos de Mongo, vive en el adapter. Si representa una regla del proceso, por ejemplo reintentar una operación de negocio con condiciones, puede pertenecer al caso de uso u orquestador.”

---

# Slide 8 — AWS Fan-Out en ASULADO

## Qué mostrar

El código que agrega `topic` a los `MessageAttributes` y el diagrama q-novedades → Pipe → EventBridge → colas independientes.

## Qué decir

“El microservicio construye un `OutboundMessage` con un topic, por ejemplo `NOVEDAD_RECIBIDA`. El adapter serializa el payload y agrega el topic y metadata como atributos SQS. Un Pipe lleva el mensaje al bus de EventBridge. Las rules evalúan el topic y envían el evento a las colas de notificación o a la entrada de la Step Function. Cada consumidor tiene backlog y ritmo propios.”

## Explicación técnica

SQS proporciona almacenamiento temporal, polling, visibility timeout y entrega al menos una vez. EventBridge proporciona un bus y reglas de routing. En este flujo no se encontró SNS; no se debe afirmar que forma parte de esta implementación.

La arquitectura muestra una excepción: el tramo desde `ms-novedades` hasta el motor de reglas sigue siendo pipe-a-pipe. El resultado del motor sí pasa por EventBridge y allí ocurre el fan-out hacia notificación core y la State Machine.

## Preguntas que me pueden hacer

¿Por qué una cola por consumidor?

## Respuesta

“Porque cada consumidor necesita disponibilidad, velocidad, retries y backlog independientes. Si notificación falla, la orquestación no debería detenerse.”

## Pregunta difícil

¿Cuándo elegirías SNS en lugar de EventBridge?

## Respuesta profunda

“SNS resulta adecuado para pub/sub directo con routing relativamente simple y alto fan-out. EventBridge ofrece reglas más expresivas, buses por dominio, integración con múltiples destinos y mejor desacoplamiento semántico. SQS complementa ambos cuando necesito durabilidad, buffer y control del consumidor.”

---

# Slide 9 — Mensajería: entrega y fallos

## Qué mostrar

El ciclo receive → visibility timeout → procesar → delete y el código real del listener.

## Qué decir

“El listener hace long polling, entrega el mensaje al processor y solo después ejecuta delete. Ese orden permite retry si el processor propaga un error. Standard SQS puede entregar duplicados, por lo que el consumidor debe ser idempotente.”

## Explicación técnica

Durante el visibility timeout, otros consumidores no ven el mensaje. Si el consumidor termina sin borrar, el mensaje reaparece. Si el tiempo es menor que la duración real del procesamiento, dos consumidores pueden procesarlo a la vez.

Una redrive policy mueve el mensaje a una DLQ después de `maxReceiveCount`. No se encontró esa política en los manifiestos revisados, por lo que se explica como configuración necesaria, no como parte confirmada del repositorio.

El `SQSProcessor` contiene ramas que convierten errores en `Mono.empty()`. Para el listener, `Mono.empty()` significa finalización exitosa y conduce al delete. Ese comportamiento debe revisarse cuando se espere retry o DLQ.

## Preguntas que me pueden hacer

¿Qué ocurre si no se ejecuta `deleteMessage`?

## Respuesta

“El mensaje vuelve a estar visible al vencer el timeout y otro polling puede recibirlo. Por eso la operación debe tolerar duplicados.”

## Pregunta difícil

¿Cómo implementarías idempotencia?

## Respuesta profunda

“Persistiría un identificador estable del evento con una operación condicional. El consumidor intenta registrar ese ID y solo ejecuta el efecto si la inserción gana. El registro y el efecto necesitan una frontera transaccional o un diseño de inbox/outbox para evitar estados intermedios.”

---

# Slide 10 — Step Functions

## Qué mostrar

El subflujo Choice → SQS `waitForTaskToken` → Liquidación → callback, el fragmento ASL y el grafo real como contexto.

## Qué decir

“`UpdatePaymentPlan` publica un mensaje SQS que incluye `$$.Task.Token`. La ejecución queda suspendida hasta recibir `SendTaskSuccess` o `SendTaskFailure`, o hasta llegar al timeout de 900 segundos. Los fallos `TaskFailed` tienen backoff exponencial y tres intentos. `Catch` persiste el error en una ruta explícita.”

## Explicación técnica

- `Task` integra un servicio o ejecuta trabajo.
- `Choice` selecciona una ruta sin ejecutar trabajo externo.
- `Retry` reintenta el mismo estado según tipo de error.
- `Catch` captura el error final y redirige.
- `ResultPath` combina el resultado con el input.
- `OutputPath` filtra lo que el estado entrega al siguiente.
- El task token convierte una interacción asíncrona en una espera administrada por Step Functions.

Standard conserva historial y permite ejecuciones largas. Express se orienta a alto volumen y corta duración, con semántica y costos diferentes.

## Preguntas que me pueden hacer

¿Por qué no una Lambda grande?

## Respuesta

“Una Lambda grande tendría que persistir progreso, implementar retries, ramificación y recuperación. La State Machine hace visibles esos estados y permite reanudar desde el punto controlado.”

## Pregunta difícil

¿Qué pasa si el callback nunca llega?

## Respuesta profunda

“El estado vence por `TimeoutSeconds`, genera un error y entra al `Catch`. Debo correlacionar el token con el trabajo externo, protegerlo como dato sensible y diseñar compensación para callbacks tardíos.”

---

# Slide 11 — Containers

## Qué mostrar

El Dockerfile multi-stage, el flujo JDK builder → JAR → JRE runtime y el YAML de Kubernetes.

## Qué decir

“La primera etapa compila con Gradle y JDK 21. La segunda contiene solamente JRE, el JAR y un usuario sin privilegios. Compose levanta la aplicación y Mongo con healthchecks. En Kubernetes se declaran requests, limits, probes, filesystem de solo lectura y HPA.”

## Explicación técnica

Una imagen es un artefacto inmutable por capas. Un contenedor es una instancia de ejecución de esa imagen. Multi-stage permite descartar Gradle, fuentes y JDK después del build.

`ENTRYPOINT` define el ejecutable principal. `CMD` suele aportar argumentos por defecto. La forma exec evita un shell, aunque el Dockerfile actual usa `sh -c` para expandir `JAVA_OPTS`.

Requests participan en scheduling y garantizan recursos. Limits ponen techo. Readiness retira temporalmente el Pod del Service. Liveness reinicia un proceso que no puede recuperarse.

## Preguntas que me pueden hacer

¿Por qué JRE y no JDK en runtime?

## Respuesta

“El JRE contiene lo necesario para ejecutar. El JDK agrega compilador y herramientas que aumentan tamaño y superficie de ataque.”

## Pregunta difícil

¿El healthcheck TCP garantiza que la aplicación funciona?

## Respuesta profunda

“No. Solo demuestra que el puerto acepta conexiones. Un endpoint de readiness puede validar dependencias esenciales con cuidado de no crear reinicios en cascada. Liveness debe comprobar que el proceso puede progresar, no que todos los servicios externos estén disponibles.”

---

# Slide 12 — Redis y cache-aside

## Qué mostrar

El flujo GET → hit/miss → DB → SET TTL y los fragmentos de `ValidarActivacionTarjetaUseCase` y `TarjetaRedisService`.

## Qué decir

“El caso de uso consulta Redis. Ante miss, consulta R2DBC y guarda la tarjeta con TTL de cinco días. Las modificaciones pueden invalidar la clave por número de tarjeta. Es el patrón cache-aside: la aplicación coordina lectura, carga y expiración.”

## Explicación técnica

TTL limita el tiempo máximo de un dato potencialmente obsoleto, pero no sustituye la invalidación cuando se conoce el cambio. Un cache hit evita la BD. Un miss agrega latencia y puede causar stampede si muchas solicitudes consultan la misma clave.

Redis distribuido permite compartir caché entre réplicas y agrega latencia de red. Una caché local es más rápida, pero cada instancia conserva una copia distinta.

El código usa Strings con objetos serializados. Redis también ofrece Hashes, Lists, Sets, Sorted Sets, Streams y estructuras probabilísticas. Elegir estructura depende del patrón de acceso, no solo del tipo del objeto Java.

## Preguntas que me pueden hacer

¿Qué ocurre si Redis falla?

## Respuesta

“El código convierte el error de lectura en un miss y consulta la base de datos. Eso preserva disponibilidad, pero puede aumentar carga sobre R2DBC.”

## Pregunta difícil

¿Qué problema tiene el `subscribe()` interno?

## Respuesta profunda

“Crea una segunda suscripción fuera de la cadena principal. La escritura queda separada de cancelación, contexto y manejo de errores. La compondría con `flatMap(tarjeta -> guardarTarjeta(tarjeta).onErrorResume(...).thenReturn(tarjeta))` para mantener una única suscripción.”

---

# Slide 13 — Testing reactivo

## Qué mostrar

Las pruebas de 201 registros y continuidad después de un timeout individual.

## Qué decir

“Las pruebas no validan solo que el publisher complete. Verifican que 201 registros generen tres consultas masivas, que el servicio externo reciba 201 llamadas y que un timeout individual produzca un registro de error sin impedir el siguiente éxito.”

## Explicación técnica

`StepVerifier` se suscribe al publisher y verifica la secuencia de señales. `verifyComplete()` espera finalización normal. `expectError` exige una señal terminal de error. Los `verify` de Mockito comprueban efectos sobre gateways.

El caso fatal de S3 emite cien registros y luego `onError`; la prueba verifica que el lote termine como parcial y conserve los cien éxitos anteriores.

## Preguntas que me pueden hacer

¿Por qué no usar `block()` en la prueba?

## Respuesta

“`StepVerifier` conserva la semántica de señales y permite probar demanda, tiempo virtual, cancelación, valores y errores sin convertir el pipeline en una llamada síncrona.”

## Pregunta difícil

¿Estas pruebas demuestran backpressure?

## Respuesta profunda

“Demuestran agrupación, orden observable y continuidad. Para probar demanda explícita usaría `StepVerifier.create(publisher, 0)`, luego `thenRequest(n)` y verificaría que la fuente no emite más allá de lo solicitado.”

---

# Slide 14 — Bases de datos: ACID, TCL, CAP y BASE

## Qué mostrar

El adaptador que aplica `TransactionalOperator`, el caso de uso que encierra varias operaciones en la unidad transaccional, el índice único parcial de Liquibase y dos accesos reales a DynamoDB: `Query` por partition key con paginación y `GetItem` con lectura fuerte.

## Qué decir

“En ASULADO la unidad de negocio entra por un puerto transaccional y el adaptador la ejecuta con `TransactionalOperator`. Si el publisher completa, Spring confirma; si emite error, revierte las operaciones R2DBC de esa transacción. La consistencia también se protege con un índice único parcial. En DynamoDB el acceso está modelado por claves: consulto por participante, recorro páginas con `LastEvaluatedKey` y, cuando necesito leer el valor más reciente, activo `consistentRead`.”

## Explicación técnica

ACID separa cuatro garantías. Atomicidad significa todo o nada dentro de la transacción. Consistencia significa que constraints, tipos e invariantes llevan la base de un estado válido a otro. Aislamiento controla qué pueden observar transacciones concurrentes; `READ COMMITTED`, `REPEATABLE READ` y `SERIALIZABLE` reducen anomalías con costos crecientes. Durabilidad significa que un commit exitoso sobrevive a fallos posteriores según las garantías del motor.

`TransactionalOperator` asocia la transacción al contexto reactivo. El `Mono.defer` evita construir o reutilizar la operación fuera del momento de suscripción. La frontera solo cubre recursos administrados por el `ReactiveTransactionManager`; una publicación SQS, una escritura DynamoDB o una llamada HTTP no se revierte con PostgreSQL.

En DynamoDB una partition key determina la distribución y una sort key permite agrupar, ordenar y consultar rangos dentro de la partición. Un GSI puede definir otra partition key y otra sort key; sus lecturas son eventualmente consistentes. Un LSI conserva la partition key, cambia la sort key, se define al crear la tabla y puede solicitar lectura fuerte. El código mostrado consulta la tabla base; GSI y LSI se explican como decisiones de modelado, no como infraestructura ya implementada.

CAP solo obliga una elección cuando hay una partición de red. BASE describe sistemas que priorizan disponibilidad y permiten estado temporal hasta converger. DynamoDB permite elegir consistencia por operación: `GetItem` y la tabla base admiten `consistentRead(true)`, mientras un GSI no.

## Preguntas que me pueden hacer

¿`@Transactional` y `TransactionalOperator` garantizan ACID por sí solos?

## Respuesta

“No. Delimitan la transacción en el framework. Las garantías reales dependen del motor, el nivel de aislamiento, los constraints y de que todas las operaciones participen en el mismo recurso transaccional.”

## Pregunta difícil

¿Cómo mantienes consistencia cuando la operación también publica un mensaje?

## Respuesta profunda

“No intentaría extender la transacción PostgreSQL sobre SQS. Persistiría el cambio y un evento outbox en la misma transacción. Otro proceso publicaría el evento de forma reintentable e idempotente. Así evito el estado imposible de confirmar la base y perder el mensaje, o publicar el mensaje y luego revertir la base.”

---

# Slide 15 — Mapa de competencias

## Qué mostrar

La tabla final que conecta cada competencia con el código o arquitectura ya presentada.

## Qué decir

“Este mapa resume el vínculo entre assessment e implementación. Frameworks aparece en configuración de beans y seguridad. Concurrencia en el pipeline S3/Reactor. Cloud en mensajería y Step Functions. Containers en Docker y Kubernetes. Caché en Redis cache-aside. Testing en las señales y efectos verificados.”

## Explicación técnica

No introducir evidencia nueva. Si el evaluador elige una fila, volver a la slide técnica correspondiente.

## Preguntas que me pueden hacer

¿Cuál caso cubre más competencias?

## Respuesta

“La recarga masiva combina Spring, Reactor, S3, R2DBC, integración externa, manejo de fallos y pruebas.”

## Pregunta difícil

¿Qué implementarías a continuación para fortalecer estas decisiones?

## Respuesta profunda

“En mensajería cerraría idempotencia y redrive policy con pruebas de duplicados. En caché eliminaría el `subscribe()` interno y probaría concurrencia de misses. En el worker distribuido comprobaría que el claim del lote sea atómico entre réplicas.”

---

# Slide 16 — Preguntas

## Qué mostrar

La frase final y la lista de tecnologías.

## Qué decir

“Las brechas se trabajaron implementando. La revisión incluye ahora transacciones R2DBC, ACID, optimización relacional y modelado DynamoDB con CAP y BASE. Quedo atento a las preguntas y puedo abrir cualquiera de las rutas mostradas.”

## Explicación técnica

No agregar un resumen oral largo. Usar esta slide como índice para navegar con teclado a la evidencia solicitada.

## Preguntas que me pueden hacer

¿Puedes abrir el código?

## Respuesta

“Sí. Cada slide indica el repositorio y el archivo principal.”

## Pregunta difícil

¿Cuál decisión cambiarías hoy?

## Respuesta profunda

“Eliminaría las suscripciones internas en los casos donde aparecen, propagaría correctamente los errores del consumidor SQS y haría explícita la idempotencia en el límite de mensajería.”
