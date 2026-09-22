# Informe de hallazgos — Review de Assessment

## Resumen ejecutivo

La historia defendible no es “aprendí más tecnologías”. Es: **después del assessment del 24 de marzo, Andrés tomó decisiones e implementó flujos reales en Spring/WebFlux, procesamiento masivo, autenticación, contenedores, IaC y orquestación AWS**. Tres brechas calificadas en 3 muestran evidencia fuerte de avance; caché no alcanza el mismo estándar y debe tratarse con transparencia.

## Evidencias más fuertes

### 1. Recarga masiva reactiva — Kire

- Autoría verificable en commits `0bfed0f`, `532e71e` y `1234318`.
- S3 se consume con `S3AsyncClient` como `Flux` de buffers; el CSV se reconstruye línea a línea.
- El caso de uso procesa lotes de 100 con `buffer` y `concatMap`, registra resultados individuales y conserva estados `COMPLETADA`, `PARCIAL` o `FALLIDA`.
- No hay métricas productivas de throughput: no afirmar “procesa miles por segundo”.
- Validación local: **162 pruebas de usecase pasaron, 0 fallos** el 21/09/2026.

### 2. Spring Boot, seguridad y multitenancy — Kire Auth

- `R2dbcConfig` muestra `@Bean`, `@Primary` y `@Qualifier` en dos conexiones R2DBC.
- `CognitoService` adapta un SDK bloqueante con `boundedElastic`.
- `IniciarSesionUseCase` combina Cognito, estado de usuario en BD y contexto Reactor por tenant.
- La extracción local de claims decodifica el payload; la validación criptográfica JWKS vive en la librería compartida. No confundir ambas responsabilidades.

### 3. Docker/Compose + AWS con Terraform — Franquicias

- Dockerfile multi-stage, JRE runtime y usuario no root.
- Compose levanta API + Mongo, volumen y healthchecks.
- Terraform implementa API Gateway público, VPC Link, ALB interno y ECS Fargate en subred privada; Secrets Manager, ECR y CloudWatch completan seguridad/operación.
- Validación local: `docker compose config`, `terraform fmt -check` y `terraform validate` exitosos; **71 pruebas Java** pasaron.

### 4. Step Functions + Lambda Python — SmartPay

- Commits de Andrés agregan Lambda de datos bancarios, ramas Choice, omisión de pasos cuando la cuenta no cambió y flujo de beneficiario.
- La state machine actual tiene **52 estados**: 30 Task, 9 Choice, 5 Pass, 5 Fail y 3 Succeed; 95 referencias de transición sin destinos faltantes.
- Integra Lambda, HTTP con EventBridge Connection, SQS `waitForTaskToken` y DynamoDB para idempotencia/locks.
- `sam validate --lint` exitoso y **39 pruebas Python** pasaron.

### 5. Resiliencia y Clean Architecture — Franquicias

- Puertos de dominio aíslan Mongo y WebFlux.
- `MongoResilienceExecutor` aplica timeout y circuit breaker de manera diferida para cada suscripción.
- El diseño transforma errores técnicos en errores de negocio controlados.

## Brechas claramente trabajadas

### Frameworks — de 3 a evidencia compatible con 4

Hay implementación frecuente de Spring Boot, DI por constructor, configuración de beans, selección con `@Qualifier/@Primary`, WebFlux, validación, manejo de errores y seguridad Cognito. El argumento debe ser “puedo explicar y mantener el wiring”, no “soy experto absoluto en todo Spring”.

### Concurrencia — de 3 a evidencia compatible con 4

Hay evidencia de event loop, offloading de SDK bloqueante, streaming asíncrono de S3, backpressure natural, lotes, orden con `concatMap`, exclusión local con `AtomicBoolean` y pruebas de continuidad parcial.

### Contenedores — de 3 a evidencia compatible con 4

Docker y Docker Compose son implementación directa y validada. Kubernetes aporta exposición real a Deployment/HPA/probes/securityContext, aunque la autoría es compartida.

## Brechas parcialmente trabajadas

### Seguridad dentro de Frameworks

Hay login, challenge, refresh, sign-out y JWT/claims. Sin embargo, “dos mecanismos de autenticación” podría significar dos esquemas distintos (por ejemplo OAuth2/OIDC y API key/mTLS), no solo varios flujos Cognito. Preparar esta aclaración.

### Kubernetes

Andrés corrigió un Deployment y conoce sus piezas, pero el grueso de los manifiestos fue creado por otros. Presentarlo como experiencia compartida, no como diseño integral propio.

## Brechas sin evidencia suficiente

### Caché — principal riesgo

- Existe Redis reactivo con TTL e invalidación dentro de Kire.
- El historial Git atribuye la implementación principal a otros autores.
- No se halló evidencia atribuible de tres estrategias (cache-aside, read-through, write-through, write-behind) ni de estructuras Redis distintas a value/string.
- **Conclusión:** no solicitar que esta brecha se considere completamente cerrada.
- Evidencia faltante: laboratorio o PR propio que implemente al menos cache-aside con TTL/jitter y una segunda estructura (Hash/Sorted Set), incluya invalidación, prueba de concurrencia/cache stampede y métricas hit/miss.

## Temas que podrían cuestionarme

1. ¿Por qué `concatMap` si WebFlux permite concurrencia? Porque Redeban y la consistencia por bono favorecen orden y presión controlada; la concurrencia debe introducirse solo con límite explícito y pruebas de idempotencia.
2. ¿La lectura S3 es realmente streaming? La descarga sí llega como publisher de `ByteBuffer`; el parser conserva solo la línea incompleta. La **carga** inicial todavía serializa una lista completa a `String`, por lo que no debe venderse como streaming de subida.
3. ¿`boundedElastic` hace no bloqueante un SDK? No. Aísla el bloqueo en un pool acotado; la alternativa preferible es un SDK async.
4. ¿`AtomicBoolean` evita duplicados en varias réplicas? No. Solo protege una JVM; la reclamación atómica en persistencia o un lock distribuido es la protección entre instancias.
5. ¿Decodificar JWT equivale a validarlo? No. Decodificar obtiene claims; validar exige firma, `iss`, `aud`, expiración y demás constraints.
6. ¿Step Functions garantiza exactly-once? No. Hay ejecución al menos una vez en varias integraciones; la idempotencia se construye con claves, estados y operaciones seguras.
7. ¿Retry siempre mejora resiliencia? No. Debe restringirse a fallos transitorios, con backoff/jitter, límites e idempotencia para evitar tormentas.
8. ¿Docker Compose es orquestación de producción? No. Es composición local; Kubernetes/ECS son runtimes de orquestación administrada para despliegue.
9. ¿El Dockerfile es Distroless? No. Usa Alpine JRE; su defensa es multi-stage, usuario no root y menor runtime, no Distroless.
10. ¿Dónde están las métricas de impacto? No se encontraron métricas productivas verificables. Hablar de propiedades del diseño y validaciones, no de porcentajes inventados.

## Riesgos durante la presentación

- **Sobreafirmar caché:** el evaluador puede pedir autoría y tres estrategias.
- **Confundir concurrencia con paralelismo:** `flatMap` concurrente no implica ejecución paralela de CPU; depende del scheduler y del trabajo.
- **Decir “todo es no bloqueante”:** Cognito usa cliente síncrono aislado; es una compatibilidad controlada.
- **Llamar transacción al lote completo:** las llamadas externas y registros parciales no forman una transacción ACID distribuida.
- **Atribuirse la librería tenant:** fue escrita por otro autor; Andrés implementó integraciones y configuración de consumo.
- **Usar el diagrama de Step Functions como única prueba:** acompañarlo siempre con ASL, commits y validación SAM.
- **Afirmar nivel 5 como conclusión:** mostrar evidencia, límites y criterio; permitir que el evaluador valore el nivel.

## Qué debo estudiar antes del jueves

### P0 — lunes 21

- Spring IoC: `BeanFactory` vs `ApplicationContext`, scopes y lifecycle.
- `@Bean` vs `@Component`; `@Qualifier` vs `@Primary` con `R2dbcConfig`.
- Concurrencia vs paralelismo; event loop; `subscribeOn` vs `publishOn`.
- Explicar de memoria el flujo de recarga masiva y sus límites.

### P0 — martes 22

- Docker layers, multi-stage, usuario no root, volúmenes, redes y healthchecks.
- Compose vs Kubernetes vs ECS.
- Redis: cache-aside, read-through, write-through, write-behind, TTL, invalidación, stampede y estructuras.
- Preparar respuesta honesta: caché todavía en consolidación.

### P1 — miércoles 23

- Step Functions: paths, Retry/Catch, task token, idempotencia y locks.
- JWT/OIDC/Cognito: firma, JWKS, claims, expiración, refresh y diferencia decodificar/validar.
- R2DBC: transacciones reactivas, pool y contexto tenant.
- Dos ensayos completos de 20 minutos y ronda de preguntas difíciles.

### P0 — jueves 24

- Repasar cheat sheet, rutas de evidencia y cinco mensajes clave.
- Ensayar respuestas de 60–90 segundos.
- No estudiar temas nuevos; revisar riesgos y límites.

## Qué no debería afirmar

- “Cerré completamente la brecha de caché”.
- “Implementé la librería JWT multitenant”.
- “La aplicación es 100 % no bloqueante”.
- “Tenemos exactly-once”.
- “El flujo masivo procesa N registros por segundo” sin medición.
- “Usamos Distroless”.
- “Diseñé todos los manifiestos Kubernetes”.
- “Implementé todos los protocolos que aparecen en el diagrama SmartPay”.
- “La Step Function de la imagen es exactamente la misma que `state-machine-novedades`”; la imagen corresponde al flujo de liquidación y sirve como material complementario.

## Mejores casos técnicos para defender

1. **Recarga masiva reactiva:** cierra concurrencia y demuestra WebFlux, S3 async, lotes, errores parciales, arquitectura y testing.
2. **Auth Cognito + R2DBC multitenant:** cierra Spring/Beans y permite hablar de seguridad, `boundedElastic`, JWT y contexto Reactor.
3. **Franquicias de local a AWS:** cierra Docker/Compose y refuerza Clean Architecture, resiliencia e IaC.
4. **Novedades orquestadas:** profundiza Cloud con Step Functions, Lambda Python, idempotencia, retry/catch y decisiones de negocio.
5. **Caché como brecha consciente:** demostrar criterio reconociendo evidencia insuficiente y proponiendo un cierre verificable.

## Validaciones realizadas

| Validación | Resultado |
|---|---|
| `gift-card-back-bulk-authorizations ./gradlew :usecase:clean :usecase:test` | 162 pruebas, 0 fallos |
| `lambda-novedad-datos-bancarios pytest -q` | 39 pruebas, 0 fallos |
| `Franquicias ./gradlew test` | 71 pruebas, 0 fallos |
| `docker compose config --quiet` | Válido |
| `terraform fmt -check` | Válido |
| `terraform validate` | Configuración válida |
| `sam validate --lint` (state-machine-novedades) | Plantilla válida |
| Validación estructural ASL | 52 estados, 95 referencias, 0 destinos faltantes |
