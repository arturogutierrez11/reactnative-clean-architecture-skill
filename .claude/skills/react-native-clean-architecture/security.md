# Seguridad mobile

**Principio:** el cliente móvil es un entorno no confiable. Cualquier cosa en el bundle o en el dispositivo puede ser leída o modificada. Ninguna configuración del cliente reemplaza los controles del backend (autenticación, autorización, validación, rate limiting).

Aplicá buenas prácticas desde el inicio; no agregues controles complejos (pinning, detección de root) sin un requisito de negocio o de riesgo concreto.

## Secretos y configuración

- Nada secreto en el bundle: ni `EXPO_PUBLIC_*`, ni constantes, ni `app.config.ts` `extra`. Todo eso es extraíble.
- Claves "públicas" por diseño (Maps API key restringida por bundle id, publishable key de pagos, DSN de crash reporting) pueden ir al cliente, con restricciones configuradas en el proveedor.
- Integraciones que requieren secreto → endpoint de tu backend.
- Secretos de build (credenciales de tiendas, tokens de EAS, keystores) en EAS secrets / secretos del CI. Nunca en el repo.
- No leer ni imprimir `.env`, keystores ni `credentials.json` durante el trabajo.

## Credenciales y sesión

- Tokens solo en SecureStore ([storage.md](storage.md)).
- Access tokens de vida corta; refresh rotado y revocable.
- Limpiar todo al logout.
- Pantallas sensibles: considerar ocultar contenido en el app switcher (bloqueo al pasar a background) si el producto lo requiere.

## Entradas y respuestas

- Validar formularios (UX) y **también** en backend.
- Validar respuestas de API, datos persistidos y parámetros de navegación con Zod.
- Deep links y payloads de notificaciones son entrada no confiable: validar, no ejecutar acciones destructivas automáticamente, exigir sesión para datos privados, no abrir URLs arbitrarias recibidas (`Linking.openURL` solo con allowlist de dominios).
- WebViews: allowlist de orígenes, no inyectar tokens en páginas de terceros, `originWhitelist` restrictivo, deshabilitar lo que no se use.

## Red

- HTTPS siempre. iOS ATS y Android cleartext están bloqueados por defecto en producción; no abrir excepciones globales (`NSAllowsArbitraryLoads`, `usesCleartextTraffic`) para "hacer andar" un entorno.
- Certificate pinning solo si el modelo de amenazas lo exige: requiere plan de rotación y puede dejar la app inutilizable si se hace mal.

## Logs y observabilidad

- Nunca loguear tokens, contraseñas, datos personales, coordenadas, contenido de formularios, headers `Authorization` ni bodies completos.
- Scrubbing en el SDK de crash reporting (`beforeSend`). Ver [observability.md](observability.md).
- En producción, `console.*` puede quedar en logs del sistema accesibles: quitar o silenciar.

## Archivos

- Guardar en directorios privados de la app.
- Validar tipo/tamaño de archivos elegidos por el usuario antes de subir.
- No guardar documentos sensibles en cache compartida ni galería sin consentimiento.

## Permisos

- Mínimos y en contexto ([device-permissions.md](device-permissions.md)).
- Revisar el manifest final de Android y quitar permisos agregados por librerías que no se usan (`android.blockedPermissions`).

## Dispositivos comprometidos

- Root/jailbreak no se detecta de forma confiable desde JS. Para apps de alto riesgo (finanzas, salud), usar atestación del sistema (App Attest / Play Integrity) validada en el backend.
- No basar decisiones de seguridad en checks del cliente.

## Dependencias

- Instalar solo librerías mantenidas, con uso real, revisando permisos nativos que agregan.
- `npm audit` (o equivalente) periódicamente; revisar el lockfile en PRs.
- No ejecutar scripts de instalación de paquetes desconocidos sin revisar.

## Código

- Sin `eval`/`new Function` con datos externos.
- Sin `dangerouslySetInnerHTML`-equivalentes en WebViews con HTML de usuario sin sanitizar.
- Ofuscación (Hermes bytecode ya dificulta lectura) no es seguridad.

## Checklist rápido

- [ ] Ningún secreto en bundle ni repo.
- [ ] Tokens en SecureStore, limpiados al logout.
- [ ] Datos externos validados.
- [ ] Deep links y notificaciones validados.
- [ ] Logs sin datos sensibles.
- [ ] HTTPS y sin excepciones de ATS/cleartext.
- [ ] Permisos mínimos.
