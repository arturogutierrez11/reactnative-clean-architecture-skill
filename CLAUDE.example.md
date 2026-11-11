# CLAUDE.md

Copiá este archivo como `CLAUDE.md` en la raíz del proyecto y completá los campos entre `<>`. Eliminá lo que no aplique.

## Proyecto

- Nombre: `<nombre de la app>`
- Propósito: `<una o dos frases sobre qué hace la app y para quién>`
- Plataformas: iOS `<versión mínima>` y Android `<versión mínima>`
- Workflow: `<Expo con CNG (ios/ y android/ generados) | Expo con carpetas nativas versionadas | React Native CLI>`
- Package manager: `<npm | pnpm | yarn | bun>`

## Metodología

Usá la skill `react-native-clean-architecture` para todo trabajo en esta app:

- Estructura `src/app`, `src/domains`, `src/shared`; organización dominio → funcionalidad → responsabilidad.
- APIs del dispositivo mediante adaptadores en `src/shared/native/`.
- La arquitectura crece con la complejidad real: no crear carpetas, interfaces, stores ni dependencias sin necesidad.

## Dominios actuales

| Dominio | Funcionalidades |
|---|---|
| `<auth>` | `<login, session>` |
| `<location>` | `<get-current-location>` |

## Entornos

| Entorno | Perfil EAS | API |
|---|---|---|
| development | `development` | `<url>` |
| preview | `preview` | `<url>` |
| production | `production` | `<url>` |

Variables públicas documentadas en `.env.example`. Nunca leer ni imprimir `.env`, keystores ni credenciales.

## Comandos

```bash
<pm> run typecheck
<pm> run lint
<pm> test
npx expo-doctor
```

## Reglas del proyecto

- `<Ej.: textos de UI en español rioplatense>`
- `<Ej.: no agregar dependencias nativas sin aprobación>`
- `<Ej.: toda funcionalidad con permisos incluye test de permiso denegado>`
- No ejecutar `eas submit`, `eas update` a producción ni publicar en tiendas sin autorización explícita.
- Al entregar, indicar qué se verificó y en qué plataforma; lo que requiere dispositivo se lista como pendiente.
