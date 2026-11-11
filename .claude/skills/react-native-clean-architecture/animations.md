# Animaciones y gestos

## Elegir herramienta

| Necesidad | Herramienta |
|---|---|
| Transiciones entre pantallas | Las del navegador (Expo Router/React Navigation): `animation`, `presentation` |
| Fade/scale simples, pocas propiedades | `Animated` de React Native con `useNativeDriver: true` |
| Animaciones ligadas a gestos, scroll, layout, interrupciones | React Native Reanimated + React Native Gesture Handler |
| Entrada/salida de elementos en listas | Reanimated layout animations (`entering`, `exiting`, `layout`) |
| Ilustraciones animadas | Lottie (`lottie-react-native`) solo si diseño entrega assets |
| Feedback de presión | `Pressable` con estilo por `pressed`, o escala con Reanimated en el `Button` del design system |

No incorpores Reanimated, Moti, Skia o Lottie si `Animated` o la transición del navegador resuelven el caso. Si el proyecto ya tiene Reanimated (muchas plantillas de Expo lo incluyen), usalo. Verificá la versión instalada: la API y su integración con `react-native-worklets` cambian entre versiones mayores.

## Principios

- Duración: 150–300 ms para microinteracciones; usar tokens `motion`.
- Curvas: `ease-out` para entradas, `ease-in` para salidas; springs para elementos que el usuario arrastra.
- Animar `transform` y `opacity` (corren en el hilo de UI). Evitar animar `width`, `height`, `top` en elementos complejos.
- Interrumpibles: un gesto nuevo cancela la animación en curso sin saltos.
- Respetar **Reduce Motion**: reemplazar movimiento por fade o mostrar el estado final.
- Nunca bloquear la interacción esperando una animación decorativa.

## Ejemplo con Animated

```tsx
import { useEffect, useRef } from 'react';
import { Animated, AccessibilityInfo } from 'react-native';

export function FadeIn({ children }: { children: React.ReactNode }) {
  const opacity = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    let cancelled = false;
    AccessibilityInfo.isReduceMotionEnabled().then((reduce) => {
      if (cancelled) return;
      if (reduce) return opacity.setValue(1);
      Animated.timing(opacity, { toValue: 1, duration: 250, useNativeDriver: true }).start();
    });
    return () => { cancelled = true; };
  }, [opacity]);

  return <Animated.View style={{ opacity }}>{children}</Animated.View>;
}
```

## Ejemplo con Reanimated

```tsx
import Animated, { useAnimatedStyle, useSharedValue, withSpring, useReducedMotion } from 'react-native-reanimated';
import { Pressable, type PressableProps, type StyleProp, type ViewStyle } from 'react-native';

const AnimatedPressable = Animated.createAnimatedComponent(Pressable);

// `style` como objeto (no función) para poder combinarlo con el estilo animado.
type Props = Omit<PressableProps, 'style'> & { style?: StyleProp<ViewStyle> };

export function PressableScale(props: Props) {
  const scale = useSharedValue(1);
  const reduceMotion = useReducedMotion();
  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));

  return (
    <AnimatedPressable
      {...props}
      onPressIn={(e) => { if (!reduceMotion) scale.value = withSpring(0.97); props.onPressIn?.(e); }}
      onPressOut={(e) => { scale.value = withSpring(1); props.onPressOut?.(e); }}
      style={[props.style, style]}
    />
  );
}
```

Verificá los nombres (`useReducedMotion`, `createAnimatedComponent`) en la versión de Reanimated instalada.

## Gestos

- `react-native-gesture-handler` con `GestureHandlerRootView` en el layout raíz (si la plantilla no lo hace ya).
- No competir con gestos del sistema (borde izquierdo en iOS, barra inferior).
- Feedback háptico al confirmar un umbral (swipe para borrar), no en cada frame.

## Rendimiento

- Medir con el Performance Monitor del menú de desarrollo y perfiladores nativos; probar en **build de release** en un Android de gama media: el modo dev no es representativo.
- Evitar `setState` en cada frame de una animación; usar shared values.
- Listas con layout animations: probar con muchos ítems.
