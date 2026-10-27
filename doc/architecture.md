# Arquitectura del sistema

## Visión general

Ocrer es un monorepo con una API FastAPI asíncrona, una SPA React y servicios
auxiliares para OCR y procesamiento de eventos. Nginx actúa como entrada HTTP;
PostgreSQL conserva el estado de negocio. El motor OCR se selecciona mediante
configuración y puede ser PaddleOCR, Tesseract o un stub de pruebas.

```text
Navegador / cliente de integración
               │ HTTP
               ▼
          Nginx :80
       ┌───────┴─────────┐
       │                 │
       ▼                 ▼
 SPA React           FastAPI
                         │
        ┌────────────────┼──────────────────┐
        ▼                ▼                  ▼
   PostgreSQL       OCR worker        Event worker
                    ├─ PaddleOCR      ├─ Inbox
                    └─ Tesseract      └─ Outbox
```

En desarrollo, Vite sirve la SPA con hot reload; en producción Nginx sirve el
bundle estático y hace proxy de la API. Los workers son procesos separados que
comparten el código y la base de datos del backend.

## Cómo describir los patrones con precisión

El backend combina **módulos organizados por capacidad**, **CQRS** para algunos
casos de uso y una **infraestructura de eventos**. Hay separación parcial entre
routers, servicios, repositorios y modelos, pero no hay una implementación
uniforme de Clean Architecture ni un dominio puro aislado de SQLAlchemy.

| Idea | Uso actual | Límite importante |
| --- | --- | --- |
| DDD | Lenguaje y módulos orientados a capacidades (documentos, trabajadores, etc.); eventos de dominio y casos de uso identificables. | No todos los módulos tienen agregados, value objects o invariantes encapsuladas. Algunos modelos ORM también hacen de modelo de negocio. |
| Clean Architecture | Separación práctica entre transporte (`routers`), lógica (`services` / handlers), acceso a datos (`repositories`) y persistencia. | La dirección de dependencias no está impuesta en todo el sistema; algunos routers llaman directamente a servicios y repositorios. |
| Hexagonal / Ports & Adapters | El motor OCR se selecciona mediante una fachada/configuración; hay repositorios que agrupan consultas SQL. El transporte HTTP no es el único disparador: también hay workers. | Las interfaces/puertos no están definidos de forma sistemática ni todos los adaptadores se pueden intercambiar sin SQLAlchemy. |
| CQRS | `CommandBus` y `QueryBus` separan escrituras y lecturas donde se han adoptado; handlers registrados por tipo de mensaje. | CQRS no cubre todos los endpoints, y lectura/escritura comparten PostgreSQL y modelos en varias áreas. No es una arquitectura de microservicios ni event sourcing. |

Una descripción resumida adecuada es **monolito modular con separación por
capas/vertical slices y adopción selectiva de CQRS y puertos/adaptadores**.

## Límites y flujo de dependencias

- **Entrada**: routers FastAPI validan HTTP, dependencias de autenticación y
  permisos, y traducen request/response a tipos de aplicación.
- **Aplicación**: `services/` contiene flujos de OCR, auth y coordinación;
  `domain/<capacidad>/` aloja commands, queries, handlers y, en algunos casos,
  eventos, subscribers o traductores.
- **Persistencia**: `repositories/` concentra consultas y mutaciones SQLAlchemy
  allí donde se usa; `models/` define el mapeo ORM; Alembic gestiona el esquema.
- **Infraestructura transversal**: `events/`, `cqrs/`, `ratelimit/`, middleware,
  observabilidad y configuración.
- **Composición**: `app/main.py` registra routers y carga `app.domain` para
  registrar handlers, subscribers y traductores. Los workers cargan el mismo
  registro al iniciar.

```text
HTTP router ──► servicio / command bus ──► handler ──► repository ──► SQLAlchemy
     │                    │                  │
     └── auth/schema      └── OCR facade     └── DomainEvent → Outbox
                                                       │
                                              event worker / subscriber
```

No se debe interpretar el diagrama como una regla absoluta: hay endpoints que
usan directamente repositorios y servicios. Al añadir lógica nueva, mantener el
router delgado y elegir el patrón existente más cercano; no introducir capas
vacías sin una necesidad de dominio o de sustitución de infraestructura.

## CQRS, transacciones y mensajería

Los comandos registrados se despachan por `app/cqrs/bus.py`. El bus ejecuta el
handler dentro de un scope de recolección de eventos y añade los eventos
resultantes a `outbox_messages` usando la misma `AsyncSession`. La transacción
real la controla el ciclo de vida de la sesión del request o el worker. Un fallo
propaga el error y permite hacer rollback.

Las queries invocan su handler sin escribir en el outbox. El event worker drena
el outbox y el inbox; los mensajes entrantes se deduplican por origen e ID y se
traducen a comandos. Los reintentos usan backoff y un límite de intentos. Esta
garantía es de entrega/reintento de eventos, **no** de event sourcing: el estado
actual de negocio reside en tablas relacionales.

## Flujo OCR

1. El router valida el archivo, los límites de tamaño y la identidad/scopes del
   cliente externo o del usuario.
2. El servicio OCR inspecciona el tipo de documento. En PDF intenta primero leer
   la capa de texto; páginas escaneadas se rasterizan y pasan al motor elegido.
3. El resultado puede clasificarse, extraerse, redactarse y serializarse como
   JSON, texto, hOCR, ALTO o PDF searchable.
4. El flujo síncrono devuelve el resultado directamente. El asíncrono persiste
   un `OcrJob`; el worker lo reclama, procesa y guarda estado/resultado.
5. Si hay callback configurado, se envía con protección SSRF, firma y política
   de reintentos. El registro de documentos y el uso por cliente se conservan
   para consulta y cuotas.

## Decisiones y restricciones

- Mantener límites de carga y procesamiento: OCR/PDF pueden consumir mucha CPU y
  memoria; evitar procesar archivos ilimitados o materializar entradas enormes.
- Tratar callbacks y proveedores LLM como límites de confianza. El envío a un
  proveedor externo sólo ocurre si la función está configurada y se invoca.
- El repositorio es un monolito desplegable como varios procesos Docker, no una
  colección de microservicios con propiedad independiente de datos.
- SQLAlchemy está presente en handlers/repositorios: aislarlo más requiere una
  evolución gradual, no sólo renombrar carpetas como `domain` o `application`.
- El motor PaddleOCR 2.x puede tener restricciones de wheels/arquitectura; revisar
  el Dockerfile y la matriz de dependencias al cambiar Python o arquitectura.

## Lecturas relacionadas

- [Backend](backend.md)
- [Operación y seguridad](operations.md)
- [README del subsistema de eventos](../backend/app/events/README.md)
