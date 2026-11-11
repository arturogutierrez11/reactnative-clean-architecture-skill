# Permisos y funcionalidades del dispositivo

Guías para integrar progresivamente. **No instales estas dependencias al crear el proyecto**: agregalas con `npx expo install` cuando la funcionalidad llegue, y verificá su documentación para el SDK instalado.

## Modelo común de permisos

Todo permiso parte del mismo modelo en el dominio, sin importar la API (un adaptador puede extender la variante `granted`, p. ej. `precision` en ubicación):

```ts
export type PermissionState =
  | { status: 'granted' }
  | { status: 'undetermined' }
  | { status: 'denied'; canAskAgain: boolean };
```

Flujo obligatorio:

1. **Consultar** (`get...PermissionsAsync`) sin pedir.
2. Si `undetermined`: mostrar contexto propio (qué, por qué, beneficio) y pedir **por acción del usuario**.
3. Si `denied` y `canAskAgain`: explicar y permitir reintentar.
4. Si `denied` y `!canAskAgain`: ofrecer `Linking.openSettings()`; re-consultar al volver a `active`.
5. La funcionalidad degrada con elegancia: la app no se bloquea entera por un permiso opcional.

Cada adaptador de `shared/native/<capacidad>/` mapea la respuesta de la librería a `PermissionState`. Pedí el mínimo necesario y cuando se necesita: App Review y Google Play rechazan permisos sin uso visible o pedidos al inicio sin contexto.

Ubicación: [geolocation.md](geolocation.md).

## Plantilla de decisión por integración

Para cada integración documentá en la entrega: cuándo usarla, librería, compatibilidad (Expo Go / Development Build / New Architecture), permisos, configuración nativa, arquitectura, errores, testing, restricciones reales y efectos en publicación.

---

## Cámara y multimedia

| Ítem | Detalle |
|---|---|
| Cuándo | Captura en la app (`expo-camera`, `CameraView`) solo si necesitás UI propia, escaneo de códigos o control fino. Para "elegir/tomar una foto" alcanza `expo-image-picker` (`launchCameraAsync`, `launchImageLibraryAsync`). |
| Permisos | Cámara; micrófono si graba video con audio. En iOS 14+ y Android 13+ el selector de fotos del sistema permite elegir sin permiso de galería completo: preferilo. |
| Config | Plugins `expo-camera` / `expo-image-picker` con textos (`cameraPermission`, `photosPermission`, `microphonePermission`; verificá los nombres exactos). Quitar `RECORD_AUDIO` con `blockedPermissions` si no grabás audio. |
| Arquitectura | `shared/native/media/` con `pickImage()`, `takePhoto()` que devuelven `{ uri, width, height, mimeType, fileSize }` propios. Compresión/redimensión con `expo-image-manipulator` en un service del dominio antes de subir. |
| Upload | Multipart con `FormData` (`{ uri, name, type }`) o `expo-file-system` para uploads en background. Validar tamaño y tipo antes. Ver [data-fetching.md](data-fetching.md#uploads-y-descargas). |
| Errores | Cancelación del usuario (`canceled: true`) no es error. Archivos HEIC en iOS: convertir si el backend no los acepta. |
| Testing | Mock del adaptador; manual en dispositivo (el simulador iOS no tiene cámara). |
| Publicación | Declarar fotos/videos en App Privacy y Data safety si se suben. |

## Notificaciones

| Ítem | Detalle |
|---|---|
| Cuándo | Locales (`scheduleNotificationAsync`) para recordatorios del dispositivo; push para eventos del servidor. |
| Librería | `expo-notifications` (+ `expo-device` para detectar dispositivo físico). Push con Expo Push Service o FCM/APNs directos (`getDevicePushTokenAsync`). |
| Compatibilidad | Push no disponible en Expo Go en Android (desde SDK 53): requiere Development Build. Push no funciona en simulador iOS de forma confiable para tokens reales; usar dispositivo. |
| Permisos | iOS: prompt explícito. Android 13+: `POST_NOTIFICATIONS`; **crear el canal (`setNotificationChannelAsync`) antes de pedir el token** para que aparezca el prompt. |
| Config | Plugin `expo-notifications` (`icon`, `color`, `defaultChannel`, `sounds`, `enableBackgroundRemoteNotifications`). Credenciales APNs/FCM en EAS, nunca en el repo. `projectId` de EAS requerido para `getExpoPushTokenAsync`. |
| Arquitectura | `domains/notifications/register-device/`: obtener token → enviarlo al backend asociado a la sesión → reenviar si cambia (`addPushTokenListener`). Revocar en logout (endpoint de baja + no mostrar notificaciones de la cuenta anterior). |
| Foreground | `setNotificationHandler({ handleNotification: async () => ({ shouldShowBanner, shouldShowList, shouldPlaySound, shouldSetBadge }) })`. Desde SDK 58 las notificaciones en foreground se muestran por defecto salvo que el handler diga otra cosa. |
| Deep link | Respuesta con `addNotificationResponseReceivedListener` (app abierta) y `useLastNotificationResponse()` (app iniciada desde la notificación; `undefined` = aún no se sabe, `null` = ninguna). Validar `data` antes de navegar. Ver [navigation.md](navigation.md#navegación-desde-notificaciones). |
| Errores | `getExpoPushTokenAsync` hace red: try/catch y reintento posterior. Permiso denegado: la app sigue funcionando. |
| Testing | Unit del registro con mocks; push real en dispositivo con la herramienta de Expo o del proveedor. |
| Publicación | iOS: capability Push. Android: canales con nombres comprensibles para el usuario. |

```ts
// Orden correcto en Android antes de pedir token
if (Platform.OS === 'android') {
  await Notifications.setNotificationChannelAsync('default', {
    name: 'General',
    importance: Notifications.AndroidImportance.DEFAULT,
  });
}
const { status } = await Notifications.requestPermissionsAsync();
```

## Biometría y autenticación local

| Ítem | Detalle |
|---|---|
| Cuándo | Desbloquear una sesión ya existente o confirmar una acción sensible. **No** reemplaza la autenticación del backend. |
| Librería | `expo-local-authentication`: `hasHardwareAsync`, `isEnrolledAsync`, `supportedAuthenticationTypesAsync`, `authenticateAsync({ promptMessage, cancelLabel, disableDeviceFallback, fallbackLabel, biometricsSecurityLevel })`. |
| Compatibilidad | Face ID no funciona en Expo Go en iOS: Development Build. |
| Config | iOS `NSFaceIDUsageDescription` vía plugin (`faceIDPermission`). Sin él, iOS cae al código del dispositivo. |
| Arquitectura | `shared/native/biometrics/` con `authenticate(reason): Promise<{ ok: true } | { ok: false; reason: 'cancelled' | 'unavailable' | 'not-enrolled' | 'lockout' | 'failed' }>`. Para proteger un secreto con biometría: `SecureStore` con `requireAuthentication: true` (las claves pueden invalidarse si cambian las biometrías). |
| Errores | `user_cancel`, `system_cancel`, `app_cancel` → cancelado (no mostrar error). `lockout` → explicar. `not_enrolled`/`passcode_not_set` → ofrecer método alternativo. |
| Testing | Simulador iOS: Features → Face ID → Enrolled/Matching. Emulador Android: huella en Extended Controls. Dispositivo real para validar. |

## Hardware y conectividad

| Integración | Librería sugerida | Notas clave |
|---|---|---|
| Conectividad | `@react-native-community/netinfo` | `isInternetReachable` puede ser `null` al inicio. Conectar con `onlineManager` de TanStack Query. Ver [offline.md](offline.md). |
| Haptics | `expo-haptics` | Solo donde aporta (confirmaciones, selección). Función en `shared/native/haptics.ts` consumida por la primitiva (`Button`); respetar que el usuario puede desactivar vibración del sistema. |
| Compartir | `Share` de React Native (texto/URL) o `expo-sharing` (archivos) | `expo-sharing` no comparte texto suelto; verificá. |
| Archivos locales | `expo-file-system` | Usar directorios de la app (documento/cache). Validar tipo y tamaño. No guardar datos sensibles sin cifrar. |
| Sensores | `expo-sensors` (acelerómetro, giroscopio, podómetro, barómetro) | Fijar `setUpdateInterval` y desuscribir al salir de pantalla o pasar a background. |
| Bluetooth LE | Librería comunitaria (p. ej. `react-native-ble-plx`) | Development Build, permisos distintos por versión de Android (`BLUETOOTH_SCAN`/`CONNECT` en 12+), texto `NSBluetoothAlwaysUsageDescription` en iOS. Verificá compatibilidad New Architecture. |
| NFC | Librería comunitaria (p. ej. `react-native-nfc-manager`) | iOS: solo dispositivos compatibles, entitlement NFC y sesión con UI del sistema; Android: chequear `isEnabled`. |

Para cada una: adaptador en `shared/native/<capacidad>/` solo si hay permisos, ciclo de vida o diferencias por plataforma; un uso trivial puede llamar a la librería desde un hook del dominio.

## Sistema operativo

| Tema | Guía |
|---|---|
| AppState | [native-platforms.md](native-platforms.md#ciclo-de-vida-appstate) |
| Tareas en background | `expo-background-task` para trabajo periódico diferible (el SO decide cuándo; no es exacto) y `expo-task-manager` para tareas disparadas por el SO. Verificá disponibilidad en el SDK. Nunca prometer horarios exactos. |
| Deep links / Universal Links / App Links | [navigation.md](navigation.md#deep-links-universal-links-y-app-links) y [configuration.md](configuration.md) |
| Teclado, orientación | [native-platforms.md](native-platforms.md) |
| Apariencia y dark mode | `useColorScheme()`; `userInterfaceStyle: 'automatic'` en `app.config.ts`. Ver [design-system.md](design-system.md). |
| Accesibilidad | `AccessibilityInfo.isReduceMotionEnabled`, `isScreenReaderEnabled`, escala de fuente. Ver [design-system.md](design-system.md#accesibilidad). |

## Errores comunes

- Pedir todos los permisos en el onboarding.
- Tratar `canceled` como error.
- Volver a llamar `request...` en bucle cuando `canAskAgain` es `false`.
- Olvidar desuscribir listeners de sensores/notificaciones.
- Asumir que algo probado en Expo Go se comporta igual en producción.
