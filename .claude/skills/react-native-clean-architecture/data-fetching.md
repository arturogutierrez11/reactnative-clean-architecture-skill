# Datos, APIs y validación

## Capas

```text
Componente ─→ Hook (useQuery/useMutation) ─→ Service del dominio ─→ shared/api/httpClient ─→ fetch
                                                   │
                                                   └─ Zod: valida DTO, mapea a modelo
```

- **Nunca** HTTP directo desde componentes o pantallas.
- `shared/api/httpClient.ts`: URL base, headers, timeout, cancelación, token, refresh, normalización de errores. Sin conocimiento de dominios: la sesión se conecta registrando funciones (`configureHttpClient`), así `shared` no importa `domains`.
- Service por operación en la funcionalidad: arma el request, valida la respuesta con el schema, mapea DTO → modelo de la app.
- Si el proyecto ya usa axios u otro cliente con interceptores, mantenelo y aplicá las mismas reglas.

Implementación completa: [examples.md](examples.md#1-fundaciones-compartidas-errores-y-cliente-http) y [examples.md](examples.md#4-integración-rest-perfil).

## Configuración

- URL base desde `process.env.EXPO_PUBLIC_API_URL`, validada al iniciar (`shared/config/env.ts`). Usá acceso estático con punto (`process.env.EXPO_PUBLIC_X`): Expo no inlinea `process.env['X']` ni destructuring.
- Todo `EXPO_PUBLIC_*` queda **en texto plano dentro del bundle**. Nunca claves privadas, secretos de API ni credenciales de servicios. Las integraciones con secretos van por tu backend. Ver [configuration.md](configuration.md).
- HTTPS obligatorio en staging y producción.

## DTOs y modelos

```ts
// DTO tal cual llega (snake_case, nullable, strings para fechas)
const orderDtoSchema = z.object({
  id: z.string(),
  total_cents: z.number().int(),
  created_at: z.iso.datetime(),
  status: z.enum(['pending', 'paid', 'cancelled']),
});

// Modelo de la app
export type Order = { id: string; total: number; createdAt: Date; status: OrderStatus };

export const toOrder = (dto: z.infer<typeof orderDtoSchema>): Order => ({
  id: dto.id,
  total: dto.total_cents / 100,
  createdAt: new Date(dto.created_at),
  status: dto.status,
});
```

- Validar **siempre** datos externos con `parse`/`safeParse`. Nunca `as Order` sobre JSON.
- Si DTO y modelo son idénticos, no crear mapper: usar `z.infer` del schema.
- Enum desconocido del backend: decidir explícitamente (fallar o mapear a `'unknown'`). Para apps que no se actualizan al instante, tolerar valores nuevos suele ser mejor (`z.enum(['pending', 'paid', 'cancelled', 'unknown']).catch('unknown')`: el valor de `catch` debe pertenecer al enum).

## Errores

Normalizar en el cliente HTTP a un `AppError` con código:

| Código | Origen | UI típica |
|---|---|---|
| `NETWORK` | `fetch` rechazado (sin red, DNS, TLS) | "Sin conexión" + reintentar |
| `TIMEOUT` | Abort por timeout | "Tardó demasiado" + reintentar |
| `CANCELLED` | Abort del consumidor | Nada |
| `UNAUTHORIZED` | 401 tras refresh fallido | Sesión expirada (lo maneja la sesión) |
| `FORBIDDEN` | 403 | "No tenés acceso" |
| `NOT_FOUND` | 404 | Estado vacío / "No existe" |
| `VALIDATION` | 400/422 con detalles | Errores por campo |
| `CONFLICT` | 409 | Mensaje de negocio |
| `SERVER` | 5xx | "Problema del servicio" + reintentar |
| `INVALID_RESPONSE` | Falló el schema de respuesta | Error genérico; reportar a observabilidad |
| `UNKNOWN` | Resto | Error genérico |

- La UI traduce códigos a mensajes; nunca muestra `error.message` técnico.
- Errores de negocio con código del backend (`{ code: 'INSUFFICIENT_STOCK' }`) se mapean en el service del dominio a una unión tipada.
- Reportar a observabilidad `SERVER`, `INVALID_RESPONSE`, `UNKNOWN`; no reportar `NETWORK`/`CANCELLED` como crash.

## Cancelación y timeouts

- TanStack Query entrega `signal` a `queryFn`: pasalo al service y al cliente.
- Timeout por defecto (p. ej. 15 s) en el cliente, configurable por request (uploads más largos).
- Implementar con `AbortController` + `setTimeout` encadenado al `signal` externo; no asumir que `AbortSignal.timeout`/`AbortSignal.any` existen en el runtime (Hermes) de la versión instalada sin verificarlo.

## Reintentos

- Lecturas (GET): reintento con backoff vía TanStack Query (`retry: 2`), sin reintentar 4xx.
- Mutaciones no idempotentes (POST de pago, creación): **no** reintentar automáticamente salvo que el backend acepte `Idempotency-Key` y la envíes.
- El cliente HTTP no reintenta por su cuenta salvo el único reintento tras refresh de token.

## Autenticación en requests

- `Authorization: Bearer <access>` agregado por el cliente cuando el request lo requiere.
- 401 → un único refresh compartido (single-flight) → reintentar una vez → si falla, `onUnauthorized()` limpia sesión. Ver [authentication.md](authentication.md).

## Paginación

- `useInfiniteQuery` con cursor (preferido) u offset; `getNextPageParam` devuelve `undefined` al final.
- `FlatList` `onEndReached` + `onEndReachedThreshold`; evitar disparos múltiples chequeando `hasNextPage && !isFetchingNextPage`.
- Pull-to-refresh con `refreshing`/`onRefresh` ligados a `refetch`.

## Uploads y descargas

- El `request()` de los ejemplos serializa JSON: para multipart agregá una función `upload()` en `shared/api` que reutilice auth, timeout y normalización de errores pero envíe `FormData` sin `JSON.stringify`.
- Uploads multipart: `FormData.append('file', { uri, name, type } as unknown as Blob)` — el cast es la frontera conocida de React Native con los tipos de DOM; documentalo en el código. No fijar `Content-Type` manualmente (el boundary lo pone fetch).
- Validar tamaño y tipo antes; comprimir imágenes ([device-permissions.md](device-permissions.md#cámara-y-multimedia)).
- Archivos grandes o que deben continuar en background: `expo-file-system` (API de upload/download de la versión instalada) o URLs prefirmadas del backend.
- Progreso: `fetch` no expone progreso de subida; usar la API de `expo-file-system` o `XMLHttpRequest` si hace falta.

## Offline

Ver [offline.md](offline.md). Mínimo: no mostrar error de pantalla completa si hay datos en cache.

## Errores comunes

- `useEffect(() => { fetch(...) }, [])` en una pantalla.
- `response.json()` sin chequear `response.ok`.
- Reintentar un POST de pago tras timeout.
- `EXPO_PUBLIC_STRIPE_SECRET_KEY`.
- Loguear la respuesta completa con datos personales.
