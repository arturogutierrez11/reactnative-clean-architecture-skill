# Navegación y sesiones

Basado en Expo Router (rutas por archivos sobre React Navigation). Verificá la versión instalada: APIs como `Stack.Protected` o su prop `redirectTo` dependen del SDK (`redirectTo` existe desde SDK 58 según la documentación oficial).

## Principios

- Las rutas son finas: leen parámetros y renderizan una pantalla del dominio.
- La autenticación se decide **en un solo lugar** (el layout raíz o el de `(app)`), nunca duplicada en cada pantalla.
- Preferí navegación declarativa (`Stack.Protected`, `<Redirect />`, `<Link />`) sobre `router.replace` en efectos.
- No pierdas estado de navegación sin necesidad: evitá remontar el navegador raíz al cambiar de sesión con `key` dinámicas salvo que sea intencional (limpiar historial al logout).

## Estructura recomendada

```text
src/app/
├── _layout.tsx            # providers, splash, guards
├── +not-found.tsx
├── (auth)/
│   ├── _layout.tsx        # Stack sin header
│   ├── login.tsx
│   └── onboarding.tsx
└── (app)/
    ├── _layout.tsx        # Stack del área autenticada
    ├── (tabs)/
    │   ├── _layout.tsx    # Tabs
    │   ├── index.tsx
    │   └── profile.tsx
    ├── orders/[orderId].tsx
    └── settings.tsx       # presentation: 'modal' si corresponde
```

Los grupos `(nombre)` organizan sin agregar segmentos a la URL. Agregá `(tabs)` solo cuando el producto tenga navegación por pestañas; [examples.md](examples.md#2-navegación-con-expo-router) usa un Stack simple.

## Rutas protegidas

Con Expo Router moderno, usá `Stack.Protected` con `guard`. Cuando el guard pasa a `false`, las entradas de historial de esas pantallas se eliminan y el router redirige a la primera pantalla disponible (o a `redirectTo` en SDKs que lo soportan).

```tsx
// src/app/_layout.tsx
import { Stack } from 'expo-router';
import { useSession } from '@/domains/auth/session/useSession';

export default function RootLayout() {
  const { status } = useSession();
  if (status === 'restoring') return null; // splash visible hasta restaurar

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Protected guard={status === 'authenticated'}>
        <Stack.Screen name="(app)" />
      </Stack.Protected>
      <Stack.Protected guard={status !== 'authenticated'}>
        <Stack.Screen name="(auth)" />
      </Stack.Protected>
    </Stack>
  );
}
```

Reglas:

- Una pantalla se declara una sola vez; usá grupos y guards complementarios, no pantallas duplicadas.
- Si la versión instalada no tiene `Stack.Protected`, usá `<Redirect href="/login" />` en el layout del grupo protegido, en un único lugar.
- Las rutas protegidas del cliente **no** reemplazan la autorización del backend.

## Inicialización y splash

1. `SplashScreen.preventAutoHideAsync()` en el scope del módulo de `_layout.tsx`.
2. Restaurar sesión (SecureStore), cargar fuentes y config crítica.
3. Ocultar splash (`SplashScreen.hideAsync()`) cuando la primera pantalla real puede renderizar.
4. Nunca bloquear indefinidamente: si la restauración falla, ir a `(auth)` y registrar el error.

Verificá en la documentación de `expo-splash-screen` de tu SDK si existe `SplashScreen.setOptions` para animaciones de salida.

## Parámetros de ruta

- Leé con `useLocalSearchParams<{ orderId: string }>()`. Los parámetros llegan como `string | string[]`: validalos con Zod antes de usarlos.
- Pasá identificadores, no objetos completos. La pantalla destino carga sus datos (TanStack Query los sirve desde cache).
- Con `typedRoutes` habilitado (experimento de Expo Router), los `href` quedan tipados; activalo si la versión lo soporta.

## Deep links, Universal Links y App Links

- Esquema propio en `app.config.ts` (`scheme`) y dominios asociados (iOS `associatedDomains`, Android `intentFilters` con `autoVerify`). Ver [configuration.md](configuration.md).
- Expo Router mapea URLs a rutas automáticamente. Un deep link es **entrada no confiable**: validá parámetros, no ejecutes acciones destructivas solo por abrir un link y exigí sesión antes de mostrar datos privados.
- Para reescribir URLs entrantes (links legacy, notificaciones), usá `+native-intent.tsx` (`redirectSystemPath`) si la versión lo soporta.
- Probar: `npx uri-scheme open <scheme>://ruta --ios|--android` o `xcrun simctl openurl booted <url>` / `adb shell am start -W -a android.intent.action.VIEW -d <url>`.

## Navegación desde notificaciones

- Incluí en el payload una ruta o identificador (`data.url` o `data.orderId`), validalo y navegá con `router.push`.
- Manejar los dos casos: app abierta (listener de respuesta) y app iniciada desde la notificación (`useLastNotificationResponse`). Ver [device-permissions.md](device-permissions.md#notificaciones).
- Esperar a que la sesión esté restaurada antes de navegar a rutas protegidas.

## Sesión expirada

- El cliente HTTP detecta 401 tras fallar el refresh → limpia sesión en el store → el guard cambia → el router saca al usuario del área protegida. Ninguna pantalla navega manualmente al login. Ver [authentication.md](authentication.md).
- Mostrar un mensaje no técnico ("Tu sesión expiró").

## Android Back y gestos

- Stack maneja el botón Back y el gesto de retroceso. No interceptar `BackHandler` salvo para confirmar descarte de cambios; en ese caso usar `usePreventRemove` de React Navigation o `beforeRemove` si la versión lo expone.
- Modales: `presentation: 'modal'` en iOS se cierra con swipe; asegurá que cerrar no pierda datos sin confirmación.

## Errores comunes

- Redirect loops: guard que depende de un estado que la misma redirección modifica. Usá un estado `restoring` explícito.
- `router.replace` en `useEffect` antes de que el navegador esté montado.
- Lógica de auth en cada pantalla.
- Pasar objetos grandes por params.
- Datos de negocio fetcheados en `_layout.tsx`.
