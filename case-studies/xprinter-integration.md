# XPrinter 네이티브 연동

## 문제

모바일 송장 출력은 단순히 Bluetooth 데이터를 전송하는 것으로 끝나지 않았습니다. Android와 iOS의 연결 방식이 다르고, 프린터 기종과 펌웨어에 따라 상태 응답 및 전송 완료 시점이 달랐습니다. 구형 Android 기기에서는 이전 연결의 콜백이 늦게 도착해 현재 세션에 영향을 주는 상황도 고려해야 했습니다.

## 구조

```text
React Native / TypeScript
        ↓ 공통 프린터 인터페이스
Expo Native Module
        ├─ Android: Kotlin + XPrinter SDK
        └─ iOS: Swift / Objective-C + BLE Bridge
        ↓
TSPL Command / Printer Status
```

앱 화면은 플랫폼별 SDK를 직접 알지 않고 검색, 연결, 상태 확인, 단일 출력, 다중 출력, 연결 해제라는 공통 인터페이스만 사용합니다.

## 주요 구현

### 연결 세션 관리

- 동시에 여러 연결 요청이 진행되지 않도록 차단
- 연결 세대(connection generation)를 증가시켜 이전 세션의 늦은 콜백 무시
- 재연결 전 기존 SDK 연결을 명시적으로 종료
- 네이티브 모듈 종료 시 현재 프린터 세션 정리

### 순차 출력

- 송장 데이터를 TSPL Base64 payload로 전달
- 여러 송장을 하나의 배치로 받고 순서대로 전송
- 최종 PRINT 명령 이후 프린터 응답 또는 상태 장벽 확인
- 전송 중 중복 출력 요청 방지

### 상태와 타임아웃

- 프린터 상태 조회에 제한 시간을 적용
- SDK의 byte status 해석 차이를 보정
- 연결, 전송, 상태 조회 오류를 구분해 앱으로 반환
- 사용자에게 재연결, 용지 확인 등 실행 가능한 안내 제공

### 안전 검증

- 허용하지 않는 TSPL 명령 차단
- 단일 및 배치 payload 크기 제한
- 프린터 capability와 media sensor 정보를 가능한 범위에서 확인
- 진단 로그에 연결 및 전송 시간을 기록

## iOS와 Android 차이 처리

Android는 XPrinter SDK의 Classic Bluetooth 연결과 콜백 수명 주기를 관리했습니다. iOS는 BLE 검색·연결·전송을 Objective-C 브리지와 Swift Expo Module로 연결했습니다. 플랫폼별 차이는 네이티브 구현에 남기고 React Native 레이어에는 동일한 계약을 제공했습니다.

## 검증

- 출력 순서와 중복 실행 방지
- 앱 재시작 후 프린터 선택 복원
- iOS 연결 실패 후 복구
- 전송 중 연결 해제
- 다중 송장 출력 순서
- 지원하지 않는 프린터 또는 명령 처리

## 결과

React Native가 제공하지 않는 하드웨어 연동 문제를 네이티브 코드와 프린터 프로토콜 수준에서 해결했습니다. 앱 사용자는 플랫폼 차이를 의식하지 않고 주문 상세 화면에서 송장을 출력할 수 있습니다.

