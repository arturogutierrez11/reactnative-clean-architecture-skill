---
name: react-native-clean-architecture
description: Estándar de ingeniería para apps React Native/Expo con TypeScript organizadas en src/app, src/domains y src/shared por dominio → funcionalidad → responsabilidad. Usar para crear apps, agregar funcionalidades, refactorizar, revisar arquitectura, integrar APIs nativas de iOS/Android (geolocalización, permisos, cámara, notificaciones, biometría, módulos nativos), navegación con Expo Router, sesiones, estado, datos, diseño, testing, builds EAS y publicación en App Store/Google Play.
---

# React Native Clean Architecture

Aplicá esta Skill al pedido concreto del usuario. Implementá código y verificaciones cuando pida implementar; entregá hallazgos con evidencia cuando pida diagnóstico. No modifiques código en un pedido de solo análisis.

## Contrato obligatorio

- Estructura objetivo: `src/app/`, `src/domains/`, `src/shared/`. Organización: **dominio → funcionalidad → responsabilidad**.
- `app/` contiene rutas, layouts, providers, guards de sesión y composición. No contiene llamadas HTTP, reglas de negocio ni APIs del dispositivo.
- `domains/` contiene funcionalidades y sus componentes, pantallas, hooks, servicios, schemas y adaptadores propios.
- `shared/` contiene solo capacidades reutilizables sin reglas de negocio: cliente HTTP, UI primitiva, tema, adaptadores nativos genéricos, almacenamiento, errores, config.
- Las APIs del dispositivo (ubicación, cámara, biometría, notificaciones, almacenamiento seguro) se consumen a través de un adaptador en `shared/native/` o del dominio. Los componentes nunca importan módulos nativos directamente.
- iOS y Android son plataformas de primera clase. Código compartido por defecto; diferencias en adaptadores o archivos `.ios.tsx`/`.android.tsx` cuando sean reales.
- Verificá versiones instaladas, compatibilidad con Expo SDK/React Native/New Architecture y restricciones de Expo Go antes de usar una API o instalar una dependencia. Nunca inventes versiones ni APIs.
- Crear solo archivos y carpetas con contenido necesario. La arquitectura crece con la complejidad real.
- No guardar secretos de backend en la app, ni tokens fuera de almacenamiento seguro, ni datos personales o coordenadas en logs.
- No afirmar que algo funciona en iOS o Android si solo se ejecutó TypeScript, lint o tests unitarios.

## Procedimiento de trabajo

1. **Inspección.** Leé `CLAUDE.md`/instrucciones del repo, `package.json`, lockfile, `app.json`/`app.config.ts`, `eas.json`, `tsconfig.json`, configuración de navegación, carpetas `ios/`/`android/` si existen, estructura de `src/` y tests. Determiná Expo SDK, React Native, package manager, si es proyecto con prebuild (CNG) o con carpetas nativas versionadas, y qué módulos nativos hay. No leas ni imprimas `.env`, keystores ni credenciales.
2. **Clasificación.** Proyecto nuevo, funcionalidad, corrección, refactor, integración nativa, diagnóstico, optimización o publicación.
3. **Diseño.** Dominio, funcionalidad, responsabilidades, contratos, permisos, diferencias iOS/Android, necesidad de Development Build, riesgos y pruebas. Conciso para tareas chicas; plan explícito para tareas complejas o que tocan configuración nativa.
4. **Implementación.** Una porción funcional completa: UI, estados (carga, vacío, error, permiso denegado, offline cuando aplique), lógica, acceso a datos/dispositivo, accesibilidad. Reutilizá lo existente. Sin esqueletos vacíos.
5. **Verificación.** Typecheck, lint, tests relacionados y `npx expo install --check`/`npx expo-doctor` cuando cambien dependencias o config. Separá claramente lo verificado de lo que requiere simulador, emulador o dispositivo. Ver [checklist.md](checklist.md).
6. **Entrega.** Qué se implementó, archivos modificados, decisiones, dependencias agregadas, verificaciones ejecutadas con resultado, pendientes y diferencias iOS/Android.

## Referencias por tarea

Leé siempre [architecture.md](architecture.md), [folder-structure.md](folder-structure.md) y [responsibilities.md](responsibilities.md) antes de crear archivos. Después, solo lo que la tarea necesita:

| Necesidad | Leer |
|---|---|
| Nombres de archivos, símbolos, imports | [naming-conventions.md](naming-conventions.md) |
| Pantallas y componentes | [components.md](components.md) |
| Rutas, layouts, guards, deep links | [navigation.md](navigation.md) |
| Diferencias iOS/Android, teclado, safe areas, Back | [native-platforms.md](native-platforms.md) |
| Swift/Kotlin, Expo Modules, config plugins, prebuild | [native-modules.md](native-modules.md) |
| Ubicación, tracking, geofencing | [geolocation.md](geolocation.md) |
| Permisos, cámara, notificaciones, biometría, hardware | [device-permissions.md](device-permissions.md) |
| useState, Context, Zustand, TanStack Query | [state-management.md](state-management.md) |
| Cliente HTTP, DTOs, errores, paginación, uploads | [data-fetching.md](data-fetching.md) |
| Login, refresh tokens, sesión, logout | [authentication.md](authentication.md) |
| SecureStore, AsyncStorage, SQLite, archivos | [storage.md](storage.md) |
| Seguridad mobile | [security.md](security.md) |
| Tokens visuales, tema, dark mode, accesibilidad | [design-system.md](design-system.md) |
| Animaciones y gestos | [animations.md](animations.md) |
| Listas, renders, imágenes, batería | [performance.md](performance.md) |
| Offline, cache, sincronización | [offline.md](offline.md) |
| Unit, componentes, integración, E2E, dispositivo | [testing.md](testing.md) |
| Entornos, `app.config.ts`, variables, identificadores | [configuration.md](configuration.md) |
| EAS Build/Submit, firmas, tiendas, OTA | [builds-and-deployment.md](builds-and-deployment.md) |
| Errores en producción, crashes, logs | [observability.md](observability.md) |
| Implementaciones de referencia | [examples.md](examples.md) |
| Repo existente, refactor, auditoría | [inconsistencies.md](inconsistencies.md) |

## Aplicación por modo

- **Proyecto nuevo:** Expo + TypeScript strict + Expo Router con `src/app`. Configurá alias `@/*` → `src/*`. Instalá TanStack Query, Zustand, React Hook Form o Zod solo cuando la primera funcionalidad que los necesita llegue. Ver [folder-structure.md](folder-structure.md#proyecto-nuevo).
- **Nueva funcionalidad:** ubicá el dominio, implementá solo las responsabilidades necesarias y conectá desde `app/`.
- **Integración nativa:** primero confirmá si una librería Expo o mantenida cubre la necesidad; documentá permisos, config plugin, Development Build y comportamiento por plataforma antes de escribir código.
- **Existente/refactor:** respetá el router, el workflow (Expo o React Native CLI) y las convenciones existentes. No migres a Expo, a Expo Router ni a `src/` sin pedido explícito. Migrá por funcionalidad. Ver [inconsistencies.md](inconsistencies.md).
- **Diagnóstico:** hechos observados, riesgos inferidos y propuestas, con archivo y evidencia.
- **Publicación:** ver [builds-and-deployment.md](builds-and-deployment.md) y [configuration.md](configuration.md).

## Contra la sobreingeniería

- Sin interfaces para servicios con una sola implementación, salvo en las fronteras con el dispositivo o proveedores externos donde hay sustitución real (tests, mocks, proveedor alternativo).
- Sin repositorios abstractos, contenedores de DI, clases ni casos de uso por defecto: una función bien nombrada suele bastar.
- Sin stores globales para estado local, ni hooks que solo reenvían una llamada.
- Sin offline-first, módulos nativos o librerías antes de que la necesidad exista.
- Sin reescribir código funcional sin una razón concreta.

## Criterios de finalización

La tarea termina cuando la funcionalidad está completa en su alcance, respeta los límites de [architecture.md](architecture.md), maneja estados y permisos relevantes, tiene pruebas proporcionales, los checks disponibles se ejecutaron (con resultado reportado) y la entrega declara qué falta verificar en dispositivo.
