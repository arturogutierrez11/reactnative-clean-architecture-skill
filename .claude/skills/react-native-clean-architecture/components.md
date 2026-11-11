# Pantallas y componentes

## Al implementar una pantalla

1. Definí los estados que la funcionalidad realmente tiene: inicial, cargando, éxito, vacío, error, permiso denegado, offline, no autorizado. No todos aplican.
2. Modelá esos estados como una unión discriminada en el hook, no como cinco booleanos sueltos.
3. La pantalla elige qué renderizar por estado; cada estado es un componente o rama simple.
4. Envolvé la pantalla en el contenedor `Screen` del proyecto (safe area, fondo del tema, scroll y teclado) si existe; si no existe y hay dos o más pantallas que lo necesitan, crealo en `shared/ui`.
5. Extraé un componente cuando tenga responsabilidad clara, reutilización real o haga comprensible al padre; no por cantidad de líneas.

```tsx
// Patrón de selección por estado
switch (state.status) {
  case 'loading':
    return <StateView loading title="Buscando tu ubicación" />;
  case 'permission-denied':
    return (
      <LocationPermissionState
        reason="permission-denied"
        canAskAgain={state.canAskAgain}
        onRequest={requestPermission}
        onOpenSettings={openSettings}
      />
    );
  case 'error':
    return <StateView title="Sin ubicación" description={ERROR_TEXT[state.code]} action={{ label: 'Reintentar', onPress: retry }} />;
  case 'success':
    return <LocationSummary location={state.location} approximate={state.precision === 'approximate'} isFallback={state.isFallback} />;
  // ...resto de estados
}
```

Versión completa: [examples.md](examples.md#10-pantalla-de-ubicación).

## Ubicación de componentes

- Primitivas sin conocimiento de negocio (`Button`, `AppText`, `TextField`, `Card`, `Screen`, `StateView`, `ListItem`) → `shared/ui`.
- Componentes con conceptos de negocio (`OrderCard`, `LocationPermissionState`) → funcionalidad, o `domains/<domain>/shared/components` si varias funcionalidades del dominio los usan.
- Un componente usado una vez no va a `shared` por ser pequeño.
- No crear un componente "universal" lleno de flags para unificar pantallas con reglas distintas. Preferir composición (`children`, slots como `left`/`right`).

## Criterios para primitivas reutilizables

Creá una primitiva cuando el segundo consumidor aparece o cuando el design system la define explícitamente. Cada primitiva:

- Consume tokens del tema ([design-system.md](design-system.md)); sin colores ni tamaños literales.
- Expone props mínimas y tipadas; extiende props nativas cuando corresponde (`PressableProps`, `TextInputProps`).
- Tiene accesibilidad por defecto: `accessibilityRole`, `accessibilityLabel` cuando no hay texto visible, `accessibilityState` (`disabled`, `busy`, `selected`).
- Área táctil mínima de 44×44 pt (iOS) / 48×48 dp (Android). Usá `hitSlop` si el visual es menor.
- Soporta estados `disabled`, `loading` y pressed (feedback visual inmediato con `Pressable` y su callback `({ pressed })`).
- Respeta escala de fuente del sistema (`allowFontScaling` por defecto en true; limitar con `maxFontSizeMultiplier` solo cuando el layout se rompe).

| Primitiva | Notas |
|---|---|
| `Text` | Variantes tipográficas del tema (`title`, `body`, `caption`). Color del tema. |
| `Button` | Variantes (`primary`, `secondary`, `destructive`, `ghost`), `loading` bloquea doble envío. |
| `TextInput` | Label asociado visible, error accesible, `returnKeyType`, `textContentType`/`autoComplete` para autocompletado del sistema. |
| `Screen` | `SafeAreaView`/insets de `react-native-safe-area-context`, scroll opcional, manejo de teclado. |
| `Card`, `ListItem` | Presentación; si es presionable, `accessibilityRole="button"`. |
| `Modal`/`BottomSheet` | Preferí `presentation: 'modal'`/`'formSheet'` de Expo Router para modales navegables. Bottom sheets de librería solo si el producto lo requiere. |
| `Toast` | Mensajes no bloqueantes; anunciar con `AccessibilityInfo.announceForAccessibility`. |
| `StateView` | Un contrato para carga, vacío y error: título, descripción y acciones (reintentar, ir a configuración). Separar en `EmptyState`/`ErrorState` solo si sus diseños divergen. |

No implementes todas por anticipado.

## Formularios

- React Hook Form + Zod cuando el formulario tiene validaciones o más de dos campos; `useState` alcanza para un campo simple.
- Labels visibles, errores debajo del campo y anunciados, foco al siguiente campo con `returnKeyType="next"` y `ref.focus()`.
- Bloqueo de doble envío mientras está pendiente; conservar valores si falla.
- La validación cliente es UX; el backend valida igual.
- Teclado: `keyboardType` adecuado (`email-address`, `number-pad`), `autoCapitalize="none"` para emails, `secureTextEntry` para contraseñas, `textContentType="password"`/`autoComplete="password"` para gestores de contraseñas.

## Listas

- `FlatList`/`SectionList` (o `FlashList` si el proyecto lo usa) para colecciones de tamaño variable; nunca `ScrollView` + `map` para listas largas.
- `keyExtractor` con identidad estable, no índices.
- Estado vacío distinto de error: cero resultados no es una falla. Ver [performance.md](performance.md#listas).

## Errores comunes

- Lógica de permisos dentro del componente visual.
- `useEffect` que hace fetch directo con `fetch()` en la pantalla.
- Textos técnicos (`Network request failed`) mostrados al usuario.
- Colores hardcodeados que rompen dark mode.
- `TouchableOpacity` sin `accessibilityRole` ni label en botones de solo ícono.
