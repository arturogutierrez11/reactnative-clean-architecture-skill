# Rendimiento y experiencia de usuario

**Regla:** medí o identificá evidencia antes de introducir optimizaciones complejas. Probá en build de release y en un Android de gama media; el modo desarrollo es mucho más lento y engaña.

## Herramientas de medición

- Performance Monitor (menú de desarrollo): FPS de JS y UI.
- React DevTools Profiler (vía React Native DevTools): qué renderiza y por qué.
- Xcode Instruments (Time Profiler, Allocations, Energy) y Android Studio Profiler (CPU, memoria, energía).
- Tiempo de arranque: medir en release desde el ícono hasta la primera pantalla interactiva.

## Listas

- `FlatList`/`SectionList` siempre para colecciones de tamaño variable. `FlashList` si el proyecto lo usa o si la medición lo justifica.
- `keyExtractor` estable.
- `renderItem` liviano; ítems memoizados (`React.memo`) **cuando** el profiler muestra renders innecesarios o el ítem es costoso.
- `getItemLayout` si la altura es fija (mejora scroll a índice y salto inicial).
- Ajustar `initialNumToRender`, `windowSize`, `maxToRenderPerBatch` solo con medición.
- Imágenes de ítems con tamaño fijo y versión reducida desde el servidor.
- Nunca `ScrollView` + `map` para listas largas ni `FlatList` dentro de `ScrollView` del mismo eje.

## Renders

`useMemo`, `useCallback` y `React.memo` **no** se aplican por defecto. Aportan cuando:

| Herramienta | Cuándo |
|---|---|
| `React.memo` | Componente costoso o ítem de lista que recibe props estables y re-renderiza por el padre |
| `useCallback` | La función se pasa a un componente memoizado o es dependencia de un efecto |
| `useMemo` | Cálculo costoso medido, o valor que es dependencia/prop de algo memoizado |

Si el proyecto usa React Compiler, la memoización manual es casi siempre innecesaria; verificá si está habilitado (`app.json` `experiments.reactCompiler` u otra configuración) antes de agregarla.

Causas comunes de renders: Context con valores que cambian a menudo, stores sin selectores, objetos/estilos inline recreados para hijos memoizados, estado demasiado arriba.

## Imágenes

- `expo-image` con `contentFit`, `placeholder`, `cachePolicy` y `recyclingKey` en listas.
- Pedir al servidor el tamaño adecuado; nunca descargar originales de 4000 px para miniaturas.
- Comprimir antes de subir.

## Memoria

- Desuscribir listeners, sensores, timers y watchers de ubicación.
- No guardar arrays enormes en estado global.
- Cuidado con closures en listeners que retienen pantallas.

## Navegación

- Pantallas pesadas: diferir trabajo no crítico hasta después de la transición (mostrar skeleton y cargar después; `InteractionManager` fue removido del core en React Native 0.88 (verificado en el paquete instalado); usá `requestIdleCallback` o el evento `transitionEnd` del navegador).
- No cargar datos de todas las tabs al iniciar.
- `lazy` en tabs (comportamiento por defecto en React Navigation; verificá).

## Arranque

- Minimizar trabajo en el scope de módulos importados por `_layout.tsx`.
- Evitar barrels que importan todo un dominio.
- Fuentes y assets críticos solamente antes de ocultar splash.

## Sensores, batería y background

- Precisión y frecuencia mínimas de ubicación ([geolocation.md](geolocation.md)).
- `setUpdateInterval` en sensores; desuscribir al pasar a background.
- Agrupar requests (batch) en lugar de uno por evento.
- Polling: solo con pantalla visible y con intervalos razonables; preferir push.
- Trabajo en background solo con APIs del sistema diseñadas para eso.

## Aplicaciones lentas: diagnóstico

1. Reproducir en release.
2. Medir (FPS, profiler) e identificar el cuello: JS, UI o red.
3. Corregir la causa (render innecesario, lista mal configurada, imagen grande, trabajo síncrono pesado).
4. Volver a medir y reportar antes/después.

No reportes "optimizado" sin medición; decí qué se cambió y qué falta medir en dispositivo.
