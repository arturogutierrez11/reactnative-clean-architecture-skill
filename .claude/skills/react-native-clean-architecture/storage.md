# Almacenamiento local

## Elegir por tipo de dato

| Dato | Solución | Por qué |
|---|---|---|
| Tokens, credenciales, claves | `expo-secure-store` | Keychain (iOS) / Keystore (Android) |
| Preferencias simples no sensibles | `@react-native-async-storage/async-storage` | Simple, asíncrono |
| Clave-valor de alta frecuencia y lectura síncrona | `react-native-mmkv` (si el proyecto lo justifica) | Rendimiento; requiere Development Build |
| Datos relacionales, colas offline, volumen | `expo-sqlite` (con o sin ORM) | Consultas, transacciones |
| Archivos (imágenes, PDFs) | `expo-file-system` | Directorios de la app |
| Cache de queries | `persistQueryClient` sobre AsyncStorage/MMKV | Solo si hace falta arranque offline |

No agregues más de un mecanismo para el mismo tipo de dato.

## SecureStore

API: `setItemAsync(key, value, options?)`, `getItemAsync(key, options?)`, `deleteItemAsync(key, options?)`; versiones síncronas `getItem`/`setItem` bloquean el hilo JS (evitarlas en render).

- Claves: solo alfanuméricos, `.`, `-` y `_`.
- Valores: strings. Algunos iOS antiguos rechazan valores grandes (~2 KB): no guardar objetos grandes; manejar errores nativos.
- `requireAuthentication: true` protege con biometría (en iOS pide al leer/actualizar; en Android en todas las operaciones). Las claves pueden invalidarse si cambian las biometrías: tratar ese error como "volver a iniciar sesión".
- Android: los datos no sobreviven a la desinstalación. iOS: el Keychain puede sobrevivir a una reinstalación; no confiar en eso, y considerar limpiar tokens en el primer arranque tras reinstalar (flag en AsyncStorage que sí se borra).
- Backup de Android: el plugin `configureAndroidBackup` (default `true`) excluye SecureStore del backup automático; verificá la configuración si cambiás reglas de backup.

## Wrapper en shared

```ts
// src/shared/storage/secureStorage.ts
import * as SecureStore from 'expo-secure-store';

export const secureStorage = {
  get: (key: string) => SecureStore.getItemAsync(key),
  set: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  remove: (key: string) => SecureStore.deleteItemAsync(key),
};
```

El wrapper es útil porque centraliza el mock en tests y permite cambiar opciones globales. Las **claves** pertenecen a cada dominio (`auth.refreshToken` en `domains/auth/session/session.persistence.ts`), no a `shared`.

## AsyncStorage

- Prefijar claves por dominio (`profile.preferences`).
- Guardar JSON versionado (`{ v: 1, data }`) y validar con Zod al leer: los datos persistidos son entrada externa tras una actualización de la app.
- Nunca datos sensibles ni tokens.

## SQLite

- Para colas offline, historial de ubicaciones, catálogos locales.
- Migraciones versionadas desde el primer día.
- Acceso desde un repositorio en la funcionalidad (aquí sí aporta: combina persistencia local con sincronización).

## Limpieza

- Logout: borrar tokens, datos del usuario, cache de queries persistido y archivos privados.
- Definir retención para datos con información personal (ubicaciones, fotos).

## Testing

- Mock de `expo-secure-store` y AsyncStorage (`@react-native-async-storage/async-storage/jest/async-storage-mock`); verificá el path del mock en la versión instalada.
