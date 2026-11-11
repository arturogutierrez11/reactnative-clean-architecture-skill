# Estado

## Elegir propietario antes que herramienta

| Necesidad | Herramienta |
|---|---|
| Toggle, input, modal, tab de una pantalla | `useState` en el componente |
| Transiciones con varias acciones relacionadas | `useReducer` |
| Compartido entre hermanos | Padre común más cercano |
| Formularios | React Hook Form (+ Zod) o estado local si es trivial |
| Datos del servidor (cache, refetch, paginación, mutaciones) | TanStack Query |
| Dependencia de un subárbol con cambios poco frecuentes (tema, i18n) | Context |
| Estado cliente transversal (sesión, preferencias, flujo multi-pantalla) | Zustand, con dueño de dominio |
| Persistente sensible (tokens) | SecureStore ([storage.md](storage.md)) |
| Persistente no sensible (preferencias, cache) | AsyncStorage / MMKV / SQLite según volumen |

## Reglas

- **Una sola fuente de verdad.** No copiar datos de TanStack Query a Zustand "para tenerlos a mano". Leer con `useQuery` o `queryClient.getQueryData`.
- No guardar valores derivables; calcularlos en render (o `useMemo` si es costoso y medido).
- No crear un store por pantalla. Un store existe cuando varias pantallas no relacionadas lo leen y tiene ciclo de vida definido (inicio, reset en logout).
- Separar estado de UI (filtro abierto) y estado de negocio (carrito confirmado).
- Stores pertenecen a su dominio (`domains/auth/session/session.store.ts`), no a `shared/stores`.
- Tokens nunca en Zustand persistido con AsyncStorage. El store puede tener `status` y datos públicos del usuario; los tokens viven en SecureStore y en memoria del servicio de sesión, que los entrega al cliente HTTP mediante `getAccessToken`.
- Al logout: `queryClient.clear()`, reset de stores de dominio, borrar persistencia del usuario.

## TanStack Query en React Native

- `QueryClientProvider` en `src/app/_layout.tsx`.
- Conectar `focusManager` con `AppState` y `onlineManager` con NetInfo; React Native no tiene eventos de foco/online del navegador. Verificá el ejemplo oficial de React Native de TanStack Query para la versión instalada.
- Query keys como arrays jerárquicos, definidas cerca del dominio: `['orders', 'list', filters]`, `['orders', 'detail', id]`. Una fábrica de keys por dominio cuando haya más de dos queries.
- `staleTime` explícito según el dato (perfil: minutos; stock: segundos). No dejar todo en 0 por defecto sin pensarlo.
- Mutaciones: invalidar solo las keys afectadas en `onSuccess`; optimistic updates solo cuando la UX lo justifica y con rollback.
- `retry`: queries sí (con límite); mutaciones no idempotentes, no ([data-fetching.md](data-fetching.md#reintentos)).
- Hook por operación en la funcionalidad: `useOrders(filters)`, `useCreateOrder()`. No crear un hook que solo envuelve `useQuery` sin key/fn propias.

```ts
// domains/orders/list-orders/hooks/useOrders.ts
export function useOrders(filters: OrderFilters) {
  return useQuery({
    queryKey: ordersKeys.list(filters),
    queryFn: ({ signal }) => listOrders(filters, { signal }),
    staleTime: 60_000,
  });
}
```

## Zustand

```ts
// domains/auth/session/session.store.ts
import { create } from 'zustand';
import type { SessionUser } from './session.schema';

export type Session =
  | { status: 'restoring' }
  | { status: 'anonymous'; reason?: 'expired' }
  | { status: 'authenticated'; user: SessionUser };

// El estado es un único campo discriminado; las acciones viven fuera del store.
export const useSessionStore = create<{ session: Session }>()(() => ({
  session: { status: 'restoring' },
}));

export const setSession = (session: Session) => useSessionStore.setState({ session });
```

Implementación completa (restauración, login, logout, refresh): [examples.md](examples.md#6-autenticación-y-sesión).

- Selectores para evitar renders: `useSessionStore((s) => s.session.status)`.
- Acciones como funciones del módulo (`setSession(...)`) para poder llamarlas fuera de React (cliente HTTP, tasks).
- `persist` middleware solo para datos no sensibles y con versión/migración.

## Context

- Bien para inyección de dependencias livianas en un subárbol (tema, adaptador en tests).
- Mal para estado de alta frecuencia (posición GPS, progreso de upload): re-renderiza a todos los consumidores. Usar un store con selectores o estado local.

## Estado en efectos

- Cancelar requests obsoletos (TanStack Query pasa `signal`).
- Limpiar suscripciones (sensores, AppState, listeners) en el cleanup del efecto.
- No sincronizar dos fuentes con efectos bidireccionales.

## Datos obsoletos y conectividad

- Mostrar datos cacheados con indicador de antigüedad cuando no hay red, en lugar de pantalla de error.
- Persistir cache de queries (`persistQueryClient`) solo si la app debe arrancar con datos offline ([offline.md](offline.md)).
