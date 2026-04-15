# App Version Manager

앱 강제 업데이트를 관리하는 경량 버전 체크 시스템.  
GitHub Pages에 호스팅된 JSON 파일 하나로 모든 앱의 버전을 관리합니다.

## 구조

```
├── index.html       # 버전 관리 대시보드
├── version.json     # 앱 버전 데이터 (앱에서 이 파일을 읽음)
└── README.md
```

## 설정 방법

### 1. GitHub 레포 생성
```bash
git init
git add .
git commit -m "init: app version manager"
git remote add origin https://github.com/YOUR_USERNAME/app-version-manager.git
git push -u origin main
```

### 2. GitHub Pages 활성화
- Settings → Pages → Source: `main` branch → Save
- 배포 완료 후 접속: `https://YOUR_USERNAME.github.io/app-version-manager/`

### 3. 엔드포인트
```
https://YOUR_USERNAME.github.io/app-version-manager/version.json
```

## version.json 형식

플랫폼별로 버전을 따로 관리합니다 (AOS/iOS 심사 일정 차이 대응).

```json
{
  "app_key": {
    "name": "앱 이름",
    "android": {
      "min_version": "1.0.0",
      "latest_version": "1.2.0",
      "app_id": "com.example.app"
    },
    "ios": {
      "min_version": "1.0.0",
      "latest_version": "1.2.0",
      "app_id": "000000000"
    }
  }
}
```

| 필드 | 설명 |
|------|------|
| `name` | 대시보드에 표시될 앱 이름 |
| `android` / `ios` | 플랫폼별 버전 정보 (둘 중 하나만 있어도 OK) |
| `min_version` | 이 버전 미만이면 **강제 업데이트** dialog (취소 불가) |
| `latest_version` | 최신 버전. `min_version`과 같으면 강제, 크면 선택 업데이트 안내 |
| `app_id` | Android: package name (`com.example.app`) / iOS: Apple ID (숫자, 예: `1234567890`) |

### 정책 예시
- **강제**: `min_version = latest_version = "1.2.0"`
- **선택**: `min_version = "1.0.0"`, `latest_version = "1.2.0"`
- **조용히**: `min_version`을 충분히 낮게 두기

## Flutter 연동

```yaml
# pubspec.yaml
dependencies:
  http: ^1.1.0
  package_info_plus: ^8.0.0
  store_redirect: ^2.0.3
```

```dart
import 'dart:convert';
import 'dart:io' show Platform;
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:package_info_plus/package_info_plus.dart';
import 'package:store_redirect/store_redirect.dart';

class VersionChecker {
  static const _url = 'https://YOUR_USERNAME.github.io/app-version-manager/version.json';
  static const _appKey = 'app_a'; // 앱마다 다르게 설정
  static DateTime? _lastCheck;
  static bool _dialogShown = false;

  static Future<void> checkIfNeeded(BuildContext context) async {
    final now = DateTime.now();
    if (_lastCheck != null && now.difference(_lastCheck!).inHours < 3) return;
    _lastCheck = now;

    try {
      final res = await http.get(Uri.parse(_url));
      if (res.statusCode != 200) return;

      final data = jsonDecode(res.body) as Map<String, dynamic>;
      final appInfo = data[_appKey] as Map<String, dynamic>?;
      if (appInfo == null) return;

      // store_redirect는 양쪽 ID를 모두 받아 플랫폼 자동 감지
      final androidId = (appInfo['android']?['app_id'] as String?) ?? '';
      final iosId = (appInfo['ios']?['app_id'] as String?) ?? '';

      final platformKey = Platform.isIOS ? 'ios' : 'android';
      final platform = appInfo[platformKey] as Map<String, dynamic>?;
      if (platform == null) return;

      final packageInfo = await PackageInfo.fromPlatform();
      final current = packageInfo.version;
      final minVersion = platform['min_version'] as String?;
      final latestVersion = platform['latest_version'] as String?;

      if (!context.mounted || _dialogShown) return;

      // 강제 업데이트
      if (minVersion != null && _isBelow(current, minVersion)) {
        _dialogShown = true;
        _showUpdateDialog(context, forceUpdate: true,
            androidId: androidId, iosId: iosId);
        return;
      }
      // 선택 업데이트
      if (latestVersion != null && _isBelow(current, latestVersion)) {
        _dialogShown = true;
        _showUpdateDialog(context, forceUpdate: false,
            androidId: androidId, iosId: iosId);
      }
    } catch (_) {}
  }

  /// "1.2.3", "1.2.3+45" 등 안전 비교. 자릿수 다르면 0으로 패딩.
  static bool _isBelow(String current, String target) {
    int parse(String s) => int.tryParse(s.split('+').first) ?? 0;
    final c = current.split('.').map(parse).toList();
    final t = target.split('.').map(parse).toList();
    final len = c.length > t.length ? c.length : t.length;
    for (var i = 0; i < len; i++) {
      final ci = i < c.length ? c[i] : 0;
      final ti = i < t.length ? t[i] : 0;
      if (ci < ti) return true;
      if (ci > ti) return false;
    }
    return false;
  }

  static void _showUpdateDialog(
    BuildContext context, {
    required bool forceUpdate,
    required String androidId,
    required String iosId,
  }) {
    showDialog(
      context: context,
      barrierDismissible: !forceUpdate,
      builder: (_) => PopScope(
        canPop: !forceUpdate,
        child: AlertDialog(
          title: const Text('업데이트 필요'),
          content: Text(
            forceUpdate
                ? '새 버전이 출시되었습니다.\n계속 사용하려면 업데이트가 필요합니다.'
                : '새 버전이 출시되었습니다.\n업데이트하시겠습니까?',
          ),
          actions: [
            if (!forceUpdate)
              TextButton(
                onPressed: () {
                  _dialogShown = false;
                  Navigator.pop(context);
                },
                child: const Text('나중에'),
              ),
            FilledButton(
              onPressed: () => StoreRedirect.redirect(
                androidAppId: androidId,
                iOSAppId: iosId,
              ),
              child: const Text('업데이트'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 호출 위치

```dart
class _AppState extends State<MyApp> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    VersionChecker.checkIfNeeded(context);
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      VersionChecker.checkIfNeeded(context);
    }
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
}
```

## 업데이트 방법

버전을 바꾸고 싶을 때:

```bash
# version.json 수정 후
git add version.json
git commit -m "update: app_a min_version to 1.2.0"
git push
```

GitHub Pages 캐시는 보통 수 분 내 갱신됩니다.
