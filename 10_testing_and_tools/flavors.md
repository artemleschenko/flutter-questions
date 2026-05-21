# Что такое flavors? Как ты организуешь работу с ними?

> Flavors (в контексте Flutter) — это механизм, позволяющий компилировать различные варианты (сборки) приложения из единой кодовой базы. Чаще всего они используются для разделения сред разработки (`Development`, `Staging`, `Production`). Каждый `flavor` может иметь свой уникальный идентификатор пакета (`Bundle ID` / `Application ID`), отдельные ключи `API`, сторонние конфигурации (`Firebase`, `AppMetrica`) и визуальные отличия (иконки, названия приложения).

## Разбор

На уровне Flutter «флейворы» — это абстракция. Фреймворк не реализует их сам, а делегирует задачу сборки нативным инструментам автоматизации `Gradle` (`Android`) и `Xcode Build System` (`iOS`). Когда вы выполняете команду `flutter run --flavor dev`, Flutter передает этот флаг в нативный слой.

### 1. Нативная реализация

#### Android

В Android флейворы настраиваются в файле `android/app/build.gradle` через блок `productFlavors` внутри `flavorDimensions`.

Каждому флейвору можно задать свой `applicationIdSuffix` (например, `.dev`).

Ресурсы (иконки, строки, файлы конфигурации вроде `google-services.json`) разделяются по директориям в `android/app/src/`: `src/dev/`, `src/prod/`, `src/main/` (общие ресурсы).

#### iOS

В iOS концепции flavor напрямую нет. Flutter маппит флейворы на `Xcode Schemes` и `Build Configurations`.

Для флейвора `dev` в `Xcode` необходимо создать три конфигурации: `Debug-dev`, `Profile-dev`, `Release-dev`, а также схему с именем `dev`.

Разделение `Bundle ID` и файлов (например, `GoogleService-Info.plist`) настраивается через `Target Build Settings` или кастомные `Build Phases` скрипты, которые копируют нужные файлы в зависимости от текущей конфигурации сборки.

### 2. Организация работы на стороне Dart

Существует два основных архитектурных подхода к инициализации конфигураций в Dart-коде.

#### 1. Раздельные точки входа 

Создаются отдельные файлы для запуска каждого окружения, которые инициализируют глобальный синглтон конфигурации перед запуском основного приложения.

```dart
// lib/environment.dart
enum EnvironmentType { dev, staging, prod }

class AppEnvironment {
  final EnvironmentType type;
  final String apiBaseUrl;

  static late AppEnvironment _instance;
  static AppEnvironment get instance => _instance;

  AppEnvironment._({required this.type, required this.apiBaseUrl});

  static void initialize({required EnvironmentType type, required String apiBaseUrl}) {
    _instance = AppEnvironment._(type: type, apiBaseUrl: apiBaseUrl);
  }
}
```

```dart
// lib/main_dev.dart
import 'package:flutter/material.dart';
import 'environment.dart';
import 'main_common.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  AppEnvironment.initialize(
    type: EnvironmentType.dev,
    apiBaseUrl: 'https://api.dev.company.com',
  );
  mainCommon(); // Запуск основного runApp()
}
```

Запуск: `flutter run lib/main_dev.dart --flavor dev`

#### 2. Использование `--dart-define-from-file` (Рекомендуемый с Flutter 3.16+)

Вместо жестко прописанного кода конфигурации выносятся в JSON-файлы, которые считываются на этапе компиляции через `String.fromEnvironment`. Этот же JSON можно использовать на нативном уровне для подстановки `Bundle ID` и названий приложений.

```json
// config/dev.json
{
  "DEFINE_API_URL": "https://api.dev.company.com",
  "DEFINE_APP_NAME": "App Dev"
}
```

Считывание в Dart:

```dart
const String apiUrl = String.fromEnvironment('DEFINE_API_URL');
```

Запуск: `flutter run --flavor dev --dart-define-from-file=config/dev.json`


### Практическое применение

В продакшн-разработке организация работы с флейворами выглядит следующим образом:

- Разделение Firebase / сервисов аналитики: В папках проекта создается структура для хранения конфигурационных файлов. На Android файлы `google-services.json` кладутся в `src/dev/` и `src/prod/`. На iOS пишется Bash-скрипт в `Xcode Build Phases`, который копирует нужный `GoogleService-Info.plist` в зависимости от конфигурации (`$CONFIGURATION`).

- Разделение иконок: С помощью пакета `flutter_launcher_icons` генерируются разные иконки. Конфигурация пакета поддерживает указание конкретного флейвора (flavors: dev: ...). Визуальное отличие иконки (например, плашка "DEV" поверх логотипа) критически важно для QA-инженеров, чтобы они не путали сборки на тестируемом устройстве.

- Автоматизация CI/CD: В пайплайнах (GitHub Actions, GitLab CI, Codemagic) сборка автоматизируется с явным указанием флейворов:

Сборка для тестировщиков: `flutter build apk --flavor staging -t lib/main_staging.dart` -> отправка в Firebase App Distribution.

Сборка для релиза: `flutter build appbundle --flavor prod -t lib/main_prod.dart` -> отправка в Google Play Console.

## Что почитать

* [Set up Flutter flavors for Android](https://docs.flutter.dev/deployment/flavors)
* [Set up Flutter flavors for iOS and macOS](https://docs.flutter.dev/deployment/flavors-ios)

