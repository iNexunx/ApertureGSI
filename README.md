# ApertureGSI

Build automatizado (GitHub Actions) de las apps de cámara de **LineageOS** para incluirlas
en una **custom ROM / GSI**.

Compila desde [`LineageOS/android_packages_apps_Aperture`](https://github.com/LineageOS/android_packages_apps_Aperture):

| Origen | Resultado | package |
|---|---|---|
| `:app` | **Aperture.apk** | `org.lineageos.aperture` |
| `:lens_launcher` | **ApertureLensLauncher.apk** | `com.google.android.apps.googlecamera.fishfood` |
| `rro_overlays/ApertureFrameworksBaseOverlay` | **ApertureFrameworksBaseOverlay.apk** (RRO) | `org.lineageos.aperture.frameworksbaseoverlay` |
| `app/*.xml` | **sysconfig** (2 XML) | — |

El **RRO** es un overlay estático que declara Aperture como escáner QR por defecto del sistema
(`config_defaultQrCodeComponent` → `org.lineageos.aperture/.QrScannerActivity`).

## Variantes (matriz)

Un solo workflow compila las **5** versiones en paralelo. Cada una con su `minSdk`/`targetSdk`,
su `versionName` y su propio release:

| Variante | Android | API (`minSdk`/`targetSdk`) | `versionName`/`versionCode` | Rama fuente |
|---|---|---|---|---|
| `android12` | 12 | 31 / 31 | 12 / 12 | `lineage-20.0` |
| `android12L` | 12L | 32 / 32 | 12.1 / 121 | `lineage-20.0` |
| `android13` | 13 | 33 / 33 | 13 / 13 | `lineage-20.0` |
| `android14` | 14 | 34 / 34 | 14 / 14 | `lineage-21.0` |
| `android15` | 15 | 35 / 35 | 15 / 15 | `lineage-22.2` |

`minSdk` = `targetSdk` = API de la versión (cada APK es para su Android).

El RRO y los 2 XML **no existen** en la rama `lineage-20.0`; se toman de `lineage-22.2`
(el contenido del overlay es idéntico en todas las ramas).

## Cómo se ejecuta

- **Automático:** cada `push` a `main` (compila las 5 variantes).
- **Manual:** **Actions → Build Aperture (Android 12 / 12L / 13 / 14 / 15) → Run workflow**.

## Salida

- **Artifacts** del run: uno por variante (`aperture-<variante>-v<version>-full`) con los 3 APKs
  y los 2 XML.
- **Releases** con tag `<variante>-build-<n>` (p. ej. `android14-build-3`).

## Firma

Los APKs se firman con las **test-keys públicas de AOSP** (de
`LineageOS/android_build/target/product/security`), usando `apksigner` tras `zipalign`.
Esquema V1+V2+V3 (sin `idsig`).

> Las claves privadas oficiales de LineageOS **no son públicas**. Si tu ROM/GSI usa otra clave de
> plataforma, edita el workflow y sube tu propio keystore como Secret.

## Incluir en ROM / GSI

1. Elige la variante que corresponde a tu ROM/GSI (fila de la tabla).
2. Coloca los archivos como prebuilts, p. ej.:
   ```
   product/app/Aperture/Aperture.apk
   product/app/ApertureLensLauncher/ApertureLensLauncher.apk
   product/overlay/ApertureFrameworksBaseOverlay/ApertureFrameworksBaseOverlay.apk
   product/etc/sysconfig/initial-package-stopped-states-org.lineageos.aperture.xml
   product/etc/sysconfig/preinstalled-packages-org.lineageos.aperture.xml
   ```
   (o las rutas equivalentes `system/product/...` en una GSI ya montada).
3. La firma debe **coincidir con la de tu ROM** si quieres permisos de nivel firma; para uso
   normal (cámara + Lens) cualquier test-key sirve.
4. El **RRO** es estático (`android:isStatic="true"`, `targetPackage="android"`): debe ir en la
   partición de overlays (`/product/overlay/`) y quedar declarado en el build (o con idmap
   precompilado) para aplicarse al boot. Sin él, la cámara funciona igual; solo se pierde que el
   escáner QR del sistema abra Aperture.
5. Los 2 XML de `sysconfig/` controlan la preinstalación (que no arranque detenida y en qué
   tipos de usuario se instala). Opcionales si instalas el APK a mano.

## Reproducir en local

```bash
git clone -b lineage-22.2 https://github.com/LineageOS/android_packages_apps_Aperture src
cd src
sed -i -E 's/minSdk = [0-9]+/minSdk = 35/'   app/build.gradle.kts lens_launcher/build.gradle.kts
sed -i -E 's/targetSdk = [0-9]+/targetSdk = 35/' app/build.gradle.kts lens_launcher/build.gradle.kts
sed -i -E 's/versionCode = [0-9]+/versionCode = 15/' app/build.gradle.kts lens_launcher/build.gradle.kts
sed -i -E 's/versionName = "[^"]*"/versionName = "15"/' app/build.gradle.kts lens_launcher/build.gradle.kts
./gradlew :app:assembleRelease :lens_launcher:assembleRelease -x lintVitalRelease
```

## Licencia

El código compilado pertenece a The LineageOS Project (Apache-2.0). Este repo solo contiene el
pipeline de build.
