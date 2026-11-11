# Configuración y entornos

## Entornos

| Entorno | Uso | Build | API |
|---|---|---|---|
| development | Desarrollo local con Development Build | `development` (dev client) | Local o dev |
| preview / staging | QA interno, TestFlight interno, Play internal testing | `preview` | Staging |
| production | Tiendas | `production` | Producción |

Cada entorno debe poder instalarse en paralelo en el mismo dispositivo cuando QA lo necesite: usar bundle id/package con sufijo (`com.acme.app.development`, `com.acme.app.preview`) y nombre visible distinto.

## `app.config.ts`

Usá `app.config.ts` (dinámico) cuando la configuración varía por entorno; `app.json` alcanza si no.

```ts
// app.config.ts
import type { ExpoConfig, ConfigContext } from 'expo/config';

const APP_VARIANT = process.env.APP_VARIANT ?? 'development';
const suffix = APP_VARIANT === 'production' ? '' : `.${APP_VARIANT}`;
const nameSuffix = APP_VARIANT === 'production' ? '' : ` (${APP_VARIANT})`;

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: `Acme${nameSuffix}`,
  slug: 'acme',
  scheme: `acme${APP_VARIANT === 'production' ? '' : `-${APP_VARIANT}`}`,
  version: '1.0.0',
  orientation: 'portrait',
  userInterfaceStyle: 'automatic',
  icon: './assets/images/icon.png',
  ios: {
    bundleIdentifier: `com.acme.app${suffix}`,
    supportsTablet: false,
  },
  android: {
    package: `com.acme.app${suffix}`,
    adaptiveIcon: { foregroundImage: './assets/images/adaptive-icon.png', backgroundColor: '#FFFFFF' },
    blockedPermissions: [],
  },
  plugins: [
    'expo-router',
    ['expo-splash-screen', { image: './assets/images/splash-icon.png', imageWidth: 200, backgroundColor: '#FFFFFF' }],
    // Agregar plugins solo cuando la funcionalidad los usa:
    // ['expo-location', { locationWhenInUsePermission: '...' }],
  ],
  experiments: { typedRoutes: true },
  extra: {
    eas: { projectId: process.env.EAS_PROJECT_ID },
  },
});
```

Ajustá nombres de assets y opciones de plugins a los que existan en el proyecto; verificá las opciones de `expo-splash-screen` y `experiments` para el SDK instalado.

`APP_VARIANT` se define por perfil en `eas.json` (`"env": { "APP_VARIANT": "preview" }`) o al correr local.

## Variables de entorno

- **Públicas (cliente):** `EXPO_PUBLIC_*`. Se inlinean en el bundle en texto plano. Solo URLs, flags y claves públicas restringidas.
- **De build:** variables sin prefijo leídas en `app.config.ts` o `eas.json` (p. ej. `APP_VARIANT`). No llegan al bundle salvo que las pongas en `extra` (y entonces son públicas).
- **Secretos:** credenciales de tiendas, tokens de servicios, keystores → EAS environment variables con visibilidad secreta / secretos del CI. Nunca en `.env` versionado ni en el bundle.
- Archivos `.env` cargados por Expo CLI según modo: `.env`, `.env.local`, `.env.development`, `.env.production` (+ `.local`). Verificá en la documentación del SDK cuáles se cargan; el SDK 58 no carga archivos con otros nombres como `.env.staging`.
- Acceso estático con punto: `process.env.EXPO_PUBLIC_API_URL`.
- `.env*.local` y `.env` en `.gitignore`; versionar `.env.example` con nombres y sin valores reales.

```bash
# .env.example
EXPO_PUBLIC_API_URL=https://api.example.com
```

Validar al iniciar:

```ts
// src/shared/config/env.ts
import { z } from 'zod';

const schema = z.object({
  apiUrl: z.url(),
});

export const env = schema.parse({
  apiUrl: process.env.EXPO_PUBLIC_API_URL,
});
```

Si falla, la app debe fallar rápido en desarrollo con un mensaje claro (sin imprimir valores).

## Identificadores

- `ios.bundleIdentifier` y `android.package`: definitivos antes del primer envío a tiendas; no cambian después sin crear otra app.
- `version` (visible, semver) vs. `ios.buildNumber` / `android.versionCode` (incrementales por build). Con EAS, preferir `"appVersionSource": "remote"` + `autoIncrement` en el perfil de producción ([builds-and-deployment.md](builds-and-deployment.md#versionado)).

## Permisos y plugins

- Textos de permisos iOS en opciones de cada plugin o `ios.infoPlist`.
- Android: `android.permissions` para agregar; `android.blockedPermissions` para quitar los que agregan librerías y no usás.
- Revisar el resultado con `npx expo config --type introspect` (verificá el flag) o con `npx expo prebuild` en una rama descartable.

## Deep links

- `scheme` para links propios.
- Universal Links: `ios.associatedDomains: ['applinks:acme.com']` + archivo `apple-app-site-association` servido por tu dominio.
- App Links: `android.intentFilters` con `autoVerify: true` + `assetlinks.json` en el dominio.

## Assets

- Icono 1024×1024 sin transparencia para iOS; adaptive icon para Android (foreground + background).
- Splash con `expo-splash-screen` (imagen + color, variantes dark si corresponde).
- Fuentes vía `expo-font`.

## Servicios externos

- Archivos de configuración de Firebase (`GoogleService-Info.plist`, `google-services.json`) referenciados desde config; decidir con el equipo si se versionan (contienen identificadores, no secretos de servidor) o se inyectan desde secretos de EAS.
- Claves de mapas restringidas por bundle id/package y huella SHA del certificado.

## EAS

```json
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "development": { "developmentClient": true, "distribution": "internal", "env": { "APP_VARIANT": "development" } },
    "preview": { "distribution": "internal", "env": { "APP_VARIANT": "preview" } },
    "production": { "autoIncrement": true, "env": { "APP_VARIANT": "production" } }
  },
  "submit": { "production": {} }
}
```

Verificá campos y versión mínima de `eas-cli` en la documentación vigente; agregá `"version"` en `cli` según lo que recomiende `eas build:configure`.
