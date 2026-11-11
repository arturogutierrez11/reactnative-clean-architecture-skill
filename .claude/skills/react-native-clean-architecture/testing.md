# Testing y validación

**Regla de honestidad:** TypeScript, lint y tests unitarios no demuestran que algo funcione en iOS o Android. Nunca afirmes "verificado en iOS/Android" sin haberlo ejecutado en simulador, emulador o dispositivo; decí exactamente qué corriste y dónde.

## Stack

- Jest con el preset `jest-expo` (proyectos Expo) o `react-native` (CLI). Verificá el preset instalado.
- React Native Testing Library (`@testing-library/react-native`) para componentes y hooks.
- E2E: Maestro (flujos YAML, simple de mantener, buen soporte de permisos y simuladores). Detox es alternativa justificada si el proyecto ya lo usa o necesita sincronización gris-box.
- No introducir un runner nuevo para una corrección trivial.

### Configuración verificada (Expo SDK 58, noviembre 2026)

Comprobado al validar [examples.md](examples.md): con `jest-expo` 58, React Native 0.88, TypeScript 6 y React Native Testing Library 14 hizo falta:

- Instalar `@react-native/jest-preset` (peer de `jest-expo`; sin él, Jest falla al cargar el preset). Usar la versión que coincide con React Native.
- `moduleNameMapper` para el alias: `{ '^@/(.*)$': '<rootDir>/src/$1' }` (Jest no lee `paths` de tsconfig).
- `"types": ["jest"]` en `tsconfig.json`: TypeScript 6 ya no incluye todos los `@types` automáticamente.
- `await render(...)` y `userEvent.setup()` con `await user.press(...)`: en RNTL 14 el render es asíncrono.

En otras versiones verificá cada punto en lugar de copiarlo.

## Qué probar por capa

| Capa | Qué | Cómo |
|---|---|---|
| Unit | Funciones puras, schemas, mappers DTO → modelo, reglas, casos de uso | Jest puro, dependencias como parámetros o `jest.mock` |
| Servicios | Request correcto, validación de respuesta, mapeo de errores HTTP | Mock de `fetch` o del cliente HTTP |
| Adaptadores | Traducción de respuestas de la librería nativa a tipos propios | `jest.mock('expo-location')` con respuestas reales de la API |
| Hooks | Transiciones de estado (loading → success/error/permiso) | `renderHook` + `QueryClientProvider` de test |
| Componentes | Render por estado, interacciones, accesibilidad | RNTL: queries por rol/label/texto, `userEvent` |
| Integración | Login completo, refresh, persistencia, navegación protegida | Varias piezas reales + mocks solo en bordes (red, nativo) |
| E2E | Login, navegación crítica, permisos, flujos principales | Maestro en simulador/emulador |
| Manual en dispositivo | Hardware, background, rendimiento | Lista de escenarios abajo |

## Reglas

- Probá comportamiento, no implementación. Queries por rol y texto accesible (`getByRole('button', { name: 'Ingresar' })`), no por `testID` salvo que no haya alternativa.
- Mockeá en los **bordes** (red, módulos nativos, tiempo), no la lógica propia.
- Cada funcionalidad con permisos: test de permiso denegado y de "no volver a preguntar".
- Cada pantalla con datos: test de loading, vacío, error y éxito cuando aplican.
- `QueryClient` nuevo por test con `retry: false`.
- Timers falsos para backoff y timeouts.
- Sin snapshots grandes como sustituto de aserciones.

## Mocks de módulos nativos

```ts
jest.mock('expo-location', () => ({
  getForegroundPermissionsAsync: jest.fn(),
  requestForegroundPermissionsAsync: jest.fn(),
  hasServicesEnabledAsync: jest.fn(),
  getLastKnownPositionAsync: jest.fn(),
  getCurrentPositionAsync: jest.fn(),
  watchPositionAsync: jest.fn(),
  Accuracy: { Lowest: 1, Low: 2, Balanced: 3, High: 4, Highest: 5, BestForNavigation: 6 },
  PermissionStatus: { GRANTED: 'granted', DENIED: 'denied', UNDETERMINED: 'undetermined' },
}));
```

Preferí mockear tu adaptador (`LocationAdapter`) en tests de dominio; reservá el mock del paquete para los tests del adaptador. Algunos paquetes Expo traen mocks automáticos con `jest-expo`; verificá.

## E2E con Maestro (ejemplo)

```yaml
# e2e/location-permission-denied.yaml
appId: com.example.app
---
- launchApp:
    clearState: true
    permissions:
      location: deny
# Login con una cuenta de prueba pasada por variables (maestro test -e TEST_USER_EMAIL=... ).
- tapOn: "Email"
- inputText: ${TEST_USER_EMAIL}
- tapOn: "Contraseña"
- inputText: ${TEST_USER_PASSWORD}
- tapOn: "Ingresar"
- tapOn: "Ver mi ubicación"
- assertVisible: "Necesitamos acceso a tu ubicación"
- assertVisible: "Abrir configuración"
```

Verificá la sintaxis de `permissions` y comandos en la documentación vigente de Maestro. El `appId` es el bundle id / package real del proyecto. Las credenciales de prueba nunca se escriben en el flujo ni en el repo.

## Testing nativo y en dispositivo

Escenarios mínimos para funcionalidades nativas:

- iOS Simulator y Android Emulator para flujo general.
- Dispositivo real para: cámara, biometría, push, Bluetooth/NFC, background location, rendimiento.
- Permiso denegado, "permitir una vez", precisión reducida, revocado desde Configuración.
- Servicios desactivados (GPS, Bluetooth).
- Sin red, red intermitente, cambio Wi-Fi ↔ datos.
- Background → foreground, app terminada y relanzada, reinicio del dispositivo (si aplica).
- Fuente grande, dark mode, VoiceOver/TalkBack en flujos críticos.
- Dispositivo chico y grande; tablet si se soporta.

## Verificación antes de entregar

1. Detectar scripts en `package.json`; usar el package manager del lockfile.
2. `tsc --noEmit` (o script `typecheck`), lint, tests relacionados.
3. `npx expo install --check` y `npx expo-doctor` si cambiaron dependencias o config.
4. Build/run en simulador o emulador cuando el entorno lo permite y el cambio toca nativo, config o navegación.
5. Reportar: comando → resultado. Lo no ejecutado se lista como pendiente con el motivo.
