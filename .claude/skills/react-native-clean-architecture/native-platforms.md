# iOS y Android como plataformas de primera clase

## Regla general

Implementación compartida por defecto. Diferencias por plataforma solo cuando el comportamiento, la API o la convención de diseño lo exige, y aisladas en el punto más bajo posible.

| Mecanismo | Usar para | Evitar |
|---|---|---|
| `Platform.OS === 'ios'` | Una diferencia puntual (un valor, un prop) | Ramas largas dentro de componentes |
| `Platform.select({ ios, android, default })` | Valores (estilos, textos, constantes) | Lógica |
| `Component.ios.tsx` / `Component.android.tsx` | Implementaciones realmente distintas con el mismo contrato de props | Duplicar 90% del código; extraé lo común |
| Adaptador por plataforma | APIs nativas con comportamiento distinto (permisos de background, servicios) | — |

Con archivos por plataforma, ambos deben exportar el mismo contrato. Si existe un archivo base `Component.tsx` además de los específicos, Metro prioriza el específico; TypeScript resuelve el base, así que mantené las firmas idénticas (o tipá con un `Component.types.ts` compartido).

## Áreas seguras

- Usá `react-native-safe-area-context` (`SafeAreaProvider` en el layout raíz; `SafeAreaView` o `useSafeAreaInsets` en `Screen`). El `SafeAreaView` de `react-native` está limitado a iOS y deprecado en versiones recientes; verificá.
- Android moderno dibuja edge-to-edge: no asumas que la barra de estado o la de navegación reservan espacio. Aplicá insets también abajo.
- Headers de Expo Router ya manejan el inset superior; no lo dupliques.

## Teclado

- Formularios: `KeyboardAvoidingView` con `behavior={Platform.OS === 'ios' ? 'padding' : 'height'}` o la solución que ya use el proyecto (p. ej. `react-native-keyboard-controller`). Verificá el comportamiento en ambas plataformas, especialmente Android edge-to-edge.
- `keyboardShouldPersistTaps="handled"` en `ScrollView` con inputs.
- `Keyboard.dismiss()` al enviar.

## Back de Android y gestos

- El stack maneja Back. Interceptar solo para confirmar descarte de datos.
- No poner acciones en el borde izquierdo que compitan con el gesto de retroceso de iOS ni en el borde inferior que compitan con la barra de gestos.

## Ciclo de vida (AppState)

- `AppState` (`active`, `background`, `inactive` en iOS). Suscribir con `AppState.addEventListener('change', ...)` y remover en cleanup.
- Usos: refrescar permisos al volver de Configuración, pausar sensores, refetch (TanStack Query con `focusManager`), bloquear pantalla sensible.
- Centralizalo en un hook (`shared/hooks/useAppState.ts`) cuando haya dos consumidores.

## Retorno desde apps externas

- Al abrir Configuración (`Linking.openSettings()`), el usuario vuelve a la app: re-chequear permisos en el evento `active`.
- Al abrir un navegador para OAuth, usar `expo-web-browser`/`expo-auth-session` y manejar cancelación.

## Permisos: diferencias clave

| Tema | iOS | Android |
|---|---|---|
| Momento del prompt | Una vez; luego solo Configuración | Se puede volver a pedir hasta que el usuario marca "no volver a preguntar" (`canAskAgain: false`) |
| Textos | Obligatorios en Info.plist (`NS...UsageDescription`); App Review los evalúa | No hay texto del sistema; explicá en tu UI antes |
| Precisión de ubicación | `ios.accuracy: 'full' | 'reduced'` | `android.accuracy: 'fine' | 'coarse'` |
| Background location | "Always" tras "While Using"; "Allow Once" impide pedir background en esa sesión | Android 11+ abre Configuración; requiere foreground service para updates continuos |
| Notificaciones | Prompt explícito | Android 13+ requiere permiso `POST_NOTIFICATIONS` y canal creado antes |

Detalle en [device-permissions.md](device-permissions.md) y [geolocation.md](geolocation.md).

## Diseño por plataforma

- Respetá convenciones: tabs abajo en ambas; headers con título centrado en iOS e izquierda en Android (React Navigation lo hace por defecto); action sheets en iOS vs. menús/bottom sheets en Android.
- Tipografía del sistema (SF Pro / Roboto) salvo identidad de marca definida.
- Feedback: `android_ripple` en `Pressable` para Android; opacidad/escala en iOS. Un `Pressable` del design system puede resolverlo una vez.
- Sombras: `shadow*` en iOS vs. `elevation` en Android; versiones recientes de RN soportan `boxShadow` en ambas, verificá. Encapsulalo en tokens ([design-system.md](design-system.md)).

## Tamaños de pantalla

- Diseñá con flexbox y porcentajes; `useWindowDimensions()` para breakpoints (teléfono chico, grande, tablet, landscape).
- Probá al menos en un dispositivo chico (iPhone SE / Android 5") y uno grande. Probá con fuente del sistema al máximo.
- Orientación: definí `orientation` en `app.config.ts`; si es `default`, verificá layouts en landscape.

## Checklist por funcionalidad

- [ ] Funciona en iOS y Android (o la diferencia está documentada).
- [ ] Safe areas superior e inferior.
- [ ] Teclado no tapa inputs.
- [ ] Back de Android y gestos iOS no rompen el flujo.
- [ ] Permiso denegado y "no volver a preguntar" manejados.
- [ ] Vuelve correctamente desde background y desde Configuración.
- [ ] Dark mode y fuente grande.
