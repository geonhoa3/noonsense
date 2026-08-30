# 눈치챙겨 (NoonSense)

**위치 기반 자동 무음 안드로이드 앱** — 조용해야 하는 장소에 들어가면 휴대폰을 알아서 무음/진동으로 바꿔줍니다.

> 도서관·병원·종교시설처럼 벨소리가 민망한 곳에서, 사용자가 매번 직접 무음을 켜고 끄지 않아도 되도록 만든 앱입니다. 특히 무음 설정 자체가 익숙지 않은 **50~80대**도 쉽게 쓸 수 있는 것을 목표로 했습니다.

- **Google Play**: https://play.google.com/store/apps/details?id=com.chobo.noonsense
- **패키지**: `com.chobo.noonsense`
- **개발**: 1인 (기획 · UI/UX · 개발 · 배포 · 운영)

---

## 만든 계기

성당에서 미사를 드릴 때 50~80대의 어르신들의 휴대폰 벨소리가 울려 서로 민망해지는 상황을 자주 봤습니다. 무음 전환을 깜빡하는 건 누구나 겪는 일이고, 특히 스마트폰이 익숙지 않은 어르신에게는 더 큰 불편이라는 점에 주목해 "신경 쓰지 않아도 조용한 곳에서는 알아서 무음이 되는" 앱을 만들었습니다.

## 주요 기능
<table>
  <tr>
    <td><img alt="image" src="https://github.com/user-attachments/assets/6806081b-02a9-490e-af35-2054d947b582" width="200"/></td>
    <td><img alt="image" src="https://github.com/user-attachments/assets/93782e66-e5d9-4ccf-a3ad-a2f06e48947e" width="200"/></td>
    <td><img alt="image" src="https://github.com/user-attachments/assets/4827997e-cc5b-4961-8c13-114a2b427346" width="200"/></td>
    <td><img alt="image" src="https://github.com/user-attachments/assets/a7d2f357-0ab8-46d5-83cf-58453af33fd9" width="200"/></td>
  </tr>
</table>


- **등록 장소 자동 무음** — 사용자가 지정한 곳을 지도 선택/검색으로 등록하면, 그 구역에 들어갈 때 자동으로 무음/진동 전환 (Geofencing)
- **카테고리 자동 무음** — 도서관·영화관·성당·교회·절·병원 등은 **등록 없이도** 주변을 인식해 자동 무음 (카카오 로컬 검색 기반)
- **무음존 부재중 알림** — 무음 지역에서 놓친 전화를, 지역을 벗어날 때 음성(TTS)으로 안내
- **어르신 배려 UX** — 그림으로 따라 하는 권한 설정 가이드 + 필수 권한 완료 전에는 사용 화면으로 넘어가지 못하게 막는 온보딩 게이트

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| 언어 | Kotlin |
| UI | Jetpack Compose (Material3), Coroutine |
| 위치 | FusedLocationProvider, Geofencing API, Foreground Service |
| 외부 연동 | Kakao Local REST API (HttpURLConnection + Gson) |
| 기타 | AdMob, Google Play In-App Update, TextToSpeech, BroadcastReceiver, SharedPreferences |
| 빌드 | Gradle (KTS), ProGuard/R8, minSdk 26 / targetSdk 36 |

## 핵심 설계

- **거리 기반 4단계 적응형 위치 추적** — 목적지까지의 거리에 따라 확인 주기·정밀도를 4단계(구역 안 / 100m 이내 / 100~500m / 500m 초과)로 차등화. 멀 때는 최소한으로 확인해 배터리 부담을 줄이고, 가까울수록 정밀하게 감지.
- **정확도 기반 진입 판정** — GPS 오차(수십 m)로 인한 오작동을 막기 위해, 카테고리 장소는 `거리 + GPS 정확도 ≤ 반경`일 때만(오차를 감안해도 확실히 안쪽일 때만) 무음으로 판정.
- **카테고리 = 가상 지오펜스** — 좌표를 미리 알 수 없는 카테고리 장소는 카카오 검색 결과를 기존 위치 판정 루프에 그대로 합류시켜, 로직 중복 없이 확장.
- **오작동 방지 체류(dwell)** — 카테고리 장소는 일정 시간(2분) 머물러야 무음으로 전환해, 옆을 지나가기만 할 때 잘못 무음되는 것을 방지.

## 측정/성과

- **배터리**: 소모가 가장 큰 조건(카테고리 감지 활성)에서도 하루 약 **0.2%** (실기기 배터리 사용량 기준, 대부분 백그라운드)
- **위치 요청 빈도**: 원거리 상태에서 상시 추적 대비 약 **95% 절감** (설계값 기준)
- **비용**: 유료 API 대신 카카오 무료 쿼터를 활용해 사용자당 API 호출 비용 **0원**

## 프로젝트 구조

```
app/src/main/java/com/chobo/noonsense/
├─ MainActivity.kt          # 화면/상태/권한 흐름 (Compose)
├─ LocationMonitorService.kt# 위치 감시 Foreground Service, 적응형 추적, 카테고리 감지
├─ GeofenceHelper.kt        # 등록 장소 Geofence 등록/검색
├─ ModeManager.kt           # 무음/진동/벨소리 모드 전환
├─ KakaoLocalApi.kt         # 카카오 로컬 REST 호출 + 파싱
├─ SilenceCategories.kt     # 카테고리(도서관·병원 등) 선택/저장
└─ PermissionGuide.kt       # 어르신용 이미지 권한 가이드
```

## 빌드 방법

1. `local.properties`에 카카오 REST API 키 추가:
   ```properties
   KAKAO_REST_API_KEY=발급받은_카카오_REST_API_키
   ```
   > 키는 소스에 하드코딩하지 않고 빌드 시 `BuildConfig`로 주입됩니다.
2. 빌드:
   ```bash
   ./gradlew assembleDebug     # 디버그 APK
   ./gradlew assembleRelease   # 릴리스 (서명 필요)
   ```

## 개발자

**LoG H** — 개인 프로젝트 (2026.01~)
