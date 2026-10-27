# Frontend

## Estructura

La SPA está en `frontend/` y usa React, TypeScript, Vite, React Router, TanStack
Query y componentes shadcn/ui. Las rutas de administración se conectan a la API
FastAPI; nginx publica la SPA junto con el proxy de API en producción.

- `src/routes/` compone las vistas principales de navegación.
- `src/pages/` agrupa pantallas y sus componentes específicos.
- `src/services/` contiene clientes HTTP y tipos de API por capacidad.
- `src/providers.tsx` configura providers transversales.
- `src/app-routes.tsx` configura el enrutamiento de la aplicación.

La interfaz separa presentación y acceso HTTP, pero no implementa una capa de
dominio frontend formal. Los servicios reflejan contratos de la API y TanStack
Query gestiona fetching, caché y estado remoto.

## Sesión y autorización

La sesión de administración se basa en cookies JWT. Las peticiones mutables
deben respetar el mecanismo CSRF del backend; el backend sigue siendo la
autoridad para autenticar y autorizar cada operación. La visibilidad de una ruta
o botón en el cliente sólo mejora UX y nunca reemplaza RBAC/scopes del servidor.

El cliente de integración externo es distinto de la sesión de navegador: usa
Bearer tokens y scopes desde la API, no las credenciales del panel.

## Cambios recomendados

- Mantener las llamadas API en `src/services/`, evitando `fetch` duplicado en
  componentes.
- Modelar tipos junto al recurso/servicio y mantenerlos alineados con las
  respuestas Pydantic/OpenAPI.
- Usar TanStack Query para datos remotos y estado local de React para estado
  efímero de interfaz.
- No codificar decisiones de permisos únicamente en el frontend.
- Verificar `pnpm lint` / `pnpm build` según scripts de `frontend/package.json`.
