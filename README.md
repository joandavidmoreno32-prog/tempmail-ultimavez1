# TempMail — App Android nativa de correo temporal

Proyecto Android nativo en **Java**, arquitectura limpia
`UI → ViewModel → UseCase → Repository → API/Database`.

## 1. Arquitectura

```
app/
├── data/            (Retrofit, Room, DTOs, RealRepository, MockRepository)
├── domain/          (modelos puros, contratos de repositorio, UseCases)
├── presentation/    (Activities, ViewModels, Adapters, componentes reutilizables)
├── notifications/   (FCM, WorkManager)
├── di/              (ServiceLocator — inyección de dependencias manual)
└── utils/           (Resource, AppError, HtmlSanitizer, SecureTokenStore, AdManager, PremiumManager)
```

- El repositorio se resuelve en `ServiceLocator` y cambia automáticamente entre
  `RealMailboxRepository` y `MockMailboxRepository` según
  `BuildConfig.USE_MOCK_REPOSITORY` (activo por defecto en `debug`).
- No hay lógica de negocio en Activities/Fragments: solo observan LiveData.

## 2. Abrir el proyecto en Android Studio

1. `File → Open` y selecciona la carpeta raíz `temp-mail-app`.
2. Espera a que Gradle sincronice (usa AGP 8.6, requiere Android Studio
   Ladybug o superior y JDK 17).
3. Ejecuta el módulo `app` en un emulador o dispositivo (minSdk 24).
4. Por defecto corre en modo Mock: no necesitas backend para probar la app.

## 3. Configurar el backend real

Cuando tengas tu backend REST desplegado:

1. Edita `app/build.gradle` → `defaultConfig`:
   ```groovy
   buildConfigField "String", "BASE_URL", "\"https://tu-backend.com/\""
   ```
2. En `buildTypes.release`, cambia:
   ```groovy
   buildConfigField "boolean", "USE_MOCK_REPOSITORY", "false"
   ```
3. Implementa en tu backend los endpoints que ya consume `ApiService`:
   - `POST /api/v1/mailbox/create`
   - `GET /api/v1/mailbox/{id}/messages`
   - `GET /api/v1/messages/{id}`
   - `DELETE /api/v1/mailbox/{id}`
   - `POST /api/v1/device/register`
4. El backend es responsable de: crear buzones, recibir correos reales,
   aplicar expiración, eliminar mensajes vencidos, autenticar con el token
   emitido en `mailbox/create`, y aplicar rate limiting.

## 4. Configurar Firebase (notificaciones push)

1. Crea un proyecto en [Firebase Console](https://console.firebase.google.com).
2. Añade una app Android con `applicationId = com.tempmail.app`.
3. Descarga el `google-services.json` real y reemplaza el archivo placeholder
   en `app/google-services.json` (el incluido en este repo es ficticio y solo
   permite que el proyecto compile).
4. Tu backend debe enviar el push con `data.mailboxId` y `data.messageId` para
   que, al tocar la notificación, se abra directamente el correo
   (`EmailDetailActivity`).

## 5. Generar el APK / AAB (obtener el instalable)

Este proyecto se preparó y verificó en un entorno sin Android SDK ni acceso a
red, así que el APK **no viene precompilado** en esta entrega. Hay dos formas
reales de obtenerlo, elige la que te resulte más cómoda:

### Opción A — GitHub Actions (sin instalar nada en tu computadora)

1. Sube esta carpeta a un repositorio de GitHub (puede ser privado).
2. El workflow `.github/workflows/android-build.yml` ya incluido se ejecuta
   automáticamente al hacer push a `main`, o puedes dispararlo a mano desde
   la pestaña **Actions → Build TempMail APK → Run workflow**.
3. Cuando termine (unos 3-5 minutos), entra al run finalizado y descarga el
   artefacto `tempmail-debug-apk` — ahí está tu `app-debug.apk`, listo para
   instalar en un Android (necesitas permitir "orígenes desconocidos" o
   transferirlo por ADB: `adb install app-debug.apk`).
4. Este workflow compila el **debug** (con `USE_MOCK_REPOSITORY=true`, es
   decir, funciona sin backend real, mostrando datos de prueba). Para compilar
   `release` habría que además configurar el keystore de firma como secreto
   de GitHub (ver Opción B, paso de firma).

### Opción B — Android Studio (en tu computadora)

```bash
# Debug (con MockRepository activo)
./gradlew assembleDebug

# Release (requiere firmar con tu keystore, ver app/build.gradle → signingConfigs)
./gradlew bundleRelease
```

1. Abre la carpeta raíz en Android Studio (Ladybug o superior, JDK 17).
2. Al sincronizar, Android Studio **generará automáticamente** el
   `gradle-wrapper.jar` que falta en este paquete (solo incluí
   `gradle-wrapper.properties`, apuntando a Gradle 8.9, porque el binario del
   wrapper no se puede generar sin red). Si prefieres generarlo tú mismo antes
   de abrir el proyecto: instala Gradle una vez (`brew install gradle` /
   `sdk install gradle`) y corre `gradle wrapper --gradle-version 8.9` en la
   raíz del proyecto.
3. Ejecuta el módulo `app` en un emulador o dispositivo, o usa
   `Build → Build Bundle(s) / APK(s) → Build APK(s)` para obtener el archivo
   sin conectar un dispositivo.

Antes de firmar en release:
- Genera tu keystore: `keytool -genkey -v -keystore release.keystore -alias tempmail -keyalg RSA -keysize 2048 -validity 10000`
- Completa `signingConfigs.release` en `app/build.gradle` (usa variables de
  entorno, nunca subas el keystore ni las contraseñas al repositorio).

## 6. Checklist para publicar en Google Play

- [ ] `google-services.json` real (no el placeholder) incluido en el build de release.
- [ ] `BASE_URL` apuntando al backend de producción, `USE_MOCK_REPOSITORY=false`.
- [ ] Política de privacidad real publicada y enlazada (reemplazar el texto de `PrivacyPolicyActivity`).
- [ ] Formulario de "Seguridad de los datos" de Play Console completado (qué datos se recopilan: ninguno personal, solo el buzón temporal).
- [ ] `minifyEnabled true` y `shrinkResources true` verificados en release (ya configurado).
- [ ] Firma con keystore de producción (no el de debug).
- [ ] Icono adaptativo (`ic_launcher`) y capturas de pantalla en modo claro y oscuro.
- [ ] Pruebas en al menos un dispositivo de gama baja (minSdk 24) y uno reciente.
- [ ] Revisar permisos declarados: solo `INTERNET`, `ACCESS_NETWORK_STATE`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED` (no hay permisos de ubicación/contactos).
- [ ] Verificar que no hay tokens ni URLs de desarrollo hardcodeadas.

## 7. Estado de esta entrega

Incluye: arquitectura completa, capa de datos (Retrofit + Room + Mock),
pantallas Home/EmailDetail/Settings/About/PrivacyPolicy/Splash, notificaciones
FCM + limpieza periódica con WorkManager, sanitización de HTML, tests unitarios
y de UI, placeholders desacoplados para AdMob/Premium, ícono de lanzador
(adaptativo + fallback raster en todas las densidades) y acceso a Ajustes desde
Home.

Cambios de esta revisión:
- Se generó el ícono de la app (`mipmap-anydpi-v26` + PNG en mdpi/hdpi/xhdpi/xxhdpi/xxxhdpi),
  que antes faltaba y hubiera impedido compilar/instalar la app.
- Se agregó el botón "⚙ Ajustes" en `HomeActivity`, ya que `openSettings()`
  existía pero no tenía ningún punto de entrada visible.
- `TempMailApplication` ahora programa la limpieza periódica con WorkManager
  desde el primer arranque (antes solo se programaba tras `BOOT_COMPLETED`,
  por lo que nunca se activaba hasta el primer reinicio del dispositivo).
- Se añadió `res/values/dimens.xml` (sección 32 del spec).
- Se agregó `.github/workflows/android-build.yml` para compilar el APK de
  debug en la nube (GitHub Actions) sin necesidad de Android Studio local, y
  `gradle/wrapper/gradle-wrapper.properties` (Gradle 8.9) para cuando lo abras
  en Android Studio.

Pendiente de personalizar antes de producción: `BASE_URL` real, `google-services.json`
real, textos legales definitivos, y (opcionalmente) integrar el SDK real de AdMob
en `AdManager` si se activa `ENABLE_ADS`. También queda como mejora opcional
extraer `ExpirationTimerView`, `PrimaryButton`, `RefreshButton` y `LoadingView`
como componentes reutilizables independientes (sección 29 del spec); hoy esa
lógica vive inline en `HomeActivity` y sus layouts, funcionalmente completa
pero sin esa capa extra de reutilización.