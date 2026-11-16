# CarMusic Diagnostic

App Android (Kotlin, minSdk 23, targetSdk 30) que **diagnostica qué metadatos multimedia puede leer una app normal** en el teléfono mientras este reproduce música y está conectado por Bluetooth a la pantalla del coche (ZXDZ_01, Android 11.1).

La app corre **en el teléfono**. No se comunica con la pantalla del coche: mide lo que el sistema Android del teléfono expone, que es el punto de partida de lo que puede salir por Bluetooth.

## Compilar el APK

Requisitos: Android Studio (Koala o superior) o JDK 17 + Android SDK 34.

```
./gradlew assembleDebug
```

Resultado: `app/build/outputs/apk/debug/CarMusic-Diagnostic.apk`
(`./gradlew assembleRelease` genera `app/build/outputs/apk/release/CarMusic-Diagnostic.apk`, firmado con la clave debug para poder instalarlo.)

En Android Studio: *File > Open* sobre esta carpeta, esperar al sync de Gradle y *Build > Build APK(s)*.

## Permisos

| Elemento | Por qué |
|---|---|
| Ningún `<uses-permission>` | La app no usa Internet, ni Bluetooth directo, ni almacenamiento. |
| Estado Bluetooth | Se deduce con `AudioManager.getDevices()` (salidas A2DP/SCO), que no requiere permisos. |
| `NotificationListenerService` + `BIND_NOTIFICATION_LISTENER_SERVICE` | Android exige un listener habilitado por el usuario para llamar a `MediaSessionManager.getActiveSessions()`. El servicio no lee notificaciones. El usuario lo activa a mano en Ajustes. |
| `<queries>` | Visibilidad de paquetes en Android 11+ para mostrar el nombre de la app de música. |

No se usan APIs privadas ni permisos de sistema.

## Instalar y probar

1. Copia `CarMusic-Diagnostic.apk` al teléfono e instálalo (permite «instalar apps de origen desconocido» para el gestor de archivos/navegador que uses).
2. Abre **CarMusic Diagnostic**. Pulsa **Acceso a notificaciones** y activa «CarMusic Diagnostic».
   - Android 13+: si el interruptor está gris, ve a Ajustes > Aplicaciones > CarMusic Diagnostic > menú ⋮ > **Permitir ajustes restringidos** y repite.
3. Conecta el teléfono a la pantalla del coche por Bluetooth (audio multimedia).
4. Reproduce una canción en **YouTube Music**.
5. Vuelve a CarMusic Diagnostic y pulsa **Actualizar** (también se actualiza solo al cambiar la sesión).
6. Lee el panel de detalles: cada clave de carátula marca ✔ o ✘ y la conclusión final lo resume.
7. Para ver errores: `adb logcat -s CarMusicDiag`.

Prueba extra recomendada: instala el mismo APK **en la pantalla del coche**. Si allí no aparece ninguna sesión del teléfono, confirma que el audio Bluetooth lo gestiona el módulo/MCU de la pantalla y no el Android visible para apps.

## Selección de la sesión (v1.1)

1. Sesión del paquete `com.google.android.apps.youtube.music`, aunque esté en pausa.
2. Si no existe, la sesión que esté reproduciendo.
3. Si ninguna reproduce, la primera según la prioridad del sistema.

El paquete que aporta los metadatos se muestra siempre en pantalla («Paquete de la sesión») y en el informe copiado.

## Tipos de carátula

Cada clave se clasifica por separado:

| Tipo | Significado |
|---|---|
| **Bitmap disponible** | Imagen incrustada en la metadata; se muestra en la vista previa. |
| **URI local disponible** | `content://`, `file://`, etc. Se intenta leer; el detalle indica si fue legible. |
| **URI HTTPS disponible** | La app publica una URL remota. **No se descarga** (la app no usa Internet). |
| **Completamente ausente** | Ninguna clave de carátula tiene valor. |

## Cómo interpretar el resultado

- **Bitmap o URI local legible**: el teléfono tiene la imagen. Si la pantalla no la muestra, el límite está en el enlace Bluetooth o en la pantalla.
- **Solo URI HTTPS**: YouTube Music **sí publica** carátula. No se puede concluir que no la proporcione; solo que esta versión no la descarga. Verificarla exigiría una versión con el permiso `INTERNET`, que no se ha añadido.
- **Completamente ausente**: en ese momento la sesión no publica carátula. Cambia de canción y pulsa Actualizar para descartar un estado transitorio.

## Botón «Copiar diagnóstico»

Copia al portapapeles un informe de texto con fecha, dispositivo, estado del acceso a notificaciones, Bluetooth, paquete, campos principales, detalle de cada sesión y clave de carátula, y la conclusión. Pégalo tal cual para compartir el resultado.

## Cambios en v1.1

- Prioridad de sesión: YouTube Music, luego la que reproduce.
- Clasificación Bitmap / URI local / URI HTTPS / ausente; la conclusión ya no afirma que no hay carátula cuando solo existe una URI HTTPS.
- Botón para copiar el diagnóstico completo y paquete visible.
- Refrescos agrupados (300 ms) y sin redibujar ni registrar en Logcat si nada cambió; los avisos de error se registran una sola vez.
- Los callbacks de sesión solo se re-registran si cambia el conjunto de sesiones.
- Sin permisos nuevos.

## Limitaciones conocidas (documentadas a propósito)

1. **Recibir el título por Bluetooth no implica acceso a todo.** AVRCP 1.3 transporta solo texto: título, artista, álbum, número de pista, total, género y duración. La carátula requiere AVRCP 1.6 con Cover Art (canal BIP), que dependen a la vez de la pila Bluetooth del teléfono (versión de Android/fabricante) y de que la pantalla del coche lo implemente. Una app de terceros no puede activar ni sustituir eso.
2. **Una app normal no puede enviar metadatos al coche.** Solo la pila Bluetooth del sistema habla AVRCP; esta app únicamente lee.
3. **Visibilidad de sesiones.** `getActiveSessions()` devuelve solo las sesiones activas. Una sesión sin reproducción reciente puede desaparecer.
4. **URIs remotas.** Si YouTube Music entrega la carátula como URL `https`, la app solo la informa (como «URI HTTPS disponible»); no la descarga porque no usa Internet.
5. **Detección de «relacionada con Bluetooth»** es una heurística: paquete del stack Bluetooth, o sesión reproduciendo mientras hay salida A2DP/SCO activa. Android no etiqueta las sesiones por transporte.
6. **targetSdk 30** es intencional (Android 11). Es válido para instalación directa; no cumple los requisitos actuales de publicación en Google Play.
