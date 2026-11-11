# Convenciones de nombres y código

Si el proyecto ya tiene una convención consistente, seguila. Lo siguiente aplica a proyectos nuevos o sin convención.

## Nombres

| Elemento | Convención | Ejemplo |
|---|---|---|
| Dominio y funcionalidad | `kebab-case`, verbo + objeto para funcionalidades | `location/get-current-location` |
| Carpeta de responsabilidad | Plural convencional | `components`, `hooks`, `services`, `use-cases` |
| Componente / pantalla | Archivo y símbolo PascalCase; pantallas con sufijo `Screen` | `LocationStatus.tsx`, `LoginScreen.tsx` |
| Hook | `use` + PascalCase, `.ts` si no tiene JSX | `useCurrentLocation.ts` |
| Service / caso de uso | camelCase, verbo + objeto | `getCurrentLocation.ts`, `startLocationTracking.ts` |
| Adaptador | `<capacidad>.adapter.ts` (contrato) y `<proveedor>-<capacidad>.adapter.ts` (implementación) | `location.adapter.ts`, `expo-location.adapter.ts` |
| Schema | `<concepto>.schema.ts`; símbolo camelCase + `Schema` | `login.schema.ts`, `loginSchema` |
| Tipos | `<concepto>.types.ts`; símbolos PascalCase | `location.types.ts`, `Coordinates` |
| Store | `<concepto>.store.ts`; hook `use<Concepto>Store` | `session.store.ts`, `useSessionStore` |
| Task de background | `<concepto>Task.ts`; nombre de tarea en UPPER_SNAKE_CASE | `locationTask.ts`, `LOCATION_TRACKING_TASK` |
| Archivo por plataforma | `.ios.tsx` / `.android.tsx` / `.native.tsx` | `DatePicker.ios.tsx` |
| Test | `<archivo>.test.ts(x)` | `getCurrentLocation.test.ts` |
| Constantes globales inmutables | UPPER_SNAKE_CASE | `DEFAULT_TIMEOUT_MS` |
| Rutas Expo Router | Convención del router: `index.tsx`, `[id].tsx`, `(group)/`, `_layout.tsx`, `+not-found.tsx` | `orders/[orderId].tsx` |

Mixto intencional: carpetas en kebab-case (estables, agrupan), archivos con el nombre del símbolo principal que exportan (búsqueda directa). Si el proyecto usa kebab-case también para archivos (`get-current-location.ts`), mantenelo.

## Exports

- Exports nombrados en módulos propios. `export default` solo donde lo exige el framework (rutas y layouts de Expo Router).
- La ruta puede reexportar la pantalla: `export { LoginScreen as default } from '@/domains/auth/login/screens/LoginScreen';`
- Sin barrels automáticos (ver [architecture.md](architecture.md#direcciones-permitidas)).

## Imports

- `@/` para cruzar ámbitos (`@/shared/ui/Button`, `@/domains/auth/session/useSession`). Relativos cortos dentro de la funcionalidad. Nada de `../../../`.
- Orden: externos, `@/`, relativos. Dejar que ESLint/Prettier lo resuelvan si están configurados.
- Detectar ciclos con `npx madge --circular --extensions ts,tsx src` si hay sospechas (herramienta opcional; no instalarla como dependencia del proyecto sin pedido).

## TypeScript

- `strict: true`. Sin `any`; usar `unknown` para datos externos y validar con Zod.
- Modelar resultados con uniones discriminadas: `{ status: 'granted' } | { status: 'denied'; canAskAgain: boolean }`.
- Tipar contratos públicos (props, retornos de hooks, services). Inferir dentro de implementaciones.
- Derivar tipos del schema (`z.infer`) en vez de duplicarlos.
- Evitar `as` salvo en fronteras justificadas con comentario. Nunca `as Product` sobre JSON externo.
- Preferir `type` para uniones y props; `interface` para contratos de adaptadores es aceptable. Ser consistente.
- `noUncheckedIndexedAccess` es recomendable en proyectos nuevos.

## Código

- Funciones pequeñas y cohesivas. Componentes con una responsabilidad clara.
- Archivos que crecen por encima de ~250–300 líneas son una señal para revisar responsabilidades, no una regla automática.
- Extraer lógica solo cuando aporta (reutilización, testabilidad, legibilidad).
- Preservar contratos existentes durante refactors.
- No dejar `console.log` de depuración. Usar el logger del proyecto ([observability.md](observability.md)).
