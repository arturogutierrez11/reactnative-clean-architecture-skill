# Arquitectura y dependencias

## Regla de ubicación

Antes de crear un archivo, respondé tres preguntas:

1. ¿Qué **dominio** lo posee? Área estable del producto: `auth`, `location`, `profile`, `orders`.
2. ¿Qué **funcionalidad** lo necesita? Resultado reconocible por el usuario: `login`, `get-current-location`, `update-profile`.
3. ¿Qué **responsabilidad** cumple? `screens`, `components`, `hooks`, `services`, `schemas`, `types`, `use-cases`, `adapters`, `stores`, `tests`, solo si existe código para esa carpeta.

Ubicación resultante: `src/domains/<domain>/<feature>/<responsibility>/<Archivo>`. Un tipo o helper de un solo consumidor puede quedar dentro del archivo que lo usa.

## Capas pragmáticas

Clean Architecture acá significa **límites explícitos de dependencia**, no cuatro carpetas por operación.

| Capa | Qué contiene | Dónde vive | Puede depender de |
|---|---|---|---|
| Presentación | Pantallas, componentes, hooks de UI | `screens/`, `components/`, `hooks/` | Aplicación, dominio, `shared/ui`, `shared/theme` |
| Aplicación | Orquestación de una operación: permisos → dispositivo → API → persistencia | `use-cases/` o la función de `services/` | Dominio, contratos de infraestructura |
| Dominio | Modelos, reglas puras, validaciones de negocio | `types/`, `schemas/`, funciones puras | Nada de React, React Native ni Expo |
| Infraestructura | HTTP, almacenamiento, adaptadores del dispositivo, SDKs | `services/`, `shared/api`, `shared/native`, `shared/storage` | Librerías externas, config |

Reglas:

- El código de dominio (tipos, reglas puras) no importa `react`, `react-native` ni `expo-*`. Así se testea con Jest puro y se reutiliza.
- Los componentes no importan `expo-location`, `expo-secure-store`, `fetch` ni el cliente HTTP. Consumen hooks.
- Los hooks coordinan estado de presentación y llaman servicios o casos de uso. No contienen reglas de negocio extensas.
- Un caso de uso aparece cuando una operación coordina **varios** pasos o fuentes (permiso + sensor + API + cache) o cuando la misma operación tiene varios disparadores (pantalla, tarea en background, notificación). Una lectura simple es un service.
- Excepción aceptada: un caso de uso puede llamar directamente a una librería cuando es la única implementación y la operación es propia del proveedor (p. ej. tracking en background con `expo-location`, ver [geolocation.md](geolocation.md#implementación)). Si aparece un segundo proveedor o hace falta testear la orquestación sin la librería, extraé el adaptador.
- Los adaptadores nativos aíslan APIs del dispositivo detrás de un contrato propio (tipos de la app, errores de la app). Ver [native-modules.md](native-modules.md) y [geolocation.md](geolocation.md).

## Direcciones permitidas

```text
app ──→ domains ──→ shared
app ──────────────→ shared
feature ──→ su propio código
feature ──→ domains/<domain>/shared ──→ shared global
```

- `shared` global nunca importa `domains` ni `app`.
- `domains` nunca importa de `app`.
- `domains/<domain>/shared` nunca importa funcionalidades concretas de su dominio.
- Una funcionalidad no importa internals de una funcionalidad hermana. Si dos la necesitan, el contrato sube a `domains/<domain>/shared`.
- Una dependencia entre dominios es explícita, acíclica y pasa por un contrato pequeño (una función o un hook público). Ejemplo válido: `orders/checkout` usa `useSession()` de `auth/session`. Inválido: leer el store interno de `auth` o su token.
- No mover reglas de negocio a `shared` para esconder una dependencia entre dominios.
- No crear `index.ts` barrel automáticamente. Generan ciclos y empeoran el tiempo de arranque de Metro. Permitido solo como API pública pequeña de un dominio cuando hay varios consumidores externos.

## Cuándo crear cada abstracción

| Abstracción | Crear cuando | No crear cuando |
|---|---|---|
| Adaptador nativo con contrato (`LocationAdapter`) | Frontera con el dispositivo: hay que mockearla en tests, puede cambiar el proveedor, o difiere por plataforma | Una llamada puntual sin permisos ni estado: basta una función en `shared/native/<capacidad>/` (p. ej. `haptics.ts` con `selectionFeedback()`), sin interface |
| Interface TS para un service | Hay dos implementaciones reales (Expo y nativo propio, real y fake de demo) | Solo existe una implementación y el test puede mockear el módulo |
| Caso de uso (`use-cases/`) | Orquesta ≥2 fuentes, tiene reglas propias o múltiples disparadores | Envuelve un único `apiClient.get` |
| Repositorio | La funcionalidad combina remoto + persistencia local con política propia (offline, cache) | Solo hay una API remota y TanStack Query ya cachea |
| Store (Zustand) | Estado cliente compartido por pantallas no relacionadas y con ciclo de vida claro | Datos remotos (TanStack Query) o estado de una sola pantalla |
| Context | Dependencia de un subárbol (tema, sesión) con cambios poco frecuentes | Estado de alta frecuencia (posición GPS a 1 Hz) que re-renderiza todo |
| Hook propio | Encapsula estado + efectos + limpieza significativos o se reutiliza | Solo reenvía `useQuery` sin agregar nada |
| Clase | Hay estado interno con ciclo de vida (p. ej. cola de sincronización) | Por costumbre: preferí funciones y objetos literales |
| Contenedor de DI | Prácticamente nunca en una app RN; usar parámetros, módulos y `jest.mock` | — |

## Compartir con evidencia

Antes de mover algo a `shared`, buscá consumidores reales (`rg`). Extraé primero a la funcionalidad, luego al compartido del dominio y recién después al global, cuando sirva a dominios distintos sin conocer sus reglas.

## Excepciones

Si una necesidad obliga a romper una regla (p. ej. un SDK que exige inicializarse en `app/_layout.tsx`), explicá la excepción, mantenela en el mínimo de código y documentala en la entrega. No reescribas la app para alcanzar una estructura ideal fuera del pedido.
