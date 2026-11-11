# Checklist de implementación y revisión

Usar sobre el alcance modificado. Marcar cada punto como pasa, falla o no aplica. No crear archivos solo para completar casillas.

## Arquitectura

- [ ] Dominio y funcionalidad identificados por propósito.
- [ ] Rutas en `src/app` finas: renderizan pantallas del dominio, sin lógica.
- [ ] Código funcional en `src/domains/<domain>/<feature>`.
- [ ] `shared` sin reglas de negocio ni imports de `domains`/`app`.
- [ ] Sin imports a internals de funcionalidades hermanas ni ciclos.
- [ ] Sin carpetas vacías, barrels automáticos, interfaces de una sola implementación (salvo fronteras nativas), stores o hooks innecesarios.
- [ ] Naming y alias consistentes ([naming-conventions.md](naming-conventions.md)).

## Plataformas y nativo

- [ ] APIs del dispositivo consumidas vía adaptador/hook, no desde componentes.
- [ ] Comportamiento iOS y Android considerado; diferencias documentadas.
- [ ] Safe areas, teclado, Back de Android.
- [ ] Permisos: undetermined, granted, denied con y sin `canAskAgain`, servicios desactivados, cambio desde Configuración.
- [ ] Textos de permisos iOS y plugins configurados.
- [ ] Requisito de Development Build identificado y comunicado.
- [ ] Sin cambios manuales en carpetas nativas generadas por prebuild.

## Datos y estado

- [ ] Sin HTTP en componentes; service + hook.
- [ ] Respuestas externas validadas; DTO mapeado cuando difiere del modelo.
- [ ] Errores normalizados; UI sin mensajes técnicos.
- [ ] Cancelación y timeout; sin reintentos de mutaciones no idempotentes.
- [ ] Una sola fuente de verdad; sin datos remotos duplicados en stores.
- [ ] Invalidación de queries tras mutaciones.

## Seguridad

- [ ] Sin secretos en bundle, `EXPO_PUBLIC_*` ni repo.
- [ ] Tokens solo en SecureStore; limpiados al logout.
- [ ] Logs sin tokens, datos personales ni coordenadas.
- [ ] Deep links y payloads validados.

## UI y UX

- [ ] Estados relevantes: loading, vacío, error, permiso denegado, offline, no autorizado.
- [ ] Tokens del tema; dark mode.
- [ ] Accesibilidad: roles, labels, área táctil, escala de fuente, Reduce Motion.
- [ ] Prevención de doble envío; formularios conservan datos ante error.

## Rendimiento

- [ ] Listas virtualizadas con keys estables.
- [ ] Listeners, sensores y watchers limpiados.
- [ ] Memoización solo con justificación.

## Verificación real

1. Detectar scripts y package manager del lockfile.
2. Typecheck, lint, tests relacionados. No usar `--fix` fuera del alcance.
3. `npx expo install --check` / `npx expo-doctor` si cambiaron dependencias o config.
4. Agregar tests de comportamiento proporcionales: permisos denegados, errores de API, estados de UI.
5. Ejecutar en simulador/emulador/dispositivo cuando el entorno lo permite y el cambio lo requiere.
6. Revisar el diff completo.

## Entrega

1. Qué se implementó.
2. Archivos creados/modificados.
3. Decisiones técnicas y alternativas descartadas.
4. Dependencias agregadas (con versión resuelta por `npx expo install`).
5. Verificaciones ejecutadas → resultado.
6. Pendientes, incluido lo que requiere dispositivo.
7. Diferencias de comportamiento iOS/Android.

Distinguí revisión estática de ejecución real. Nunca declares verificado en una plataforma lo que no se ejecutó en ella.
