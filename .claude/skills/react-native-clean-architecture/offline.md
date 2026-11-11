# Offline y conectividad

## ¿Hace falta?

| Nivel | Cuándo | Qué implica |
|---|---|---|
| 0. Online con buena degradación | Por defecto en toda app | Detectar sin red, mostrar mensajes claros, reintentar, no perder formularios |
| 1. Lectura offline | Usuario consulta datos en campo/sin señal | Cache persistida de queries, indicador de antigüedad |
| 2. Escritura offline | Usuario crea/edita sin red (inspecciones, entregas, tracking) | Cola persistente de operaciones, sincronización, conflictos |
| 3. Offline-first | La base local es la fuente de verdad (sync bidireccional) | SQLite + motor de sync; decisión arquitectónica mayor |

Empezá en el nivel 0. Subí solo con un requisito real. No construyas el nivel 3 para una app online sencilla.

## Nivel 0 (siempre)

- `@react-native-community/netinfo` conectado a `onlineManager` de TanStack Query: las queries se pausan sin red y se reanudan.
- Banner "Sin conexión" no bloqueante cuando `isInternetReachable === false` (tratar `null` como desconocido, no como offline).
- Errores `NETWORK` con acción de reintentar ([data-fetching.md](data-fetching.md#errores)).
- Formularios conservan valores si el envío falla.
- Mostrar datos cacheados en memoria antes que un error de pantalla completa.

## Nivel 1: cache persistida

- `persistQueryClient` (o `PersistQueryClientProvider`) con AsyncStorage/MMKV, `maxAge` acotado y `buster` ligado a la versión de la app.
- Persistir solo queries útiles offline (`shouldDehydrateQuery`), nunca datos sensibles sin evaluar.
- Indicar "Actualizado hace X".

## Nivel 2: operaciones pendientes

```text
domains/<dominio>/<feature>/
├── services/pendingOperations.ts   # cola persistente (SQLite o AsyncStorage para colas chicas)
└── services/syncPendingOperations.ts
```

Reglas:

- Cada operación con id generado en el cliente (UUID) enviado como `Idempotency-Key` o id del recurso: los reintentos no duplican.
- Persistir **antes** de intentar enviar. Marcar enviada solo tras respuesta exitosa.
- Disparadores de sync: reconexión (NetInfo), app vuelve a `active`, inicio de la app, tarea en background si existe.
- Backoff exponencial con límite; errores 4xx permanentes salen de la cola y se reportan al usuario.
- Orden: respetar dependencias (crear antes de editar).
- UI: mostrar estado por ítem (pendiente, sincronizado, falló) y permitir reintentar/descartar.
- TanStack Query: `useMutation` con `networkMode` y mutaciones pausadas puede cubrir casos simples; verificá su soporte de persistencia de mutaciones antes de construir una cola propia.

## Conflictos

Definí la política con el backend antes de implementar:

- **Last-write-wins** con timestamps del servidor: simple, puede perder ediciones.
- **Versionado optimista** (`version`/`ETag`): el servidor rechaza con 409 y la app pide resolver.
- **Merge por campo** solo si el dominio lo permite.

Nunca descartar silenciosamente una edición del usuario.

## Evitar pérdida de datos

- Escribir a persistencia en transacciones.
- No borrar datos locales hasta confirmar el envío.
- Al logout con operaciones pendientes: advertir al usuario.
- Migraciones de la cola versionadas.

## Testing

- Unit: cola (enqueue, retry, backoff, idempotencia) con reloj falso.
- Integración: mock de NetInfo cambiando online/offline.
- Manual: modo avión, red intermitente (Network Link Conditioner en iOS, emulador con red limitada en Android), app cerrada con pendientes y reabierta.
