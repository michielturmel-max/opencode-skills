---
name: android-apk-no-gradle
description: Use when building or modifying an Android APK without Gradle - aapt2 resource compilation, javac against android.jar, d8 dexing, apksigner, Docker build image, manifest and resources.
---

Build an Android APK with SDK tools only (no Gradle): aapt2 → javac → d8 →
aapt add → zipalign → apksigner. Reference: `news-android`, one script
`scripts/build-apk.sh` does the whole chain, `set -euo pipefail`.

Names below come from that reference app — substitute your own: package id
`nl.michiel.nieuws`, output `dist/nieuws.apk`, the keytool `-dname`, and the
Docker image tag `android-news:build`.

## SDK location

- `SDK="${ANDROID_HOME:-/opt/android-sdk}"`; tools in `$SDK/build-tools/35.0.0`
  (`aapt2`, legacy `aapt`, `d8`, `zipalign`, `apksigner`), platform jar
  `$SDK/platforms/android-35/android.jar`.
- Script aborts early: `[ -d "$BT" ] || exit 1`.

## Project layout

```
app/src/main/AndroidManifest.xml        hand-written manifest
app/src/main/java/nl/michiel/nieuws/*.java
app/src/main/res/{drawable,mipmap-anydpi-v26,values}/   hand-written XML
debug.keystore                          repo root, stable signing key
scripts/build-apk.sh                    whole build
build/                                  wiped and recreated each run
dist/                                   final signed APK
```

No layout XML — UI is built in Java (`setContentView(buildLayout())`).
Manifest: `package="nl.michiel.nieuws"`, permissions `INTERNET` and
`ACCESS_NETWORK_STATE`, one launcher activity `.MainActivity`.

## Build pipeline (exact commands)

`OUT="$ROOT/build"`, wiped then `mkdir -p "$OUT"/{gen,classes,dex,apk}`.

1. Resources → flat + link (generates R.java):

   ```sh
   "$BT/aapt2" compile --dir "$APP/src/main/res" -o "$OUT/gen/res.zip"
   "$BT/aapt2" link -o "$OUT/apk/base.apk" \
     -I "$PLATFORM/android.jar" \
     --manifest "$APP/src/main/AndroidManifest.xml" \
     --java "$OUT/gen" \
     --min-sdk-version 26 --target-sdk-version 35 \
     --version-code 1 --version-name 1.0.0 \
     "$OUT/gen/res.zip"
   ```

2. javac — sources are app java plus generated `$OUT/gen/**/R.java`:

   ```sh
   find "$APP/src/main/java" "$OUT/gen" -name '*.java' > "$OUT/sources.txt"
   javac -classpath "$PLATFORM/android.jar" -d "$OUT/classes" \
     -encoding UTF-8 -nowarn @"$OUT/sources.txt"
   ```

3. d8 dexing:

   ```sh
   find "$OUT/classes" -name '*.class' > "$OUT/classes.txt"
   "$BT/d8" --release --min-api 26 --lib "$PLATFORM/android.jar" \
     --output "$OUT/dex" @"$OUT/classes.txt"
   ```

4. Dex into APK + align. The dex must be added from inside the APK dir, else
   `aapt` writes the entry as an absolute path (`/work/build/dex/classes.dex`)
   and the package parser won't find `classes.dex`:

   ```sh
   cp "$OUT/dex/classes.dex" "$OUT/apk/classes.dex"
   cd "$OUT/apk"
   "$BT/aapt" add -f base.apk classes.dex >/dev/null
   rm -f classes.dex
   "$BT/zipalign" -f 4 base.apk base-aligned.apk
   ```

5. Signing — one repo-root `debug.keystore`, created once and kept in git:
   a fresh key each build causes `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, and the
   SDK's own debug.keystore is per-machine unique:

   ```sh
   keytool -genkeypair -v -keystore "$ROOT/debug.keystore" \
     -storepass android -keypass android \
     -alias androiddebugkey -keyalg RSA -keysize 2048 -validity 10000 \
     -dname "CN=Nieuws Debug, OU=Dev, O=michiel, C=NL"
   "$BT/apksigner" sign \
     --ks "$KEY" --ks-pass pass:android --key-pass pass:android \
     --ks-key-alias androiddebugkey \
     --min-sdk-version 26 \
     --out "$DIST/nieuws.apk" base-aligned.apk
   "$BT/apksigner" verify --print-certs "$DIST/nieuws.apk" | head -3
   ```

Output: `dist/nieuws.apk`, ~33 kB signed (plus a `.idsig` alongside);
install with `adb install -r dist/nieuws.apk`.

## Docker build image

Image `android-news:build` (Ubuntu 24.04 base), `WORKDIR /work`, env
`ANDROID_HOME`/`ANDROID_SDK_ROOT=/opt/android-sdk`,
`JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64` (PATH includes
`cmdline-tools/latest/bin`). It installs: apt
`openjdk-17-jdk-headless unzip zip curl ca-certificates`; Android
`cmdline-tools` unzipped to `/opt/android-sdk/cmdline-tools/latest`,
licenses accepted, then
`sdkmanager "platforms;android-35" "build-tools;35.0.0"`.

Build (repo mounted at /work, root-owned outputs):

```sh
docker run --rm -v "$PWD:/work" android-news:build bash /work/scripts/build-apk.sh
```

## Tests (no device; run on host JVM inside the image)

Both scripts compile the needed production classes against
`android.jar` + `kxml2.jar` (override via `KXML_JAR`, default
`~/.cache/kxml2.jar`), generate a `Main.java` in a tmpdir, then `java`.
Reason: `android.util.Xml.newPullParser()` is a `Stub!` stub in `android.jar`,
so `Feed.setParserFactory(...)` is swapped to `org.kxml2.io.KXmlParser`
(the reference implementation Android itself uses); parse logic is identical.

```sh
# parser vs saved feed files (mount ~/.cache/feeds:/feeds)
docker run --rm -v "$PWD:/work" -v ~/.cache/feeds:/feeds \
  -v ~/.cache/kxml2.jar:/kxml2.jar:ro -e KXML_JAR=/kxml2.jar \
  android-news:build bash /work/scripts/test-feed.sh /feeds/tweakers.xml /feeds/lobsters.xml

# full chain vs live feeds (Fetcher + Feed)
docker run --rm -v "$PWD:/work" -v ~/.cache/kxml2.jar:/kxml2.jar:ro \
  -e KXML_JAR=/kxml2.jar android-news:build bash /work/scripts/test-live.sh
```

`scripts/probe-gate.sh` is a separate curl probe against DPG Media's consent
gate (Tweakers article pages return an empty ~8 kB page before the content).
