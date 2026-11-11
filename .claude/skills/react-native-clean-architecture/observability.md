# Observabilidad y mantenimiento

Sin proveedor impuesto (Sentry, Datadog, Bugsnag, Firebase Crashlytics, etc.). Usá el que el proyecto ya tenga; si no hay, proponé opciones al usuario antes de instalar.

## Qué cubrir

| Necesidad | Qué buscar en el proveedor |
|---|---|
| Errores JS | Captura automática + manual con contexto |
| Crashes nativos | SDK con soporte iOS/Android nativo (no solo JS) |
| Símbolos | Subida de source maps (Hermes) y dSYM/mapping de ProGuard/R8 en cada build/update |
| Releases | Asociar eventos a versión, build y update id OTA |
| Rendimiento | Arranque, navegación, requests lentos |
| Breadcrumbs | Navegación y acciones, sin datos personales |

## Logger

Un único logger en `src/shared/observability/logger.ts`, que en desarrollo usa `console` y en producción envía al proveedor:

```ts
// src/shared/observability/logger.ts
type LogContext = Record<string, string | number | boolean | null | undefined>;

type Reporter = {
  breadcrumb: (message: string, context?: LogContext) => void;
  captureError: (error: unknown, context?: LogContext) => void;
};

let reporter: Reporter | null = null;

/** Se llama una vez al iniciar la app con el SDK elegido. */
export function setErrorReporter(next: Reporter) {
  reporter = next;
}

export const logger = {
  info(message: string, context?: LogContext) {
    if (__DEV__) console.info(message, context);
    reporter?.breadcrumb(message, context);
  },
  warn(message: string, context?: LogContext) {
    if (__DEV__) console.warn(message, context);
    reporter?.breadcrumb(message, context);
  },
  error(error: unknown, context?: LogContext) {
    if (__DEV__) console.error(error, context);
    reporter?.captureError(error, context);
  },
};
```

- El contexto es un objeto plano con valores primitivos: obliga a elegir campos y evita volcar objetos completos (respuestas, usuarios, ubicaciones).
- Eventos con nombre estable (`location_task_error`, `login_failed`) para poder agregarlos.

## Datos sensibles

- Nunca enviar tokens, contraseñas, emails, teléfonos, coordenadas, contenido de formularios ni bodies HTTP.
- Configurar scrubbing en el SDK (`beforeSend`/`beforeBreadcrumb`) para headers `Authorization`, query params y campos conocidos.
- Identificar usuarios con un id interno opaco, no con email; respetar consentimiento si la regulación lo exige.
- Revisar qué captura automáticamente el SDK (requests de red, capturas de pantalla, session replay) y desactivar lo que exponga datos.

## Error boundaries

- Expo Router permite exportar `ErrorBoundary` desde un archivo de ruta/layout: úsalo en el layout raíz y en áreas críticas para mostrar una pantalla de recuperación ("Algo salió mal" + reintentar) y reportar.
- Errores esperados (red, permisos, validación) se manejan como estados, no llegan al boundary.

## Diagnóstico en producción

1. Agrupar por versión/build/update para detectar regresiones.
2. Correlacionar con release notes y updates OTA (posible rollback del update).
3. Revisar diferencias por plataforma, versión de SO y fabricante (Android).
4. Reproducir en build de release; agregar un test de regresión.

## Mantenimiento

- Actualizar el SDK de Expo de una versión mayor a la siguiente siguiendo su guía (`npx expo install expo@^X` + `npx expo install --fix`), en un PR dedicado, sin mezclar con funcionalidades.
- Revisar changelog de cada SDK: deprecaciones, cambios de comportamiento (permisos, notificaciones, edge-to-edge).
- Dependencias nativas: verificar compatibilidad antes de actualizar el SDK.
- Monitorear la tasa de crashes y ANRs (Play Console, App Store Connect / Xcode Organizer).
