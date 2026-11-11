# Diseño de interfaces y design system

Referencias: Apple Human Interface Guidelines, Material Design 3, guías de accesibilidad de iOS y Android. El objetivo es una UI que se sienta nativa y propia del producto, no un prototipo genérico.

## Principios

- **Jerarquía clara:** un objetivo principal por pantalla; acción primaria evidente; contenido antes que decoración.
- **Consistencia:** mismos tokens, mismos componentes, mismo comportamiento en toda la app.
- **Estados diseñados:** carga (skeleton o indicador con contexto), vacío (explica y ofrece acción), error (comprensible y con recuperación), éxito (feedback breve).
- **Feedback inmediato:** estado pressed visible; háptica sutil donde confirma algo significativo.
- **Movimiento con propósito:** transiciones que explican de dónde viene y a dónde va el contenido ([animations.md](animations.md)).
- **Nativo:** respetar convenciones de cada plataforma ([native-platforms.md](native-platforms.md#diseño-por-plataforma)).

Señales de UI "genérica": gradientes y sombras por todos lados, tarjetas idénticas para todo, emojis como iconos, textos de relleno, espaciado irregular, botones sin estados, ignorar dark mode. Evitalas.

## Tokens

Definidos en `src/shared/theme/`. Adaptá valores a la identidad de cada producto; la estructura es lo que se mantiene.

```ts
// src/shared/theme/tokens.ts
export const palette = {
  blue600: '#2563EB', blue400: '#60A5FA',
  gray950: '#0B0D12', gray900: '#111827', gray600: '#4B5563', gray400: '#9CA3AF',
  gray100: '#F3F4F6', white: '#FFFFFF', red600: '#DC2626', red400: '#F87171',
} as const;

export const spacing = { xs: 4, sm: 8, md: 12, lg: 16, xl: 24, xxl: 32 } as const;
export const radius = { sm: 6, md: 10, lg: 16, full: 999 } as const;

export const typography = {
  title: { fontSize: 28, lineHeight: 34, fontWeight: '700' },
  heading: { fontSize: 20, lineHeight: 26, fontWeight: '600' },
  body: { fontSize: 16, lineHeight: 22, fontWeight: '400' },
  label: { fontSize: 14, lineHeight: 18, fontWeight: '500' },
  caption: { fontSize: 12, lineHeight: 16, fontWeight: '400' },
} as const;

export const motion = { fast: 150, normal: 250, slow: 400 } as const;

export type ThemeColors = Record<
  'background' | 'surface' | 'text' | 'textMuted' | 'primary' | 'onPrimary' | 'danger' | 'border',
  string
>;

const light: ThemeColors = {
  background: palette.white, surface: palette.gray100,
  text: palette.gray900, textMuted: palette.gray600,
  primary: palette.blue600, onPrimary: palette.white,
  danger: palette.red600, border: '#E5E7EB',
};

const dark: ThemeColors = {
  background: palette.gray950, surface: palette.gray900,
  text: palette.gray100, textMuted: palette.gray400,
  primary: palette.blue400, onPrimary: palette.gray950,
  danger: palette.red400, border: '#1F2937',
};

export const colorSchemes = { light, dark };
```

- **Colores semánticos** (`text`, `surface`, `danger`), nunca valores de paleta directos en componentes.
- **Espaciado** en escala de 4.
- **Tipografía** por rol; respetar Dynamic Type/escala de fuente.
- **Radios, sombras/elevación, motion** como tokens. Sombras: encapsular la diferencia iOS (`shadow*`) / Android (`elevation`) o usar `boxShadow` si la versión de RN lo soporta en ambas (verificá).

```ts
// src/shared/theme/useTheme.ts
import { useColorScheme } from 'react-native';
import { colorSchemes, spacing, radius, typography, motion } from './tokens';

export function useTheme() {
  const scheme = useColorScheme() === 'dark' ? 'dark' : 'light';
  return { scheme, colors: colorSchemes[scheme], spacing, radius, typography, motion };
}
```

Si el producto permite elegir tema manualmente, mové la preferencia a un store/Context de preferencias y combinala con `useColorScheme`. Si el proyecto usa una librería de estilos (NativeWind, Tamagui, Unistyles, Restyle), mapeá estos tokens a su sistema en lugar de duplicarlos.

## Dark mode

- `userInterfaceStyle: 'automatic'` en `app.config.ts`.
- Toda superficie, texto, borde e ícono usa tokens. Imágenes/ilustraciones con variante oscura si hace falta.
- `StatusBar` (`expo-status-bar` con `style="auto"`) acorde al fondo.
- Probar contraste en ambos modos.

## Accesibilidad

- Contraste mínimo WCAG AA (4.5:1 texto normal, 3:1 texto grande e iconos significativos).
- `accessibilityLabel` en controles sin texto; `accessibilityHint` solo si la acción no es obvia.
- `accessibilityRole` correcto (`button`, `link`, `header`, `switch`, `image`...), `accessibilityState` para disabled/selected/checked/busy.
- Orden de foco lógico; agrupar elementos relacionados (`accessible` en el contenedor) cuando el lector debe leerlos juntos.
- Área táctil 44×44 pt / 48×48 dp.
- Escala de fuente: layouts que crecen verticalmente; probar con el tamaño máximo.
- Reduce Motion: `AccessibilityInfo.isReduceMotionEnabled()` o `useReducedMotion` de Reanimated → simplificar animaciones.
- Anunciar cambios importantes (`AccessibilityInfo.announceForAccessibility`).
- No transmitir información solo con color.
- Probar con VoiceOver y TalkBack al menos los flujos críticos.

## Iconos, imágenes y fuentes

- Un solo set de iconos coherente (`@expo/vector-icons` o SF Symbols vía `expo-symbols` en iOS si el proyecto lo adopta).
- `expo-image` para imágenes remotas (cache, placeholders, transiciones).
- Fuentes con `expo-font` (config plugin para incrustarlas en build cuando es posible); cargar antes de ocultar el splash.

## Layout responsive

- Flexbox, `gap`, porcentajes; `useWindowDimensions` para adaptar a tablet/landscape.
- Ancho máximo de contenido en tablets para legibilidad.
- No dimensiones absolutas basadas en un dispositivo.
