# Detección y migración de inconsistencias

Aplica a proyectos existentes, refactors y auditorías.

## Inspección operativa

Comandos orientativos (adaptar a las carpetas reales; omitir las ausentes):

```bash
rg --files src app | head -200
rg -n "from 'expo-(location|camera|secure-store|notifications|local-authentication|sensors)'" src app
rg -n "fetch\(|axios\." src app
rg -n "AsyncStorage\.(setItem|getItem)" src app
rg -n "Platform\.OS" src app | wc -l
rg -n "console\.(log|warn|error)" src app
rg -n "process\.env\." src app
rg -n "create\(\(|createContext\(" src app
```

Leé el contexto de cada resultado antes de clasificarlo. Un exit code sin coincidencias no es un error.

También revisá: `package.json` (dependencias sin uso, duplicadas: dos clientes HTTP, dos librerías de estado), `app.config.ts` (permisos sin uso), si `ios/`/`android/` están versionados con cambios manuales.

## Señales y acciones

| Señal | Riesgo | Acción |
|---|---|---|
| Token en AsyncStorage o store persistido | Seguridad | Migrar a SecureStore (leer el valor viejo una vez, moverlo, borrarlo) — prioridad alta |
| `EXPO_PUBLIC_*` con secreto | Seguridad | Rotar el secreto y moverlo al backend — prioridad crítica; avisar al usuario |
| Coordenadas/tokens en logs | Privacidad | Quitar y agregar scrubbing |
| Pantalla con `fetch` + `useEffect` + parseo | Arquitectura | Extraer service + hook (TanStack Query si está instalado) |
| Componente que importa `expo-location` directamente | Arquitectura/testabilidad | Introducir adaptador y hook de dominio |
| Lógica de auth en varias pantallas/layouts | Bugs de redirección | Centralizar en guard de layout |
| Datos remotos copiados en Zustand | Inconsistencia | Leer de TanStack Query; borrar copia |
| `shared/` con componentes de negocio de un solo dominio | Acoplamiento | Mover al dominio |
| Carpetas globales `services/`, `hooks/` gigantes | Organización | Migrar por propietario funcional |
| `Platform.OS` disperso en muchos componentes | Mantenibilidad | Agrupar en adaptador o archivos por plataforma |
| Cambios manuales en `ios/`/`android/` con CNG | Se pierden en prebuild | Convertir a config plugin |
| Permisos pedidos al iniciar la app | Rechazo en tiendas / UX | Pedir en contexto |
| `ScrollView` + `map` en listas largas | Rendimiento | `FlatList` |
| `any` en respuestas | Bugs en runtime | Schema Zod |
| Barrels que causan ciclos | Arranque lento, ciclos | Imports directos |

## Priorizar

1. Seguridad y privacidad (secretos, tokens, logs).
2. Fallos funcionales y crashes.
3. Permisos y comportamiento por plataforma.
4. Contratos y dependencias (límites de capas).
5. Ubicación y naming.

No convertir una preferencia cosmética en bloqueante. No inventar fallos por ausencia de auth, offline o stores si el producto no los necesita.

## Migración por funcionalidad

1. Registrar entrada (ruta), consumidores y comportamiento actual.
2. Mover una funcionalidad completa a `src/domains/<dominio>/<feature>`, actualizando imports.
3. Mantener temporalmente un reexport de compatibilidad solo si hay muchos consumidores; indicar cuándo retirarlo.
4. Verificar tipos, lint, tests y el flujo en la app.
5. Eliminar código viejo solo sin consumidores.

No mezclar en el mismo cambio: actualización del SDK, migración a Expo Router, migración a `src/`, rediseño visual y reorganización. Cada uno es un PR separado y requiere autorización si no fue pedido.

### Proyectos React Native CLI

- No migrar a Expo sin pedido. Se puede adoptar módulos Expo individuales (`npx install-expo-modules`) si el usuario lo aprueba.
- `ios/`/`android/` son fuente: cambios nativos con revisión de diff.
- Navegación con React Navigation: aplicar los mismos principios (rutas finas, guard único) en el módulo de navegadores.

## Formato de hallazgo

```text
archivo:línea → responsabilidad actual → problema concreto → impacto → corrección propuesta → cómo verificar
```

Diferenciá hechos observados, riesgos inferidos y propuestas. En diagnóstico no modifiques archivos.
