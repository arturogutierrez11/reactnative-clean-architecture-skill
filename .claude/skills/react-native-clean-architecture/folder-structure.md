# Estructura de carpetas

## Objetivo ilustrativo

```text
<raíz del proyecto>
├── app.config.ts            # o app.json
├── eas.json                 # cuando se usa EAS
├── .env.example
├── assets/                  # iconos, splash, fuentes
├── e2e/                     # flujos Maestro u otra herramienta
├── ios/ android/            # solo si el proyecto las versiona (sin CNG)
└── src/
    ├── app/                 # Expo Router
    │   ├── _layout.tsx
    │   ├── (auth)/
    │   │   ├── _layout.tsx
    │   │   └── login.tsx
    │   └── (app)/
    │       ├── _layout.tsx
    │       ├── index.tsx        # perfil
    │       └── location.tsx
    ├── domains/
    │   ├── auth/
    │   │   ├── login/
    │   │   │   ├── screens/LoginScreen.tsx
    │   │   │   ├── components/LoginForm.tsx
    │   │   │   ├── schemas/login.schema.ts
    │   │   │   ├── hooks/useLogin.ts
    │   │   │   └── services/login.ts
    │   │   └── session/
    │   │       ├── session.schema.ts
    │   │       ├── session.persistence.ts
    │   │       ├── session.store.ts
    │   │       ├── session.service.ts
    │   │       ├── configureAuthHttp.ts
    │   │       └── useSession.ts
    │   └── location/
    │       ├── get-current-location/
    │       │   ├── hooks/useCurrentLocation.ts
    │       │   ├── services/getCurrentLocation.ts
    │       │   ├── components/LocationPermissionState.tsx
    │       │   ├── screens/CurrentLocationScreen.tsx
    │       │   └── tests/
    │       └── watch-location/
    └── shared/
        ├── api/             # cliente HTTP y errores de transporte
        ├── config/          # env pública validada
        ├── ui/              # primitivas sin negocio
        ├── theme/           # tokens y hook de tema
        ├── native/          # adaptadores del dispositivo
        │   └── location/
        ├── storage/         # SecureStore / AsyncStorage
        ├── errors/          # AppError y normalización
        ├── observability/   # logger y reporte de errores
        ├── hooks/           # hooks genéricos (useAppState)
        ├── utils/
        └── types/
```

**No generes este árbol entero.** Cada carpeta existe solo si contiene archivos que la tarea necesita.

## Escala según complejidad

Una funcionalidad simple puede ser dos archivos:

```text
domains/profile/get-profile/
├── useProfile.ts          # useQuery + service inline o importado
└── ProfileCard.tsx
```

Una de complejidad media agrupa por responsabilidad cuando hay más de un archivo por tipo:

```text
domains/profile/update-profile/
├── components/ProfileForm.tsx
├── schemas/profile.schema.ts
├── services/updateProfile.ts
└── hooks/useUpdateProfile.ts
```

Una compleja (background, varias fuentes, reglas) suma `use-cases/`, `tasks/`, `tests/` (y `types/` si los tipos propios crecen):

```text
domains/location/watch-location/
├── components/LocationStatus.tsx
├── hooks/useLocationTracking.ts
├── use-cases/startLocationTracking.ts
├── services/locationQueue.ts
├── services/trackingPreference.ts
├── tasks/locationTask.ts
└── tests/
```

Regla: pasá de archivos sueltos a carpetas por responsabilidad cuando la funcionalidad supera ~4–5 archivos o cuando hay dos o más del mismo tipo.

## Pasos para ubicar código

1. Determiná la funcionalidad por su propósito, no por el nombre de una pantalla o una tecnología.
2. Las **rutas** viven en `src/app` y solo renderizan una pantalla del dominio: `export { CurrentLocationScreen as default }` o un componente que la compone.
3. La **pantalla** (`screens/`) compone componentes y hooks de la funcionalidad. No conoce la ruta física salvo a través de `router`/`Link`.
4. Los **tests** van junto al código (`*.test.ts(x)`) o en `tests/` de la funcionalidad, según la convención existente. E2E en `e2e/` en la raíz.
5. `assets/` queda en la raíz porque `app.config.ts` lo referencia. Fuentes e imágenes de un componente pueden importarse desde ahí.
6. Tareas en background (`TaskManager.defineTask`) se definen en un archivo del dominio (`tasks/`) y se importan una sola vez en el scope global, desde `src/app/_layout.tsx`. Ver [geolocation.md](geolocation.md#segundo-plano).

## Proyecto nuevo

1. Creá el proyecto con la plantilla oficial vigente (`npx create-expo-app@latest`). Verificá en la documentación cuál es la plantilla por defecto y si ya usa `src/app`.
2. Si la plantilla deja `app/` en la raíz, movelo a `src/app/`: Expo Router detecta `src/app` automáticamente cuando no existe `app/` en la raíz. Verificalo en la documentación de la versión instalada.
3. `tsconfig.json`: `"strict": true` y `"paths": { "@/*": ["./src/*"] }`. Expo soporta `paths` de tsconfig sin plugin de Babel adicional; verificalo para la versión instalada.
4. Eliminá pantallas y componentes de demo de la plantilla antes de construir.
5. Agregá dependencias recién cuando una funcionalidad las use, con `npx expo install <paquete>` para obtener versiones compatibles con el SDK.

## Proyecto existente

- Respetá la ubicación de rutas existente (`app/` raíz, `src/app`, o React Navigation en `src/navigation`). No mantengas dos árboles de rutas.
- Con React Navigation, el equivalente de `src/app` es el módulo de navegadores (`src/app/navigation/` o la carpeta existente): define stacks/tabs y registra pantallas del dominio. Los conceptos de este documento aplican igual.
- En React Native CLI, `ios/` y `android/` son código fuente. En Expo con CNG (`prebuild`), son artefactos generados: no los edites a mano. Ver [native-modules.md](native-modules.md#prebuild-y-cng).
