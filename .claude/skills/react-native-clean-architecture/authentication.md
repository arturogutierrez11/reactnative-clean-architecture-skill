# Autenticación y sesión

## Estructura

```text
src/domains/auth/
├── login/
│   ├── screens/LoginScreen.tsx
│   ├── components/LoginForm.tsx
│   ├── schemas/login.schema.ts
│   ├── hooks/useLogin.ts
│   └── services/login.ts
└── session/
    ├── session.schema.ts        # usuario y tokens (Zod)
    ├── session.persistence.ts   # tokens en SecureStore
    ├── session.store.ts         # estado: restoring | anonymous | authenticated
    ├── session.service.ts       # restore, signIn, signOut, refresh
    ├── configureAuthHttp.ts     # conecta la sesión con shared/api
    └── useSession.ts            # API pública del dominio
```

No hay funcionalidad `logout/` separada: cerrar sesión es `signOut({ revoke: true })` del servicio de sesión, llamado desde la pantalla que tenga el botón. Crearla solo si el logout gana lógica propia (confirmaciones, sincronización pendiente).

Implementación completa: [examples.md](examples.md#6-autenticación-y-sesión).

## Modelo de sesión

```ts
type Session =
  | { status: 'restoring' }        // leyendo SecureStore al iniciar
  | { status: 'anonymous'; reason?: 'expired' }
  | { status: 'authenticated'; user: SessionUser };
```

- Un único estado `restoring` evita loops de redirección y parpadeos.
- El store guarda solo datos públicos del usuario. Los tokens viven en SecureStore y en memoria del módulo de sesión.

## Tokens

| Token | Dónde | Notas |
|---|---|---|
| Access token (corto) | Memoria + SecureStore | Se adjunta en cada request autenticado |
| Refresh token | SecureStore | Nunca en AsyncStorage, Zustand persistido, logs ni params de URL |
| Datos públicos del usuario | SecureStore o AsyncStorage | Permite arrancar offline con la sesión conocida |

- `keychainAccessible`: el default (`WHEN_UNLOCKED`) es correcto para la mayoría. Si una tarea en background necesita el token con el dispositivo bloqueado, usar `AFTER_FIRST_UNLOCK` y documentarlo.
- El backend debe rotar refresh tokens y poder revocarlos.

## Flujos

**Restauración (arranque):** leer tokens → si no hay, `anonymous` → si hay, `authenticated` con el usuario guardado → los requests validarán el token; si el refresh falla, se cierra la sesión. Así la app abre sin red.

**Login:** validar formulario (Zod) → `POST /auth/login` → validar respuesta → guardar tokens → `authenticated`. El guard de navegación redirige solo; la pantalla no navega.

**Refresh:** el cliente HTTP recibe 401 → llama al refresh registrado (una sola promesa compartida entre requests concurrentes) → guarda nuevos tokens → reintenta una vez. Si el refresh devuelve 401/400 → `signOut()`. Si falla por red → no cerrar sesión; propagar `NETWORK`.

**Logout:** best-effort `POST /auth/logout` (revocar refresh) → borrar SecureStore → `queryClient.clear()` → reset de stores → desregistrar push token del usuario → `anonymous`.

**Sesión expirada:** igual que logout pero disparado por el cliente HTTP, con mensaje "Tu sesión expiró".

## OAuth / proveedores externos

- `expo-auth-session` + `expo-web-browser` con PKCE. Nunca embebas `client_secret` en la app.
- Sign in with Apple es obligatorio en iOS si ofrecés otros logins sociales de terceros (regla de App Review; verificá la versión vigente de las guidelines).
- El intercambio de código por tokens lo hace preferentemente tu backend.

## Biometría

Desbloqueo local de una sesión existente, no autenticación con el servidor. Ver [device-permissions.md](device-permissions.md#biometría-y-autenticación-local).

## Reglas

- Lógica de auth en un solo lugar (dominio `auth` + guard en un layout).
- Ninguna pantalla lee tokens.
- No confiar en roles guardados localmente para proteger datos: el backend autoriza.
- No loguear credenciales, tokens ni respuestas de login.
- Formularios con `textContentType`/`autoComplete` correctos para gestores de contraseñas.
