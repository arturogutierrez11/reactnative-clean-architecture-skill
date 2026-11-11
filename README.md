# React Native Clean Architecture Skill

Skill para Claude Code que establece un estándar de ingeniería para aplicaciones **React Native + Expo + TypeScript** organizadas por **dominio → funcionalidad → responsabilidad**, con iOS y Android como plataformas de primera clase.

No es un boilerplate ni una plantilla rígida: es una metodología que Claude aplica al crear apps, agregar funcionalidades, refactorizar, auditar, integrar capacidades nativas y preparar publicaciones, haciendo crecer la arquitectura junto con la complejidad real.

## Qué problemas resuelve

- Pantallas que mezclan UI, permisos, sensores, HTTP y persistencia.
- APIs del dispositivo (ubicación, cámara, biometría, notificaciones) acopladas a componentes e imposibles de testear.
- Permisos pedidos sin contexto, `canAskAgain` ignorado, background location prometido sin garantías del sistema operativo.
- Tokens en AsyncStorage, secretos en `EXPO_PUBLIC_*`, coordenadas en logs.
- Lógica de autenticación duplicada en layouts y loops de redirección.
- Sobreingeniería: repositorios, interfaces, casos de uso y stores para operaciones triviales.
- Afirmaciones de "funciona en iOS/Android" sin haberlo ejecutado.

## Arquitectura

```text
src/
├── app/       # Expo Router: rutas, layouts, providers, guards de sesión
├── domains/   # dominio → funcionalidad → responsabilidad
│   └── location/
│       └── get-current-location/
│           ├── hooks/ services/ components/ screens/ tests/
└── shared/    # solo capacidades reutilizables sin negocio
    ├── api/ config/ errors/ observability/
    ├── native/   # adaptadores del dispositivo
    ├── storage/ theme/ ui/
```

Dependencias permitidas: `app → domains → shared`. `shared` nunca importa `domains` ni `app`. Las APIs del dispositivo se consumen mediante adaptadores; los componentes no importan módulos nativos.

## Stack recomendado

| Área | Preferencia |
|---|---|
| Plataforma | Expo (Development Builds cuando hay nativo) |
| Lenguaje | TypeScript strict |
| Navegación | Expo Router |
| Estado remoto / global | TanStack Query / Zustand |
| Formularios y validación | React Hook Form + Zod |
| Almacenamiento seguro | Expo SecureStore |
| Ubicación | Expo Location + Task Manager |
| Testing | Jest + React Native Testing Library, Maestro para E2E |
| Builds | EAS Build / Submit / Update |

Son preferencias, no requisitos: la skill obliga a verificar versiones instaladas, compatibilidad con Expo SDK, React Native y New Architecture antes de usar cualquier API, y funciona con proyectos React Native CLI sin forzar migraciones.

## Principios

1. La arquitectura crece con la complejidad real: dos archivos para una funcionalidad simple, capas solo cuando aportan.
2. Abstracciones solo en fronteras reales (dispositivo, proveedores externos).
3. Permisos en contexto, con todos sus estados.
4. Seguridad desde el inicio, sin controles innecesarios.
5. Honestidad en la verificación: qué se ejecutó y dónde.

## Instalación

Cloná el repositorio en una carpeta de herramientas:

```bash
git clone https://github.com/arturogutierrez11/reactnative-clean-architecture-skill.git
```

### Por proyecto

Desde la raíz del proyecto móvil (reemplazá `/ruta/` por la ubicación del clon):

```bash
mkdir -p .claude/skills
cp -R /ruta/reactnative-clean-architecture-skill/.claude/skills/react-native-clean-architecture .claude/skills/
```

Versionalo junto al proyecto para que todo el equipo use la misma metodología.

### Para tu usuario (todos tus proyectos)

```bash
mkdir -p ~/.claude/skills
cp -R /ruta/reactnative-clean-architecture-skill/.claude/skills/react-native-clean-architecture ~/.claude/skills/
```

Si ya existe una skill con ese nombre, compará las diferencias antes de reemplazarla. Una skill del proyecto y una del usuario con el mismo nombre pueden convivir; preferí una sola fuente para evitar versiones distintas.

Opcional: copiá [CLAUDE.example.md](CLAUDE.example.md) como `CLAUDE.md` en la raíz del proyecto y completá los datos del proyecto.

## Uso con Claude Code

Claude la activa automáticamente cuando la tarea coincide con su descripción (React Native, Expo, funcionalidades nativas, etc.). Para invocarla explícitamente:

```text
/react-native-clean-architecture Agregá una pantalla que muestre mi ubicación actual.
```

```text
/react-native-clean-architecture Analizá este repositorio sin modificar archivos.
```

```text
/react-native-clean-architecture Refactorizá el login preservando su comportamiento.
```

```text
/react-native-clean-architecture Prepará la app para enviar a TestFlight e internal testing.
```

### Aplicación nueva

```text
/react-native-clean-architecture Creá una app Expo nueva llamada "Acme" con login y una pantalla de perfil.
```

Claude crea el proyecto con la plantilla oficial vigente, configura TypeScript strict, `src/app` y el alias `@/`, y agrega dependencias solo cuando la primera funcionalidad las necesita.

### Aplicación existente

Claude inspecciona `package.json`, lockfile, `app.config.ts`, navegación, carpetas nativas y tests; respeta el router y el workflow existentes, y migra por funcionalidad sin reorganizar todo el proyecto.

## Contenido

La entrada es [SKILL.md](.claude/skills/react-native-clean-architecture/SKILL.md): contrato, procedimiento, tabla de referencias y criterios de finalización. Las referencias se cargan según la tarea:

| Área | Referencias |
|---|---|
| Arquitectura | `architecture`, `folder-structure`, `responsibilities`, `naming-conventions`, `inconsistencies`, `checklist` |
| UI | `components`, `design-system`, `animations`, `performance` |
| Navegación y datos | `navigation`, `state-management`, `data-fetching`, `authentication`, `storage`, `offline` |
| Nativo | `native-platforms`, `native-modules`, `geolocation`, `device-permissions` |
| Calidad y entrega | `security`, `testing`, `configuration`, `builds-and-deployment`, `observability` |

[examples.md](.claude/skills/react-native-clean-architecture/examples.md) contiene una app mínima coherente: cliente HTTP con refresh de tokens, sesión con SecureStore y guards de Expo Router, perfil remoto, componentes reutilizables, ubicación actual con adaptador nativo y manejo completo de permisos, tests y un refactor antes/después.

### Cómo se validaron los ejemplos

Los bloques de código de los ejemplos se extrajeron a un proyecto con Expo SDK 58 (React Native 0.88.1, TypeScript 6, Zod 4, TanStack Query 5, Zustand 5, React Hook Form 7, jest-expo 58, React Native Testing Library 14). Se ejecutó `tsc --noEmit` sin errores, 13 tests de Jest pasaron, `expo export` generó bundles Hermes para iOS y Android y `expo-doctor` pasó 20/20 checks (noviembre 2026).

**No** se ejecutaron en simulador, emulador ni dispositivo. Los comportamientos de permisos, ubicación y background deben validarse en dispositivos reales.

## Mantenimiento

- Al salir un nuevo Expo SDK, revisá su changelog y actualizá las secciones afectadas (permisos, notificaciones, APIs de ubicación, configuración de Jest).
- Volvé a validar `examples.md` con el SDK nuevo antes de publicar cambios.
- Para actualizar una instalación: `git pull` en el clon y volvé a copiar la carpeta de la skill.

## Limitaciones

- Las APIs citadas se verificaron contra la documentación de Expo SDK 58; versiones anteriores o posteriores pueden diferir. La skill instruye a Claude a verificar la versión instalada.
- No prescribe proveedores de observabilidad, mapas ni autenticación.
- Las guías de Bluetooth, NFC y otros módulos comunitarios indican criterios, no implementaciones probadas.
- Las políticas de App Store y Google Play cambian: verificá los requisitos vigentes antes de publicar.

## Contribuciones

1. Abrí un issue describiendo el problema o la mejora.
2. Mantené el estilo: reglas concretas, cuándo aplicar y cuándo evitar, ejemplos reales.
3. Si cambiás código de `examples.md`, verificalo con TypeScript y Jest en un proyecto Expo real e indicá versiones.
4. Verificá que los enlaces relativos y anclas sigan funcionando.

## Licencia

[MIT](LICENSE)
