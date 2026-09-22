# Guion para presentar — Review de Assessment

## Cómo usar este guion

- No memorizar palabra por palabra. Recordar por slide: **mensaje → evidencia → trade-off → transición**.
- Ritmo sugerido: 18–22 minutos para 21 slides; detenerse más en los cuatro casos.
- Si preguntan por una tecnología no implementada, responder: “No tengo evidencia suficiente para afirmarlo; sí puedo explicar el concepto y qué faltaría validar”.

---

## Slide 1 — De conocimiento técnico a criterio y dominio

### Objetivo
Establecer una conversación basada en evidencia y no en una solicitud de nota.

### Qué decir
“Quiero mostrar la evolución desde el último assessment. No preparé un inventario de tecnologías: seleccioné implementaciones donde puedo explicar el problema, la decisión, el código, los trade-offs y los límites.”

### Evidencia
Los cinco entregables y los repositorios analizados.

### Conceptos importantes
Evidencia, trazabilidad, criterio técnico.

### Posible pregunta del evaluador
¿Buscas justificar directamente un nivel 5?

### Respuesta recomendada
“Busco que la valoración surja de la evidencia. Voy a diferenciar lo consolidado, lo compartido y lo que todavía debo fortalecer.”

### Pregunta de profundización
¿Qué entiendes por dominio?

### Respuesta
“Poder explicar funcionamiento interno, justificar la elección, anticipar fallos, proponer alternativas y orientar a otros; no solo hacer que el código funcione.”

### Transición
“Con ese criterio, esta es la tesis que conecta toda la presentación.”

---

## Slide 2 — Mensaje central

### Objetivo
Resumir el cambio de conocimiento a aplicación y criterio.

### Qué decir
“El crecimiento ocurrió cuando las brechas aparecieron dentro de problemas reales: autenticación multitenant, procesamiento masivo, despliegue de contenedores y orquestación de procesos. La teoría se volvió decisión y la decisión dejó rastros verificables.”

### Evidencia
Commits, código, pruebas y validaciones locales.

### Conceptos importantes
Brecha, problema, decisión, resultado, aprendizaje.

### Posible pregunta del evaluador
¿Cuál fue la evidencia más transformadora?

### Respuesta recomendada
“La recarga masiva: integra Reactor, S3, lotes, persistencia, errores parciales y pruebas; me obliga a defender concurrencia y límites, no solo operadores.”

### Pregunta de profundización
¿Qué diferencia hay entre una implementación y una evidencia de dominio?

### Respuesta
“La implementación prueba aplicación. El dominio aparece cuando puedo explicar por qué se diseñó así, cuándo fallaría y qué cambiaría ante otro contexto.”

### Transición
“Para evaluar el cambio, primero recordemos el punto de partida exacto.”

---

## Slide 3 — Punto de partida

### Objetivo
Mostrar que las brechas provienen literalmente del assessment.

### Qué decir
“El assessment dejó cuatro prioridades calificadas en 3: Frameworks, Concurrencia, Contenedores y Caché. Protocolos, Pruebas y Cloud estaban en 4. Por eso mi defensa se concentra en las cuatro brechas, no en todo lo que aparece en los repositorios.”

### Evidencia
`last_assestment.md`.

### Conceptos importantes
Nivel previo, brecha directa, foco.

### Posible pregunta del evaluador
¿Cuál era la observación específica de Frameworks?

### Respuesta recomendada
“Profundizar Spring Boot, beans, DI, `Qualifier` y mecanismos de autenticación/JWT.”

### Pregunta de profundización
¿Por qué no centrarte en las competencias que ya estaban en 4?

### Respuesta
“Las uso como soporte para demostrar profundidad, pero la evolución medible debe compararse con las brechas de nivel 3.”

### Transición
“Para no confundir presencia de código con experiencia propia, apliqué un método de evidencia.”

---

## Slide 4 — Método de evidencia

### Objetivo
Dar credibilidad al análisis y explicar las cifras.

### Qué decir
“Verifiqué tres capas: código real, autoría Git y ejecución. Corrí 162 pruebas del módulo de use cases de Kire, 71 en Franquicias y 39 en la Lambda Python. También validé Compose, Terraform, SAM y referencias de la máquina de estados.”

### Evidencia
- Kire: `./gradlew :usecase:clean :usecase:test`.
- Franquicias: `./gradlew test`.
- Python: `pytest -q`.
- `terraform validate`, `sam validate --lint`.

### Conceptos importantes
Reproducibilidad, autoría, validación.

### Posible pregunta del evaluador
¿Esas cifras representan cobertura?

### Respuesta recomendada
“No. Representan pruebas ejecutadas y exitosas. No encontré un porcentaje de cobertura consolidado y no lo invento.”

### Pregunta de profundización
¿Una suite verde garantiza calidad?

### Respuesta
“No. Da confianza sobre comportamientos cubiertos; todavía necesito revisar calidad de assertions, integración real, contratos y riesgos no modelados.”

### Transición
“Con esa regla, agrupé la evidencia en cuatro contextos.”

---

## Slide 5 — Mapa de proyectos

### Objetivo
Ubicar rápidamente los casos y separar autoría propia de trabajo de equipo.

### Qué decir
“Kire aporta recarga masiva y auth; Franquicias muestra un proyecto de extremo a extremo; SmartPay novedades aporta orquestación AWS y Python; ASULADO complementa con experiencia de equipo en WebFlux, R2DBC y Kubernetes.”

### Evidencia
Commits posteriores al 24/03/2026 y estructura de cada repositorio.

### Conceptos importantes
Caso de estudio, ownership, trabajo colaborativo.

### Posible pregunta del evaluador
¿Todo el código mostrado es tuyo?

### Respuesta recomendada
“No. La matriz distingue autoría alta, compartida y exposición. Por ejemplo, la librería tenant fue creada por otro autor; yo implementé integraciones y configuraciones de consumo.”

### Pregunta de profundización
¿Cómo atribuyes una clase modificada por varios autores?

### Respuesta
“Reviso el historial del archivo y los diffs de los commits que introducen la decisión concreta que defiendo.”

### Transición
“Esa separación produce un mapa honesto de cierre de brechas.”

---

## Slide 6 — Tres avances y una brecha consciente

### Objetivo
Adelantar la conclusión sin sobreafirmar.

### Qué decir
“Spring, concurrencia y contenedores tienen evidencia fuerte. Caché tiene código real en el entorno, pero no suficiente autoría ni variedad de estrategias. No la presentaré como consolidada.”

### Evidencia
`matriz_evidencias_assessment.md`.

### Conceptos importantes
Fortaleza de evidencia, evidencia insuficiente.

### Posible pregunta del evaluador
¿Reconocer una brecha no debilita tu caso?

### Respuesta recomendada
“Al contrario: demuestra criterio y confiabilidad. Prefiero defender tres cierres sólidos y un plan verificable que atribuirme trabajo ajeno.”

### Pregunta de profundización
¿Qué cerraría la brecha de caché?

### Respuesta
“Una implementación propia con cache-aside, TTL/jitter, invalidación, estructura Hash o Sorted Set, prueba de stampede y métricas hit/miss.”

### Transición
“Empiezo con el caso que conecta más competencias: recarga masiva.”

---

## Slide 7 — Recarga masiva sin cargar todo en memoria

### Objetivo
Demostrar Reactor, procesamiento masivo, S3, control de errores y testing.

### Qué decir
“El archivo se recupera con `S3AsyncClient` como publisher de buffers. El adapter reconstruye líneas incrementalmente. El caso de uso agrupa 100 registros, consulta bonos por lote, procesa secuencialmente y persiste un resultado por registro. Al final deriva COMPLETADA, PARCIAL o FALLIDA.”

### Evidencia
- `ArchivoRecargaMasivaS3Adapter.java`, 69–110.
- `EjecucionRecargaMasivaUseCase.java`, 46–69 y 196–230.
- Commit `0bfed0f`.

### Conceptos importantes
Streaming, backpressure, `buffer`, `concatMap`, partial success.

### Posible pregunta del evaluador
¿Cómo evitas cargar el archivo completo?

### Respuesta recomendada
“En lectura, S3 emite `ByteBuffer`; solo conservo la línea incompleta y emito registros. Después `buffer(100)` acota el lote en memoria. La carga inicial sí serializa una lista completa y es una mejora pendiente.”

### Pregunta de profundización
¿Qué pasa si falla la recarga 500?

### Respuesta
“La llamada se `materialize`, se construye un resultado de error, se persiste y el flujo continúa. El lote termina PARCIAL si hubo éxitos. Un reproceso debe ser selectivo e idempotente.”

### Transición
“La parte más importante no es que haya operadores, sino por qué escogí esa semántica.”

---

## Slide 8 — Criterio reactivo

### Objetivo
Mostrar comprensión de concurrencia vs orden y límites de la solución.

### Qué decir
“`concatMap` no busca máxima velocidad. Busca orden y presión predecible sobre Redeban. Cambiarlo a `flatMap` con concurrencia puede mejorar throughput, pero solo después de validar capacidad, idempotencia y orden de efectos.”

### Evidencia
Uso de `concatMap` en lectura/lotes/registros y `AtomicBoolean` en el worker.

### Conceptos importantes
Concurrencia, paralelismo, orden, downstream capacity.

### Posible pregunta del evaluador
¿Reactivo no debería ser concurrente por defecto?

### Respuesta recomendada
“Reactivo describe composición y demanda; la concurrencia depende del operador y scheduler. `concatMap` es reactivo aunque serialice publishers.”

### Pregunta de profundización
¿Cómo aumentarías concurrencia de forma segura?

### Respuesta
“`flatMap(fn, N)` con N bajo y configurable, rate limiter, timeout, métricas, pruebas de idempotencia y validación de orden requerido.”

### Transición
“El segundo caso muestra que también profundicé en cómo Spring construye y conecta la solución.”

---

## Slide 9 — Spring y autenticación

### Objetivo
Cerrar la brecha de beans/DI y vincularla con seguridad real.

### Qué decir
“En Auth no solo uso anotaciones. Configuro dos `ConnectionFactory`, marco escritura como `@Primary` y selecciono lectura/escritura con `@Qualifier`. El flujo de login combina Cognito, validación de usuario activo y contexto Reactor por tenant.”

### Evidencia
- `R2dbcConfig.java`, 26–99; commit `5ecbb00`.
- `CognitoService.java`, 58–195.
- `IniciarSesionUseCase.java`, 26–74.

### Conceptos importantes
IoC, beans, qualifier/primary, DI, Cognito, Reactor Context.

### Posible pregunta del evaluador
¿Diferencia entre `@Primary` y `@Qualifier`?

### Respuesta recomendada
“`@Primary` define el candidato por defecto por tipo. `@Qualifier` expresa exactamente cuál dependencia quiero en ese punto.”

### Pregunta de profundización
¿Por qué `boundedElastic` en Cognito?

### Respuesta
“El cliente usado es síncrono y bloquea. Lo encapsulo para no bloquear el event loop. No lo convierte en async; preferiría el cliente async si está disponible.”

### Transición
“Ese criterio también aparece al empaquetar y operar el software.”

---

## Slide 10 — Contenedores

### Objetivo
Demostrar Docker y Compose con trade-offs.

### Qué decir
“Franquicias usa build multi-stage: compila con Gradle/JDK y ejecuta solo el JAR en un JRE. El runtime usa usuario no root y límite de memoria consciente del contenedor. Compose agrega Mongo, volumen y healthchecks.”

### Evidencia
- `deployment/Dockerfile`, 1–26.
- `deployment/docker-compose.yml`, 1–40.
- `docker compose config --quiet` exitoso.

### Conceptos importantes
Image layers, multi-stage, non-root, network, volume, healthcheck.

### Posible pregunta del evaluador
¿Qué optimización aplicaste?

### Respuesta recomendada
“Separé toolchain de build del runtime y usé JRE en vez de JDK. También ejecuto sin root. No uso Distroless y no lo afirmo.”

### Pregunta de profundización
¿Compose es un orquestador?

### Respuesta
“Coordina servicios locales, pero no resuelve despliegue distribuido, autoscaling o reconciliación como Kubernetes/ECS.”

### Transición
“El contenedor se vuelve relevante cuando explico dónde y cómo corre.”

---

## Slide 11 — Franquicias en AWS

### Objetivo
Conectar contenedores, IaC, seguridad y observabilidad.

### Qué decir
“Terraform publica API Gateway, entra por VPC Link al ALB interno y ejecuta ECS Fargate sin IP pública. Secrets Manager entrega la URI de Mongo; CloudWatch y access logs dan trazabilidad.”

### Evidencia
`deployment/terraform/service/main.tf`; commits `72673fb`, `3dca907`, `aa7ef75`; `terraform validate` exitoso.

### Conceptos importantes
VPC, subred pública/privada, SG, IAM, ECS, ECR, observabilidad.

### Posible pregunta del evaluador
¿Por qué API Gateway delante de ALB?

### Respuesta recomendada
“Mantiene el ALB privado y concentra capacidades de borde como throttling, auth, dominio y WAF. El costo es una capa y precio adicional.”

### Pregunta de profundización
¿Dónde aplicas mínimo privilegio?

### Respuesta
“El rol de ejecución solo obtiene el secreto Mongo específico; SG permiten VPC Link→ALB y ALB→ECS por puertos definidos.”

### Transición
“En SmartPay, el reto no era solo ejecutar un servicio sino coordinar un proceso multietapa.”

---

## Slide 12 — Step Functions

### Objetivo
Demostrar orquestación, resiliencia e idempotencia con evidencia de autoría.

### Qué decir
“La máquina actual tiene 52 estados y combina Lambda, HTTP, SQS con task token y DynamoDB. Mis commits agregaron la Lambda bancaria, la decisión de dispersión, el caso cuenta-sin-cambio y beneficiarios. Retry y Catch hacen explícitos los fallos.”

### Evidencia
`novedades.asl.json`; commits `c00c265`, `f665417`, `2c72a17`, `a534c14`, `892ad67`; SAM válido.

### Conceptos importantes
Task, Choice, ResultPath, Retry/Catch, task token, idempotencia.

### Posible pregunta del evaluador
¿Por qué no una Lambda única?

### Respuesta recomendada
“Hay múltiples decisiones, integraciones y callbacks. Step Functions externaliza estado, retries y observabilidad; una Lambda larga concentraría todo y sería más difícil de recuperar.”

### Pregunta de profundización
¿Step Functions garantiza exactly-once?

### Respuesta
“No. Diseño idempotencia por correlationId, conditional writes/locks y operaciones seguras. SQS y varios servicios son al menos una vez.”

### Transición
“La máquina no es solo una gráfica: una Lambda implementa parte de la decisión de negocio.”

---

## Slide 13 — Lambda Python

### Objetivo
Demostrar Python, integración y optimización de flujo.

### Qué decir
“La Lambda normaliza el evento, consulta la novedad y el siguiente pago, compara la cuenta existente y evita escribir cuando no cambió. Devuelve campos que permiten a Step Functions saltar el paso correcto.”

### Evidencia
`src/lambda_handler.py`, `register_handler.py`, commits de mayo/junio y 39 tests.

### Conceptos importantes
Mapping, clientes, errores tipados, decisión idempotente, observabilidad con traceId.

### Posible pregunta del evaluador
¿Qué patrón ves en los handlers?

### Respuesta recomendada
“Un registry selecciona una implementación de un contrato de handler. La base fue de equipo; mis cambios ampliaron el flujo y la comparación de cuentas.”

### Pregunta de profundización
¿Dónde colocarías retries HTTP?

### Respuesta
“Preferentemente en la orquestación o cliente con política explícita solo para fallos transitorios/idempotentes. Evitaría duplicar retries entre Lambda y Step Functions.”

### Transición
“Los casos anteriores se sostienen porque el dominio está aislado de infraestructura.”

---

## Slide 14 — Arquitectura y resiliencia

### Objetivo
Explicar Clean/Hexagonal y circuit breaker con código.

### Qué decir
“El Handler conoce HTTP y llama un input port. El use case conoce contratos del dominio. Mongo implementa el output port. Por eso puedo probar negocio sin Spring/Mongo y concentrar timeout/circuit breaker en el adapter.”

### Evidencia
`ProductUseCase.java`, gateways, `Handler.java`, `MongoResilienceExecutor.java`.

### Conceptos importantes
DIP, ports/adapters, cohesión, circuit breaker, timeout.

### Posible pregunta del evaluador
¿Clean Architecture es solo la estructura de carpetas?

### Respuesta recomendada
“No. La prueba es la dirección de dependencias: el dominio no importa WebFlux ni Mongo; infraestructura implementa contratos definidos hacia adentro.”

### Pregunta de profundización
¿Por qué `transformDeferred` para el circuit breaker?

### Respuesta
“Aplica el operador por suscripción y conserva semántica lazy; evita compartir estado de transformación incorrectamente en la cadena.”

### Transición
“Esa separabilidad se refleja en las pruebas que ejecuté.”

---

## Slide 15 — Testing

### Objetivo
Mostrar calidad mediante escenarios significativos.

### Qué decir
“Las pruebas no solo cubren happy path: verifican 201 registros en tres lotes, continuidad tras error individual y estado parcial después de 100 éxitos si S3 falla.”

### Evidencia
`EjecucionRecargaMasivaUseCaseTest.java`, 73–207.

### Conceptos importantes
StepVerifier, error signal, completion, mocks, efectos.

### Posible pregunta del evaluador
¿Por qué StepVerifier?

### Respuesta recomendada
“Porque se suscribe y verifica señales del publisher; puedo comprobar emisión, completion y error sin romper el modelo con `block()`.”

### Pregunta de profundización
¿Esas pruebas demuestran TDD?

### Respuesta
“Demuestran cobertura de comportamientos. Para afirmar TDD describiría el ciclo Red–Green–Refactor usado; no lo deduzco solo del estado final.”

### Transición
“La misma regla de honestidad aplica a la brecha donde la evidencia no alcanza.”

---

## Slide 16 — Caché: evidencia insuficiente

### Objetivo
Mostrar autoconocimiento y plan de cierre.

### Qué decir
“Encontré Redis reactivo, TTL e invalidación, pero el historial atribuye esa implementación a otros autores. Tampoco hay tres estrategias ni estructuras diferentes. Por eso no la presento como cerrada.”

### Evidencia
`UsuarioVentaMasivaRedisService`, `RedisCacheService` e historial Git.

### Conceptos importantes
Cache-aside, TTL, invalidación, stampede, Hash/ZSet.

### Posible pregunta del evaluador
¿Entonces qué aprendiste de caché?

### Respuesta recomendada
“Puedo explicar estrategias y riesgos, pero todavía debo convertirlo en evidencia propia. Mi siguiente práctica debe medir hits/misses, stale data y stampede.”

### Pregunta de profundización
¿Cómo diseñarías cache-aside robusto?

### Respuesta
“Key versionada, TTL con jitter, single-flight/lock por key, fallback controlado, invalidación por evento y métricas. La fuente sigue siendo autoridad.”

### Transición
“Con esa precisión, el antes y el ahora quedan así.”

---

## Slide 17 — Antes vs ahora

### Objetivo
Comparar evolución sin convertirla en auto-calificación.

### Qué decir
“Ahora puedo vincular cada brecha con una implementación y también con un límite. Spring tiene wiring real; concurrencia tiene streaming y fronteras de bloqueo; contenedores llegan a ECS; caché tiene un plan, no una afirmación.”

### Evidencia
Tabla resumida y matriz completa.

### Conceptos importantes
Evolución, profundidad, limitaciones.

### Posible pregunta del evaluador
¿Qué te falta para nivel 5?

### Respuesta recomendada
“Más evidencia de liderazgo repetible: métricas productivas, decisiones registradas, mentoring y cierre de caché con implementación propia.”

### Pregunta de profundización
¿Cuál evidencia muestra capacidad de diseño?

### Respuesta
“Terraform de Franquicias y ampliaciones de Step Functions: conectan seguridad, operación, fallos y negocio, no solo clases aisladas.”

### Transición
“La matriz sintetiza la fuerza de cada argumento.”

---

## Slide 18 — Matriz de cierre

### Objetivo
Presentar una conclusión trazable y rápida.

### Qué decir
“La fuerza alta exige código, autoría y validación. Frameworks, concurrencia, contenedores, Cloud y testing cumplen. Caché queda baja. Esta tabla no asigna una nota nueva: organiza evidencia para la conversación.”

### Evidencia
`matriz_evidencias_assessment.md`.

### Conceptos importantes
Nivel de evidencia, no nivel autoasignado.

### Posible pregunta del evaluador
¿Por qué Kubernetes aparece compartido?

### Respuesta recomendada
“Modifiqué un Deployment, pero la base y otros manifiestos son del equipo. Mi evidencia directa de contenedores es Docker/Compose/ECS.”

### Pregunta de profundización
¿Qué evidencia rechazarías como débil?

### Respuesta
“Un README sin código/commit, una tecnología solo en dependencias o una métrica no reproducible.”

### Transición
“Más importante que la tabla es lo que hoy puedo sostener frente a preguntas.”

---

## Slide 19 — Lo que puedo explicar y defender

### Objetivo
Mostrar profundidad y comunicación para distintas audiencias.

### Qué decir
“A desarrollo puedo explicar operadores, lifecycle, contratos y tests. A arquitectura, aislamiento, resiliencia y costos. A negocio, continuidad parcial, riesgo y por qué una decisión protege la operación.”

### Evidencia
Los cuatro casos y sus trade-offs.

### Conceptos importantes
Comunicación técnica, criterio, impacto.

### Posible pregunta del evaluador
Explícale backpressure a negocio.

### Respuesta recomendada
“Es como una banda transportadora: el receptor indica cuánto puede manejar para que el productor no le arroje más trabajo del que puede procesar.”

### Pregunta de profundización
Ahora explícalo a un desarrollador.

### Respuesta
“El `Subscriber` solicita N elementos mediante `Subscription.request(n)` y el `Publisher` respeta esa demanda; operadores pueden transformar o acumularla.”

### Transición
“Para llegar con esa claridad al jueves, cierro con un plan corto y priorizado.”

---

## Slide 20 — Plan al jueves

### Objetivo
Mostrar preparación deliberada, no improvisación.

### Qué decir
“Hoy cierro Spring y concurrencia; mañana contenedores y caché; el miércoles Step Functions/JWT y dos simulaciones; el jueves solo repaso evidencia y riesgos.”

### Evidencia
`guia_estudio_assessment.md`.

### Conceptos importantes
P0/P1, active recall, simulación.

### Posible pregunta del evaluador
¿Qué priorizarías si solo tuvieras una hora?

### Respuesta recomendada
“Recarga masiva, beans/Qualifier y respuesta honesta de caché; son las brechas más probables.”

### Pregunta de profundización
¿Cómo sabes que puedes explicarlo y no solo leerlo?

### Respuesta
“Ensayo respuestas de 60–90 segundos sin pantalla, después verifico contra la ruta de evidencia y corrijo omisiones.”

### Transición
“Con eso, cierro con la idea que quiero dejar.”

---

## Slide 21 — Cierre

### Objetivo
Dejar una conclusión firme, honesta y no defensiva.

### Qué decir
“La evolución que presento es pasar de implementar piezas a entender y defender sistemas: cómo fluyen, dónde fallan, qué cuestan y cómo mejorarlos. La evidencia es fuerte en Spring, concurrencia y contenedores; profundiza Cloud y arquitectura; y reconoce caché como el siguiente cierre.”

### Evidencia
Matriz, validaciones y rutas mostradas.

### Conceptos importantes
Autonomía, criterio, siguiente nivel.

### Posible pregunta del evaluador
Resume en una frase por qué tu nivel evolucionó.

### Respuesta recomendada
“Porque hoy puedo conectar una brecha con una decisión implementada, una validación, un trade-off y una mejora siguiente.”

### Pregunta de profundización
¿Qué compromiso técnico asumes después del review?

### Respuesta
“Cerrar caché con evidencia propia y añadir métricas operativas a los casos que hoy solo puedo validar por código y tests.”

### Transición
“Quedo atento a profundizar en cualquiera de los casos y abrir el código exacto.”
