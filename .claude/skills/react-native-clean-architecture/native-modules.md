# Módulos nativos, config plugins y builds de desarrollo

## Árbol de decisión

1. ¿Lo resuelve una API de React Native o un paquete del SDK de Expo (`expo-*`)? → Usalo con `npx expo install`.
2. ¿Lo resuelve una librería comunitaria mantenida, compatible con la New Architecture y con el SDK instalado? → Revisá [React Native Directory](https://reactnative.directory) (compatibilidad New Architecture, última release, plataformas), issues abiertos y si trae config plugin.
3. ¿Hace falta una API nativa no cubierta? → Módulo propio con **Expo Modules API** (Swift/Kotlin) en proyectos Expo; Turbo Module/Fabric con codegen en proyectos CLI que no usan Expo Modules.
4. ¿Solo hace falta cambiar configuración nativa (Info.plist, AndroidManifest, Gradle, Podfile, entitlements)? → **Config plugin**, no edición manual.

Documentá la decisión en la entrega: qué alternativa se descartó y por qué.

## Expo Go vs. Development Build

Expo Go trae un conjunto fijo de módulos nativos y solo soporta el SDK que acompaña. Requieren **Development Build** (`expo-dev-client` + `eas build --profile development` o `npx expo run:ios|android`):

- Cualquier librería con código nativo que no esté en Expo Go.
- Config plugins propios o que cambian permisos/entitlements.
- Ubicación en background y TaskManager con ejecución en background.
- Push notifications en Android (desde SDK 53).
- Face ID con texto propio (`NSFaceIDUsageDescription`).
- Módulos nativos propios.

Antes de implementar, decí explícitamente si la funcionalidad necesitará Development Build. No asumas que "funciona en Expo Go" implica que funciona en producción, ni al revés.

## Prebuild y CNG

- **Continuous Native Generation (CNG):** `ios/` y `android/` se generan con `npx expo prebuild` a partir de `app.config.ts` + config plugins. Suelen estar en `.gitignore`. **Cualquier edición manual se pierde** en el próximo prebuild.
- **Bare / carpetas versionadas:** `ios/` y `android/` son fuente. Se editan con Xcode/Android Studio y los cambios se revisan en el diff. Config plugins no se reaplican salvo que se corra prebuild.

Cómo saberlo: si `ios/`/`android/` están en `.gitignore` o no existen → CNG. Si están versionadas → bare. EAS Build también lo detecta y advierte si hay configuración nativa en `app.config.ts` que no se aplicará.

Regla: **nunca modificar archivos nativos generados sin determinar primero si sobrevivirán a un prebuild.** En CNG, todo cambio nativo va en un config plugin.

## Config plugins

- Primero usá las opciones de plugin del paquete (`["expo-location", { "locationWhenInUsePermission": "..." }]`).
- Plugin propio: archivo en `plugins/with<Capacidad>.ts` (o `.js` según cómo cargue la versión instalada), usando `@expo/config-plugins` (`withInfoPlist`, `withAndroidManifest`, `withEntitlementsPlist`, `withAppBuildGradle`, `withPodfile`...). Verificá los nombres de mods en la documentación vigente.
- Los plugins deben ser idempotentes (correr prebuild dos veces da el mismo resultado).
- Probá con `npx expo prebuild --clean` y revisá el diff generado (aunque no se versione).

## Expo Modules API (módulo propio)

```bash
npx create-expo-module@latest --local   # crea modules/<nombre> en el proyecto
```

- Swift en `ios/`, Kotlin en `android/`, contrato TS en `src/`. API declarativa (`Module`, `Name`, `Function`, `AsyncFunction`, `Events`, `View`).
- El contrato TS del módulo es infraestructura: consumirlo a través de un adaptador en `shared/native/<capacidad>/`, igual que cualquier API del dispositivo.
- Implementá ambas plataformas o documentá y tipá el soporte parcial (lanzar un error tipado `UNSUPPORTED_PLATFORM`).
- Permisos y capacidades del módulo se declaran en un config plugin del propio módulo.

## Dependencias nativas

- iOS: CocoaPods (o SPM si la versión de RN/Expo lo soporta). Mínimo de iOS definido por el SDK; no lo bajes.
- Android: Gradle; `minSdkVersion`/`compileSdkVersion`/`targetSdkVersion` los define el SDK. Para cambiarlos en CNG usá `expo-build-properties`.
- Tras agregar una librería nativa: nuevo Development Build. Un update OTA **no** puede agregar código nativo ([builds-and-deployment.md](builds-and-deployment.md#updates-ota)).

## New Architecture

- Las versiones actuales de React Native y Expo usan la New Architecture (Fabric, TurboModules, JSI). Verificá en el changelog del SDK instalado si la arquitectura legacy sigue siendo opcional.
- Antes de agregar una librería nativa, confirmá su compatibilidad. Si no es compatible, buscá alternativa; no desactives la New Architecture para una sola librería sin aprobación explícita.

## Capacidades de iOS y configuración Android

- iOS: entitlements (Push, Associated Domains, Sign in with Apple, App Groups, HealthKit...) y `UIBackgroundModes`. En CNG se declaran vía `ios.entitlements`, `ios.infoPlist` o plugins.
- Android: `android.permissions` (agregar) y `android.blockedPermissions` (quitar permisos que una librería agrega y no usás). Revisá el manifest final: Google Play revisa permisos sensibles.

## Verificación

- `npx expo-doctor` y `npx expo install --check` tras cambios de dependencias.
- Build real (`npx expo run:ios` / `run:android` o EAS development) para validar compilación nativa. TypeScript no valida nada nativo.
- Reportá qué plataforma compilaste y en qué destino (simulador/emulador/dispositivo).
