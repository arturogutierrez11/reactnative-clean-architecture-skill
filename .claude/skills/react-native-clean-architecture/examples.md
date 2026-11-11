# Ejemplos operativos

## Alcance y adaptación

Los ejemplos forman una app mínima coherente: login con sesión persistente, perfil remoto y una pantalla de ubicación actual. Usan Expo Router con `src/app`, TypeScript strict, TanStack Query 5, Zustand 5, React Hook Form 7, Zod 4, `expo-location`, `expo-secure-store` y `@react-native-community/netinfo`. **Verificá las versiones instaladas y adaptá las APIs antes de copiar.** Instalá dependencias solo cuando la funcionalidad las use (`npx expo install <paquete>`).

Convenciones: alias `@/` → `src/`. La primera línea de cada bloque indica la ruta del archivo. El backend ilustrativo expone `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout` y `GET /me/profile` con DTOs en snake_case; adaptá schemas al contrato real.

Archivos usados por los ejemplos y definidos en otras referencias:

- `src/shared/config/env.ts` → [configuration.md](configuration.md#variables-de-entorno)
- `src/shared/storage/secureStorage.ts` → [storage.md](storage.md#wrapper-en-shared)
- `src/shared/observability/logger.ts` → [observability.md](observability.md#logger)
- `src/shared/theme/tokens.ts` y `useTheme.ts` → [design-system.md](design-system.md#tokens)

Contenido:

1. [Fundaciones compartidas: errores y cliente HTTP](#1-fundaciones-compartidas-errores-y-cliente-http)
2. [Navegación con Expo Router](#2-navegación-con-expo-router)
3. [Componentes reutilizables](#3-componentes-reutilizables)
4. [Integración REST: perfil](#4-integración-rest-perfil)
5. [Creación de una pantalla con manejo de estados](#5-creación-de-una-pantalla-con-manejo-de-estados)
6. [Autenticación y sesión](#6-autenticación-y-sesión)
7. [Geolocalización: adaptador nativo](#7-geolocalización-adaptador-nativo)
8. [Caso de uso: obtener la ubicación actual](#8-caso-de-uso-obtener-la-ubicación-actual)
9. [Hook personalizado y solicitud de permisos](#9-hook-personalizado-y-solicitud-de-permisos)
10. [Pantalla de ubicación](#10-pantalla-de-ubicación)
11. [Testing](#11-testing)
12. [Refactor de un componente con demasiadas responsabilidades](#12-refactor-de-un-componente-con-demasiadas-responsabilidades)

---

## 1. Fundaciones compartidas: errores y cliente HTTP

`AppError` normaliza fallas para que la UI decida mensajes por código.

```ts
// src/shared/errors/AppError.ts
export type AppErrorCode =
  | 'NETWORK'
  | 'TIMEOUT'
  | 'CANCELLED'
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'
  | 'NOT_FOUND'
  | 'VALIDATION'
  | 'CONFLICT'
  | 'SERVER'
  | 'INVALID_RESPONSE'
  | 'UNKNOWN';

type AppErrorOptions = { status?: number; details?: unknown; originalError?: unknown };

export class AppError extends Error {
  readonly code: AppErrorCode;
  readonly status?: number;
  readonly details?: unknown;
  readonly originalError?: unknown;

  constructor(code: AppErrorCode, options: AppErrorOptions = {}) {
    super(code);
    this.name = 'AppError';
    this.code = code;
    this.status = options.status;
    this.details = options.details;
    this.originalError = options.originalError;
  }
}

export function errorCodeOf(error: unknown): AppErrorCode {
  return error instanceof AppError ? error.code : 'UNKNOWN';
}
```

El cliente HTTP no conoce el dominio `auth`: la sesión se conecta registrando funciones con `configureHttpClient` ([sección 6](#6-autenticación-y-sesión)).

```ts
// src/shared/api/httpClient.ts
import type { ZodType } from 'zod';
import { env } from '@/shared/config/env';
import { AppError, type AppErrorCode } from '@/shared/errors/AppError';

type AuthHooks = {
  getAccessToken: () => string | null;
  /** Devuelve true si obtuvo un token nuevo. Lanza AppError('NETWORK') si no pudo intentarlo. */
  refreshAccessToken: () => Promise<boolean>;
  onUnauthorized: () => void;
};

let authHooks: AuthHooks | null = null;

export function configureHttpClient(hooks: AuthHooks) {
  authHooks = hooks;
}

export type RequestOptions = {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  body?: unknown;
  signal?: AbortSignal;
  timeoutMs?: number;
  /** false para endpoints públicos (login, refresh). */
  auth?: boolean;
  headers?: Record<string, string>;
};

const DEFAULT_TIMEOUT_MS = 15_000;
const baseUrl = env.apiUrl.replace(/\/$/, '');

export async function request<T>(path: string, schema: ZodType<T>, options: RequestOptions = {}): Promise<T> {
  const useAuth = options.auth !== false;
  let response = await send(path, options, useAuth);

  if (response.status === 401 && useAuth && authHooks) {
    const refreshed = await authHooks.refreshAccessToken();
    if (refreshed) response = await send(path, options, useAuth);
    if (response.status === 401) authHooks.onUnauthorized();
  }

  return parseResponse(response, schema);
}

async function send(path: string, options: RequestOptions, useAuth: boolean): Promise<Response> {
  const { method = 'GET', body, signal, timeoutMs = DEFAULT_TIMEOUT_MS, headers } = options;
  const controller = new AbortController();
  let timedOut = false;
  const timer = setTimeout(() => {
    timedOut = true;
    controller.abort();
  }, timeoutMs);
  const abortFromCaller = () => controller.abort();
  if (signal?.aborted) controller.abort();
  else signal?.addEventListener('abort', abortFromCaller);

  const token = useAuth ? authHooks?.getAccessToken() : null;

  try {
    return await fetch(`${baseUrl}${path}`, {
      method,
      headers: {
        Accept: 'application/json',
        ...(body !== undefined ? { 'Content-Type': 'application/json' } : {}),
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...headers,
      },
      body: body === undefined ? undefined : JSON.stringify(body),
      signal: controller.signal,
    });
  } catch (error) {
    if (timedOut) throw new AppError('TIMEOUT', { originalError: error });
    if (controller.signal.aborted) throw new AppError('CANCELLED', { originalError: error });
    throw new AppError('NETWORK', { originalError: error });
  } finally {
    clearTimeout(timer);
    signal?.removeEventListener('abort', abortFromCaller);
  }
}

async function parseResponse<T>(response: Response, schema: ZodType<T>): Promise<T> {
  const payload = await readBody(response);

  if (!response.ok) {
    throw new AppError(codeForStatus(response.status), { status: response.status, details: payload });
  }

  const result = schema.safeParse(payload);
  if (!result.success) {
    throw new AppError('INVALID_RESPONSE', { status: response.status, originalError: result.error });
  }
  return result.data;
}

async function readBody(response: Response): Promise<unknown> {
  if (response.status === 204) return undefined;
  const text = await response.text();
  if (!text) return undefined;
  try {
    return JSON.parse(text) as unknown;
  } catch (error) {
    if (response.ok) throw new AppError('INVALID_RESPONSE', { status: response.status, originalError: error });
    return undefined; // cuerpo de error no JSON (p. ej. HTML de un proxy)
  }
}

function codeForStatus(status: number): AppErrorCode {
  if (status === 400 || status === 422) return 'VALIDATION';
  if (status === 401) return 'UNAUTHORIZED';
  if (status === 403) return 'FORBIDDEN';
  if (status === 404) return 'NOT_FOUND';
  if (status === 409) return 'CONFLICT';
  if (status >= 500) return 'SERVER';
  return 'UNKNOWN';
}
```

TanStack Query: un cliente, reintentos solo para errores transitorios, y conexión con `AppState`/NetInfo porque React Native no tiene eventos de foco ni de red del navegador.

```ts
// src/shared/api/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { AppError, type AppErrorCode } from '@/shared/errors/AppError';

const NOT_RETRYABLE: AppErrorCode[] = ['UNAUTHORIZED', 'FORBIDDEN', 'NOT_FOUND', 'VALIDATION', 'CANCELLED', 'INVALID_RESPONSE'];

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      retry: (failureCount, error) =>
        failureCount < 2 && !(error instanceof AppError && NOT_RETRYABLE.includes(error.code)),
    },
    mutations: { retry: false },
  },
});
```

```ts
// src/shared/api/reactQueryNative.ts
import NetInfo from '@react-native-community/netinfo';
import { focusManager, onlineManager } from '@tanstack/react-query';
import { AppState } from 'react-native';

/** Conecta TanStack Query con el ciclo de vida de la app y la red. Devuelve cleanup. */
export function setupReactQueryForNative(): () => void {
  onlineManager.setEventListener((setOnline) =>
    NetInfo.addEventListener((state) => setOnline(state.isConnected !== false)),
  );
  const subscription = AppState.addEventListener('change', (status) => {
    focusManager.setFocused(status === 'active');
  });
  return () => subscription.remove();
}
```

---

## 2. Navegación con Expo Router

```text
src/app/
├── _layout.tsx          # providers + guard único
├── (auth)/
│   ├── _layout.tsx
│   └── login.tsx
└── (app)/
    ├── _layout.tsx
    ├── index.tsx        # perfil
    └── location.tsx     # ubicación actual
```

El layout raíz es el único lugar que decide acceso. Mientras la sesión está `restoring`, el splash sigue visible.

```tsx
// src/app/_layout.tsx
import '@/domains/auth/session/configureAuthHttp';
import { QueryClientProvider } from '@tanstack/react-query';
import { Stack } from 'expo-router';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { restoreSession } from '@/domains/auth/session/session.service';
import { useSession } from '@/domains/auth/session/useSession';
import { queryClient } from '@/shared/api/queryClient';
import { setupReactQueryForNative } from '@/shared/api/reactQueryNative';

void SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const session = useSession();

  useEffect(() => {
    void restoreSession();
    return setupReactQueryForNative();
  }, []);

  useEffect(() => {
    if (session.status !== 'restoring') void SplashScreen.hideAsync();
  }, [session.status]);

  if (session.status === 'restoring') return null;

  const isAuthenticated = session.status === 'authenticated';

  return (
    <SafeAreaProvider>
      <QueryClientProvider client={queryClient}>
        <Stack screenOptions={{ headerShown: false }}>
          <Stack.Protected guard={isAuthenticated}>
            <Stack.Screen name="(app)" />
          </Stack.Protected>
          <Stack.Protected guard={!isAuthenticated}>
            <Stack.Screen name="(auth)" />
          </Stack.Protected>
        </Stack>
      </QueryClientProvider>
    </SafeAreaProvider>
  );
}
```

```tsx
// src/app/(auth)/_layout.tsx
import { Stack } from 'expo-router';

export default function AuthLayout() {
  return <Stack screenOptions={{ headerShown: false }} />;
}
```

```tsx
// src/app/(auth)/login.tsx
export { LoginScreen as default } from '@/domains/auth/login/screens/LoginScreen';
```

```tsx
// src/app/(app)/_layout.tsx
import { Stack } from 'expo-router';

export default function AppLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Mi perfil' }} />
      <Stack.Screen name="location" options={{ title: 'Mi ubicación' }} />
    </Stack>
  );
}
```

```tsx
// src/app/(app)/index.tsx
export { ProfileScreen as default } from '@/domains/profile/get-profile/screens/ProfileScreen';
```

```tsx
// src/app/(app)/location.tsx
export { CurrentLocationScreen as default } from '@/domains/location/get-current-location/screens/CurrentLocationScreen';
```

Notas:

- Ninguna pantalla navega al login ni al área privada: el guard reacciona al cambio de sesión.
- Las rutas solo reexportan pantallas. Si una ruta necesita leer parámetros, los valida y los pasa como props.
- Si la versión de Expo Router no tiene `Stack.Protected`, reemplazá los guards por `<Redirect />` en `(app)/_layout.tsx` (un solo lugar).

---

## 3. Componentes reutilizables

Primitivas sin negocio en `shared/ui`, basadas en los tokens de [design-system.md](design-system.md#tokens). Se crean cuando hay consumidores; acá los usan login, perfil y ubicación.

```tsx
// src/shared/ui/AppText.tsx
import { Text, type TextProps } from 'react-native';
import { useTheme } from '@/shared/theme/useTheme';
import type { typography } from '@/shared/theme/tokens';

type Props = TextProps & {
  variant?: keyof typeof typography;
  tone?: 'default' | 'muted' | 'danger';
};

export function AppText({ variant = 'body', tone = 'default', style, ...props }: Props) {
  const { colors, typography } = useTheme();
  const color = tone === 'muted' ? colors.textMuted : tone === 'danger' ? colors.danger : colors.text;
  return <Text {...props} style={[typography[variant], { color }, style]} />;
}
```

```tsx
// src/shared/ui/Button.tsx
import { ActivityIndicator, Pressable, type PressableProps } from 'react-native';
import { useTheme } from '@/shared/theme/useTheme';
import { AppText } from './AppText';

type Props = Omit<PressableProps, 'children' | 'style'> & {
  label: string;
  variant?: 'primary' | 'secondary';
  loading?: boolean;
};

export function Button({ label, variant = 'primary', loading = false, disabled, ...props }: Props) {
  const { colors, spacing, radius } = useTheme();
  const isPrimary = variant === 'primary';
  const isDisabled = disabled || loading;

  return (
    <Pressable
      accessibilityRole="button"
      accessibilityState={{ disabled: isDisabled, busy: loading }}
      disabled={isDisabled}
      android_ripple={{ color: colors.border }}
      style={({ pressed }) => ({
        minHeight: 48,
        paddingHorizontal: spacing.lg,
        borderRadius: radius.md,
        alignItems: 'center',
        justifyContent: 'center',
        backgroundColor: isPrimary ? colors.primary : 'transparent',
        borderWidth: isPrimary ? 0 : 1,
        borderColor: colors.border,
        opacity: isDisabled ? 0.5 : pressed ? 0.85 : 1,
      })}
      {...props}
    >
      {loading ? (
        <ActivityIndicator color={isPrimary ? colors.onPrimary : colors.primary} />
      ) : (
        <AppText variant="label" style={{ color: isPrimary ? colors.onPrimary : colors.primary }}>
          {label}
        </AppText>
      )}
    </Pressable>
  );
}
```

```tsx
// src/shared/ui/TextField.tsx
import type { ComponentRef, Ref } from 'react';
import { TextInput, View, type TextInputProps } from 'react-native';
import { useTheme } from '@/shared/theme/useTheme';
import { AppText } from './AppText';

type Props = TextInputProps & {
  label: string;
  error?: string;
  ref?: Ref<ComponentRef<typeof TextInput>>;
};

export function TextField({ label, error, style, ...props }: Props) {
  const { colors, spacing, radius, typography } = useTheme();
  return (
    <View style={{ gap: spacing.xs }}>
      <AppText variant="label">{label}</AppText>
      <TextInput
        accessibilityLabel={label}
        placeholderTextColor={colors.textMuted}
        style={[
          typography.body,
          {
            minHeight: 48,
            paddingHorizontal: spacing.md,
            borderRadius: radius.md,
            borderWidth: 1,
            borderColor: error ? colors.danger : colors.border,
            color: colors.text,
            backgroundColor: colors.surface,
          },
          style,
        ]}
        {...props}
      />
      {error ? (
        <AppText variant="caption" tone="danger" accessibilityLiveRegion="polite">
          {error}
        </AppText>
      ) : null}
    </View>
  );
}
```

```tsx
// src/shared/ui/Screen.tsx
import type { ReactNode } from 'react';
import { ScrollView } from 'react-native';
import { SafeAreaView, type Edge } from 'react-native-safe-area-context';
import { useTheme } from '@/shared/theme/useTheme';

type Props = {
  children: ReactNode;
  /** Bordes con inset; las pantallas con header nativo no necesitan 'top'. */
  edges?: Edge[];
};

export function Screen({ children, edges = ['bottom'] }: Props) {
  const { colors, spacing } = useTheme();
  return (
    <SafeAreaView edges={edges} style={{ flex: 1, backgroundColor: colors.background }}>
      <ScrollView
        keyboardShouldPersistTaps="handled"
        contentContainerStyle={{ flexGrow: 1, padding: spacing.lg, gap: spacing.lg }}
      >
        {children}
      </ScrollView>
    </SafeAreaView>
  );
}
```

`StateView` cubre carga, vacío y error con un solo contrato en lugar de tres componentes casi iguales:

```tsx
// src/shared/ui/StateView.tsx
import { ActivityIndicator, View } from 'react-native';
import { useTheme } from '@/shared/theme/useTheme';
import { AppText } from './AppText';
import { Button } from './Button';

type Props = {
  title: string;
  description?: string;
  loading?: boolean;
  action?: { label: string; onPress: () => void };
  secondaryAction?: { label: string; onPress: () => void };
};

export function StateView({ title, description, loading = false, action, secondaryAction }: Props) {
  const { colors, spacing } = useTheme();
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'stretch', gap: spacing.md }}>
      {loading ? <ActivityIndicator color={colors.primary} accessibilityLabel={title} /> : null}
      <AppText variant="heading" accessibilityRole="header" style={{ textAlign: 'center' }}>
        {title}
      </AppText>
      {description ? (
        <AppText tone="muted" style={{ textAlign: 'center' }}>
          {description}
        </AppText>
      ) : null}
      {action ? <Button label={action.label} onPress={action.onPress} /> : null}
      {secondaryAction ? (
        <Button variant="secondary" label={secondaryAction.label} onPress={secondaryAction.onPress} />
      ) : null}
    </View>
  );
}
```

---

## 4. Integración REST: perfil

Funcionalidad simple: un service que valida y mapea el DTO, un hook con la query y una pantalla. Sin repositorio, sin interface, sin caso de uso.

```ts
// src/domains/profile/get-profile/services/getProfile.ts
import { z } from 'zod';
import { request } from '@/shared/api/httpClient';

const profileDtoSchema = z.object({
  id: z.string(),
  full_name: z.string(),
  email: z.email(),
  avatar_url: z.url().nullable(),
});

export type Profile = {
  id: string;
  fullName: string;
  email: string;
  avatarUrl: string | null;
};

export async function getProfile({ signal }: { signal?: AbortSignal } = {}): Promise<Profile> {
  const dto = await request('/me/profile', profileDtoSchema, { signal });
  return { id: dto.id, fullName: dto.full_name, email: dto.email, avatarUrl: dto.avatar_url };
}
```

```ts
// src/domains/profile/get-profile/hooks/useProfile.ts
import { useQuery } from '@tanstack/react-query';
import { getProfile } from '../services/getProfile';

export const profileKeys = { me: ['profile', 'me'] as const };

export function useProfile() {
  return useQuery({
    queryKey: profileKeys.me,
    queryFn: ({ signal }) => getProfile({ signal }),
    staleTime: 5 * 60_000,
  });
}
```

El hook aporta la key, la política de cache y el contrato para la pantalla; no es un reenvío vacío.

---

## 5. Creación de una pantalla con manejo de estados

```tsx
// src/domains/profile/get-profile/screens/ProfileScreen.tsx
import { Link } from 'expo-router';
import { useState } from 'react';
import { View } from 'react-native';
import { signOut } from '@/domains/auth/session/session.service';
import { errorCodeOf } from '@/shared/errors/AppError';
import { AppText } from '@/shared/ui/AppText';
import { Button } from '@/shared/ui/Button';
import { Screen } from '@/shared/ui/Screen';
import { StateView } from '@/shared/ui/StateView';
import { useProfile } from '../hooks/useProfile';

const ERROR_MESSAGES: Partial<Record<ReturnType<typeof errorCodeOf>, string>> = {
  NETWORK: 'Revisá tu conexión e intentá de nuevo.',
  TIMEOUT: 'El servidor tardó demasiado. Intentá de nuevo.',
  SERVER: 'Tenemos un problema en el servicio. Intentá en unos minutos.',
};

export function ProfileScreen() {
  const profile = useProfile();
  const [signingOut, setSigningOut] = useState(false);

  const handleSignOut = async () => {
    setSigningOut(true);
    await signOut({ revoke: true }); // el guard de navegación lleva al login
  };

  if (profile.isPending) {
    return (
      <Screen>
        <StateView loading title="Cargando tu perfil" />
      </Screen>
    );
  }

  if (profile.isError) {
    return (
      <Screen>
        <StateView
          title="No pudimos cargar tu perfil"
          description={ERROR_MESSAGES[errorCodeOf(profile.error)] ?? 'Ocurrió un error inesperado.'}
          action={{ label: 'Reintentar', onPress: () => void profile.refetch() }}
        />
      </Screen>
    );
  }

  return (
    <Screen>
      <View style={{ gap: 4 }}>
        <AppText variant="title" accessibilityRole="header">
          {profile.data.fullName}
        </AppText>
        <AppText tone="muted">{profile.data.email}</AppText>
      </View>
      <Link href="/location" asChild>
        <Button label="Ver mi ubicación" />
      </Link>
      <Button variant="secondary" label="Cerrar sesión" loading={signingOut} onPress={handleSignOut} />
    </Screen>
  );
}
```

Decisiones:

- Estados derivados de la query: `isPending` → carga, `isError` → error con mensaje por código, éxito. No hay estado vacío porque el perfil siempre existe para un usuario autenticado.
- `UNAUTHORIZED` no tiene mensaje: la sesión expirada la resuelve el guard.
- Cerrar sesión llama a la API pública del dominio `auth` (`signOut`). No se creó `useLogout`: sería un hook que solo reenvía una llamada.

---

## 6. Autenticación y sesión

```text
src/domains/auth/
├── session/
│   ├── session.schema.ts
│   ├── session.persistence.ts
│   ├── session.store.ts
│   ├── session.service.ts
│   ├── configureAuthHttp.ts
│   └── useSession.ts
└── login/
    ├── schemas/login.schema.ts
    ├── services/login.ts
    ├── hooks/useLogin.ts
    ├── components/LoginForm.tsx
    └── screens/LoginScreen.tsx
```

```ts
// src/domains/auth/session/session.schema.ts
import { z } from 'zod';

export const sessionUserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.email(),
});
export type SessionUser = z.infer<typeof sessionUserSchema>;

export type Tokens = { accessToken: string; refreshToken: string };

export const tokensDtoSchema = z
  .object({ access_token: z.string().min(1), refresh_token: z.string().min(1) })
  .transform((dto): Tokens => ({ accessToken: dto.access_token, refreshToken: dto.refresh_token }));
```

Tokens y usuario en SecureStore. Las claves pertenecen al dominio `auth`.

```ts
// src/domains/auth/session/session.persistence.ts
import { secureStorage } from '@/shared/storage/secureStorage';
import { sessionUserSchema, type SessionUser, type Tokens } from './session.schema';

const KEYS = {
  accessToken: 'auth.accessToken',
  refreshToken: 'auth.refreshToken',
  user: 'auth.user',
} as const;

function parseUser(raw: string): SessionUser | null {
  try {
    const result = sessionUserSchema.safeParse(JSON.parse(raw));
    return result.success ? result.data : null;
  } catch {
    return null;
  }
}

export const sessionPersistence = {
  async load(): Promise<{ tokens: Tokens; user: SessionUser } | null> {
    const [accessToken, refreshToken, rawUser] = await Promise.all([
      secureStorage.get(KEYS.accessToken),
      secureStorage.get(KEYS.refreshToken),
      secureStorage.get(KEYS.user),
    ]);
    if (!accessToken || !refreshToken || !rawUser) return null;
    const user = parseUser(rawUser);
    return user ? { tokens: { accessToken, refreshToken }, user } : null;
  },

  async saveTokens(tokens: Tokens) {
    await Promise.all([
      secureStorage.set(KEYS.accessToken, tokens.accessToken),
      secureStorage.set(KEYS.refreshToken, tokens.refreshToken),
    ]);
  },

  async save(tokens: Tokens, user: SessionUser) {
    await Promise.all([this.saveTokens(tokens), secureStorage.set(KEYS.user, JSON.stringify(user))]);
  },

  async clear() {
    await Promise.all(Object.values(KEYS).map((key) => secureStorage.remove(key)));
  },
};
```

El store solo guarda el estado público de la sesión. Los tokens quedan en el módulo de servicio y en SecureStore.

```ts
// src/domains/auth/session/session.store.ts
import { create } from 'zustand';
import type { SessionUser } from './session.schema';

export type Session =
  | { status: 'restoring' }
  | { status: 'anonymous'; reason?: 'expired' }
  | { status: 'authenticated'; user: SessionUser };

export const useSessionStore = create<{ session: Session }>()(() => ({
  session: { status: 'restoring' },
}));

export const setSession = (session: Session) => useSessionStore.setState({ session });
```

```ts
// src/domains/auth/session/session.service.ts
import { z } from 'zod';
import { request } from '@/shared/api/httpClient';
import { queryClient } from '@/shared/api/queryClient';
import { AppError, errorCodeOf } from '@/shared/errors/AppError';
import { logger } from '@/shared/observability/logger';
import { sessionPersistence } from './session.persistence';
import { tokensDtoSchema, type SessionUser, type Tokens } from './session.schema';
import { setSession } from './session.store';

let tokens: Tokens | null = null;
let refreshInFlight: Promise<boolean> | null = null;

export const getAccessToken = () => tokens?.accessToken ?? null;

export async function restoreSession() {
  try {
    const saved = await sessionPersistence.load();
    if (!saved) {
      setSession({ status: 'anonymous' });
      return;
    }
    tokens = saved.tokens;
    // Se confía en la sesión guardada para abrir offline; el primer request la valida.
    setSession({ status: 'authenticated', user: saved.user });
  } catch (error) {
    logger.error(error, { event: 'session_restore_failed' });
    setSession({ status: 'anonymous' });
  }
}

export async function signIn(nextTokens: Tokens, user: SessionUser) {
  await sessionPersistence.save(nextTokens, user);
  tokens = nextTokens;
  setSession({ status: 'authenticated', user });
}

export async function signOut({ revoke = false, reason }: { revoke?: boolean; reason?: 'expired' } = {}) {
  const refreshToken = tokens?.refreshToken;
  if (revoke && refreshToken) {
    try {
      await request('/auth/logout', z.unknown(), {
        method: 'POST',
        body: { refresh_token: refreshToken },
        auth: false,
      });
    } catch (error) {
      // Best effort: el logout local continúa aunque falle la revocación remota.
      logger.warn('logout_revoke_failed', { code: errorCodeOf(error) });
    }
  }
  tokens = null;
  await sessionPersistence.clear();
  queryClient.clear();
  setSession({ status: 'anonymous', reason });
}

/** Un único refresh compartido entre requests concurrentes. */
export function refreshAccessToken(): Promise<boolean> {
  refreshInFlight ??= performRefresh().finally(() => {
    refreshInFlight = null;
  });
  return refreshInFlight;
}

async function performRefresh(): Promise<boolean> {
  const refreshToken = tokens?.refreshToken;
  if (!refreshToken) return false;
  try {
    const next = await request('/auth/refresh', tokensDtoSchema, {
      method: 'POST',
      body: { refresh_token: refreshToken },
      auth: false,
    });
    tokens = next;
    await sessionPersistence.saveTokens(next);
    return true;
  } catch (error) {
    // Refresh rechazado → sesión inválida. Error de red → se propaga sin cerrar sesión.
    if (error instanceof AppError && (error.code === 'UNAUTHORIZED' || error.code === 'VALIDATION')) return false;
    throw error;
  }
}
```

```ts
// src/domains/auth/session/configureAuthHttp.ts
import { configureHttpClient } from '@/shared/api/httpClient';
import { getAccessToken, refreshAccessToken, signOut } from './session.service';
import { useSessionStore } from './session.store';

configureHttpClient({
  getAccessToken,
  refreshAccessToken,
  onUnauthorized: () => {
    if (useSessionStore.getState().session.status === 'authenticated') {
      void signOut({ reason: 'expired' });
    }
  },
});
```

```ts
// src/domains/auth/session/useSession.ts
import { useSessionStore } from './session.store';

/** API pública del dominio auth para otros dominios y para app/. */
export const useSession = () => useSessionStore((state) => state.session);
```

`useSession` existe para que otros dominios no dependan de la forma interna del store.

### Login

```ts
// src/domains/auth/login/schemas/login.schema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.email('Ingresá un email válido'),
  password: z.string().min(8, 'La contraseña tiene al menos 8 caracteres'),
});

export type LoginInput = z.infer<typeof loginSchema>;
```

```ts
// src/domains/auth/login/services/login.ts
import { z } from 'zod';
import { request } from '@/shared/api/httpClient';
import { sessionUserSchema, tokensDtoSchema } from '../../session/session.schema';
import { signIn } from '../../session/session.service';
import type { LoginInput } from '../schemas/login.schema';

const loginResponseSchema = z.object({
  tokens: tokensDtoSchema,
  user: sessionUserSchema,
});

export async function login(credentials: LoginInput) {
  const result = await request('/auth/login', loginResponseSchema, {
    method: 'POST',
    body: credentials,
    auth: false,
  });
  await signIn(result.tokens, result.user);
}
```

El hook agrega valor: traduce errores a mensajes de UI.

```ts
// src/domains/auth/login/hooks/useLogin.ts
import { useMutation } from '@tanstack/react-query';
import { errorCodeOf } from '@/shared/errors/AppError';
import type { LoginInput } from '../schemas/login.schema';
import { login } from '../services/login';

function messageFor(error: unknown): string {
  switch (errorCodeOf(error)) {
    case 'UNAUTHORIZED':
    case 'VALIDATION':
      return 'Email o contraseña incorrectos.';
    case 'NETWORK':
    case 'TIMEOUT':
      return 'No pudimos conectarnos. Revisá tu conexión.';
    default:
      return 'No pudimos iniciar sesión. Intentá de nuevo.';
  }
}

export function useLogin() {
  const mutation = useMutation({ mutationFn: (input: LoginInput) => login(input) });
  return {
    submit: mutation.mutateAsync,
    isPending: mutation.isPending,
    errorMessage: mutation.isError ? messageFor(mutation.error) : null,
  };
}
```

```tsx
// src/domains/auth/login/components/LoginForm.tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { Controller, useForm } from 'react-hook-form';
import { View } from 'react-native';
import { AppText } from '@/shared/ui/AppText';
import { Button } from '@/shared/ui/Button';
import { TextField } from '@/shared/ui/TextField';
import { useLogin } from '../hooks/useLogin';
import { loginSchema, type LoginInput } from '../schemas/login.schema';

export function LoginForm() {
  const { submit, isPending, errorMessage } = useLogin();
  const { control, handleSubmit, setFocus, formState } = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  const onSubmit = handleSubmit(async (values) => {
    try {
      await submit(values);
    } catch {
      // El mensaje se muestra desde useLogin; los valores del formulario se conservan.
    }
  });

  return (
    <View style={{ gap: 16 }}>
      <Controller
        control={control}
        name="email"
        render={({ field }) => (
          <TextField
            ref={field.ref}
            label="Email"
            value={field.value}
            onChangeText={field.onChange}
            onBlur={field.onBlur}
            error={formState.errors.email?.message}
            keyboardType="email-address"
            autoCapitalize="none"
            autoComplete="email"
            textContentType="emailAddress"
            returnKeyType="next"
            onSubmitEditing={() => setFocus('password')}
          />
        )}
      />
      <Controller
        control={control}
        name="password"
        render={({ field }) => (
          <TextField
            ref={field.ref}
            label="Contraseña"
            value={field.value}
            onChangeText={field.onChange}
            onBlur={field.onBlur}
            error={formState.errors.password?.message}
            secureTextEntry
            autoComplete="password"
            textContentType="password"
            returnKeyType="go"
            onSubmitEditing={() => void onSubmit()}
          />
        )}
      />
      {errorMessage ? (
        <AppText tone="danger" accessibilityLiveRegion="polite">
          {errorMessage}
        </AppText>
      ) : null}
      <Button label="Ingresar" loading={isPending} onPress={() => void onSubmit()} />
    </View>
  );
}
```

```tsx
// src/domains/auth/login/screens/LoginScreen.tsx
import { KeyboardAvoidingView, Platform } from 'react-native';
import { useSession } from '../../session/useSession';
import { AppText } from '@/shared/ui/AppText';
import { Screen } from '@/shared/ui/Screen';
import { LoginForm } from '../components/LoginForm';

export function LoginScreen() {
  const session = useSession();
  const expired = session.status === 'anonymous' && session.reason === 'expired';

  return (
    <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
      <Screen edges={['top', 'bottom']}>
        <AppText variant="title" accessibilityRole="header">
          Iniciá sesión
        </AppText>
        {expired ? <AppText tone="muted">Tu sesión expiró. Ingresá de nuevo.</AppText> : null}
        <LoginForm />
      </Screen>
    </KeyboardAvoidingView>
  );
}
```

Puntos clave: el login no navega (el guard lo hace), los tokens nunca llegan a la UI, `UNAUTHORIZED` del login no dispara refresh porque el request es `auth: false`.

---

## 7. Geolocalización: adaptador nativo

Contrato propio en `shared/native/location`. Ningún tipo de `expo-location` sale de este módulo.

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

export type LocationErrorCode = 'PERMISSION_DENIED' | 'SERVICES_DISABLED' | 'TIMEOUT' | 'UNAVAILABLE' | 'UNKNOWN';

export class LocationError extends Error {
  readonly code: LocationErrorCode;

  constructor(code: LocationErrorCode) {
    super(code);
    this.name = 'LocationError';
    this.code = code;
  }
}
```

```ts
// src/shared/native/location/location.adapter.ts
import { expoLocationAdapter } from './expo-location.adapter';
import type { LocationAccuracyLevel, LocationFix, LocationPermission } from './location.types';

export interface LocationAdapter {
  getPermission(): Promise<LocationPermission>;
  requestPermission(): Promise<LocationPermission>;
  areServicesEnabled(): Promise<boolean>;
  getLastKnownPosition(options: { maxAgeMs: number; requiredAccuracyMeters: number }): Promise<LocationFix | null>;
  getCurrentPosition(options: { accuracy: LocationAccuracyLevel; timeoutMs: number }): Promise<LocationFix>;
  watchPosition(
    options: { accuracy: LocationAccuracyLevel; distanceIntervalMeters: number },
    onFix: (fix: LocationFix) => void,
    onError: (error: Error) => void,
  ): Promise<() => void>;
}

/** Implementación por defecto. Sustituible en tests o por otro proveedor sin tocar el dominio. */
export const locationAdapter: LocationAdapter = expoLocationAdapter;
```

```ts
// src/shared/native/location/expo-location.adapter.ts
import * as Location from 'expo-location';
import type { LocationAdapter } from './location.adapter';
import {
  LocationError,
  type LocationAccuracyLevel,
  type LocationFix,
  type LocationPermission,
} from './location.types';

const ACCURACY: Record<LocationAccuracyLevel, Location.LocationAccuracy> = {
  low: Location.Accuracy.Low,
  balanced: Location.Accuracy.Balanced,
  high: Location.Accuracy.High,
  navigation: Location.Accuracy.BestForNavigation,
};

export function toLocationFix(location: Location.LocationObject): LocationFix {
  return {
    latitude: location.coords.latitude,
    longitude: location.coords.longitude,
    accuracyMeters: location.coords.accuracy ?? null,
    timestamp: location.timestamp,
  };
}

function toPermission(response: Location.LocationPermissionResponse): LocationPermission {
  if (response.granted) {
    const reduced = response.ios?.accuracy === 'reduced' || response.android?.accuracy === 'coarse';
    return { status: 'granted', precision: reduced ? 'approximate' : 'precise' };
  }
  if (response.status === Location.PermissionStatus.UNDETERMINED) return { status: 'undetermined' };
  return { status: 'denied', canAskAgain: response.canAskAgain };
}

/**
 * Códigos observados en expo-location 58: ERR_LOCATION_UNAUTHORIZED, ERR_LOCATION_SETTINGS_UNSATISFIED (Android),
 * ERR_LOCATION_UNAVAILABLE y ERR_CURRENT_LOCATION_IS_UNAVAILABLE. Revisalos al actualizar la librería.
 */
function toLocationError(error: unknown): LocationError {
  if (error instanceof LocationError) return error;
  const code = typeof error === 'object' && error !== null && 'code' in error ? String(error.code) : '';
  if (code.includes('SETTINGS_UNSATISFIED')) return new LocationError('SERVICES_DISABLED');
  if (code.includes('UNAUTHORIZED')) return new LocationError('PERMISSION_DENIED');
  if (code.includes('UNAVAILABLE')) return new LocationError('UNAVAILABLE');
  return new LocationError('UNKNOWN');
}

function withTimeout<T>(promise: Promise<T>, timeoutMs: number): Promise<T> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new LocationError('TIMEOUT')), timeoutMs);
    promise.then(
      (value) => {
        clearTimeout(timer);
        resolve(value);
      },
      (error: unknown) => {
        clearTimeout(timer);
        reject(error);
      },
    );
  });
}

export const expoLocationAdapter: LocationAdapter = {
  async getPermission() {
    return toPermission(await Location.getForegroundPermissionsAsync());
  },

  async requestPermission() {
    return toPermission(await Location.requestForegroundPermissionsAsync());
  },

  areServicesEnabled() {
    return Location.hasServicesEnabledAsync();
  },

  async getLastKnownPosition({ maxAgeMs, requiredAccuracyMeters }) {
    const location = await Location.getLastKnownPositionAsync({
      maxAge: maxAgeMs,
      requiredAccuracy: requiredAccuracyMeters,
    });
    return location ? toLocationFix(location) : null;
  },

  async getCurrentPosition({ accuracy, timeoutMs }) {
    try {
      // expo-location no expone timeout: si vence, la request nativa sigue y su resultado se descarta.
      const location = await withTimeout(Location.getCurrentPositionAsync({ accuracy: ACCURACY[accuracy] }), timeoutMs);
      return toLocationFix(location);
    } catch (error) {
      throw toLocationError(error);
    }
  },

  async watchPosition({ accuracy, distanceIntervalMeters }, onFix, onError) {
    const subscription = await Location.watchPositionAsync(
      { accuracy: ACCURACY[accuracy], distanceInterval: distanceIntervalMeters },
      (location) => onFix(toLocationFix(location)),
      (reason) => onError(new LocationError(toLocationError({ code: reason }).code)),
    );
    return () => subscription.remove();
  },
};
```

Por qué un adaptador acá sí: es una frontera con el dispositivo con permisos, errores nativos y diferencias por plataforma; el dominio se testea con un adaptador falso y el proveedor puede cambiar.

---

## 8. Caso de uso: obtener la ubicación actual

Orquesta permiso → servicios → posición actual → fallback a la última conocida, y devuelve un resultado discriminado. No conoce React ni textos. Vive en `services/` porque es la única operación de la funcionalidad; si la funcionalidad sumara varias operaciones orquestadas con múltiples disparadores, se separaría en `use-cases/` ([architecture.md](architecture.md#cuándo-crear-cada-abstracción)).

```ts
// src/domains/location/get-current-location/services/getCurrentLocation.ts
import { locationAdapter, type LocationAdapter } from '@/shared/native/location/location.adapter';
import { LocationError, type LocationAccuracyLevel, type LocationFix } from '@/shared/native/location/location.types';

export type CurrentLocationResult =
  | { status: 'success'; location: LocationFix; precision: 'precise' | 'approximate'; isFallback: boolean }
  | { status: 'permission-required' }
  | { status: 'permission-denied'; canAskAgain: boolean }
  | { status: 'services-disabled' }
  | { status: 'error'; code: 'TIMEOUT' | 'UNAVAILABLE' | 'UNKNOWN' };

type Options = { accuracy?: LocationAccuracyLevel; timeoutMs?: number };

/** Última posición conocida aceptable como fallback: hasta 2 minutos y 100 metros. */
const FALLBACK = { maxAgeMs: 2 * 60_000, requiredAccuracyMeters: 100 };

export async function getCurrentLocation(
  { accuracy = 'balanced', timeoutMs = 15_000 }: Options = {},
  adapter: LocationAdapter = locationAdapter,
): Promise<CurrentLocationResult> {
  const permission = await adapter.getPermission();
  if (permission.status === 'undetermined') return { status: 'permission-required' };
  if (permission.status === 'denied') return { status: 'permission-denied', canAskAgain: permission.canAskAgain };

  if (!(await adapter.areServicesEnabled())) return { status: 'services-disabled' };

  try {
    const location = await adapter.getCurrentPosition({ accuracy, timeoutMs });
    return { status: 'success', location, precision: permission.precision, isFallback: false };
  } catch (error) {
    const code = error instanceof LocationError ? error.code : 'UNKNOWN';
    if (code === 'SERVICES_DISABLED') return { status: 'services-disabled' };
    if (code === 'PERMISSION_DENIED') return { status: 'permission-denied', canAskAgain: false };

    const lastKnown = await adapter.getLastKnownPosition(FALLBACK).catch(() => null);
    if (lastKnown) return { status: 'success', location: lastKnown, precision: permission.precision, isFallback: true };

    return { status: 'error', code: code === 'TIMEOUT' || code === 'UNAVAILABLE' ? code : 'UNKNOWN' };
  }
}
```

---

## 9. Hook personalizado y solicitud de permisos

El hook coordina la presentación: estado, solicitud de permiso por acción del usuario, apertura de Configuración, re-chequeo al volver a la app y descarte de resultados obsoletos.

```ts
// src/domains/location/get-current-location/hooks/useCurrentLocation.ts
import { useCallback, useEffect, useRef, useState } from 'react';
import { AppState, Linking } from 'react-native';
import { locationAdapter, type LocationAdapter } from '@/shared/native/location/location.adapter';
import { getCurrentLocation, type CurrentLocationResult } from '../services/getCurrentLocation';

export type CurrentLocationState = { status: 'loading' } | CurrentLocationResult;

const RECHECK_ON_RETURN: CurrentLocationState['status'][] = ['permission-denied', 'services-disabled'];

export function useCurrentLocation(adapter: LocationAdapter = locationAdapter) {
  const [state, setState] = useState<CurrentLocationState>({ status: 'loading' });
  const requestId = useRef(0);
  const stateRef = useRef(state);
  stateRef.current = state;

  const load = useCallback(async () => {
    const id = ++requestId.current;
    setState({ status: 'loading' });
    const result = await getCurrentLocation({}, adapter);
    if (id === requestId.current) setState(result); // ignora respuestas obsoletas
  }, [adapter]);

  const requestPermission = useCallback(async () => {
    const permission = await adapter.requestPermission();
    if (permission.status === 'granted') {
      await load();
    } else if (permission.status === 'denied') {
      setState({ status: 'permission-denied', canAskAgain: permission.canAskAgain });
    }
  }, [adapter, load]);

  const openSettings = useCallback(() => {
    void Linking.openSettings();
  }, []);

  useEffect(() => {
    void load();
    return () => {
      requestId.current += 1; // invalida resultados pendientes al desmontar
    };
  }, [load]);

  // El usuario puede cambiar permisos o activar el GPS desde Configuración.
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (next) => {
      if (next === 'active' && RECHECK_ON_RETURN.includes(stateRef.current.status)) void load();
    });
    return () => subscription.remove();
  }, [load]);

  return { state, retry: load, requestPermission, openSettings };
}
```

El componente de permisos solo muestra contexto y acciones; no conoce la API de ubicación.

```tsx
// src/domains/location/get-current-location/components/LocationPermissionState.tsx
import { StateView } from '@/shared/ui/StateView';

type Props =
  | { reason: 'permission-required'; onRequest: () => void }
  | { reason: 'permission-denied'; canAskAgain: boolean; onRequest: () => void; onOpenSettings: () => void }
  | { reason: 'services-disabled'; onRetry: () => void; onOpenSettings: () => void };

const PURPOSE = 'Usamos tu ubicación solo para mostrarte dónde estás. No la guardamos.';

export function LocationPermissionState(props: Props) {
  switch (props.reason) {
    case 'permission-required':
      return (
        <StateView
          title="¿Compartís tu ubicación?"
          description={PURPOSE}
          action={{ label: 'Permitir ubicación', onPress: props.onRequest }}
        />
      );
    case 'permission-denied':
      return props.canAskAgain ? (
        <StateView
          title="Necesitamos acceso a tu ubicación"
          description={PURPOSE}
          action={{ label: 'Intentar de nuevo', onPress: props.onRequest }}
        />
      ) : (
        <StateView
          title="Necesitamos acceso a tu ubicación"
          description="Activá el permiso de ubicación para esta app en Configuración."
          action={{ label: 'Abrir configuración', onPress: props.onOpenSettings }}
        />
      );
    case 'services-disabled':
      return (
        <StateView
          title="La ubicación del teléfono está desactivada"
          description="Activá los servicios de ubicación y volvé a intentar."
          action={{ label: 'Reintentar', onPress: props.onRetry }}
          secondaryAction={{ label: 'Abrir configuración', onPress: props.onOpenSettings }}
        />
      );
  }
}
```

`Linking.openSettings()` abre la configuración de la app; en Android los servicios de ubicación del sistema están en otra pantalla, por eso el usuario vuelve y reintenta (o se ofrece `enableNetworkProviderAsync` desde el adaptador si el producto lo necesita).

---

## 10. Pantalla de ubicación

```tsx
// src/domains/location/get-current-location/screens/CurrentLocationScreen.tsx
import { View } from 'react-native';
import { AppText } from '@/shared/ui/AppText';
import { Button } from '@/shared/ui/Button';
import { Screen } from '@/shared/ui/Screen';
import { StateView } from '@/shared/ui/StateView';
import type { LocationFix } from '@/shared/native/location/location.types';
import { LocationPermissionState } from '../components/LocationPermissionState';
import { useCurrentLocation } from '../hooks/useCurrentLocation';

const ERROR_TEXT = {
  TIMEOUT: 'No pudimos obtener tu ubicación a tiempo. Probá en un lugar abierto.',
  UNAVAILABLE: 'Tu ubicación no está disponible en este momento.',
  UNKNOWN: 'Ocurrió un error al obtener tu ubicación.',
} as const;

export function CurrentLocationScreen() {
  const { state, retry, requestPermission, openSettings } = useCurrentLocation();

  return <Screen>{renderState()}</Screen>;

  function renderState() {
    switch (state.status) {
      case 'loading':
        return <StateView loading title="Buscando tu ubicación" />;
      case 'permission-required':
        return <LocationPermissionState reason="permission-required" onRequest={requestPermission} />;
      case 'permission-denied':
        return (
          <LocationPermissionState
            reason="permission-denied"
            canAskAgain={state.canAskAgain}
            onRequest={requestPermission}
            onOpenSettings={openSettings}
          />
        );
      case 'services-disabled':
        return <LocationPermissionState reason="services-disabled" onRetry={retry} onOpenSettings={openSettings} />;
      case 'error':
        return (
          <StateView
            title="Sin ubicación"
            description={ERROR_TEXT[state.code]}
            action={{ label: 'Reintentar', onPress: retry }}
          />
        );
      case 'success':
        return (
          <>
            <LocationSummary
              location={state.location}
              approximate={state.precision === 'approximate'}
              isFallback={state.isFallback}
            />
            <Button variant="secondary" label="Actualizar" onPress={retry} />
          </>
        );
    }
  }
}

function LocationSummary({
  location,
  approximate,
  isFallback,
}: {
  location: LocationFix;
  approximate: boolean;
  isFallback: boolean;
}) {
  const accuracy = location.accuracyMeters === null ? null : Math.round(location.accuracyMeters);
  return (
    <View accessible style={{ gap: 8 }}>
      <AppText variant="heading" accessibilityRole="header">
        Tu ubicación
      </AppText>
      <AppText>
        {location.latitude.toFixed(5)}, {location.longitude.toFixed(5)}
      </AppText>
      {accuracy !== null ? <AppText tone="muted">Precisión aproximada: {accuracy} m</AppText> : null}
      {approximate ? (
        <AppText tone="muted">Compartiste una ubicación aproximada. Podés activar la precisa en Configuración.</AppText>
      ) : null}
      {isFallback ? <AppText tone="muted">Mostramos tu última ubicación conocida.</AppText> : null}
    </View>
  );
}
```

`app.config.ts` necesita el texto de permiso de iOS (ver [geolocation.md](geolocation.md#permisos)). Ubicación en primer plano funciona en Expo Go; validá igualmente en dispositivo real.

---

## 11. Testing

Caso de uso con un adaptador falso: sin mocks del SO.

```ts
// src/domains/location/get-current-location/tests/getCurrentLocation.test.ts
import type { LocationAdapter } from '@/shared/native/location/location.adapter';
import { LocationError, type LocationFix } from '@/shared/native/location/location.types';
import { getCurrentLocation } from '../services/getCurrentLocation';

const fix: LocationFix = { latitude: -34.6037, longitude: -58.3816, accuracyMeters: 12, timestamp: 1_700_000_000_000 };

function createAdapter(overrides: Partial<LocationAdapter> = {}): LocationAdapter {
  return {
    getPermission: jest.fn().mockResolvedValue({ status: 'granted', precision: 'precise' }),
    requestPermission: jest.fn().mockResolvedValue({ status: 'granted', precision: 'precise' }),
    areServicesEnabled: jest.fn().mockResolvedValue(true),
    getLastKnownPosition: jest.fn().mockResolvedValue(null),
    getCurrentPosition: jest.fn().mockResolvedValue(fix),
    watchPosition: jest.fn().mockResolvedValue(() => {}),
    ...overrides,
  };
}

describe('getCurrentLocation', () => {
  it('pide contexto antes de solicitar el permiso si nunca se pidió', async () => {
    const adapter = createAdapter({ getPermission: jest.fn().mockResolvedValue({ status: 'undetermined' }) });

    await expect(getCurrentLocation({}, adapter)).resolves.toEqual({ status: 'permission-required' });
    expect(adapter.requestPermission).not.toHaveBeenCalled();
    expect(adapter.getCurrentPosition).not.toHaveBeenCalled();
  });

  it('informa permiso denegado permanentemente', async () => {
    const adapter = createAdapter({
      getPermission: jest.fn().mockResolvedValue({ status: 'denied', canAskAgain: false }),
    });

    await expect(getCurrentLocation({}, adapter)).resolves.toEqual({ status: 'permission-denied', canAskAgain: false });
  });

  it('distingue servicios de ubicación desactivados', async () => {
    const adapter = createAdapter({ areServicesEnabled: jest.fn().mockResolvedValue(false) });

    await expect(getCurrentLocation({}, adapter)).resolves.toEqual({ status: 'services-disabled' });
  });

  it('devuelve la posición actual con la precisión concedida', async () => {
    const adapter = createAdapter({
      getPermission: jest.fn().mockResolvedValue({ status: 'granted', precision: 'approximate' }),
    });

    await expect(getCurrentLocation({ accuracy: 'high' }, adapter)).resolves.toEqual({
      status: 'success',
      location: fix,
      precision: 'approximate',
      isFallback: false,
    });
    expect(adapter.getCurrentPosition).toHaveBeenCalledWith({ accuracy: 'high', timeoutMs: 15_000 });
  });

  it('usa la última ubicación conocida si la actual vence', async () => {
    const adapter = createAdapter({
      getCurrentPosition: jest.fn().mockRejectedValue(new LocationError('TIMEOUT')),
      getLastKnownPosition: jest.fn().mockResolvedValue(fix),
    });

    await expect(getCurrentLocation({}, adapter)).resolves.toMatchObject({ status: 'success', isFallback: true });
  });

  it('informa timeout si no hay ubicación de respaldo', async () => {
    const adapter = createAdapter({
      getCurrentPosition: jest.fn().mockRejectedValue(new LocationError('TIMEOUT')),
    });

    await expect(getCurrentLocation({}, adapter)).resolves.toEqual({ status: 'error', code: 'TIMEOUT' });
  });
});
```

Componente: comportamiento por rol accesible. En React Native Testing Library 14 `render` y `userEvent` son asíncronos; en versiones anteriores `render` es síncrono: seguí la API de la versión instalada.

```tsx
// src/domains/location/get-current-location/tests/LocationPermissionState.test.tsx
import { render, screen, userEvent } from '@testing-library/react-native';
import { LocationPermissionState } from '../components/LocationPermissionState';

describe('LocationPermissionState', () => {
  it('ofrece abrir configuración cuando el permiso no puede volver a pedirse', async () => {
    const user = userEvent.setup();
    const onOpenSettings = jest.fn();
    await render(
      <LocationPermissionState
        reason="permission-denied"
        canAskAgain={false}
        onRequest={jest.fn()}
        onOpenSettings={onOpenSettings}
      />,
    );

    await user.press(screen.getByRole('button', { name: 'Abrir configuración' }));

    expect(onOpenSettings).toHaveBeenCalledTimes(1);
    expect(screen.queryByRole('button', { name: 'Intentar de nuevo' })).toBeNull();
  });

  it('explica el motivo antes de pedir el permiso', async () => {
    const user = userEvent.setup();
    const onRequest = jest.fn();
    await render(<LocationPermissionState reason="permission-required" onRequest={onRequest} />);

    expect(screen.getByText(/solo para mostrarte dónde estás/)).toBeOnTheScreen();
    await user.press(screen.getByRole('button', { name: 'Permitir ubicación' }));
    expect(onRequest).toHaveBeenCalledTimes(1);
  });
});
```

Cliente HTTP: errores normalizados y refresh.

```ts
// src/shared/api/httpClient.test.ts
import { z } from 'zod';
import { AppError } from '@/shared/errors/AppError';
import { configureHttpClient, request } from './httpClient';

jest.mock('@/shared/config/env', () => ({ env: { apiUrl: 'https://api.test' } }));

const fetchMock = jest.fn();
globalThis.fetch = fetchMock;

const jsonResponse = (status: number, body?: unknown) =>
  ({
    ok: status >= 200 && status < 300,
    status,
    text: async () => (body === undefined ? '' : JSON.stringify(body)),
  }) as Response;

const schema = z.object({ id: z.string() });

describe('request', () => {
  const hooks = {
    getAccessToken: jest.fn(() => 'access-1'),
    refreshAccessToken: jest.fn(async () => true),
    onUnauthorized: jest.fn(),
  };

  beforeEach(() => {
    jest.clearAllMocks();
    configureHttpClient(hooks);
  });

  it('valida la respuesta con el schema', async () => {
    fetchMock.mockResolvedValueOnce(jsonResponse(200, { id: 'a' }));
    await expect(request('/x', schema)).resolves.toEqual({ id: 'a' });
    expect(fetchMock).toHaveBeenCalledWith(
      'https://api.test/x',
      expect.objectContaining({ headers: expect.objectContaining({ Authorization: 'Bearer access-1' }) }),
    );
  });

  it('rechaza respuestas que no cumplen el contrato', async () => {
    fetchMock.mockResolvedValueOnce(jsonResponse(200, { id: 1 }));
    await expect(request('/x', schema)).rejects.toMatchObject({ code: 'INVALID_RESPONSE' });
  });

  it('mapea errores de servidor y de red', async () => {
    fetchMock.mockResolvedValueOnce(jsonResponse(503));
    await expect(request('/x', schema)).rejects.toMatchObject({ code: 'SERVER', status: 503 });

    fetchMock.mockRejectedValueOnce(new TypeError('Network request failed'));
    await expect(request('/x', schema)).rejects.toBeInstanceOf(AppError);
  });

  it('refresca el token una vez y reintenta ante 401', async () => {
    fetchMock.mockResolvedValueOnce(jsonResponse(401)).mockResolvedValueOnce(jsonResponse(200, { id: 'a' }));
    await expect(request('/x', schema)).resolves.toEqual({ id: 'a' });
    expect(hooks.refreshAccessToken).toHaveBeenCalledTimes(1);
    expect(hooks.onUnauthorized).not.toHaveBeenCalled();
  });

  it('notifica sesión expirada si el refresh falla', async () => {
    hooks.refreshAccessToken.mockResolvedValueOnce(false);
    fetchMock.mockResolvedValueOnce(jsonResponse(401));
    await expect(request('/x', schema)).rejects.toMatchObject({ code: 'UNAUTHORIZED' });
    expect(hooks.onUnauthorized).toHaveBeenCalledTimes(1);
  });
});
```

Configuración de Jest necesaria para el alias (además del preset):

```js
// jest.config.js
module.exports = {
  preset: 'jest-expo',
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
};
```

E2E de permiso denegado con Maestro: ver [testing.md](testing.md#e2e-con-maestro-ejemplo).

---

## 12. Refactor de un componente con demasiadas responsabilidades

### Antes

```tsx
// app/location.tsx — todo en la ruta
import * as Location from 'expo-location';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useEffect, useState } from 'react';
import { Text, TouchableOpacity, View } from 'react-native';

export default function LocationPage() {
  const [coords, setCoords] = useState<any>(null);
  const [error, setError] = useState('');

  useEffect(() => {
    (async () => {
      const { status } = await Location.requestForegroundPermissionsAsync(); // pide sin contexto
      if (status !== 'granted') return setError('Permission denied');       // sin distinguir canAskAgain
      const pos = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Highest });
      console.log('position', pos);                                          // coordenadas en logs
      await AsyncStorage.setItem('lastPosition', JSON.stringify(pos));       // persiste sin necesidad
      const token = await AsyncStorage.getItem('token');                     // token inseguro
      await fetch('https://api.example.com/positions', {                     // HTTP en la UI, sin timeout
        method: 'POST',
        headers: { Authorization: `Bearer ${token}` },
        body: JSON.stringify(pos.coords),
      });
      setCoords(pos.coords);
    })();
  }, []);

  return (
    <View>
      {error ? <Text style={{ color: 'red' }}>{error}</Text> : null}
      {coords && <Text>{coords.latitude}, {coords.longitude}</Text>}
      <TouchableOpacity onPress={() => {}}><Text>⟳</Text></TouchableOpacity>
    </View>
  );
}
```

### Problemas

| Problema | Impacto |
|---|---|
| Ruta con permisos, sensor, persistencia y HTTP | Imposible de testear y reutilizar |
| Permiso pedido al montar sin explicación; `canAskAgain` ignorado | Rechazo de App Review; usuario bloqueado |
| `Accuracy.Highest` sin necesidad, sin timeout | Batería y espera indefinida |
| Coordenadas en `console.log` y AsyncStorage | Privacidad |
| Token en AsyncStorage | Seguridad |
| `fetch` directo sin manejo de errores ni validación | Fallas silenciosas |
| `any`, estado de error como string técnico | Bugs y mensajes en inglés técnico |
| Botón de solo ícono sin rol ni label; color hardcodeado | Accesibilidad y dark mode |

### Después

| Responsabilidad | Archivo |
|---|---|
| Ruta | `src/app/(app)/location.tsx` (una línea, [sección 2](#2-navegación-con-expo-router)) |
| API del dispositivo | `src/shared/native/location/*` ([sección 7](#7-geolocalización-adaptador-nativo)) |
| Orquestación | `getCurrentLocation.ts` ([sección 8](#8-caso-de-uso-obtener-la-ubicación-actual)) |
| Estado de UI y permisos | `useCurrentLocation.ts`, `LocationPermissionState.tsx` ([sección 9](#9-hook-personalizado-y-solicitud-de-permisos)) |
| Presentación | `CurrentLocationScreen.tsx` ([sección 10](#10-pantalla-de-ubicación)) |
| Token | SecureStore vía sesión ([sección 6](#6-autenticación-y-sesión)) |
| Envío al backend | Solo si el producto lo requiere: un service `reportPosition.ts` en la funcionalidad que usa `request()`, invocado con `useMutation` |

Lo que **no** se creó: repositorio de ubicaciones, interface para el service HTTP, store global de coordenadas, caso de uso separado del service, persistencia local de la posición. Ninguno tenía un consumidor o requisito real.

Migración segura: mover primero adaptador y service con tests, luego hook y pantalla, reemplazar la ruta, migrar el token de AsyncStorage a SecureStore (leer una vez, guardar, borrar el viejo) y eliminar el código anterior cuando no tenga consumidores.
