# Geolocalización

Módulo de referencia. Las APIs citadas son de `expo-location` y `expo-task-manager`, verificadas contra la documentación oficial del SDK 58 (noviembre 2026). **Antes de implementar, confirmá nombres y opciones en la documentación de la versión instalada.** En proyectos CLI sin Expo Modules, el mismo diseño aplica con otra librería detrás del adaptador.

## Arquitectura

```text
src/shared/native/location/
├── location.types.ts                   # Contrato propio: LocationFix, LocationPermission, LocationError
├── location.adapter.ts                 # Interface LocationAdapter + instancia por defecto
└── expo-location.adapter.ts            # Implementación con expo-location

src/domains/location/
├── get-current-location/
│   ├── services/getCurrentLocation.ts  # Orquesta permiso → servicios → actual → fallback a última conocida
│   ├── hooks/useCurrentLocation.ts     # Estado discriminado para la UI
│   ├── components/LocationPermissionState.tsx
│   ├── screens/CurrentLocationScreen.tsx
│   └── tests/
└── watch-location/                     # Solo si hay tracking
    ├── hooks/useLocationTracking.ts
    ├── use-cases/startLocationTracking.ts
    ├── tasks/locationTask.ts
    ├── services/locationQueue.ts
    └── services/trackingPreference.ts
```

- El adaptador es una frontera real con el dispositivo: justifica una interface (mock en tests, proveedor alternativo, diferencias por plataforma).
- La lógica de negocio (qué precisión pedir, cuándo usar la última conocida, cómo interpretar un permiso) vive en el dominio y no conoce `expo-location`.
- Los componentes reciben estados ya interpretados.

Implementación completa del adaptador, service, hook, pantalla y tests: [examples.md](examples.md#7-geolocalización-adaptador-nativo).

## Contrato del adaptador

```ts
// src/shared/native/location/location.types.ts
export type LocationAccuracyLevel = 'low' | 'balanced' | 'high' | 'navigation';

export type LocationFix = {
  latitude: number;
  longitude: number;
  accuracyMeters: number | null;
  timestamp: number;
};

export type LocationPermission =
  | { status: 'granted'; precision: 'precise' | 'approximate' }
  | { status: 'undetermined' }
  | { status: 'denied'; canAskAgain: boolean };

export type LocationErrorCode =
  | 'PERMISSION_DENIED'
  | 'SERVICES_DISABLED'
  | 'TIMEOUT'
  | 'UNAVAILABLE'
  | 'UNKNOWN';
```

`LocationError` (clase con `code: LocationErrorCode`) completa este archivo; ver [examples.md](examples.md#7-geolocalización-adaptador-nativo).

El adaptador traduce: enum `Accuracy` de Expo ↔ `LocationAccuracyLevel`, `PermissionResponse` ↔ `LocationPermission`, errores nativos ↔ `LocationErrorCode`. Ningún tipo de `expo-location` cruza la frontera.

## Obtención de ubicación

| Necesidad | API (`expo-location`) | Notas |
|---|---|---|
| Rápida, aproximada, sin esperar GPS | `getLastKnownPositionAsync({ maxAge, requiredAccuracy })` | Devuelve `null` si el cache no cumple. Ideal como primer valor. |
| Posición actual | `getCurrentPositionAsync({ accuracy })` | Puede tardar decenas de segundos con GPS frío o en interiores. No tiene opción de timeout: aplicalo en el adaptador. |
| Actualizaciones en primer plano | `watchPositionAsync(options, callback, errorHandler)` → `LocationSubscription` | Llamar `.remove()` al desmontar o al pasar a background. |
| Servicios del sistema activos | `hasServicesEnabledAsync()` | GPS apagado ≠ permiso denegado. |
| Pedir activar proveedor (Android) | `enableNetworkProviderAsync()` | Solo Android; rechaza si el usuario cancela. |

Precisión (`Accuracy`: `Lowest`, `Low`, `Balanced`, `High`, `Highest`, `BestForNavigation`):

- `Balanced` por defecto: suficiente para "cerca de mí", ciudad, barrio.
- `High` solo si la funcionalidad necesita metros (check-in en un punto, entrega).
- `BestForNavigation` solo durante navegación activa y con el dispositivo cargando idealmente.
- Mayor precisión = más batería y más tiempo hasta el primer fix.

Estrategia recomendada para "mi ubicación actual":

1. Verificar permiso (sin pedir); si `undetermined`, mostrar explicación y pedir con acción del usuario.
2. Verificar servicios activos.
3. Pedir la actual con timeout (p. ej. 15 s).
4. Si falla o expira, usar la última conocida si es reciente (p. ej. ≤ 2 min y ≤ 100 m) e indicarlo en la UI; si no hay, error `TIMEOUT` con reintento.
5. Si la UX necesita mostrar algo antes del fix, la última conocida puede usarse además como valor provisional mientras llega la actual.

### Ubicación simulada en desarrollo

- iOS Simulator: menú Features → Location (Custom Location, City Run...), o un archivo GPX en el esquema de Xcode.
- Android Emulator: Extended Controls → Location (puntos y rutas GPX/KML), o `adb emu geo fix <longitud> <latitud>`.
- Tests: mock del adaptador, nunca del SO.
- No dejar coordenadas fijas en código de producción como fallback silencioso.

## Permisos

API: `getForegroundPermissionsAsync`, `requestForegroundPermissionsAsync`, `getBackgroundPermissionsAsync`, `requestBackgroundPermissionsAsync` (y hooks `useForegroundPermissions`/`useBackgroundPermissions`). Respuesta: `status` (`granted` | `denied` | `undetermined`), `granted`, `canAskAgain`, `ios.scope` (`whenInUse` | `always` | `none`), `ios.accuracy` (`full` | `reduced`), `android.accuracy` (`fine` | `coarse` | `none`).

| Situación | Cómo se detecta | Qué hacer |
|---|---|---|
| Nunca pedido | `status: 'undetermined'` | Pantalla de contexto ("Usamos tu ubicación para...") + botón que dispara el request. Nunca pedir al abrir la app sin contexto. |
| Concedido preciso | `granted` y `ios.accuracy: 'full'` / `android.accuracy: 'fine'` | Continuar. |
| Precisión reducida | `ios.accuracy: 'reduced'` / `android.accuracy: 'coarse'` | Funcionar con precisión aproximada si es posible; si la funcionalidad necesita precisión, explicar y ofrecer ir a Configuración. |
| Denegado, se puede volver a pedir | `denied` + `canAskAgain: true` | Explicar y ofrecer reintentar. |
| Denegado permanente / restringido | `denied` + `canAskAgain: false` (iOS después del primer rechazo, Android tras "no volver a preguntar", o restricciones parentales/MDM) | Ofrecer `Linking.openSettings()`. No volver a llamar request en bucle. |
| Servicios desactivados | `hasServicesEnabledAsync() === false` | Mensaje específico; en Android ofrecer `enableNetworkProviderAsync()`. |
| Cambios desde Configuración | Al volver a `AppState` `active` | Re-chequear permiso con `get...PermissionsAsync` (no request). |

Textos de iOS obligatorios vía config plugin (`app.config.ts`):

```ts
plugins: [
  ['expo-location', {
    locationWhenInUsePermission: 'Usamos tu ubicación para mostrarte comercios cercanos.',
    // Solo si hay tracking en background real:
    // locationAlwaysAndWhenInUsePermission: 'Usamos tu ubicación en segundo plano para registrar el recorrido de tus entregas, incluso con la app cerrada.',
    // isIosBackgroundLocationEnabled: true,
    // isAndroidBackgroundLocationEnabled: true,
    // isAndroidForegroundServiceEnabled: true, // por defecto true si el anterior es true; explicitarlo documenta la intención
  }],
],
```

Los textos deben explicar el beneficio concreto: App Review rechaza descripciones genéricas.

## Segundo plano

**No prometas tracking continuo: el sistema operativo no lo garantiza.** Diseñá para eventos intermitentes, pérdidas y recuperación.

### Requisitos

- Development Build. TaskManager no ejecuta en background en Expo Go (no está disponible en Android en Expo Go).
- Plugin con `isIosBackgroundLocationEnabled: true` (agrega `location` a `UIBackgroundModes`) y `isAndroidBackgroundLocationEnabled: true` (habilita además el foreground service por defecto).
- Permiso foreground concedido **antes** de pedir background.
- Justificación de producto real: App Store y Google Play exigen declarar y justificar ubicación en background (Google Play tiene un formulario específico y puede pedir video).

### Diferencias iOS / Android

| Aspecto | iOS | Android |
|---|---|---|
| Pedir background | Prompt "Always" (puede llegar diferido). Si el usuario eligió "Allow Once", el pedido de background en esa sesión se deniega silenciosamente. | Android 11+: `requestBackgroundPermissionsAsync` lleva a Configuración; explicá antes qué opción elegir ("Permitir todo el tiempo"). |
| Ejecución continua | Indicador azul opcional (`showsBackgroundLocationIndicator`). El sistema puede pausar updates (`pausesUpdatesAutomatically`). | Requiere **foreground service** con notificación persistente (`foregroundService: { notificationTitle, notificationBody }`). Sin él, Android limita fuertemente los updates. |
| App terminada por el usuario | Las location updates estándar se detienen. El sistema puede relanzar la app para eventos de geofencing. | La app no se reinicia para eventos de ubicación o geofencing. |
| Reinicio del dispositivo | No asumir reanudación. | No asumir reanudación. |
| Batería | Low Power Mode reduce frecuencia. | Doze, App Standby y optimizaciones del fabricante (Xiaomi, Samsung, Huawei) pueden matar el servicio. |

### Implementación

1. Definir la tarea **en el scope global** (no dentro de un componente) en un archivo del dominio, e importarlo al inicio de `src/app/_layout.tsx`. Cuando el SO despierta la app en background no hay componentes montados.

```ts
// src/domains/location/watch-location/tasks/locationTask.ts
import * as TaskManager from 'expo-task-manager';
import { locationFixesFromTaskData } from '@/shared/native/location/expo-location.adapter';
import { enqueueLocations } from '../services/locationQueue';
import { logger } from '@/shared/observability/logger';

export const LOCATION_TRACKING_TASK = 'location-tracking';

TaskManager.defineTask(LOCATION_TRACKING_TASK, async ({ data, error }) => {
  if (error) {
    logger.warn('location_task_error', { code: error.code });
    return;
  }
  const fixes = locationFixesFromTaskData(data);
  if (fixes.length > 0) await enqueueLocations(fixes);
});
```

La traducción del payload nativo vive en el adaptador, así ningún tipo de `expo-location` llega al dominio:

```ts
// Agregar a src/shared/native/location/expo-location.adapter.ts
/** expo-location entrega a la tarea { locations: LocationObject[] }. */
export function locationFixesFromTaskData(data: unknown): LocationFix[] {
  const locations = (data as { locations?: Location.LocationObject[] } | null)?.locations;
  return Array.isArray(locations) ? locations.map(toLocationFix) : [];
}
```

```tsx
// src/app/_layout.tsx (entre los primeros imports, antes de cualquier código de componentes)
import '@/domains/location/watch-location/tasks/locationTask';
```

2. Caso de uso de inicio:

```ts
// src/domains/location/watch-location/use-cases/startLocationTracking.ts
import * as Location from 'expo-location';
import { LOCATION_TRACKING_TASK } from '../tasks/locationTask';
import { trackingPreference } from '../services/trackingPreference';

export type StartTrackingResult =
  | { status: 'started' }
  | { status: 'foreground-permission-required' }
  | { status: 'background-permission-denied'; canAskAgain: boolean }
  | { status: 'unavailable' };

export async function startLocationTracking(): Promise<StartTrackingResult> {
  const foreground = await Location.getForegroundPermissionsAsync();
  if (!foreground.granted) return { status: 'foreground-permission-required' };

  if (!(await Location.isBackgroundLocationAvailableAsync())) return { status: 'unavailable' };

  // Consultar antes de pedir: si el usuario ya lo negó de forma permanente, no repetir el pedido.
  let background = await Location.getBackgroundPermissionsAsync();
  if (!background.granted && background.canAskAgain) {
    background = await Location.requestBackgroundPermissionsAsync();
  }
  if (!background.granted) {
    return { status: 'background-permission-denied', canAskAgain: background.canAskAgain };
  }

  if (!(await Location.hasStartedLocationUpdatesAsync(LOCATION_TRACKING_TASK))) {
    await Location.startLocationUpdatesAsync(LOCATION_TRACKING_TASK, {
      accuracy: Location.Accuracy.Balanced,
      distanceInterval: 50,             // metros entre updates
      deferredUpdatesInterval: 60_000,  // agrupar en background
      pausesUpdatesAutomatically: true, // iOS
      activityType: Location.ActivityType.Other,
      showsBackgroundLocationIndicator: true,
      foregroundService: {
        notificationTitle: 'Registrando tu recorrido',
        notificationBody: 'Tocá para abrir la app y detener el seguimiento.',
      },
    });
  }
  await trackingPreference.setEnabled(true);
  return { status: 'started' };
}
```

Este caso de uso llama `expo-location` directamente porque es la única implementación y el background es específico del proveedor. Si el proyecto ya tiene un adaptador de background, usalo. Si aparece un segundo proveedor, extraé `startBackgroundUpdates`/`stopBackgroundUpdates` al adaptador.

3. Detener: `Location.stopLocationUpdatesAsync(LOCATION_TRACKING_TASK)` y `trackingPreference.setEnabled(false)`. Siempre ofrecer al usuario una forma visible de detenerlo.

### Persistencia y recuperación

- La tarea en background puede ejecutarse en un contexto JS recién iniciado: **no dependas de stores en memoria**. Escribí en persistencia (SQLite o AsyncStorage para colas chicas) y subí por lotes cuando haya red.
- Subida idempotente: cada lote con un id generado en el cliente para que el backend deduplique reintentos.
- Reintentos con backoff exponencial y límite; descartar fixes más viejos que la ventana útil de negocio.
- Recuperación al iniciar la app: si `trackingPreference` dice "activo" pero `hasStartedLocationUpdatesAsync` es `false` y el permiso sigue concedido, reiniciar y registrar el evento. Si el permiso fue revocado, marcar como detenido y avisar en la UI.
- Mostrar al usuario el estado real ("Último registro hace 12 min") en lugar de asumir continuidad.

### Geofencing

`startGeofencingAsync(taskName, regions)`, `stopGeofencingAsync`, `hasStartedGeofencingAsync`. Límites: iOS 20 regiones monitoreadas, Android 100 geofences activas. Preferilo a tracking continuo cuando el negocio solo necesita entrar/salir de zonas: consume mucha menos batería.

## Seguridad y privacidad

- Pedir permiso en el momento en que la funcionalidad lo necesita, con explicación previa.
- No pedir background ("Always") sin una funcionalidad visible que lo justifique.
- Mínima precisión y frecuencia suficientes. Redondear coordenadas antes de enviarlas si el backend no necesita metros.
- No persistir coordenadas que no se van a usar; definir retención y borrar al logout.
- **Nunca** loguear coordenadas en consola, analytics, crash reports ni breadcrumbs.
- Ubicación del usuario es dato personal: declararla en App Privacy (App Store) y Data safety (Google Play).
- Respetar la revocación: si el usuario quita el permiso, detener todo y no insistir.

## Testing

- Unit: el service con un `LocationAdapter` falso; casos `granted`, `undetermined`, `denied` (`canAskAgain` true/false), servicios desactivados, timeout con y sin última conocida.
- Componentes: `LocationPermissionState` muestra "Abrir configuración" cuando `canAskAgain` es false.
- Manual en dispositivo real (obligatorio para background): denegar, "Allow Once", precisión reducida, GPS apagado, modo avión, app en background 30+ min, app terminada, reinicio del teléfono, ahorro de batería.
- E2E: Maestro puede controlar permisos al lanzar la app (verificá la sintaxis en su documentación) y simular ubicación en simulador/emulador.

Reportá explícitamente qué escenarios se probaron en dispositivo y cuáles no.
