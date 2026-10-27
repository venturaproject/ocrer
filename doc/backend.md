# Backend

## Organización

El backend vive en `backend/` y se ejecuta como aplicación FastAPI asíncrona.
`app/main.py` es el composition root práctico: configura middleware, carga los
routers y fuerza el registro de handlers y suscriptores del dominio.

```text
backend/app/
├── routers/        Adaptadores HTTP: rutas, dependencias y serialización
├── domain/         Capacidades y handlers CQRS (adopción selectiva)
├── cqrs/           Messages, registros y buses de commands/queries
├── services/       Flujos de aplicación, OCR, auth e integraciones
├── repositories/   Acceso SQLAlchemy agrupado por recurso
├── models/         Modelos ORM
├── schemas/        Contratos Pydantic de entrada/salida
├── events/         Eventos, inbox/outbox, suscriptores y workers
├── middleware/     CSRF, media privada y middleware HTTP
└── observability/  Logs, request ID y métricas
```

Los nombres de carpetas indican intención, no una garantía de dominio puro. En
algunas rutas el router llama a un servicio/repositorio directamente; otras usan
command/query bus. La guía de [arquitectura](architecture.md) detalla estos
límites y sus excepciones.

## Commands y queries

El patrón CQRS es optativo por caso de uso. Los tipos de mensaje viven junto a la
capacidad y sus handlers se registran con decoradores de `app.cqrs`:

```python
@command_handler(DeleteDocument)
async def handle_delete(session: AsyncSession, cmd: DeleteDocument) -> bool:
    document = await repo.get_document(session, cmd.id)
    if document is None:
        return False
    await repo.delete_document(session, document)
    return True
```

Los routers pueden despachar commands y consultas mediante `command_bus` y
`query_bus`. No crear un command sólo para envolver una llamada trivial; usarlo
cuando el caso de uso se beneficia de un handler explícito, transacción/eventos o
aislamiento del transporte.

## Eventos y consistencia

`app/events/` implementa un outbox transaccional, un inbox idempotente y un bus
local de subscribers. El worker procesa filas pendientes con locking de
PostgreSQL, reintentos con backoff y estado de fallo para intervención/reentrega.
Ver [README de eventos](../backend/app/events/README.md) para añadir mensajes.

Esto evita publicar un evento antes de confirmar la transacción local, pero no
proporciona exactamente-una-vez en sistemas externos. Los consumers deben ser
idempotentes; una entrega puede repetirse si falla después del efecto y antes de
registrar el éxito.

## Persistencia

- SQLAlchemy async con PostgreSQL, sesión inyectada con `get_db`.
- `repositories/` contiene funciones de consulta/mutación reutilizadas por los
  handlers y algunos routers.
- `models/` son entidades ORM; `schemas/` son DTOs/contratos API y no deben
  confundirse con entidades de dominio.
- Alembic mantiene migraciones en `backend/migrations/`.
- La sesión del request confirma al finalizar cuando no hay excepción; los
  workers controlan transacciones explícitamente con `session.begin()`.

## OCR como capacidad intercambiable

`services/ocr/` coordina extracción de texto PDF, motores, formatos, clasificación,
extracción estructurada, redacción, almacenamiento, callbacks y caché. La
selección del motor se hace con `OCR_ENGINE` (`paddle`, `tesseract` o `fake`).
Aunque existe una frontera de motor, Paddle sigue siendo una dependencia pesada
instalada en la imagen habitual; no todos los detalles de terceros están aislados
por puertos formales.

Los flujos síncronos se atienden en la API. Los trabajos durables se escriben en
`ocr_jobs` y el proceso `app.ocr.worker` los reclama/ejecuta; el resultado se
consulta por API y puede activar callback firmado.

## Cómo añadir un cambio

1. Identificar la capacidad y los contratos de API involucrados.
2. Poner validación HTTP y traducción de datos en router/schema; no incrustar ahí
   reglas de negocio complejas.
3. Mantener SQL en el repositorio cuando exista un repositorio correspondiente.
   Para un caso de uso con varias escrituras/eventos, preferir un handler y
   command bus coherentes con los módulos existentes.
4. Añadir migración Alembic si cambia el esquema; no modificar el esquema sólo en
   runtime.
5. Añadir pruebas de comportamiento y, si corresponde, de rollback, duplicados,
   reintentos y permisos.
6. Actualizar esta documentación y la guía del evento/capacidad afectada.

Los tests están en `backend/tests/`; normalmente requieren PostgreSQL de prueba.
