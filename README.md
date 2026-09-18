# Соучастник

Помогает разобраться какой срок вы получите за то что пишете в интернете. Также помечает плашками иноагентов и запрещенные соцсети. Создатель - https://t.me/neuromikhail


<img width="348" height="384" alt="photo_5249203751392845220_y" src="https://github.com/user-attachments/assets/3c5b5409-845e-4954-bb6b-6a612488fd60" />
<img width="324" height="276" alt="photo_5249203751392845221_y" src="https://github.com/user-attachments/assets/0b24ad33-231d-4a6c-b9b0-fbf2ab90c5b2" />
<img width="324" height="278" alt="photo_5249203751392845222_y" src="https://github.com/user-attachments/assets/828e34d7-bc58-4828-acae-7b9967e78b49" />


Это не юридическая консультация. Настоящий суд смешнее. 
AI can make mistakes 

## Приватность

**У приложения нет разрешения `android.permission.INTERNET`.** и нет доступа в интернет. Все ваши данные остаются на устройстве.

Проверьте сами
```bash
aapt dump permissions app-release.apk
```

ИИ-модель работает целиком на устройстве. Поля с паролями не разбираются.

только для Android

## Как это работает

| | |
|---|---|
| Модель | квантованная Qwen3.5-0.8B локально на устройстве на llama.cpp|
| Кандидаты | словарь триггеров статей `assets/triggers.json` сужает и ускоряет поиск, потом ИИ модель решает есть ли состав преступления |
| Требования | Android 9+, arm64. Проверено на 4 ГБ RAM (Honor 9X); на процессорах без dotprod (Cortex-A53/A73: Kirin 710, Snapdragon 680/662, Helio G35) работает, но разбор фразы занимает 15–20 с вместо 4–6. На Snapdragon 8 Elite (Galaxy S25, Hexagon v79) слои уезжают на NPU: медиана ~0.5 с, повторный набор той же фразы ~80 мс |

Тумблер в строке выключает разбор 

## Границы

Приложение констатирует, а не помогает обойти. Выдуманных статей в
справочнике нет: статьи за использование VPN не существует, за использование
Telegram тоже.

Слова в `assets/triggers.json` — триггеры для статей. Тк за них меня самого могут посадить в репозитории их нет, могу выдать по запросу

> за идею спасибо ув. Кибердед и ув. Лука Ебков

## Сборка релиза

Отдельного скрипта нет: релизный APK собирается на машине, где руками лежат
веса, ассеты и дерево llama.cpp. В git этого нет (см. `.gitignore`).

Нужны JDK 17, Android SDK, NDK, CMake 3.31.6 из SDK.

1. **Веса.** GGUF Q4_0 кладётся как
   `app/src/main/jniLibs/arm64-v8a/libmodel-qwen35-08b-q40.so`.
   Это не библиотека: файл так называется, чтобы установщик распаковал его
   в `nativeLibraryDir` и llama могла `mmap`. Без него приложение собирается,
   но строка показывает «модель не установлена». Спайк — Unsloth Q4_0,
   свой квант — `bash tools/make_gguf.sh`. Подробности:
   [`app/src/main/jniLibs/README.md`](app/src/main/jniLibs/README.md).
   В `app/build.gradle.kts` обязательно `useLegacyPackaging = true`,
   в манифесте — `extractNativeLibs=true`.

2. **Ассеты.** `app/src/main/assets/` (`triggers.json`, `articles.json` и
   остальное) должны лежать локально. В репозитории их нет.

3. **llama.cpp.** Дерево в `third_party/` тоже не в git. Пин и патч:

```bash
git clone https://github.com/ggml-org/llama.cpp third_party/llama.cpp
git -C third_party/llama.cpp checkout 5bda51bfbc62e64193221e639f6ad4e08767d760
git -C third_party/llama.cpp apply ../../tools/patches/llama.cpp-hexagon-ninja.patch
```

SHA пина — в `tools/llama-cpp-pin.txt`. Без патча вложенный cmake HTP
не находит ninja.

4. **NPU.** Если в `local.properties` есть `hexagon.sdk.dir` (или задан
   `HEXAGON_SDK_ROOT`) — в APK попадут `libggml-hexagon.so` и
   `libggml-htp-v*.so`. Без SDK сборка только CPU. Hexagon SDK 6.6.0.0,
   путь к тулчейну — `tools/HEXAGON_Tools/19.0.07`.

5. **Подпись.** `keystore.properties` в корне (файл вне git):

```
storeFile=souchastnik-release.jks
storePassword=...
keyAlias=souchastnik
keyPassword=...
```

Нет файла — Gradle подписывает debug-ключом и пишет предупреждение.
Такой APK можно отдать тестерам сбоку, публиковать в GitHub Releases
и F-Droid нельзя: следующее обновление с другим ключом не встанет.

6. **Сборка.**

```bash
./gradlew :app:assembleRelease
```

APK: `app/build/outputs/apk/release/app-release.apk` (сотни мегабайт
из‑за весов). `BenchActivity` в релиз не входит.

7. **Проверка до установки.**

```bash
unzip -l app/build/outputs/apk/release/app-release.apk | grep libmodel
aapt dump permissions app/build/outputs/apk/release/app-release.apk
```

В APK должен быть `lib/arm64-v8a/libmodel-qwen35-08b-q40.so` (~507 МБ).
Список разрешений пустой: `INTERNET` нет. Если задан Hexagon SDK —
ещё `libggml-hexagon.so` и `libggml-htp-v*.so`.

Строка «модель не установлена» при наборе — не только отсутствующий файл.
Тот же текст, если `LlamaBridge.init` вернул 0 или процесс `:engine` умер.
Смотреть `adb logcat -s souchastnik-engine:I souchastnik-native:I`.

## Лицензия

GPL-3.0. Делайте что хотите
