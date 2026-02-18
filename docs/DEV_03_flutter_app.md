# BetShark — Fejlesztői dokumentáció #3: Flutter Mobilalkalmazás

> **Verzió:** 1.0
> **Utoljára frissítve:** 2026-02-18
> **Célközönség:** Flutter fejlesztő

---

## 1. Áttekintés

A Flutter alkalmazás a BetShark végfelhasználói felülete iOS és Android platformon. Feladatai:

1. **Esemény böngészés (ingyenes, vendégként is):** közelgő meccsek listája, piacok megtekintése, kimenetel részletek és AI elemzések.
2. **Prémium Toplist:** napi legjobb fogadások ranglistája, csak előfizetőknek.
3. **Bejelentkezés és regisztráció:** email/jelszó, Google Sign-In, Apple Sign-In, Vendég mód.
4. **Előfizetés kezelés:** Apple IAP, Google Play Billing, 7 napos trial.
5. **Profil képernyő:** nyelv váltás, előfizetés kezelés, jogi dokumentumok, kijelentkezés, fiók törlés.
6. **Kétnyelvűség:** magyar (alapértelmezett) és angol.

Az alkalmazás REST API-n kommunikál a Node.js backenddel. A design fekete hátterű, narancssárga akcentszínekkel, cápa motívumokkal, csempealapú navigációval.

---

## 2. Tech stack

| Csomag | Verzió | Leírás |
|---|---|---|
| Flutter SDK | 3.22.x (stable) | Cross-platform UI framework |
| Dart | 3.4.x | Programozási nyelv |
| `flutter_riverpod` | 2.5.1 | State management |
| `go_router` | 14.1.4 | Deklaratív routing |
| `dio` | 5.4.3 | HTTP kliens |
| `flutter_secure_storage` | 9.0.0 | JWT token biztonságos tárolása |
| `google_sign_in` | 6.2.1 | Google Sign-In |
| `sign_in_with_apple` | 6.1.0 | Apple Sign-In |
| `in_app_purchase` | 3.1.13 | Apple IAP + Google Play Billing |
| `flutter_localizations` | (SDK) | i18n alap |
| `intl` | 0.19.0 | Dátum- és számformázás |
| `shared_preferences` | 2.2.3 | Nem érzékeny beállítások (pl. nyelv) |
| `cached_network_image` | 3.3.1 | Képek cache-elése |
| `flutter_svg` | 2.0.10+1 | SVG ikonok (cápa logo) |
| `timeago` | 3.6.1 | Relatív időformázás |
| `shimmer` | 3.0.0 | Skeleton loading animáció |

`pubspec.yaml` dependencies szekció:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  flutter_riverpod: ^2.5.1
  go_router: ^14.1.4
  dio: ^5.4.3
  flutter_secure_storage: ^9.0.0
  google_sign_in: ^6.2.1
  sign_in_with_apple: ^6.1.0
  in_app_purchase: ^3.1.13
  intl: ^0.19.0
  shared_preferences: ^2.2.3
  cached_network_image: ^3.3.1
  flutter_svg: ^2.0.10+1
  timeago: ^3.6.1
  shimmer: ^3.0.0
```

---

## 3. Projektstruktúra

```
betshark_app/
├── pubspec.yaml
├── pubspec.lock
├── analysis_options.yaml
├── android/                        # Android natív konfiguráció
│   └── app/
│       ├── build.gradle
│       └── src/main/AndroidManifest.xml
├── ios/                            # iOS natív konfiguráció
│   └── Runner/
│       ├── Info.plist
│       └── GoogleService-Info.plist
├── assets/
│   ├── images/
│   │   └── shark_logo.svg
│   └── translations/
│       ├── hu.json
│       └── en.json
└── lib/
    ├── main.dart                   # Belépési pont
    ├── app.dart                    # MaterialApp, GoRouter, Riverpod setup
    ├── core/
    │   ├── constants/
    │   │   ├── app_colors.dart     # Színek
    │   │   ├── app_text_styles.dart
    │   │   └── api_endpoints.dart  # API URL konstansok
    │   ├── theme/
    │   │   └── app_theme.dart      # ThemeData
    │   ├── network/
    │   │   ├── api_client.dart     # Dio singleton + interceptorok
    │   │   └── api_exception.dart  # Hibaosztályok
    │   ├── storage/
    │   │   └── secure_storage.dart # JWT, felhasználói adat
    │   └── l10n/
    │       ├── app_localizations.dart
    │       └── app_localizations_hu.dart
    ├── features/
    │   ├── auth/
    │   │   ├── data/
    │   │   │   └── auth_repository.dart
    │   │   ├── domain/
    │   │   │   └── user_model.dart
    │   │   └── presentation/
    │   │       ├── login_screen.dart
    │   │       ├── register_screen.dart
    │   │       └── forgot_password_screen.dart
    │   ├── events/
    │   │   ├── data/
    │   │   │   └── events_repository.dart
    │   │   ├── domain/
    │   │   │   ├── event_model.dart
    │   │   │   └── market_model.dart
    │   │   └── presentation/
    │   │       ├── events_screen.dart
    │   │       ├── markets_screen.dart
    │   │       └── outcome_detail_screen.dart
    │   ├── toplist/
    │   │   ├── data/
    │   │   │   └── toplist_repository.dart
    │   │   └── presentation/
    │   │       ├── toplist_screen.dart
    │   │       └── toplist_locked_screen.dart
    │   ├── profile/
    │   │   ├── data/
    │   │   │   └── profile_repository.dart
    │   │   └── presentation/
    │   │       └── profile_screen.dart
    │   └── subscription/
    │       ├── data/
    │       │   └── subscription_repository.dart
    │       └── presentation/
    │           └── subscription_screen.dart
    └── shared/
        ├── providers/
        │   ├── auth_provider.dart          # Riverpod: bejelentkezési állapot
        │   └── subscription_provider.dart  # Riverpod: előfizetési állapot
        └── widgets/
            ├── event_tile.dart
            ├── market_tile.dart
            ├── outcome_tile.dart
            ├── score_badge.dart
            ├── value_badge.dart
            ├── shark_icon.dart
            └── loading_shimmer.dart
```

---

## 4. Környezeti változók / Konfiguráció

Flutter alkalmazásnál nincs `.env` fájl. A konfiguráció a `lib/core/constants/api_endpoints.dart` fájlban kerül:

```dart
// lib/core/constants/api_endpoints.dart
class ApiEndpoints {
  static const String baseUrl = 'https://api.betshark.app'; // produkció
  // Fejlesztéshez: 'http://10.0.2.2:3000' (Android emulátor), 'http://localhost:3000' (iOS szimulátor)

  static const String register         = '/api/auth/register';
  static const String login            = '/api/auth/login';
  static const String googleLogin      = '/api/auth/google';
  static const String appleLogin       = '/api/auth/apple';
  static const String forgotPassword   = '/api/auth/forgot-password';
  static const String resetPassword    = '/api/auth/reset-password';
  static const String events           = '/api/events';
  static const String toplist          = '/api/toplist';
  static const String profile          = '/api/user/profile';
  static const String updateLanguage   = '/api/user/language';
  static const String acceptLegal      = '/api/user/accept-legal';
  static const String startTrial       = '/api/subscription/start-trial';
  static const String validateReceipt  = '/api/subscription/validate-receipt';
  static const String deleteAccount    = '/api/user';

  static String eventMarkets(int eventId) => '/api/events/$eventId/markets';
  static String outcomeDetail(int outcomeId) => '/api/outcomes/$outcomeId';
}
```

**Flavor-alapú konfiguráció** (ha dev/prod különbség kell):

```bash
# Fejlesztési futtatás
flutter run --dart-define=BASE_URL=http://10.0.2.2:3000

# Produkciós build
flutter build apk --dart-define=BASE_URL=https://api.betshark.app
```

```dart
// Kódból olvasva:
static const String baseUrl = String.fromEnvironment('BASE_URL', defaultValue: 'https://api.betshark.app');
```

---

## 5. Adatbázis beállítás

A Flutter alkalmazás **nem kommunikál közvetlenül az adatbázissal** — minden adatot a Node.js REST API-n keresztül kap. Nincs lokális adatbázis beállítás.

---

## 6. Lépésről lépésre implementáció

### 6.1 lépés: Flutter projekt létrehozása és alapkonfiguráció

```bash
flutter create betshark_app --org com.yourcompany --project-name betshark_app
cd betshark_app
```

A `pubspec.yaml`-ban add hozzá a függőségeket (ld. 2. fejezet), majd:

```bash
flutter pub get
```

Az `analysis_options.yaml`-ban:

```yaml
include: package:flutter_lints/flutter.yaml
linter:
  rules:
    prefer_const_constructors: true
    prefer_const_literals_to_create_immutables: true
```

---

### 6.2 lépés: Színek és téma (`core/constants/app_colors.dart`, `core/theme/app_theme.dart`)

**Mit csináljon:** Definiálja a BetShark vizuális identitását: fekete háttér, narancssárga akcentszín.

```dart
// lib/core/constants/app_colors.dart
import 'package:flutter/material.dart';

class AppColors {
  static const Color background    = Color(0xFF0A0A0A); // Mélyfekete
  static const Color surface       = Color(0xFF1A1A1A); // Kártya háttér
  static const Color surfaceLight  = Color(0xFF2A2A2A); // Emelkedettebb felület
  static const Color accent        = Color(0xFFFF6B00); // Narancssárga
  static const Color accentGold    = Color(0xFFFFD700); // Arany (Strong Value)
  static const Color accentSilver  = Color(0xFFC0C0C0); // Ezüst (Value)
  static const Color textPrimary   = Color(0xFFFFFFFF);
  static const Color textSecondary = Color(0xFFAAAAAA);
  static const Color textMuted     = Color(0xFF666666);
  static const Color success       = Color(0xFF4CAF50);
  static const Color error         = Color(0xFFE53935);
  static const Color divider       = Color(0xFF2A2A2A);
}
```

```dart
// lib/core/theme/app_theme.dart
import 'package:flutter/material.dart';
import '../constants/app_colors.dart';

class AppTheme {
  static ThemeData get dark => ThemeData(
    useMaterial3: true,
    brightness: Brightness.dark,
    scaffoldBackgroundColor: AppColors.background,
    colorScheme: const ColorScheme.dark(
      primary: AppColors.accent,
      surface: AppColors.surface,
      onPrimary: Colors.white,
      onSurface: AppColors.textPrimary,
    ),
    cardTheme: const CardTheme(
      color: AppColors.surface,
      margin: EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.all(Radius.circular(12))
      ),
    ),
    appBarTheme: const AppBarTheme(
      backgroundColor: AppColors.background,
      foregroundColor: AppColors.textPrimary,
      elevation: 0,
      centerTitle: true,
    ),
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        backgroundColor: AppColors.accent,
        foregroundColor: Colors.white,
        minimumSize: const Size(double.infinity, 50),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(10)),
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      filled: true,
      fillColor: AppColors.surfaceLight,
      border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(10),
        borderSide: BorderSide.none,
      ),
      hintStyle: const TextStyle(color: AppColors.textMuted),
    ),
    bottomNavigationBarTheme: const BottomNavigationBarThemeData(
      backgroundColor: AppColors.surface,
      selectedItemColor: AppColors.accent,
      unselectedItemColor: AppColors.textMuted,
    ),
  );
}
```

---

### 6.3 lépés: Biztonságos tárolás (`core/storage/secure_storage.dart`)

**Mit csináljon:** JWT token és a felhasználói azonosító biztonságos tárolása `flutter_secure_storage`-ban.

```dart
// lib/core/storage/secure_storage.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );

  static const _keyToken = 'jwt_token';
  static const _keyUserId = 'user_id';
  static const _keyUserEmail = 'user_email';
  static const _keyLanguage = 'user_language';

  static Future<void> saveToken(String token) =>
      _storage.write(key: _keyToken, value: token);

  static Future<String?> getToken() => _storage.read(key: _keyToken);

  static Future<void> saveUserData({
    required String userId,
    required String? email,
    required String language,
  }) async {
    await _storage.write(key: _keyUserId, value: userId);
    await _storage.write(key: _keyUserEmail, value: email ?? '');
    await _storage.write(key: _keyLanguage, value: language);
  }

  static Future<void> clearAll() => _storage.deleteAll();

  static Future<String?> getLanguage() => _storage.read(key: _keyLanguage);
}
```

---

### 6.4 lépés: API kliens (`core/network/api_client.dart`)

**Mit csináljon:** Egy Dio singleton, amely:
1. Minden kéréshez hozzáfűzi a JWT tokent (ha van).
2. 401 válasz esetén töröl a helyi tokenből és visszanavigál a bejelentkezési képernyőre.
3. Hálózati és szerver hibákat egységes `ApiException`-ba csomagolja.

```dart
// lib/core/network/api_exception.dart
class ApiException implements Exception {
  final String message;
  final int? statusCode;
  const ApiException(this.message, {this.statusCode});

  @override
  String toString() => 'ApiException($statusCode): $message';
}
```

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';
import '../constants/api_endpoints.dart';
import '../storage/secure_storage.dart';
import 'api_exception.dart';

class ApiClient {
  static final Dio _dio = _createDio();

  static Dio _createDio() {
    final dio = Dio(BaseOptions(
      baseUrl: ApiEndpoints.baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 15),
      headers: {'Content-Type': 'application/json'},
    ));

    // Auth interceptor: Bearer token automatikus hozzáfűzése
    dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        final token = await SecureStorage.getToken();
        if (token != null) {
          options.headers['Authorization'] = 'Bearer $token';
        }
        return handler.next(options);
      },
      onError: (error, handler) {
        final statusCode = error.response?.statusCode;
        final message = error.response?.data?['error']
            ?? error.message
            ?? 'Ismeretlen hálózati hiba';

        if (statusCode == 401) {
          // Token lejárt vagy érvénytelen: törlés
          SecureStorage.clearAll();
          // A navigáció a provider szintjén kezelendő (ld. auth_provider.dart)
        }

        return handler.next(DioException(
          requestOptions: error.requestOptions,
          error: ApiException(message, statusCode: statusCode),
          response: error.response,
          type: error.type,
        ));
      }
    ));

    return dio;
  }

  static Future<dynamic> get(String path, {Map<String, dynamic>? params}) async {
    try {
      final response = await _dio.get(path, queryParameters: params);
      return response.data;
    } on DioException catch (e) {
      throw e.error ?? ApiException('Network error');
    }
  }

  static Future<dynamic> post(String path, {dynamic data}) async {
    try {
      final response = await _dio.post(path, data: data);
      return response.data;
    } on DioException catch (e) {
      throw e.error ?? ApiException('Network error');
    }
  }

  static Future<dynamic> put(String path, {dynamic data}) async {
    try {
      final response = await _dio.put(path, data: data);
      return response.data;
    } on DioException catch (e) {
      throw e.error ?? ApiException('Network error');
    }
  }

  static Future<dynamic> delete(String path) async {
    try {
      final response = await _dio.delete(path);
      return response.data;
    } on DioException catch (e) {
      throw e.error ?? ApiException('Network error');
    }
  }
}
```

---

### 6.5 lépés: Domain modellek

**Mit hozz létre:** Egyszerű Dart data class-ok a JSON deserializáláshoz.

```dart
// lib/features/auth/domain/user_model.dart
class UserModel {
  final int id;
  final String? email;
  final String language;
  final String? subscriptionStatus;

  const UserModel({
    required this.id,
    this.email,
    required this.language,
    this.subscriptionStatus,
  });

  factory UserModel.fromJson(Map<String, dynamic> json) => UserModel(
    id: json['id'],
    email: json['email'],
    language: json['language'] ?? 'hu',
    subscriptionStatus: json['subscriptionStatus'],
  );

  bool get isPremium =>
    subscriptionStatus == 'active' || subscriptionStatus == 'trial';
}
```

```dart
// lib/features/events/domain/event_model.dart
class EventModel {
  final int id;
  final String homeTeam;
  final String awayTeam;
  final DateTime kickoffTime;
  final String leagueName;
  final String? country;

  const EventModel({
    required this.id,
    required this.homeTeam,
    required this.awayTeam,
    required this.kickoffTime,
    required this.leagueName,
    this.country,
  });

  factory EventModel.fromJson(Map<String, dynamic> json) => EventModel(
    id: json['id'],
    homeTeam: json['home_team'],
    awayTeam: json['away_team'],
    kickoffTime: DateTime.parse(json['kickoff_time']),
    leagueName: json['league_name'],
    country: json['country'],
  );
}
```

```dart
// lib/features/events/domain/market_model.dart
class OutcomeSummary {
  final int id;
  final String nameHu;
  final String nameEn;
  final double bestOdds;
  final double overallScore;
  final String? valueLabel;

  const OutcomeSummary({
    required this.id,
    required this.nameHu,
    required this.nameEn,
    required this.bestOdds,
    required this.overallScore,
    this.valueLabel,
  });

  factory OutcomeSummary.fromJson(Map<String, dynamic> json) => OutcomeSummary(
    id: json['id'],
    nameHu: json['name_hu'],
    nameEn: json['name_en'],
    bestOdds: (json['best_odds'] as num).toDouble(),
    overallScore: (json['overall_score'] as num).toDouble(),
    valueLabel: json['value_label'],
  );
}

class MarketModel {
  final int id;
  final String nameHu;
  final String nameEn;
  final List<OutcomeSummary> outcomes;

  const MarketModel({
    required this.id,
    required this.nameHu,
    required this.nameEn,
    required this.outcomes,
  });

  factory MarketModel.fromJson(Map<String, dynamic> json) => MarketModel(
    id: json['id'],
    nameHu: json['name_hu'],
    nameEn: json['name_en'],
    outcomes: (json['outcomes'] as List)
        .map((o) => OutcomeSummary.fromJson(o))
        .toList(),
  );
}

class OutcomeDetail {
  final int id;
  final String nameHu;
  final String nameEn;
  final double likelihoodScore;
  final double oddsScore;
  final double overallScore;
  final double bestOdds;
  final String? valueLabel;
  final String? analysisHu;
  final String? analysisEn;
  final Map<String, dynamic> event;
  final Map<String, dynamic> market;

  const OutcomeDetail({
    required this.id,
    required this.nameHu,
    required this.nameEn,
    required this.likelihoodScore,
    required this.oddsScore,
    required this.overallScore,
    required this.bestOdds,
    this.valueLabel,
    this.analysisHu,
    this.analysisEn,
    required this.event,
    required this.market,
  });

  factory OutcomeDetail.fromJson(Map<String, dynamic> json) => OutcomeDetail(
    id: json['id'],
    nameHu: json['name_hu'],
    nameEn: json['name_en'],
    likelihoodScore: (json['likelihood_score'] as num).toDouble(),
    oddsScore: (json['odds_score'] as num).toDouble(),
    overallScore: (json['overall_score'] as num).toDouble(),
    bestOdds: (json['best_odds'] as num).toDouble(),
    valueLabel: json['value_label'],
    analysisHu: json['analysis_hu'],
    analysisEn: json['analysis_en'],
    event: json['event'],
    market: json['market'],
  );
}
```

---

### 6.6 lépés: Repository-k

**Auth repository:**

```dart
// lib/features/auth/data/auth_repository.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';
import '../../../core/network/api_client.dart';
import '../../../core/constants/api_endpoints.dart';
import '../../../core/storage/secure_storage.dart';
import '../domain/user_model.dart';

class AuthRepository {
  final _googleSignIn = GoogleSignIn(scopes: ['email']);

  Future<UserModel> registerWithEmail(String email, String password) async {
    final data = await ApiClient.post(ApiEndpoints.register,
        data: {'email': email, 'password': password});
    await SecureStorage.saveToken(data['token']);
    return UserModel.fromJson(data['user']);
  }

  Future<UserModel> loginWithEmail(String email, String password) async {
    final data = await ApiClient.post(ApiEndpoints.login,
        data: {'email': email, 'password': password});
    await SecureStorage.saveToken(data['token']);
    return UserModel.fromJson(data['user']);
  }

  Future<UserModel> loginWithGoogle() async {
    final googleUser = await _googleSignIn.signIn();
    if (googleUser == null) throw Exception('Google bejelentkezés megszakítva');
    final auth = await googleUser.authentication;
    final data = await ApiClient.post(ApiEndpoints.googleLogin,
        data: {'id_token': auth.idToken});
    await SecureStorage.saveToken(data['token']);
    return UserModel.fromJson(data['user']);
  }

  Future<UserModel> loginWithApple() async {
    final credential = await SignInWithApple.getAppleIDCredential(
      scopes: [AppleIDAuthorizationScopes.email],
    );
    final data = await ApiClient.post(ApiEndpoints.appleLogin, data: {
      'identity_token': credential.identityToken,
      'user': credential.email != null ? {'email': credential.email} : null,
    });
    await SecureStorage.saveToken(data['token']);
    return UserModel.fromJson(data['user']);
  }

  Future<void> forgotPassword(String email) async {
    await ApiClient.post(ApiEndpoints.forgotPassword, data: {'email': email});
  }

  Future<void> logout() async {
    await SecureStorage.clearAll();
    await _googleSignIn.signOut();
  }
}

final authRepositoryProvider = Provider<AuthRepository>((ref) => AuthRepository());
```

**Events repository:**

```dart
// lib/features/events/data/events_repository.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/network/api_client.dart';
import '../../../core/constants/api_endpoints.dart';
import '../domain/event_model.dart';
import '../domain/market_model.dart';

class EventsRepository {
  Future<({List<EventModel> events, int total, int page})> getEvents({
    int page = 1,
    int limit = 20,
  }) async {
    final data = await ApiClient.get(
      ApiEndpoints.events,
      params: {'page': page, 'limit': limit},
    );
    return (
      events: (data['events'] as List).map((e) => EventModel.fromJson(e)).toList(),
      total: data['total'] as int,
      page: data['page'] as int,
    );
  }

  Future<({EventModel event, List<MarketModel> markets})> getEventMarkets(int eventId) async {
    final data = await ApiClient.get(ApiEndpoints.eventMarkets(eventId));
    return (
      event: EventModel.fromJson(data['event']),
      markets: (data['markets'] as List).map((m) => MarketModel.fromJson(m)).toList(),
    );
  }

  Future<OutcomeDetail> getOutcomeDetail(int outcomeId) async {
    final data = await ApiClient.get(ApiEndpoints.outcomeDetail(outcomeId));
    return OutcomeDetail.fromJson(data);
  }
}

final eventsRepositoryProvider = Provider<EventsRepository>((ref) => EventsRepository());
```

---

### 6.7 lépés: Riverpod state providerek

**Auth provider:**

```dart
// lib/shared/providers/auth_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../core/storage/secure_storage.dart';
import '../../features/auth/data/auth_repository.dart';
import '../../features/auth/domain/user_model.dart';

// Az aktuális bejelentkezési állapot
final authStateProvider = StateNotifierProvider<AuthNotifier, AsyncValue<UserModel?>>(
  (ref) => AuthNotifier(ref.watch(authRepositoryProvider)),
);

class AuthNotifier extends StateNotifier<AsyncValue<UserModel?>> {
  final AuthRepository _repo;

  AuthNotifier(this._repo) : super(const AsyncValue.loading()) {
    _init();
  }

  Future<void> _init() async {
    // Ha van mentett token, betöltjük az állapotot
    final token = await SecureStorage.getToken();
    if (token != null) {
      // Opcionálisan: /api/user/profile-t hívjuk a friss user adatokhoz
      // Egyszerűsített verzió: jelezzük, hogy be van jelentkezve
      state = const AsyncValue.data(null); // user adatok a profileból jönnek
    } else {
      state = const AsyncValue.data(null);
    }
  }

  Future<void> loginWithEmail(String email, String password) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(
      () => _repo.loginWithEmail(email, password),
    );
  }

  Future<void> registerWithEmail(String email, String password) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(
      () => _repo.registerWithEmail(email, password),
    );
  }

  Future<void> loginWithGoogle() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _repo.loginWithGoogle());
  }

  Future<void> loginWithApple() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _repo.loginWithApple());
  }

  Future<void> logout() async {
    await _repo.logout();
    state = const AsyncValue.data(null);
  }

  bool get isLoggedIn => state.valueOrNull != null;
}

// Vendég mód flag
final isGuestProvider = StateProvider<bool>((ref) => false);
```

---

### 6.8 lépés: GoRouter navigáció (`app.dart`)

**Mit csináljon:** Definiálja az összes navigációs útvonalat. A Toplist route redirect-el bejelentkezési képernyőre, ha a user nem prémium.

```dart
// lib/app.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'core/theme/app_theme.dart';
import 'features/auth/presentation/login_screen.dart';
import 'features/auth/presentation/register_screen.dart';
import 'features/auth/presentation/forgot_password_screen.dart';
import 'features/events/presentation/events_screen.dart';
import 'features/events/presentation/markets_screen.dart';
import 'features/events/presentation/outcome_detail_screen.dart';
import 'features/toplist/presentation/toplist_screen.dart';
import 'features/profile/presentation/profile_screen.dart';
import 'features/subscription/presentation/subscription_screen.dart';
import 'shared/providers/auth_provider.dart';

final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/events',
    redirect: (context, state) {
      // A Toplist oldal hitelesítést igényel (a képernyőn belül kezeljük a premium ellenőrzést)
      return null;
    },
    routes: [
      GoRoute(path: '/login',          builder: (_, __) => const LoginScreen()),
      GoRoute(path: '/register',       builder: (_, __) => const RegisterScreen()),
      GoRoute(path: '/forgot-password', builder: (_, __) => const ForgotPasswordScreen()),
      ShellRoute(
        builder: (context, state, child) => MainShell(child: child),
        routes: [
          GoRoute(
            path: '/events',
            builder: (_, __) => const EventsScreen(),
            routes: [
              GoRoute(
                path: ':eventId/markets',
                builder: (_, state) => MarketsScreen(
                  eventId: int.parse(state.pathParameters['eventId']!),
                ),
                routes: [
                  GoRoute(
                    path: 'outcome/:outcomeId',
                    builder: (_, state) => OutcomeDetailScreen(
                      outcomeId: int.parse(state.pathParameters['outcomeId']!),
                    ),
                  ),
                ],
              ),
            ],
          ),
          GoRoute(path: '/toplist', builder: (_, __) => const ToplistScreen()),
          GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
        ],
      ),
      GoRoute(path: '/subscription', builder: (_, __) => const SubscriptionScreen()),
    ],
  );
});

class BetSharkApp extends ConsumerWidget {
  const BetSharkApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      title: 'BetShark',
      theme: AppTheme.dark,
      routerConfig: router,
      supportedLocales: const [Locale('hu'), Locale('en')],
      localizationsDelegates: const [
        // AppLocalizations.delegate,  // ld. 6.9-es lépés
      ],
    );
  }
}

// Alső navigációs sáv shell widget
class MainShell extends StatelessWidget {
  final Widget child;
  const MainShell({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    final location = GoRouterState.of(context).uri.toString();
    final currentIndex = location.startsWith('/toplist') ? 1
        : location.startsWith('/profile') ? 2
        : 0;

    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: currentIndex,
        onTap: (i) {
          if (i == 0) context.go('/events');
          if (i == 1) context.go('/toplist');
          if (i == 2) context.go('/profile');
        },
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.sports_soccer), label: 'Meccsek'),
          BottomNavigationBarItem(icon: Icon(Icons.leaderboard), label: 'Toplista'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: 'Profil'),
        ],
      ),
    );
  }
}
```

---

### 6.9 lépés: Lokalizáció

**Mit hozz létre:** Magyar és angol fordítási fájlok, és egy egyszerű fordítási segédosztály.

`assets/translations/hu.json`:

```json
{
  "app_name": "BetShark",
  "events": "Meccsek",
  "toplist": "Toplista",
  "profile": "Profil",
  "login": "Bejelentkezés",
  "register": "Regisztráció",
  "email": "Email cím",
  "password": "Jelszó",
  "forgot_password": "Elfelejtett jelszó?",
  "login_with_google": "Folytatás Google-lal",
  "login_with_apple": "Folytatás Apple-lel",
  "continue_as_guest": "Vendégként folytatom",
  "likelihood_score": "Valószínűségi pontszám",
  "odds_score": "Odds pontszám",
  "overall_score": "Összesített pontszám",
  "best_odds": "Legjobb odds",
  "value": "Value",
  "strong_value": "Strong Value",
  "analysis": "Elemzés",
  "premium_required": "Ez a funkció prémium előfizetést igényel.",
  "start_trial": "7 napos ingyenes próba",
  "subscribe": "Előfizetés (2 990 Ft/hó)",
  "subscription_active": "Aktív előfizetés",
  "subscription_trial": "Próbaidőszak",
  "logout": "Kijelentkezés",
  "delete_account": "Fiók törlése",
  "language": "Nyelv",
  "legal": "Jogi dokumentumok",
  "no_events": "Nincs megjeleníthető esemény.",
  "loading": "Betöltés...",
  "error_generic": "Hiba történt. Kérjük próbálja újra.",
  "kickoff": "Kezdés"
}
```

`assets/translations/en.json`:

```json
{
  "app_name": "BetShark",
  "events": "Events",
  "toplist": "Toplist",
  "profile": "Profile",
  "login": "Sign In",
  "register": "Register",
  "email": "Email address",
  "password": "Password",
  "forgot_password": "Forgot password?",
  "login_with_google": "Continue with Google",
  "login_with_apple": "Continue with Apple",
  "continue_as_guest": "Continue as Guest",
  "likelihood_score": "Likelihood score",
  "odds_score": "Odds score",
  "overall_score": "Overall score",
  "best_odds": "Best odds",
  "value": "Value",
  "strong_value": "Strong Value",
  "analysis": "Analysis",
  "premium_required": "This feature requires a premium subscription.",
  "start_trial": "Start 7-day free trial",
  "subscribe": "Subscribe (HUF 2,990/mo)",
  "subscription_active": "Active subscription",
  "subscription_trial": "Trial period",
  "logout": "Sign out",
  "delete_account": "Delete account",
  "language": "Language",
  "legal": "Legal documents",
  "no_events": "No events to display.",
  "loading": "Loading...",
  "error_generic": "Something went wrong. Please try again.",
  "kickoff": "Kickoff"
}
```

A `pubspec.yaml`-ban add hozzá az assets szekciót:

```yaml
flutter:
  assets:
    - assets/translations/hu.json
    - assets/translations/en.json
    - assets/images/shark_logo.svg
```

Egyszerű fordítási segédosztály (flutter_localizations beépített implementáció helyett):

```dart
// lib/core/l10n/app_localizations.dart
import 'dart:convert';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../core/storage/secure_storage.dart';

final localeProvider = StateNotifierProvider<LocaleNotifier, String>((ref) {
  return LocaleNotifier();
});

class LocaleNotifier extends StateNotifier<String> {
  LocaleNotifier() : super('hu') {
    _load();
  }

  Future<void> _load() async {
    final saved = await SecureStorage.getLanguage();
    if (saved != null) state = saved;
  }

  Future<void> setLanguage(String lang) async {
    state = lang;
    await SecureStorage.saveUserData(userId: '', email: null, language: lang);
  }
}

// Fordítások betöltése
final translationsProvider = FutureProvider<Map<String, String>>((ref) async {
  final locale = ref.watch(localeProvider);
  final jsonStr = await rootBundle.loadString('assets/translations/$locale.json');
  final Map<String, dynamic> data = json.decode(jsonStr);
  return data.map((key, value) => MapEntry(key, value.toString()));
});

// Fordítás segédfüggvény widgetekben
extension TranslateExtension on Map<String, String> {
  String t(String key) => this[key] ?? key;
}
```

---

### 6.10 lépés: Közös widgetek

**Event tile (`shared/widgets/event_tile.dart`):**

```dart
// lib/shared/widgets/event_tile.dart
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';
import '../../core/constants/app_colors.dart';
import '../../features/events/domain/event_model.dart';

class EventTile extends StatelessWidget {
  final EventModel event;
  final VoidCallback onTap;

  const EventTile({super.key, required this.event, required this.onTap});

  @override
  Widget build(BuildContext context) {
    final kickoffFormatted = DateFormat('MMM d. HH:mm', 'hu').format(event.kickoffTime.toLocal());
    return Card(
      child: ListTile(
        onTap: onTap,
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        title: Text(
          '${event.homeTeam} — ${event.awayTeam}',
          style: const TextStyle(
            color: AppColors.textPrimary,
            fontWeight: FontWeight.w600,
            fontSize: 15,
          ),
        ),
        subtitle: Padding(
          padding: const EdgeInsets.only(top: 4),
          child: Text(
            '${event.leagueName} · $kickoffFormatted',
            style: const TextStyle(color: AppColors.textSecondary, fontSize: 13),
          ),
        ),
        trailing: const Icon(Icons.chevron_right, color: AppColors.textMuted),
      ),
    );
  }
}
```

**Value badge widget (`shared/widgets/value_badge.dart`):**

```dart
// lib/shared/widgets/value_badge.dart
import 'package:flutter/material.dart';
import 'package:flutter_svg/flutter_svg.dart';
import '../../core/constants/app_colors.dart';

class ValueBadge extends StatelessWidget {
  final String? valueLabel; // 'value', 'strong_value', vagy null

  const ValueBadge({super.key, this.valueLabel});

  @override
  Widget build(BuildContext context) {
    if (valueLabel == null) return const SizedBox.shrink();

    final isStrong = valueLabel == 'strong_value';
    final color = isStrong ? AppColors.accentGold : AppColors.accentSilver;
    final label = isStrong ? 'Strong Value' : 'Value';

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: color.withOpacity(0.15),
        borderRadius: BorderRadius.circular(6),
        border: Border.all(color: color, width: 1),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          SvgPicture.asset('assets/images/shark_logo.svg', width: 14, height: 14,
              colorFilter: ColorFilter.mode(color, BlendMode.srcIn)),
          const SizedBox(width: 4),
          Text(label, style: TextStyle(color: color, fontSize: 12, fontWeight: FontWeight.w600)),
        ],
      ),
    );
  }
}
```

**Score badge (`shared/widgets/score_badge.dart`):**

```dart
// lib/shared/widgets/score_badge.dart
import 'package:flutter/material.dart';
import '../../core/constants/app_colors.dart';

class ScoreBadge extends StatelessWidget {
  final double score;
  final String label;

  const ScoreBadge({super.key, required this.score, required this.label});

  Color get _color {
    if (score >= 7.0) return AppColors.success;
    if (score >= 5.0) return AppColors.accent;
    return AppColors.error;
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Container(
          width: 52,
          height: 52,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            border: Border.all(color: _color, width: 2.5),
          ),
          child: Center(
            child: Text(
              score.toStringAsFixed(1),
              style: TextStyle(color: _color, fontWeight: FontWeight.bold, fontSize: 16),
            ),
          ),
        ),
        const SizedBox(height: 4),
        Text(label, style: const TextStyle(color: AppColors.textSecondary, fontSize: 10)),
      ],
    );
  }
}
```

---

### 6.11 lépés: Bejelentkezési képernyő (`features/auth/presentation/login_screen.dart`)

**Mit csináljon:** Email/jelszó form, Google gomb, Apple gomb (csak iOS-en jelenik meg), Vendég gomb. Hiba esetén SnackBar.

```dart
// lib/features/auth/presentation/login_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/constants/app_colors.dart';
import '../../../shared/providers/auth_provider.dart';
import 'dart:io' show Platform;

class LoginScreen extends ConsumerStatefulWidget {
  const LoginScreen({super.key});

  @override
  ConsumerState<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends ConsumerState<LoginScreen> {
  final _emailCtrl = TextEditingController();
  final _passCtrl = TextEditingController();
  bool _loading = false;

  Future<void> _login() async {
    setState(() => _loading = true);
    try {
      await ref.read(authStateProvider.notifier).loginWithEmail(
        _emailCtrl.text.trim(), _passCtrl.text,
      );
      if (mounted) context.go('/events');
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(e.toString()), backgroundColor: AppColors.error),
        );
      }
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  Future<void> _loginWithGoogle() async {
    setState(() => _loading = true);
    try {
      await ref.read(authStateProvider.notifier).loginWithGoogle();
      if (mounted) context.go('/events');
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(e.toString()), backgroundColor: AppColors.error),
        );
      }
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              const SizedBox(height: 48),
              // Logo
              Center(child: Image.asset('assets/images/shark_logo.svg', height: 80)),
              const SizedBox(height: 12),
              const Center(
                child: Text('BetShark',
                  style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold, color: AppColors.accent)),
              ),
              const SizedBox(height: 40),
              // Email mező
              TextField(
                controller: _emailCtrl,
                keyboardType: TextInputType.emailAddress,
                decoration: const InputDecoration(hintText: 'Email cím'),
              ),
              const SizedBox(height: 12),
              // Jelszó mező
              TextField(
                controller: _passCtrl,
                obscureText: true,
                decoration: const InputDecoration(hintText: 'Jelszó'),
              ),
              Align(
                alignment: Alignment.centerRight,
                child: TextButton(
                  onPressed: () => context.push('/forgot-password'),
                  child: const Text('Elfelejtett jelszó?',
                    style: TextStyle(color: AppColors.textSecondary)),
                ),
              ),
              const SizedBox(height: 8),
              ElevatedButton(
                onPressed: _loading ? null : _login,
                child: _loading
                    ? const CircularProgressIndicator(color: Colors.white)
                    : const Text('Bejelentkezés'),
              ),
              const SizedBox(height: 12),
              // Google
              OutlinedButton.icon(
                onPressed: _loading ? null : _loginWithGoogle,
                icon: const Icon(Icons.g_mobiledata, size: 22),
                label: const Text('Folytatás Google-lal'),
                style: OutlinedButton.styleFrom(
                  foregroundColor: AppColors.textPrimary,
                  side: const BorderSide(color: AppColors.surfaceLight),
                  minimumSize: const Size(double.infinity, 50),
                ),
              ),
              // Apple (csak iOS-en)
              if (Platform.isIOS) ...[
                const SizedBox(height: 12),
                OutlinedButton.icon(
                  onPressed: _loading ? null : () async {
                    setState(() => _loading = true);
                    try {
                      await ref.read(authStateProvider.notifier).loginWithApple();
                      if (mounted) context.go('/events');
                    } finally {
                      if (mounted) setState(() => _loading = false);
                    }
                  },
                  icon: const Icon(Icons.apple, size: 22),
                  label: const Text('Folytatás Apple-lel'),
                  style: OutlinedButton.styleFrom(
                    foregroundColor: AppColors.textPrimary,
                    side: const BorderSide(color: AppColors.surfaceLight),
                    minimumSize: const Size(double.infinity, 50),
                  ),
                ),
              ],
              const SizedBox(height: 16),
              TextButton(
                onPressed: () {
                  ref.read(isGuestProvider.notifier).state = true;
                  context.go('/events');
                },
                child: const Text('Vendégként folytatom',
                  style: TextStyle(color: AppColors.textMuted)),
              ),
              const SizedBox(height: 12),
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Text('Nincs fiókod? ', style: TextStyle(color: AppColors.textSecondary)),
                  GestureDetector(
                    onTap: () => context.push('/register'),
                    child: const Text('Regisztráció',
                      style: TextStyle(color: AppColors.accent, fontWeight: FontWeight.w600)),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### 6.12 lépés: Esemény lista képernyő (`features/events/presentation/events_screen.dart`)

**Mit csináljon:** Paginated lista az eseményekből. Infinite scroll: az utolsó elemre görgetve a következő oldalt tölti be. Shimmer loading animáció betöltés közben.

```dart
// lib/features/events/presentation/events_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/constants/app_colors.dart';
import '../data/events_repository.dart';
import '../domain/event_model.dart';
import '../../../shared/widgets/event_tile.dart';
import '../../../shared/widgets/loading_shimmer.dart';

// Provider: esemény lista állapot
final eventsProvider = StateNotifierProvider<EventsNotifier, AsyncValue<List<EventModel>>>(
  (ref) => EventsNotifier(ref.watch(eventsRepositoryProvider)),
);

class EventsNotifier extends StateNotifier<AsyncValue<List<EventModel>>> {
  final EventsRepository _repo;
  int _page = 1;
  int _total = 0;
  bool _isLoadingMore = false;

  EventsNotifier(this._repo) : super(const AsyncValue.loading()) {
    loadEvents();
  }

  Future<void> loadEvents() async {
    _page = 1;
    state = const AsyncValue.loading();
    try {
      final result = await _repo.getEvents(page: _page, limit: 20);
      _total = result.total;
      state = AsyncValue.data(result.events);
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }

  Future<void> loadMore() async {
    if (_isLoadingMore || (state.value?.length ?? 0) >= _total) return;
    _isLoadingMore = true;
    _page++;
    try {
      final result = await _repo.getEvents(page: _page, limit: 20);
      state = AsyncValue.data([...state.value!, ...result.events]);
    } finally {
      _isLoadingMore = false;
    }
  }
}

class EventsScreen extends ConsumerWidget {
  const EventsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final eventsState = ref.watch(eventsProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('BetShark — Meccsek')),
      body: eventsState.when(
        loading: () => const LoadingShimmer(),
        error: (e, _) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('Hiba történt.', style: TextStyle(color: AppColors.error)),
              TextButton(
                onPressed: () => ref.read(eventsProvider.notifier).loadEvents(),
                child: const Text('Újrapróbálás'),
              ),
            ],
          ),
        ),
        data: (events) {
          if (events.isEmpty) {
            return const Center(
              child: Text('Nincs megjeleníthető esemény.',
                style: TextStyle(color: AppColors.textSecondary)),
            );
          }
          return NotificationListener<ScrollNotification>(
            onNotification: (notification) {
              if (notification.metrics.pixels >=
                  notification.metrics.maxScrollExtent - 200) {
                ref.read(eventsProvider.notifier).loadMore();
              }
              return false;
            },
            child: RefreshIndicator(
              onRefresh: () => ref.read(eventsProvider.notifier).loadEvents(),
              color: AppColors.accent,
              child: ListView.builder(
                itemCount: events.length,
                itemBuilder: (context, index) => EventTile(
                  event: events[index],
                  onTap: () => context.push('/events/${events[index].id}/markets'),
                ),
              ),
            ),
          );
        },
      ),
    );
  }
}
```

---

### 6.13 lépés: Piacok képernyő (`features/events/presentation/markets_screen.dart`)

**Mit csináljon:** Egy esemény összes piaca kártyákba rendezve. Minden piac kártyán látszik az összes kimenetel a best_odds és overall_score értékekkel. Egy kimenetre koppintva a részletes nézetre navigál.

```dart
// lib/features/events/presentation/markets_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/constants/app_colors.dart';
import '../data/events_repository.dart';
import '../domain/market_model.dart';
import '../domain/event_model.dart';
import '../../../shared/widgets/value_badge.dart';

final marketsProvider = FutureProvider.family<
    ({EventModel event, List<MarketModel> markets}), int>((ref, eventId) {
  return ref.watch(eventsRepositoryProvider).getEventMarkets(eventId);
});

class MarketsScreen extends ConsumerWidget {
  final int eventId;
  const MarketsScreen({super.key, required this.eventId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(marketsProvider(eventId));

    return Scaffold(
      appBar: AppBar(title: const Text('Fogadási piacok')),
      body: state.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text(e.toString(),
          style: const TextStyle(color: AppColors.error))),
        data: (data) => ListView.builder(
          padding: const EdgeInsets.symmetric(vertical: 8),
          itemCount: data.markets.length,
          itemBuilder: (context, i) => _MarketCard(
            market: data.markets[i],
            eventId: eventId,
          ),
        ),
      ),
    );
  }
}

class _MarketCard extends StatelessWidget {
  final MarketModel market;
  final int eventId;
  const _MarketCard({required this.market, required this.eventId});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(market.nameHu,
              style: const TextStyle(
                color: AppColors.textPrimary,
                fontWeight: FontWeight.w600,
                fontSize: 14,
              ),
            ),
            const SizedBox(height: 8),
            ...market.outcomes.map((outcome) => InkWell(
              onTap: () => context.push(
                '/events/$eventId/markets/outcome/${outcome.id}'
              ),
              borderRadius: BorderRadius.circular(8),
              child: Padding(
                padding: const EdgeInsets.symmetric(vertical: 6, horizontal: 4),
                child: Row(
                  children: [
                    Expanded(
                      child: Text(outcome.nameHu,
                        style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    ),
                    Text(
                      outcome.bestOdds.toStringAsFixed(2),
                      style: const TextStyle(
                        color: AppColors.textPrimary,
                        fontWeight: FontWeight.w600,
                        fontSize: 14,
                      ),
                    ),
                    const SizedBox(width: 8),
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                      decoration: BoxDecoration(
                        color: AppColors.surfaceLight,
                        borderRadius: BorderRadius.circular(4),
                      ),
                      child: Text(
                        outcome.overallScore.toStringAsFixed(1),
                        style: const TextStyle(color: AppColors.accent, fontSize: 12),
                      ),
                    ),
                    const SizedBox(width: 8),
                    ValueBadge(valueLabel: outcome.valueLabel),
                    const Icon(Icons.chevron_right, color: AppColors.textMuted, size: 18),
                  ],
                ),
              ),
            )),
          ],
        ),
      ),
    );
  }
}
```

---

### 6.14 lépés: Kimenetel részlet képernyő (`features/events/presentation/outcome_detail_screen.dart`)

**Mit csináljon:** Megmutatja a likelihood_score, odds_score, overall_score kördiagramokat, best_odds, value_label, és az AI elemzés szövegét (az app aktuális nyelvén).

```dart
// lib/features/events/presentation/outcome_detail_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/constants/app_colors.dart';
import '../data/events_repository.dart';
import '../domain/market_model.dart';
import '../../../shared/widgets/score_badge.dart';
import '../../../shared/widgets/value_badge.dart';
import '../../../core/l10n/app_localizations.dart';

final outcomeDetailProvider = FutureProvider.family<OutcomeDetail, int>((ref, id) {
  return ref.watch(eventsRepositoryProvider).getOutcomeDetail(id);
});

class OutcomeDetailScreen extends ConsumerWidget {
  final int outcomeId;
  const OutcomeDetailScreen({super.key, required this.outcomeId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(outcomeDetailProvider(outcomeId));
    final locale = ref.watch(localeProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Kimenetel részletei')),
      body: state.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text(e.toString())),
        data: (outcome) {
          final analysis = locale == 'hu' ? outcome.analysisHu : outcome.analysisEn;
          return SingleChildScrollView(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Esemény fejléc
                Text(
                  '${outcome.event['home_team']} — ${outcome.event['away_team']}',
                  style: const TextStyle(
                    color: AppColors.textPrimary,
                    fontWeight: FontWeight.bold,
                    fontSize: 18,
                  ),
                ),
                const SizedBox(height: 4),
                Text(outcome.event['league_name'],
                  style: const TextStyle(color: AppColors.textSecondary)),
                const SizedBox(height: 16),
                // Piac és kimenetel neve
                Text(
                  locale == 'hu' ? outcome.market['name_hu'] : outcome.market['name_en'],
                  style: const TextStyle(color: AppColors.textMuted, fontSize: 13),
                ),
                const SizedBox(height: 4),
                Text(
                  locale == 'hu' ? outcome.nameHu : outcome.nameEn,
                  style: const TextStyle(
                    color: AppColors.accent,
                    fontWeight: FontWeight.w600,
                    fontSize: 16,
                  ),
                ),
                const SizedBox(height: 24),
                // Pontszám körök
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceAround,
                  children: [
                    ScoreBadge(score: outcome.likelihoodScore, label: 'Valószínűség'),
                    ScoreBadge(score: outcome.oddsScore, label: 'Odds érték'),
                    ScoreBadge(score: outcome.overallScore, label: 'Összesített'),
                  ],
                ),
                const SizedBox(height: 24),
                // Best odds + value label
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const Text('Legjobb odds',
                          style: TextStyle(color: AppColors.textMuted, fontSize: 12)),
                        Text(
                          outcome.bestOdds.toStringAsFixed(2),
                          style: const TextStyle(
                            color: AppColors.textPrimary,
                            fontWeight: FontWeight.bold,
                            fontSize: 28,
                          ),
                        ),
                      ],
                    ),
                    ValueBadge(valueLabel: outcome.valueLabel),
                  ],
                ),
                const SizedBox(height: 24),
                const Divider(color: AppColors.divider),
                const SizedBox(height: 16),
                // AI elemzés
                const Text('Elemzés',
                  style: TextStyle(
                    color: AppColors.textPrimary,
                    fontWeight: FontWeight.w600,
                    fontSize: 16,
                  ),
                ),
                const SizedBox(height: 8),
                Text(
                  analysis ?? 'Elemzés nem elérhető.',
                  style: const TextStyle(
                    color: AppColors.textSecondary,
                    fontSize: 14,
                    height: 1.6,
                  ),
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```

---

### 6.15 lépés: Toplist képernyő (`features/toplist/presentation/toplist_screen.dart`)

**Mit csináljon:** Ha a felhasználó prémium, lekéri és megmutatja a toplistát. Ha nem prémium (vagy vendég), egy "zárolt" képernyőt mutat az előfizetési felhívással.

```dart
// lib/features/toplist/presentation/toplist_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/constants/app_colors.dart';
import '../../../core/network/api_client.dart';
import '../../../core/constants/api_endpoints.dart';
import '../../../shared/providers/auth_provider.dart';
import '../../../shared/widgets/value_badge.dart';

final toplistProvider = FutureProvider<List<dynamic>>((ref) async {
  final data = await ApiClient.get(ApiEndpoints.toplist);
  return data['outcomes'] as List;
});

class ToplistScreen extends ConsumerWidget {
  const ToplistScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isGuest = ref.watch(isGuestProvider);
    // Bejelentkezési/előfizetési állapot ellenőrzése
    // Egyszerűsítve: ha vendég, locked képernyőt mutatunk
    if (isGuest) return const _ToplistLockedScreen();

    final toplistState = ref.watch(toplistProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Toplista — Legjobb fogadások')),
      body: toplistState.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) {
          // 401 esetén (nem prémium) zárolás
          return const _ToplistLockedScreen();
        },
        data: (outcomes) {
          if (outcomes.isEmpty) {
            return const Center(
              child: Text('Ma még nincsenek toplista tételek.',
                style: TextStyle(color: AppColors.textSecondary)),
            );
          }
          return ListView.builder(
            itemCount: outcomes.length,
            itemBuilder: (context, i) {
              final o = outcomes[i];
              return Card(
                child: ListTile(
                  onTap: () => context.push('/events/0/markets/outcome/${o['id']}'),
                  leading: CircleAvatar(
                    backgroundColor: AppColors.accent.withOpacity(0.15),
                    child: Text('${i + 1}',
                      style: const TextStyle(color: AppColors.accent, fontWeight: FontWeight.bold)),
                  ),
                  title: Text(
                    '${o['home_team']} — ${o['away_team']}',
                    style: const TextStyle(color: AppColors.textPrimary, fontSize: 14),
                  ),
                  subtitle: Text(
                    '${o['name_hu']} · ${o['market_name_hu']}',
                    style: const TextStyle(color: AppColors.textSecondary, fontSize: 12),
                  ),
                  trailing: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    crossAxisAlignment: CrossAxisAlignment.end,
                    children: [
                      Text(
                        (o['overall_score'] as num).toStringAsFixed(1),
                        style: const TextStyle(
                          color: AppColors.accent,
                          fontWeight: FontWeight.bold,
                          fontSize: 18,
                        ),
                      ),
                      ValueBadge(valueLabel: o['value_label']),
                    ],
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }
}

class _ToplistLockedScreen extends StatelessWidget {
  const _ToplistLockedScreen();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Toplista')),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(32),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.lock, size: 64, color: AppColors.accent),
              const SizedBox(height: 20),
              const Text('Prémium tartalom',
                style: TextStyle(
                  color: AppColors.textPrimary,
                  fontSize: 22,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 12),
              const Text(
                'A Toplista a nap legjobb fogadásait tartalmazza.\n'
                'Aktiválj előfizetést az eléréséhez.',
                textAlign: TextAlign.center,
                style: TextStyle(color: AppColors.textSecondary, fontSize: 15, height: 1.5),
              ),
              const SizedBox(height: 32),
              ElevatedButton(
                onPressed: () => context.push('/subscription'),
                child: const Text('Előfizetés megtekintése'),
              ),
              const SizedBox(height: 12),
              TextButton(
                onPressed: () => context.go('/login'),
                child: const Text('Bejelentkezés',
                  style: TextStyle(color: AppColors.textSecondary)),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### 6.16 lépés: Előfizetési képernyő (`features/subscription/presentation/subscription_screen.dart`)

**Mit csináljon:** Az `in_app_purchase` csomag segítségével megmutatja az elérhető terméket (havidíjas előfizetés), kezeli a vásárlási folyamatot, és a receipt-et elküldi a backend-re validálásra.

```dart
// lib/features/subscription/presentation/subscription_screen.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import '../../../core/constants/app_colors.dart';
import '../../../core/network/api_client.dart';
import '../../../core/constants/api_endpoints.dart';

// A termék ID-k egyezniük kell az App Store Connect / Google Play Console beállításaival
const String _subscriptionProductId = 'betshark_premium_monthly';
const String _trialProductId = 'betshark_trial'; // Opcionális, ha külön termék

class SubscriptionScreen extends ConsumerStatefulWidget {
  const SubscriptionScreen({super.key});

  @override
  ConsumerState<SubscriptionScreen> createState() => _SubscriptionScreenState();
}

class _SubscriptionScreenState extends ConsumerState<SubscriptionScreen> {
  final InAppPurchase _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _subscription;
  ProductDetails? _product;
  bool _loading = true;
  bool _purchasing = false;

  @override
  void initState() {
    super.initState();
    _initIAP();
  }

  Future<void> _initIAP() async {
    final available = await _iap.isAvailable();
    if (!available) {
      setState(() => _loading = false);
      return;
    }

    // Vásárlási stream figyelése
    _subscription = _iap.purchaseStream.listen(_onPurchaseUpdate);

    // Termék lekérése
    final response = await _iap.queryProductDetails({_subscriptionProductId});
    if (response.productDetails.isNotEmpty) {
      setState(() {
        _product = response.productDetails.first;
        _loading = false;
      });
    } else {
      setState(() => _loading = false);
    }
  }

  Future<void> _onPurchaseUpdate(List<PurchaseDetails> purchases) async {
    for (final purchase in purchases) {
      if (purchase.status == PurchaseStatus.purchased ||
          purchase.status == PurchaseStatus.restored) {
        // Receipt elküldése a backend-re
        await _validateReceipt(purchase);
        await _iap.completePurchase(purchase);
      }
      if (purchase.status == PurchaseStatus.error) {
        setState(() => _purchasing = false);
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Vásárlás sikertelen.'), backgroundColor: AppColors.error),
          );
        }
      }
    }
  }

  Future<void> _validateReceipt(PurchaseDetails purchase) async {
    try {
      // iOS: verificationData.serverVerificationData = base64 receipt
      // Android: verificationData.serverVerificationData = purchase token
      final platform = purchase.verificationData.source == 'app_store' ? 'apple' : 'google';
      await ApiClient.post(ApiEndpoints.validateReceipt, data: {
        'platform': platform,
        'receipt': purchase.verificationData.serverVerificationData,
      });
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Előfizetés aktiválva!'), backgroundColor: AppColors.success),
        );
        Navigator.of(context).pop();
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Receipt validálás hiba: $e'), backgroundColor: AppColors.error),
        );
      }
    } finally {
      setState(() => _purchasing = false);
    }
  }

  Future<void> _startPurchase() async {
    if (_product == null) return;
    setState(() => _purchasing = true);
    final param = PurchaseParam(productDetails: _product!);
    await _iap.buyNonConsumable(purchaseParam: param);
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('BetShark Prémium')),
      body: _loading
          ? const Center(child: CircularProgressIndicator())
          : SingleChildScrollView(
              padding: const EdgeInsets.all(24),
              child: Column(
                children: [
                  const Icon(Icons.workspace_premium, size: 72, color: AppColors.accent),
                  const SizedBox(height: 16),
                  const Text('BetShark Prémium',
                    style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold, color: AppColors.textPrimary)),
                  const SizedBox(height: 16),
                  // Funkciók listája
                  _FeatureRow(icon: Icons.leaderboard, text: 'Napi Toplista — legjobb fogadások'),
                  _FeatureRow(icon: Icons.trending_up, text: 'Value és Strong Value jelölések'),
                  _FeatureRow(icon: Icons.psychology, text: 'Részletes AI elemzések minden kimenetelre'),
                  const SizedBox(height: 32),
                  if (_product != null) ...[
                    Text(_product!.price,
                      style: const TextStyle(
                        fontSize: 32,
                        fontWeight: FontWeight.bold,
                        color: AppColors.accent,
                      ),
                    ),
                    const Text('/ hónap', style: TextStyle(color: AppColors.textSecondary)),
                    const SizedBox(height: 20),
                    ElevatedButton(
                      onPressed: _purchasing ? null : _startPurchase,
                      child: _purchasing
                          ? const CircularProgressIndicator(color: Colors.white)
                          : const Text('Előfizetés indítása'),
                    ),
                    const SizedBox(height: 12),
                    OutlinedButton(
                      onPressed: () async {
                        await _iap.restorePurchases();
                      },
                      style: OutlinedButton.styleFrom(
                        minimumSize: const Size(double.infinity, 50),
                        foregroundColor: AppColors.textSecondary,
                      ),
                      child: const Text('Vásárlások visszaállítása'),
                    ),
                  ] else
                    const Text('A termék jelenleg nem elérhető.',
                      style: TextStyle(color: AppColors.error)),
                ],
              ),
            ),
    );
  }
}

class _FeatureRow extends StatelessWidget {
  final IconData icon;
  final String text;
  const _FeatureRow({required this.icon, required this.text});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6),
      child: Row(
        children: [
          Icon(icon, color: AppColors.accent, size: 20),
          const SizedBox(width: 12),
          Expanded(child: Text(text, style: const TextStyle(color: AppColors.textSecondary))),
        ],
      ),
    );
  }
}
```

---

### 6.17 lépés: Profil képernyő (`features/profile/presentation/profile_screen.dart`)

**Mit csináljon:** Megjeleníti az előfizetési állapotot, lehetővé teszi a nyelv váltást, jogi dokumentumok megtekintését, kijelentkezést és fiók törlést.

```dart
// lib/features/profile/presentation/profile_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/constants/app_colors.dart';
import '../../../core/network/api_client.dart';
import '../../../core/constants/api_endpoints.dart';
import '../../../shared/providers/auth_provider.dart';
import '../../../core/l10n/app_localizations.dart';

final profileProvider = FutureProvider<Map<String, dynamic>>((ref) async {
  final data = await ApiClient.get(ApiEndpoints.profile);
  return data;
});

class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isGuest = ref.watch(isGuestProvider);
    final locale = ref.watch(localeProvider);

    if (isGuest) {
      return Scaffold(
        appBar: AppBar(title: const Text('Profil')),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('Vendégként böngészel.',
                style: TextStyle(color: AppColors.textSecondary)),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => context.go('/login'),
                child: const Text('Bejelentkezés / Regisztráció'),
              ),
            ],
          ),
        ),
      );
    }

    final profileState = ref.watch(profileProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Profil')),
      body: profileState.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text(e.toString())),
        data: (data) {
          final user = data['user'];
          final sub = data['subscription'];
          return ListView(
            padding: const EdgeInsets.all(16),
            children: [
              // Email
              _ProfileTile(
                icon: Icons.email_outlined,
                label: 'Email',
                value: user['email'] ?? '—',
              ),
              // Előfizetési állapot
              _ProfileTile(
                icon: Icons.workspace_premium,
                label: 'Előfizetés',
                value: _subscriptionLabel(sub),
                onTap: () => context.push('/subscription'),
              ),
              const Divider(color: AppColors.divider),
              // Nyelv
              _ProfileTile(
                icon: Icons.language,
                label: 'Nyelv',
                value: locale == 'hu' ? 'Magyar' : 'English',
                onTap: () async {
                  final newLang = locale == 'hu' ? 'en' : 'hu';
                  await ref.read(localeProvider.notifier).setLanguage(newLang);
                  await ApiClient.put(ApiEndpoints.updateLanguage,
                    data: {'language': newLang});
                },
              ),
              // Jogi dokumentumok
              _ProfileTile(
                icon: Icons.description_outlined,
                label: 'Jogi dokumentumok',
                onTap: () {
                  // Megnyitja a jogi dokumentumokat (URL Launcher vagy WebView)
                },
              ),
              const Divider(color: AppColors.divider),
              // Kijelentkezés
              _ProfileTile(
                icon: Icons.logout,
                label: 'Kijelentkezés',
                color: AppColors.error,
                onTap: () async {
                  await ref.read(authStateProvider.notifier).logout();
                  if (context.mounted) context.go('/login');
                },
              ),
              // Fiók törlés
              _ProfileTile(
                icon: Icons.delete_forever_outlined,
                label: 'Fiók törlése',
                color: AppColors.error,
                onTap: () => _confirmDeleteAccount(context, ref),
              ),
            ],
          );
        },
      ),
    );
  }

  String _subscriptionLabel(Map<String, dynamic>? sub) {
    if (sub == null) return 'Ingyenes';
    if (sub['status'] == 'active') return 'Aktív prémium';
    if (sub['status'] == 'trial') return 'Próbaidőszak';
    return 'Lejárt';
  }

  Future<void> _confirmDeleteAccount(BuildContext context, WidgetRef ref) async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        backgroundColor: AppColors.surface,
        title: const Text('Fiók törlése', style: TextStyle(color: AppColors.textPrimary)),
        content: const Text(
          'Biztosan törölni szeretnéd a fiókodat? Ez a művelet nem vonható vissza.',
          style: TextStyle(color: AppColors.textSecondary),
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx, false), child: const Text('Mégse')),
          TextButton(
            onPressed: () => Navigator.pop(ctx, true),
            child: const Text('Törlés', style: TextStyle(color: AppColors.error)),
          ),
        ],
      ),
    );

    if (confirmed == true) {
      await ApiClient.delete(ApiEndpoints.deleteAccount);
      await ref.read(authStateProvider.notifier).logout();
      if (context.mounted) context.go('/login');
    }
  }
}

class _ProfileTile extends StatelessWidget {
  final IconData icon;
  final String label;
  final String? value;
  final VoidCallback? onTap;
  final Color? color;

  const _ProfileTile({
    required this.icon,
    required this.label,
    this.value,
    this.onTap,
    this.color,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Icon(icon, color: color ?? AppColors.textSecondary),
      title: Text(label, style: TextStyle(color: color ?? AppColors.textPrimary, fontSize: 15)),
      subtitle: value != null
          ? Text(value!, style: const TextStyle(color: AppColors.textSecondary, fontSize: 13))
          : null,
      trailing: onTap != null
          ? const Icon(Icons.chevron_right, color: AppColors.textMuted)
          : null,
      onTap: onTap,
    );
  }
}
```

---

### 6.18 lépés: `main.dart` belépési pont

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'app.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(
    const ProviderScope(
      child: BetSharkApp(),
    ),
  );
}
```

---

### 6.19 lépés: Android natív konfiguráció

**`android/app/build.gradle`** — minSdkVersion beállítása (Google Sign-In és IAP miatt legalább 21):

```gradle
android {
    defaultConfig {
        minSdkVersion 21
        targetSdkVersion 34
    }
}
```

**`android/app/src/main/AndroidManifest.xml`** — Internet engedély:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

Google Sign-In-hez add hozzá a `google-services.json` fájlt az `android/app/` könyvtárba (letöltés a Firebase Console-ból), és az `android/build.gradle`-ban:

```gradle
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.4.1'
    }
}
```

Az `android/app/build.gradle` végére:

```gradle
apply plugin: 'com.google.gms.google-services'
```

---

### 6.20 lépés: iOS natív konfiguráció

**`ios/Runner/Info.plist`** — URL scheme Google Sign-In-hez:

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <!-- A REVERSED_CLIENT_ID értéket a GoogleService-Info.plist-ből kell kiolvasni -->
      <string>com.googleusercontent.apps.YOUR_CLIENT_ID</string>
    </array>
  </dict>
</array>

<!-- Apple Sign-In entitlement -->
<key>com.apple.developer.applesignin</key>
<array>
  <string>Default</string>
</array>
```

Az Apple Sign-In képességet az Xcode-ban is be kell kapcsolni: **Signing & Capabilities → + Capability → Sign in with Apple**.

Az `ios/Runner/Runner.entitlements` fájlban:

```xml
<key>com.apple.developer.in-app-payments</key>
<array>
    <string>betshark_premium_monthly</string>
</array>
```

---

## 7. Futtatás és deployment

### Lokális futtatás

```bash
cd betshark_app
flutter pub get
flutter run                    # Alapértelmezett eszközön
flutter run -d android         # Android emulátoron
flutter run -d ios             # iOS szimulátoron
```

### Build

```bash
# Android APK
flutter build apk --release --dart-define=BASE_URL=https://api.betshark.app

# Android App Bundle (Play Store)
flutter build appbundle --release --dart-define=BASE_URL=https://api.betshark.app

# iOS (Xcode-ból archive és feltöltés)
flutter build ios --release --dart-define=BASE_URL=https://api.betshark.app
```

### App Store / Google Play előkészítés

**App Store Connect:**
1. Hozd létre az alkalmazást az App Store Connect-en.
2. Hozd létre az előfizetési terméket: **App Store Connect → Features → In-App Purchases → Subscriptions**.
   - Termék ID: `betshark_premium_monthly`
   - Ár: 2990 HUF / hó
3. A Sandbox tesztelők beállítása a teszteléshez.

**Google Play Console:**
1. Hozd létre az alkalmazást a Google Play Console-on.
2. **Monetization → Subscriptions → Create subscription**.
   - Termék ID: `betshark_premium_monthly`

### Fontos megfontolások

- **iOS szimulátoron az Apple Sign-In és az IAP nem működik** — fizikai eszköz szükséges a teszteléshez.
- **Android emulátron a Google IAP szintén fizikai eszközt igényelhet** tesztelési módban.
- **flutter_secure_storage iOS-en a Keychain-t használja**, Android-on az EncryptedSharedPreferences-t — mindkettő megbízható, de az iOS-en a backup-ból visszaállítás törölheti a tokent.
- **A GoRouter `ShellRoute`** biztosítja, hogy az BottomNavigationBar megmaradjon az aloldalak navigációjakor.
- **Riverpod `FutureProvider.family`** az esemény- és kimenetel-detailoknál biztosítja, hogy minden különböző ID külön cache-t kap, és a widget eltávolításakor automatikusan felszabadítja.
- **In-app purchase lifecycle:** A `purchaseStream`-et az `initState`-ben kell feliratkozni és a `dispose`-ban lemondani. Ne add a `build()` metódusba.
- **Trial abuse megelőzés:** A backend kezeli ezt (hashed payment identifier). A Flutter oldalon csak annyit kell megtenni, hogy a `startTrial` API híváshoz átadjuk az `originalTransactionId` (Apple) vagy `purchaseToken` (Google) értéket `payment_identifier` paraméterként.
