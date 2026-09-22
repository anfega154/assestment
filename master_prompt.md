# OBJETIVO

Quiero que reconstruyas completamente mi presentación de Review de Assessment.

La presentación actual NO cumple el objetivo.

No quiero una presentación comercial, ejecutiva ni centrada en storytelling genérico.

La persona que me evaluará tendrá un perfil técnico, probablemente:

* Tech Lead
* Líder técnico
* Arquitecto
* Desarrollador Senior

Por lo tanto, la presentación debe hablarle a alguien que puede cuestionarme técnicamente y que espera ver:

* código;
* arquitectura;
* diagramas;
* decisiones;
* implementación;
* problemas reales;
* trade-offs;
* conceptos técnicos;
* evidencia concreta.

El objetivo principal es demostrar que las brechas encontradas en mi assessment anterior fueron trabajadas y cerradas mediante implementación real.

---

# PRINCIPIO CENTRAL

La presentación debe seguir esta fórmula:

```text
BRECHA ANTERIOR
      ↓
QUÉ IMPLEMENTÉ
      ↓
DÓNDE LO IMPLEMENTÉ
      ↓
CÓMO FUNCIONA
      ↓
FRAGMENTO DE CÓDIGO / ARQUITECTURA
      ↓
DECISIÓN TÉCNICA
      ↓
TRADE-OFF
      ↓
QUÉ DEMUESTRA
```

NO quiero:

```text
Brecha
↓
mucho texto
↓
conclusión abstracta
```

---

# FUENTES

Analiza:

```text
/Users/andresganan/Desktop/Assestment/last_assestment.md
```

Proyectos:

```text
/Users/andresganan/Desktop/Kire
/Users/andresganan/Desktop/Java/ASULADO
/Users/andresganan/Desktop/Python
/Users/andresganan/Desktop/Retos/Franquicias
```

Material de apoyo:

```text
/Users/andresganan/Desktop/Assestment/programacion_reactiva_webflux.excalidraw
/Users/andresganan/Desktop/Assestment/smartpay-arquitectura.html
/Users/andresganan/Desktop/Assestment/stepfunctions_graph.png
```

Presentación actual a reemplazar/mejorar:

```text
/Users/andresganan/Desktop/Assestment/review_assessment.html
```

---

# REGLA IMPORTANTE SOBRE AUTORÍA

NO utilices como criterio para descartar evidencia:

* quién aparece como autor de un commit;
* quién creó originalmente un archivo;
* si la implementación fue compartida;
* si Git muestra autoría parcial;
* si otra persona también trabajó en el componente.

El objetivo del assessment es demostrar mi conocimiento, experiencia y dominio técnico.

No conviertas la presentación en una auditoría de Git.

No incluyas expresiones como:

```text
Autoría verificable
Autoría compartida
Commits directos
No se puede demostrar autoría
Evidencia insuficiente por commits
```

Si una implementación forma parte de un proyecto donde participé y puedo explicar:

* arquitectura;
* flujo;
* código;
* decisiones;
* problemas;
* solución;
* trade-offs;

puede ser usada como evidencia técnica.

Los commits pueden usarse internamente para localizar cambios, pero NO deben convertirse en el centro de la presentación.

---

# NO MOSTRAR FECHAS DE CORTE DE EVIDENCIA

Eliminar conceptos como:

```text
Evidencia al 21/09/2026
Evidencia hasta la fecha
Validado al...
Estado de evidencia...
```

La presentación no es una auditoría.

Es una defensa técnica.

---

# NO PRESENTAR BRECHAS COMO ABIERTAS

La finalidad de esta presentación es demostrar cómo trabajé las brechas identificadas.

No quiero slides diciendo:

```text
Brecha abierta
Brecha pendiente
Evidencia insuficiente
No demostrado
Pendiente de fortalecer
```

Si una competencia necesita mejor evidencia:

1. busca primero profundamente en todos los proyectos;
2. encuentra código relacionado;
3. encuentra configuración;
4. encuentra pruebas;
5. encuentra arquitectura;
6. encuentra aplicaciones indirectas del concepto;
7. constrúyeme una explicación técnica defendible.

Solo si absolutamente no existe nada relacionado, NO hagas una diapositiva negativa.

En ese caso simplemente no la conviertas en protagonista.

---

# TONO

Quiero una presentación:

* técnica;
* directa;
* concreta;
* sin relleno;
* sin frases comerciales;
* sin frases motivacionales;
* sin autoelogios;
* sin lenguaje corporativo innecesario.

NO quiero frases como:

> Mi evolución no consiste en saber más nombres.

> Senioridad también es saber qué no afirmar.

> De conocimiento técnico a criterio y dominio.

> Mi viaje de crecimiento.

> Mi transformación profesional.

Quiero frases como:

> Procesamiento de archivos sin cargar 10.000 registros en memoria.

> Control de concurrencia con Reactor.

> Propagación del tenant mediante Reactor Context.

> Orquestación de procesos con Step Functions.

> Fan-out con SNS/EventBridge/SQS para desacoplar aplicaciones.

---

# ESTRUCTURA GENERAL

La presentación debe durar aproximadamente:

```text
20–30 minutos
```

Debe contener aproximadamente:

```text
12–16 slides
```

No quiero 30 slides superficiales.

Prefiero pocas slides técnicamente densas pero fáciles de seguir.

---

# SLIDE 1 — PORTADA

Algo muy simple:

```text
Review de Assessment

Andrés Gañan

Evidencias de implementación técnica
```

Nada más.

---

# SLIDE 2 — ASSESSMENT ANTERIOR

Mostrar solamente:

* competencias;
* calificación anterior;
* qué debía profundizar.

Utiliza una tabla compacta.

Ejemplo:

| Competencia  | Nivel anterior | Foco                         |
| ------------ | -------------: | ---------------------------- |
| Frameworks   |              3 | Spring, DI, seguridad        |
| Concurrencia |              3 | async, parallelism, reactive |
| Protocolos   |              4 | ampliar implementación       |
| Cloud        |              X | arquitectura distribuida     |
| Containers   |              3 | Docker / Kubernetes          |
| Cache        |              3 | Redis y estrategias          |

Usa exclusivamente las competencias reales del assessment.

NO dedicar más de 1 slide.

---

# DESDE LA SLIDE 3: SOLO EVIDENCIA

A partir de aquí cada slide debe demostrar algo implementado.

---

# FORMATO OBLIGATORIO PARA CADA CASO

Cada caso técnico debe tener:

## 1. Problema

Una frase.

Ejemplo:

```text
Procesar archivos de miles de registros sin mantener el archivo completo en memoria ni saturar servicios externos.
```

## 2. Solución

Una frase.

```text
Streaming desde S3 + Reactor + procesamiento en lotes + control explícito de concurrencia.
```

## 3. Diagrama

Ejemplo:

```text
S3
 │
 ▼
S3AsyncClient
 │ ByteBuffer
 ▼
Flux<DataBuffer>
 │
 ▼
Parser CSV incremental
 │
 ▼
buffer(100)
 │
 ▼
concatMap
 │
 ├── Validación
 ├── Persistencia R2DBC
 └── API externa
 │
 ▼
Estado de ejecución
```

## 4. Código real

Entre 5 y 15 líneas.

## 5. Decisión técnica

Máximo 3 bullets.

## 6. Qué competencia demuestra

Una línea.

---

# CASO OBLIGATORIO 1 — WEBFLUX / REACTOR

Busca la mejor implementación real dentro de Kire.

Quiero mostrar código real parecido a:

```java
return archivoPort.leer(llave)
    .buffer(TAMANO_LOTE)
    .concatMap(registros -> procesarLote(lote, registros))
    .then(Mono.defer(() -> resumir(lote)))
    .onErrorResume(error -> finalizarParcial(error));
```

Pero usa el código real encontrado.

Quiero poder explicar:

* por qué Flux;
* por qué Mono;
* cómo funciona el pipeline;
* qué operador controla qué;
* qué es lazy execution;
* cuándo ocurre subscribe;
* qué es backpressure;
* diferencia entre map y flatMap;
* diferencia entre flatMap y concatMap;
* cómo controlo concurrencia;
* cómo evito saturar downstream;
* qué ocurre si un registro falla;
* cómo continúa el flujo;
* dónde existe bloqueo potencial;
* cómo aislarlo.

---

# DIAGRAMA REACTOR

Incluye un diagrama técnico como:

```text
Publisher
   │
   ▼
Flux<Registro>
   │
   ├── buffer(100)
   │
   ▼
Flux<List<Registro>>
   │
   ├── concatMap()
   │
   ▼
Mono<Resultado>
   │
   ├── onErrorResume()
   │
   ▼
Completion
```

Y otro pequeño diagrama:

```text
flatMap
A ────────┐
B ──┐     ├── resultados posiblemente desordenados
C ─────┐  │
       ▼  ▼

concatMap
A ─────► B ─────► C
orden garantizado
```

---

# CASO OBLIGATORIO 2 — SPRING BOOT

No quiero una slide diciendo simplemente:

```text
Uso Spring Boot
```

Quiero mostrar un problema real del proyecto.

Ejemplo:

```java
@Bean
@Primary
R2dbcEntityTemplate writeTemplate(
    @Qualifier("writeConnectionFactory") ConnectionFactory cf) {

    return new R2dbcEntityTemplate(cf);
}
```

Explicar:

```text
¿Por qué existe @Qualifier?
```

Porque hay múltiples beans compatibles.

```text
¿Por qué @Primary?
```

Para resolver la selección por defecto.

```text
¿Qué está haciendo Spring?
```

Construcción y resolución del grafo de dependencias desde el ApplicationContext.

Mostrar visualmente:

```text
ApplicationContext

ConnectionFactory READ ─────┐
                            ├── R2dbcEntityTemplate
ConnectionFactory WRITE ────┘
               ▲
          @Qualifier
```

---

# CASO OBLIGATORIO 3 — REACTOR CONTEXT + MULTITENANCY

Si existe en Kire, mostrarlo.

Quiero explicar:

```text
HTTP Request
    │
    ▼
JWT
    │
    ├── tenant_id
    └── tenant_profile
    │
    ▼
Reactor Context
    │
    ▼
ConnectionFactory
    │
    ▼
SET search_path
    │
    ▼
schema tenant
```

Explicar por qué:

```text
ThreadLocal
```

NO es una solución segura en un pipeline reactivo.

Explicar:

```text
Reactor Context
```

como contexto asociado a la suscripción y no al thread.

Mostrar código real.

---

# CASO OBLIGATORIO 4 — AWS STEP FUNCTIONS

Usa:

```text
stepfunctions_graph.png
```

Pero no solo pongas la imagen.

Selecciona una sección relevante del flujo y redibújala de manera legible.

Ejemplo:

```text
Request
   │
   ▼
Lambda
   │
   ▼
Choice
 ┌─┴───────────┐
 │             │
 ▼             ▼
Task A        Task B
 │             │
 └──────┬──────┘
        ▼
       SQS
        │
        ▼
Callback
```

Mostrar:

* Task;
* Choice;
* Retry;
* Catch;
* callback;
* manejo de errores;
* estado observable.

Quiero poder explicar:

> ¿Por qué Step Functions y no una Lambda gigante?

Respuesta basada en:

* separación de estados;
* retry declarativo;
* observabilidad;
* manejo explícito de errores;
* desacoplamiento;
* reanudación;
* seguimiento.

---

# CASO OBLIGATORIO 5 — FAN-OUT AWS EN ASULADO

Esto debe agregarse explícitamente.

Busca dentro de:

```text
/Users/andresganan/Desktop/Java/ASULADO
```

evidencia de arquitectura basada en:

* SNS Topics;
* EventBridge;
* SQS;
* productores;
* consumidores;
* eventos;
* colas;
* DLQ si existe;
* retries;
* filtros;
* integración entre aplicaciones.

La arquitectura que quiero demostrar conceptualmente es un patrón:

```text
                         ┌───────────────► SQS App A ───► Consumer A
                         │
Producer ──► SNS Topic ──┼───────────────► SQS App B ───► Consumer B
                         │
                         └───────────────► SQS App C ───► Consumer C
```

Y/o:

```text
Application
     │
     ▼
EventBridge
     │
     ├── Rule A ──► SQS A ──► Servicio A
     │
     ├── Rule B ──► SQS B ──► Servicio B
     │
     └── Rule C ──► SNS ─────► múltiples consumidores
```

Adapta el diagrama a la implementación REAL encontrada.

---

# EXPLICAR FAN-OUT

Quiero que la presentación explique claramente:

## Problema

Varias aplicaciones necesitan reaccionar al mismo evento sin generar acoplamiento directo entre ellas.

## Solución

Publicar una vez y distribuir a múltiples consumidores independientes.

## Resultado arquitectónico

```text
Antes

Servicio A
 ├── HTTP Servicio B
 ├── HTTP Servicio C
 └── HTTP Servicio D
```

Problemas:

```text
acoplamiento
fallos propagados
latencia
dependencias temporales
```

Después:

```text
Servicio A
    │
    ▼
Evento
    │
    ▼
Topic / Event Bus
    │
 ┌──┼──────────────┐
 ▼  ▼              ▼
Q1  Q2             Q3
│   │              │
▼   ▼              ▼
B   C              D
```

Cada consumidor procesa a su ritmo.

---

# SNS VS EVENTBRIDGE VS SQS

La presentación debe permitirme responder:

## SNS

Pub/Sub.

Ideal para:

```text
1 evento
→ múltiples suscriptores
```

---

## SQS

Queue.

Ideal para:

```text
buffer
desacoplamiento temporal
retry
backpressure natural del consumidor
```

---

## EventBridge

Event Bus.

Ideal para:

```text
routing
reglas
filtrado
integración entre productores y múltiples destinos
```

---

# FAN-OUT CON SNS + SQS

Explicar:

```text
SNS
 │
 ├── SQS servicio A
 ├── SQS servicio B
 └── SQS servicio C
```

¿Por qué una cola por consumidor?

Porque cada consumidor tiene:

* velocidad independiente;
* retries independientes;
* disponibilidad independiente;
* backlog independiente.

Si B falla:

```text
A continúa
C continúa
B conserva mensajes
```

Eso es desacoplamiento real.

---

# DELIVERY SEMANTICS

Si aplica a la implementación, explicar:

```text
at-least-once delivery
```

Por lo tanto:

```text
Consumer debe ser idempotente
```

Mostrar patrón:

```text
messageId
   │
   ▼
¿procesado?
 │      │
sí      no
 │      │
ACK     procesar
        │
        ▼
      persistir ID
        │
        ▼
       ACK
```

---

# DLQ

Si existe una DLQ:

mostrar:

```text
SQS
 │
 ├── intento 1
 ├── intento 2
 ├── intento 3
 │
 ▼
DLQ
```

Explicar:

* maxReceiveCount;
* mensajes poison;
* reproceso;
* observabilidad.

Si no existe en el proyecto, no inventarla.

---

# EVENTBRIDGE

Si ASULADO utiliza EventBridge, mostrar la regla real de routing.

Ejemplo conceptual:

```json
{
  "source": ["asulado.payment"],
  "detail-type": ["PaymentCompleted"]
}
```

Y explicar:

```text
evento
  ↓
matching rule
  ↓
target
```

Usa configuración real si existe.

---

# CASO OBLIGATORIO 6 — CONTAINERS

Quiero código real del Dockerfile.

Por ejemplo:

```dockerfile
FROM gradle:... AS builder

RUN gradle clean build

FROM eclipse-temurin:...

COPY --from=builder ...

USER app

ENTRYPOINT [...]
```

Explicar:

```text
build image
       ↓
artefacto
       ↓
runtime image
```

Y responder:

* por qué multi-stage;
* JDK vs JRE;
* tamaño;
* superficie de ataque;
* usuario no root;
* layers;
* cache;
* variables;
* healthcheck.

---

# KUBERNETES

Si existe evidencia:

mostrar un YAML real pequeño.

Ejemplo:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

o:

```yaml
livenessProbe:
readinessProbe:
```

Explicar técnicamente la diferencia.

---

# CASO CACHE / REDIS

No quiero una diapositiva diciendo que no tengo evidencia.

Busca primero dentro de TODOS los proyectos.

Si existe Redis, mostrar:

```text
Request
   │
   ▼
Redis GET
 │    │
hit   miss
 │    │
 ▼    ▼
return DB
       │
       ▼
     SET TTL
```

Explicar:

```text
Cache Aside
```

Mostrar código real.

También explicar:

* TTL;
* invalidación;
* cache hit;
* cache miss;
* stale data;
* cache stampede;
* Redis distribuido.

Si el código no tiene implementadas todas las estrategias, puedes demostrar conocimiento con la implementación que sí exista y explicar las variantes desde ese caso.

---


# CASO OBLIGATORIO 7 — BASES DE DATOS

Incluir la brecha de Bases de datos con nivel 3 y sus dos focos:

1. Transacciones robustas en RDBMS: TCL, ACID, aislamiento, índices y procedimientos almacenados.
2. CRUD en NoSQL: DynamoDB, llaves simples y compuestas, GSI, LSI, CAP y BASE.

No presentar una diapositiva negativa. Buscar implementación real y partir de ella.

Para RDBMS mostrar evidencia como:

```text
Use case
   │
   ▼
ReactiveTransactionPort
   │
   ▼
TransactionalOperator
   │
   ├── commit si completa
   └── rollback si emite error
```

Explicar ACID con precisión:

* Atomicidad: todas las operaciones de la unidad confirman o se revierten.
* Consistencia: constraints e invariantes mantienen estados válidos.
* Aislamiento: el nivel elegido controla dirty reads, non-repeatable reads y phantoms.
* Durabilidad: después del commit, el motor conserva los cambios.

Mostrar un índice real y explicar que acelera lecturas a cambio de almacenamiento y costo de escritura. Si existe un índice único parcial, explicar que también protege una invariante.

Explicar que la transacción R2DBC cubre la base relacional y no incluye automáticamente SQS, DynamoDB ni APIs externas.

Para DynamoDB mostrar operaciones reales como `PutItem`, `GetItem` y `Query`, además de paginación mediante `LastEvaluatedKey`.

Explicar conceptualmente:

* partition key y sort key;
* llave compuesta;
* GSI: partition/sort key alternativas, creación posterior y lecturas eventualmente consistentes;
* LSI: misma partition key, sort key alternativa, definido al crear la tabla y con lectura consistente opcional;
* Query frente a Scan;
* lecturas eventual y fuertemente consistentes;
* CAP como decisión durante una partición;
* BASE como disponibilidad básica, estado suave y convergencia eventual.

No afirmar que GSI o LSI están implementados si no aparecen en la infraestructura. Presentarlos como criterio de diseño derivado del acceso real por claves.

---

# SEGURIDAD

Si Cognito es una evidencia fuerte, hacer una slide técnica.

```text
Client
  │
  ▼
Cognito
  │
  ├── ID Token
  ├── Access Token
  └── Refresh Token
```

Mostrar claims reales conceptualmente:

```json
{
  "sub": "...",
  "custom:tenant_id": "2",
  "custom:tenant_profile": "schema_standard"
}
```

NO usar secretos ni tokens reales.

Explicar:

* autenticación;
* autorización;
* firma JWT;
* claims;
* expiración;
* refresh;
* JWKS;
* diferencia entre decodificar y validar.

---

# PRUEBAS

No pongas solamente:

```text
162 tests
```

Eso no demuestra profundidad.

Selecciona pruebas importantes.

Ejemplo Reactor:

```java
StepVerifier.create(useCase.execute(...))
    .expectNextMatches(...)
    .verifyComplete();
```

Explicar:

```text
onNext
  ↓
onNext
  ↓
onComplete
```

Y prueba de error:

```text
onNext
  ↓
onError
```

Mostrar qué comportamiento técnico se está validando.

---

# CÓDIGO

Cada slide técnica debe incluir al menos uno de:

* código;
* configuración;
* JSON;
* YAML;
* Terraform;
* diagrama.

Preferiblemente dos.

---

# CÓDIGO REAL

NO inventes snippets.

Obtén snippets de los proyectos.

Puedes reducir código irrelevante con:

```java
...
```

pero conserva la lógica real.

---

# RUTAS

En una pequeña zona inferior puedes indicar:

```text
gift-card-back-bulk-authorizations
EjecucionRecargaMasivaUseCase.java
```

No necesito:

* hashes;
* autores;
* fechas;
* 10 rutas diferentes.

Solo suficiente para ubicar la evidencia.

---

# DIAGRAMAS

Los diagramas deben ser técnicamente útiles.

Evita diagramas decorativos.

Prioriza:

## Reactor

```text
Publisher → operators → Subscriber
```

## Multitenancy

```text
JWT → Reactor Context → Connection → schema
```

## Fan-out

```text
Producer → SNS/EventBridge → SQS × N → Consumers
```

## Step Functions

```text
Task → Choice → Retry/Catch → callback
```

## Arquitectura hexagonal

```text
HTTP Adapter
     │
     ▼
   Port
     │
     ▼
 UseCase
     │
     ▼
 Gateway
     │
     ▼
R2DBC / AWS / API
```

---

# CLEAN ARCHITECTURE

Busca una implementación clara.

Mostrar paquetes reales:

```text
domain/
   model/
   usecase/

infrastructure/
   entrypoints/
   driven-adapters/
```

Diagrama:

```text
        HTTP
         │
         ▼
    Entry Point
         │
         ▼
      UseCase
         │
         ▼
      Gateway
      ▲     ▲
      │     │
   R2DBC    AWS
```

Explicar:

```text
Dominio no depende de infraestructura.
Infraestructura depende de contratos definidos hacia adentro.
```

---

# TRADE-OFFS

Los trade-offs deben ser técnicos y breves.

Ejemplo:

```text
concatMap
+ orden
+ presión controlada
- menor throughput que flatMap
```

```text
SNS + SQS
+ desacoplamiento
+ consumidores independientes
- consistencia eventual
- duplicados posibles
```

```text
Step Functions
+ observabilidad
+ retry declarativo
- costo por transición
- mayor dependencia AWS
```

```text
WebFlux
+ alta concurrencia I/O
- complejidad mental
- pierde beneficios si el pipeline contiene blocking calls
```

---

# FRASES QUE NO QUIERO

Eliminar de la presentación:

```text
Evidencia alta
Evidencia media
Evidencia baja
Autoría verificable
Autoría compartida
Proyecto propio
Brecha abierta
Brecha consciente
No afirmar
Senioridad es...
Mi evolución...
Estado actual...
```

---

# FRASES QUE SÍ QUIERO

```text
Implementación
```

```text
Flujo
```

```text
Decisión
```

```text
Trade-off
```

```text
Código
```

```text
Arquitectura
```

```text
Failure mode
```

```text
Escalabilidad
```

```text
Idempotencia
```

```text
Concurrencia
```

```text
Desacoplamiento
```

---

# ESTRUCTURA PROPUESTA

## 01 — Review de Assessment

Portada.

---

## 02 — Brechas a demostrar

Tabla compacta.

---

## 03 — WebFlux: procesamiento masivo

Código + flujo.

---

## 04 — Reactor: concurrencia y backpressure

Operadores + comparación.

---

## 05 — Spring: DI y configuración

Código + grafo de beans.

---

## 06 — Multitenancy reactivo

JWT → Context → R2DBC → schema.

---

## 07 — Arquitectura limpia

Caso real.

---

## 08 — AWS Fan-Out — ASULADO

SNS/EventBridge/SQS.

Esta slide debe ser especialmente fuerte.

---

## 09 — Fan-out: resiliencia e idempotencia

Retries, DLQ si existe, delivery semantics.

---

## 10 — Step Functions

Arquitectura y código/configuración.

---

## 11 — Containers

Docker + Compose/K8s.

---

## 12 — Cache / Redis

Implementación + estrategia.

---

## 13 — Testing

StepVerifier + pruebas reales.

---

## 14 — Bases de datos: ACID, TCL, CAP y BASE

Transacción R2DBC + ACID + índice real + DynamoDB + claves e índices secundarios.

---

## 15 — Mapa final de competencias

Tabla:

| Competencia  | Evidencia                                  |
| ------------ | ------------------------------------------ |
| Frameworks   | Spring DI + R2DBC + Security               |
| Concurrencia | Reactor + streaming + control de presión   |
| Cloud        | SNS + SQS + EventBridge + Step Functions   |
| Arquitectura | Clean Architecture + ports/adapters        |
| Containers   | Docker + Kubernetes                        |
| Cache        | Redis                                      |
| Bases de datos | R2DBC + ACID + índices + DynamoDB + CAP/BASE |
| Protocolos   | REST / WebSocket / eventos según evidencia |

NO usar:

```text
alta
media
baja
```

Solo evidencia.

---

# SLIDE FINAL

Muy simple:

```text
Las brechas se trabajaron implementando.

WebFlux · Spring · AWS · Arquitectura · Containers · Redis · Testing
```

Debajo:

```text
Preguntas
```

Nada de discursos.

---

# ENTREGABLE 1

Genera:

```text
/Users/andresganan/Desktop/Assestment/review_assessment_v2.html
```

Debe conservar o mejorar:

* navegación con teclado;
* fullscreen;
* progreso;
* responsive;
* modo presentación.

Pero rehacer completamente el contenido.

---

# DISEÑO

Visualmente quiero algo similar a una presentación técnica de ingeniería.

Dark mode.

Código protagonista.

Diagramas protagonistas.

Poco texto.

Cada slide debe poder entenderse visualmente en menos de 10 segundos.

---

# ENTREGABLE 2 — GUION

Genera:

```text
/Users/andresganan/Desktop/Assestment/guion_review_assessment_v2.md
```

Para cada slide:

```text
# Slide X

## Qué mostrar

## Qué decir

## Explicación técnica

## Preguntas que me pueden hacer

## Respuesta

## Pregunta difícil

## Respuesta profunda
```

---

# MUY IMPORTANTE

En las respuestas no quiero frases superficiales.

Ejemplo incorrecto:

> WebFlux permite desarrollar aplicaciones reactivas.

Ejemplo correcto:

> WebFlux utiliza un modelo no bloqueante sobre Reactor. El pipeline trabaja mediante Publisher/Subscriber y normalmente Netty procesa I/O con un número pequeño de event-loop threads. Si ejecuto una llamada bloqueante en esos threads reduzco drásticamente la capacidad de concurrencia, por eso las operaciones bloqueantes deben eliminarse o aislarse.

---

# ENTREGABLE 3 — GUÍA DE ESTUDIO

Genera:

```text
/Users/andresganan/Desktop/Assestment/guia_tecnica_assessment_v2.md
```

Debe estar basada EXCLUSIVAMENTE en las tecnologías que aparecen en la presentación.

No quiero estudiar 100 conceptos.

Quiero dominar profundamente lo que voy a mostrar.

---

# PARA CADA TEMA

Ejemplo:

```text
# WebFlux

Qué es
Cómo funciona
Dónde lo usé
Código
Qué ocurre internamente
Trade-offs
Failure modes
Preguntas típicas
Preguntas Senior
```

---

# PREGUNTAS TÉCNICAS IMPORTANTES

Prepárame especialmente para:

### Reactor

* ¿Qué es Reactive Streams?
* ¿Qué diferencia Publisher y Subscriber?
* ¿Qué es backpressure?
* ¿Cómo se representa demand?
* ¿Qué hace request(n)?
* ¿Mono ejecuta inmediatamente?
* ¿Quién realiza subscribe?
* ¿map vs flatMap?
* ¿flatMap vs concatMap?
* ¿publishOn vs subscribeOn?
* ¿boundedElastic?
* ¿parallel?
* ¿qué ocurre si bloqueo el event loop?

### Spring

* ¿Cómo funciona DI?
* ¿Cómo resuelve Spring múltiples beans?
* ¿Qué diferencia @Primary y @Qualifier?
* ¿Qué es ApplicationContext?
* ¿Qué scopes existen?
* ¿Cómo se crea un bean?
* ¿Qué proxies utiliza Spring?

### AWS messaging

* ¿SNS vs SQS?
* ¿SNS vs EventBridge?
* ¿EventBridge vs SQS?
* ¿qué es fan-out?
* ¿por qué una SQS por consumidor?
* ¿qué ocurre si un consumidor está caído?
* ¿cómo manejar duplicados?
* ¿qué es idempotencia?
* ¿qué es DLQ?
* ¿qué es visibility timeout?
* ¿qué ocurre si no hago delete del mensaje?
* ¿qué es at-least-once?
* ¿FIFO vs Standard?
* ¿cómo conservar orden?
* ¿cómo evitar procesamiento duplicado?

### Step Functions

* ¿Standard vs Express?
* ¿Task?
* ¿Choice?
* ¿Map?
* ¿Parallel?
* ¿Retry?
* ¿Catch?
* ¿ResultPath?
* ¿OutputPath?
* ¿callback task token?

### Docker

* ¿image vs container?
* ¿layers?
* ¿multi-stage?
* ¿ENTRYPOINT vs CMD?
* ¿por qué no root?
* ¿JDK vs JRE?
* ¿qué hace Compose?

### Kubernetes

* Pod;
* Deployment;
* Service;
* ConfigMap;
* Secret;
* requests;
* limits;
* liveness;
* readiness;
* HPA.

### Redis

* cache aside;
* TTL;
* invalidación;
* distributed cache;
* cache stampede;
* eviction policies.

### Bases de datos

* ¿qué significa cada propiedad ACID?;
* ¿cómo funciona una transacción reactiva con TransactionalOperator?;
* ¿cuándo ocurre commit y cuándo rollback?;
* ¿qué recursos quedan fuera de una transacción R2DBC?;
* ¿qué niveles de aislamiento existen y qué anomalías evitan?;
* ¿qué costo tiene un índice?;
* ¿cuándo usar un procedimiento almacenado?;
* ¿partition key vs sort key?;
* ¿GSI vs LSI?;
* ¿Query vs Scan?;
* ¿qué cambia con ConsistentRead?;
* ¿cómo se relacionan CAP y BASE con DynamoDB?;

---

# ARCHIVO ADICIONAL

Genera:

```text
/Users/andresganan/Desktop/Assestment/preguntas_tecnicas_assessment.md
```

Con preguntas tipo entrevista técnica.

Para cada pregunta:

```text
Pregunta

Respuesta corta — 30 segundos

Respuesta profunda — 2 minutos

Ejemplo de mi proyecto

Posible repregunta
```

---

# OBJETIVO FINAL

El evaluador debe terminar la presentación habiendo visto:

```text
Código real
Arquitectura real
Decisiones reales
Problemas reales
Soluciones reales
```

No quiero convencerlo mediante discurso.

Quiero que la implementación hable por mí.

---

# EMPIEZA

Primero:

1. lee `last_assestment.md`;
2. determina exactamente las brechas;
3. revisa profundamente los proyectos;
4. identifica código que permita responder cada brecha;
5. selecciona los mejores snippets;
6. construye diagramas basados en ese código;
7. identifica especialmente la arquitectura fan-out de ASULADO;
8. identifica uso real de SNS, SQS y EventBridge;
9. construye la presentación alrededor de implementación;
10. genera todos los entregables.

No generes la presentación hasta haber identificado qué código real va a aparecer en cada slide.
