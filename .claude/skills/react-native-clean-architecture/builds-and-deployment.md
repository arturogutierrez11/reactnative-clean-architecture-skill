# Builds y publicación

No implementes pipelines completos hasta que sean necesarios. Este documento indica cómo hacerlo bien cuando llegue el momento. **Nunca ejecutes `eas submit`, publiques updates OTA a producción ni subas builds a tiendas sin autorización explícita del usuario.**

## Tipos de build

| Build | Para | Comando |
|---|---|---|
| Development Build | Desarrollo con módulos nativos propios | `eas build --profile development` o `npx expo run:ios` / `run:android` |
| Preview | QA interno (instalable por link, ad hoc/internal) | `eas build --profile preview` |
| Production | Tiendas (`.ipa` / `.aab`) | `eas build --profile production` |

Builds locales: `eas build --local` o Xcode/Gradle si el equipo no usa EAS.

## iOS

- **Bundle identifier** definitivo y registrado en Apple Developer.
- **Signing:** EAS gestiona certificados de distribución y provisioning profiles (recomendado) o usás credenciales propias (`credentials.json`, nunca versionado).
- **Capabilities** (Push, Associated Domains, Sign in with Apple...) declaradas en config; EAS las sincroniza con el App ID.
- **App Store Connect:** crear la app, completar App Privacy (datos recolectados: ubicación, identificadores, contacto...), clasificación de edad, screenshots por tamaño requerido.
- **TestFlight:** builds de producción pasan por revisión beta para testers externos; internos no.
- **Textos de permisos** (`NS*UsageDescription`) específicos y en el idioma de la app: motivo frecuente de rechazo.
- **Revisión:** cuenta demo si hay login; explicar en las notas funcionalidades en background (ubicación, audio); eliminación de cuenta in-app si permite crear cuentas.
- **Versionado:** `version` (CFBundleShortVersionString) y `buildNumber` (CFBundleVersion) incremental.
- Requisitos de SDK/Xcode mínimos para subir cambian cada año: verificá los vigentes.

## Android

- **Package name** definitivo.
- **Signing:** upload key (EAS la genera y guarda) + Play App Signing (Google firma la versión final). Perder la upload key requiere proceso de reset con Google.
- **Android App Bundle (`.aab`)** para Play; `.apk` solo para distribución interna.
- **Google Play Console:** ficha, Data safety, clasificación de contenido, público objetivo, declaraciones de permisos sensibles (ubicación en background, exact alarms, foreground services con su tipo).
- **Internal testing** → closed → open → producción. Cuentas personales nuevas pueden requerir un período de prueba cerrada con testers antes de producción; verificá la política vigente.
- **versionCode** entero incremental; **versionName** visible.
- **targetSdkVersion:** Google Play exige un mínimo que sube anualmente; el SDK de Expo lo define. Verificá el requisito vigente al publicar.

## Versionado

- `version` semver manual por release.
- Números de build automáticos: `"cli": { "appVersionSource": "remote" }` y `"autoIncrement": true` en el perfil `production` de `eas.json`.
- Tag de git por release publicada.

## Automatización

### Validaciones antes de un build

```bash
npx tsc --noEmit
npx expo lint
npx jest --ci
npx expo-doctor
```

Usá los scripts definidos en `package.json` si existen.

### GitHub Actions (cuando se necesite)

- Workflow de PR: install con el lockfile, typecheck, lint, tests. No requiere secretos.
- Workflow de release (manual o por tag): `expo/expo-github-action` con `EXPO_TOKEN` en secretos de GitHub → `eas build --platform all --profile production --non-interactive` → opcional `--auto-submit`.
- Alternativa: EAS Workflows (`.eas/workflows/*.yml`) si el equipo prefiere ejecutar todo en EAS; verificá la sintaxis vigente.
- Nunca imprimir secretos en logs del CI.

## Updates OTA

- `expo-updates` + EAS Update permiten publicar cambios **solo de JS y assets**.
- **runtimeVersion** determina compatibilidad: un update solo se aplica a builds con el mismo runtime. Política recomendada: `"runtimeVersion": { "policy": "fingerprint" }` (cambia automáticamente cuando cambia el nativo) o `appVersion`. Verificá las políticas disponibles.
- Si el cambio agrega o actualiza un módulo nativo, modifica config plugins, permisos o versión del SDK → **nuevo build**, no OTA.
- Canales por entorno (`preview`, `production`) ligados a perfiles de build.
- Publicar primero en preview, probar, luego promover. Tener plan de rollback (`eas update:rollback` o republicar el anterior; verificá comando).
- Las políticas de tiendas permiten OTA para correcciones y mejoras, no para cambiar el propósito de la app.

## Checklist de release

- [ ] Versión y build number correctos.
- [ ] Variables y API de producción; sin flags de debug.
- [ ] Permisos y textos revisados; permisos no usados bloqueados.
- [ ] Iconos, splash, nombre.
- [ ] Crash reporting con source maps/símbolos subidos ([observability.md](observability.md)).
- [ ] Flujos críticos probados en build de release en iOS y Android reales.
- [ ] App Privacy / Data safety actualizados.
- [ ] Notas de revisión y cuenta demo.
