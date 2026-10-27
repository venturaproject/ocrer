# Operación y seguridad

## Topología

`compose.dev.yml` levanta el modo de desarrollo, incluido Vite. `compose.yml`
define el despliegue de producción con Nginx y backend; el worker OCR comparte
la imagen/código del backend y consume PostgreSQL. El worker de eventos procesa
inbox y outbox. `compose.observability.yml` añade Prometheus y Grafana.

PostgreSQL es el almacenamiento durable. Los archivos temporales/modelos OCR y
el directorio de media se montan en volúmenes según el Compose y la configuración.
Las migraciones se aplican con Alembic; la inicialización de usuarios/permisos se
realiza mediante `seed.py`.

```bash
cp .env.example .env
# Configurar secretos y credenciales antes de exponer servicios
make build
make up
make migrate
make seed
```

Consulta `README.md` para el quick start completo, endpoints y variables de
entorno.

## Seguridad y límites de confianza

- `SECRET_KEY`, contraseñas, secretos de clientes, tokens y credenciales S3 deben
  proporcionarse por entorno/secret manager; nunca versionarlos.
- Las sesiones de navegador usan cookie JWT y protección CSRF. Los clientes
  externos usan Bearer tokens con scopes, límites de frecuencia y cuotas.
- Los callbacks salientes requieren validación anti-SSRF y política de hosts/IP;
  se firman y reintentan. Mantener allowlists explícitas en producción.
- Las integraciones LLM son servicios externos: redactar PII según política y
  evitar enviar datos si no se ha habilitado expresamente la extracción.
- Limitar tamaño del upload, megapíxeles y páginas PDF. OCR/PDF es trabajo
  intensivo y los workers deben dimensionarse en función de la carga.
- Deshabilitar Swagger/OpenAPI en producción si la política del despliegue así lo
  requiere (`DOCS_ENABLED=false`). TLS y terminación segura corresponden al
  proxy/ingress del entorno.

## Observabilidad

La API expone métricas Prometheus, logs configurables (incluido JSON) y request
IDs para correlación. Grafana puede cargar un dashboard provisionado. Proteger
`/metrics` con token si se expone fuera de una red de monitorización confiable.

## Despliegues y verificaciones

- Revisar `docker compose config` antes de aplicar cambios en Compose.
- Aplicar migraciones antes de arrancar versiones que dependan de columnas nuevas.
- Confirmar health checks de PostgreSQL/API y que ambos workers procesen sus
  colas; vigilar reintentos y mensajes en estado `failed`.
- Asegurar persistencia y backup tanto de PostgreSQL como de los volúmenes de
  modelos/media necesarios para el despliegue.
- Probar cambios de versión Python y librerías nativas en la misma arquitectura
  que producción. PaddlePaddle requiere atención especial por compatibilidad de
  wheels y tamaño de imagen.
