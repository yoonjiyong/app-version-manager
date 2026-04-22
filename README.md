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
| `latest_version` | 최신 버전. 현재 Flutter 코드에서는 **사용 안 함** (선택 업데이트 미제공, 스키마는 호환용으로 유지) |
| `app_id` | Android: package name (`com.example.app`) / iOS: Apple ID (숫자, 예: `1234567890`) |

> **선택 업데이트는 제공하지 않음**: UX 복잡도 대비 가치가 낮아 강제 업데이트 한 가지만 지원합니다. 최신 버전 유도는 스토어의 자동 업데이트에 맡깁니다.

### 정책 예시
- **강제**: `min_version`을 새 버전과 동일하게 올리면 구버전 차단
- **조용히**: `min_version`을 충분히 낮게 두기

## Flutter 연동

- **강제 업데이트만 지원** (선택 업데이트는 제공 안 함)
- 3시간 throttle, 앱 `resumed` 때 재확인
- 뒤로가기/바깥 탭으로 닫을 수 없는 blocking dialog

### pubspec.yaml

```yaml
dependencies:
  http: ^1.2.0                 # JSON fetch
  package_info_plus: ^9.0.1    # 현재 앱 버전 조회
  store_redirect: ^2.0.3       # Play / App Store 이동
```

### VersionChecker 서비스

`lib/core/services/version_checker.dart`

```dart
import 'dart:convert';
import 'dart:io' show Platform;

import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:package_info_plus/package_info_plus.dart';
import 'package:store_redirect/store_redirect.dart';

/// 원격 version.json을 읽어 강제 업데이트 다이얼로그를 띄운다.
/// - min_version 미만: 강제 업데이트 (취소/뒤로가기 불가)
/// - latest_version은 현재 사용 안 함 (선택 업데이트 비활성)
class VersionChecker {
  VersionChecker._();

  static const _url =
      'https://YOUR_USERNAME.github.io/app-version-manager/version.json';
  static const _appKey = 'my_app_key'; // version.json 의 앱 키

  static DateTime? _lastCheck;
  static bool _dialogShown = false;

  /// 3시간마다 최대 1번만 호출되도록 throttling.
  /// 다이얼로그가 이미 떠 있는 동안에는 중복 호출 무시.
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

      final androidId = (appInfo['android']?['app_id'] as String?) ?? '';
      final iosId = (appInfo['ios']?['app_id'] as String?) ?? '';

      final platformKey = Platform.isIOS ? 'ios' : 'android';
      final platform = appInfo[platformKey] as Map<String, dynamic>?;
      if (platform == null) return;

      final packageInfo = await PackageInfo.fromPlatform();
      final current = packageInfo.version;
      final minVersion = platform['min_version'] as String?;

      if (!context.mounted || _dialogShown) return;

      if (minVersion != null && _isBelow(current, minVersion)) {
        _dialogShown = true;
        await _showUpdateDialog(
          context,
          androidId: androidId,
          iosId: iosId,
        );
      }
    } catch (e) {
      debugPrint('[VersionChecker] check failed: $e');
    }
  }

  /// 테스트용: 실제 버전 비교를 우회하고 강제 업데이트 다이얼로그를 바로 띄움.
  /// 프로덕션 UI에는 노출하지 말 것.
  static Future<void> showTestDialog(BuildContext context) {
    _dialogShown = false;
    return _showUpdateDialog(
      context,
      androidId: 'com.example.app',
      iosId: '0000000000',
    );
  }

  /// "1.2.3", "1.2.3+45" 등 semver 안전 비교. 자릿수 다르면 0으로 패딩.
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

  static Future<void> _showUpdateDialog(
    BuildContext context, {
    required String androidId,
    required String iosId,
  }) {
    final scheme = Theme.of(context).colorScheme;
    return showDialog<void>(
      context: context,
      barrierDismissible: false,
      barrierColor: Colors.black.withValues(alpha: 0.55),
      builder: (dialogCtx) => PopScope(
        canPop: false,
        child: Dialog(
          backgroundColor: scheme.surface,
          elevation: 0,
          insetPadding:
              const EdgeInsets.symmetric(horizontal: 32, vertical: 24),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(20),
          ),
          child: Padding(
            padding: const EdgeInsets.fromLTRB(24, 28, 24, 20),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Container(
                  width: 64,
                  height: 64,
                  decoration: BoxDecoration(
                    color: scheme.primaryContainer,
                    shape: BoxShape.circle,
                    border: Border.all(
                      color: scheme.primary.withValues(alpha: 0.25),
                    ),
                  ),
                  alignment: Alignment.center,
                  child: Icon(
                    Icons.rocket_launch_rounded,
                    size: 32,
                    color: scheme.primary,
                  ),
                ),
                const SizedBox(height: 18),
                Text(
                  '업데이트가 필요해요',
                  style: TextStyle(
                    fontSize: 18,
                    fontWeight: FontWeight.w700,
                    color: scheme.onSurface,
                    letterSpacing: -0.2,
                  ),
                ),
                const SizedBox(height: 10),
                Text(
                  '더 나은 경험을 위해\n최신 버전으로 업데이트해 주세요.',
                  textAlign: TextAlign.center,
                  style: TextStyle(
                    fontSize: 14,
                    height: 1.45,
                    color: scheme.onSurfaceVariant,
                  ),
                ),
                const SizedBox(height: 24),
                SizedBox(
                  width: double.infinity,
                  height: 50,
                  child: FilledButton(
                    style: FilledButton.styleFrom(
                      backgroundColor: scheme.primary,
                      foregroundColor: scheme.onPrimary,
                      elevation: 0,
                      shape: RoundedRectangleBorder(
                        borderRadius: BorderRadius.circular(12),
                      ),
                    ),
                    onPressed: () {
                      StoreRedirect.redirect(
                        androidAppId: androidId,
                        iOSAppId: iosId,
                      );
                    },
                    child: const Text(
                      '지금 업데이트',
                      style: TextStyle(
                        fontSize: 15,
                        fontWeight: FontWeight.w600,
                      ),
                    ),
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

> 프로젝트에 고유 팔레트가 있다면 `scheme.*` 자리를 `AppColors.card / accent / onSurface` 등 프로젝트 토큰으로 치환하세요.

### 앱 진입점 통합

`initState`의 `addPostFrameCallback`에서 첫 체크, `didChangeAppLifecycleState.resumed`에서 재체크.
GoRouter 쓰는 경우 root navigator key의 context로 전역 다이얼로그 오픈.

```dart
class _MyAppState extends State<MyApp> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    WidgetsBinding.instance.addPostFrameCallback((_) => _runVersionCheck());
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) _runVersionCheck();
  }

  void _runVersionCheck() {
    final ctx = _rootNavigatorKey.currentContext; // GoRouter 사용 시
    if (ctx == null) return;
    VersionChecker.checkIfNeeded(ctx);
  }

  @override
  Widget build(BuildContext context) {
    // ... MaterialApp.router
    return const SizedBox.shrink();
  }
}
```

### QA 테스트 버튼 (릴리즈 전 삭제)

실기기에서 다이얼로그 모양을 바로 확인하려면 마이페이지 등 개발용 메뉴에 임시 버튼 추가.
`showTestDialog`는 네트워크/버전 비교를 우회한다.

```dart
ListTile(
  title: const Text('업데이트 다이얼로그 미리보기'),
  onTap: () => VersionChecker.showTestDialog(context),
),
```

### 체크리스트

- [ ] `pubspec.yaml`에 `http`, `package_info_plus`, `store_redirect` 추가
- [ ] `VersionChecker._url`을 자기 GitHub Pages URL로 교체
- [ ] `_appKey`를 `version.json`에 쓰는 키로 교체
- [ ] `showTestDialog`의 `androidId`, `iosId`를 실제 앱 ID로 교체
- [ ] 앱 진입점에서 `WidgetsBindingObserver`로 lifecycle 훅 연결
- [ ] GoRouter 쓰는 경우 root navigator context 사용
- [ ] 다크/라이트 모드 모두에서 다이얼로그 시각 확인
- [ ] 실기기에서 뒤로가기로 닫히지 않는지 확인 (`PopScope canPop: false`)

## 업데이트 방법

버전을 바꾸고 싶을 때:

```bash
# version.json 수정 후
git add version.json
git commit -m "update: app_a min_version to 1.2.0"
git push
```

GitHub Pages 캐시는 보통 수 분 내 갱신됩니다.
