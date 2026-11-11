# Responsabilidades

| Unidad | Debe hacer | No debe hacer |
|---|---|---|
| Ruta `src/app/**.tsx` | Leer parámetros, renderizar la pantalla del dominio, configurar `Stack.Screen options` | Fetch, permisos, reglas, formularios completos |
| Layout `_layout.tsx` | Navegador (Stack/Tabs), providers del alcance, guards de sesión, splash | Implementar funcionalidades o lógica de autenticación duplicada |
| Pantalla (`screens/`) | Componer hooks y componentes de una funcionalidad, decidir qué estado se muestra | Llamar APIs del dispositivo o HTTP directamente |
| Componente | Render, interacción, accesibilidad, estado visual local | Importar `expo-*` de infraestructura, cliente HTTP o stores de otro dominio |
| Hook | Estado + efectos + limpieza, adaptar servicios/queries a la UI | Contener reglas de negocio extensas o ser un wrapper vacío |
| Caso de uso | Orquestar varios pasos (permiso → sensor → API → persistencia) con resultado tipado | Conocer React, navegación o textos de UI |
| Service | Una operación externa con contrato explícito: request, validación, mapeo DTO → modelo | Importar componentes/hooks o decidir mensajes visuales |
| Adaptador nativo | Traducir una API del dispositivo a tipos y errores de la app | Contener negocio o decidir cuándo pedir permisos para una pantalla |
| Schema (Zod) | Validar datos en runtime (formularios, respuestas, params, deep links) | Hacer I/O |
| Tipo | Contrato estático | Reemplazar validación de datos externos |
| Store | Estado cliente compartido con ciclo de vida definido | Duplicar datos remotos o guardar tokens |
| Task de background | Recibir eventos del SO y delegar en un caso de uso/servicio sin React | Montar componentes o depender de estado en memoria de la UI |
| Config plugin | Modificar configuración nativa de forma reproducible | Contener lógica de runtime |

## Flujo típico

```text
Ruta ─→ Pantalla ─→ Hook ─→ (Caso de uso) ─→ Service / Adaptador ─→ API / Dispositivo
                     │
                     └─→ Componentes (props tipadas, sin infraestructura)
```

## Criterios

- Una lectura simple: un service que hace request, valida con Zod y mapea, más un hook con `useQuery`. No exigir repositorio + caso de uso + mapper + service para el mismo fetch.
- Separar cuando haya reglas independientes, varios disparadores, proveedores alternativos reales o necesidad de testear la orquestación sin dispositivo.
- La autorización real ocurre en el backend. La app nunca decide privilegios por su cuenta ni confía en roles guardados localmente para proteger datos. Ver [security.md](security.md).
- Mensajes para el usuario se deciden en presentación a partir de errores tipados ([data-fetching.md](data-fetching.md#errores)). Services y adaptadores devuelven códigos, no textos de UI.
- Antes de extraer código, identificá qué cambiaría independientemente y quién lo consumiría. Sin respuesta concreta, mantené la implementación local.
