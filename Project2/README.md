# 프로젝트 2: 자유 주제 자동화 — 오마이보카 신규 사용내역 알림 (2시간 스로틀)

## 반복 업무 정의

"오마이보카"(단어 학습 앱) 사용자가 새로운 학습 활동(단어 학습 등)을 하면 이를 실시간으로 팀 Slack에 공유하고 싶다. 다만 같은 사용자가 짧은 시간에 여러 번 활동해도 그때마다 알림이 오면 스팸이 되므로, **같은 사용자 기준 2시간 이내에는 추가 알림을 보내지 않는다.**

## 도구 선정 및 이유

**n8n**을 선택했다. 이유:
- Project 1에서 이미 n8n 인프라(셀프호스팅, Google Sheets/Slack 연동)를 구축해둔 상태라 재사용 가능
- Webhook 트리거와 Code 노드(자바스크립트 실행)를 자유롭게 조합할 수 있어, "직전 알림 시각과 비교해 2시간 경과 여부 판단"이라는 커스텀 로직을 구현하기에 적합
- REST API가 공개돼 있어 워크플로우를 스크립트로 만들고 curl로 즉시 테스트할 수 있었음

## 워크플로우 흐름

```
Webhook (Trigger, POST /webhook/ohmyvoca-usage)
  → Google Sheets 조회 (Action 1: "ohmyvoca-alerts" 탭에서 전체 사용자 알림 이력 읽기)
  → Code 노드 (해당 userId의 마지막 알림 시각을 찾아 현재와 비교, 2시간 경과 여부 판단)
  → IF 분기 (shouldAlert == true?)
      True  → Slack 알림 발송 (Action 2) + Google Sheets에 알림 시각 기록 (Action 3)
      False → 아무 것도 하지 않음 (스킵)
```

- **Trigger**: Webhook (오마이보카 서버가 신규 사용내역 발생 시 `{ "userId": "...", "detail": "..." }` 형태로 POST 호출한다고 가정)
- **Action 1**: Google Sheets API로 `ohmyvoca-alerts` 탭 전체를 조회
- **분기 로직**: Code 노드에서 해당 `userId`의 가장 최근 알림 시각을 찾아 현재 시각과의 차이(시간)를 계산 — 기록이 없거나 2시간 이상 지났으면 `shouldAlert: true`
- **Action 2**: Slack `#notification` 채널에 알림 발송 (True일 때만)
- **Action 3**: Google Sheets에 `[userId, 알림시각]` 행 추가 — 다음 판단의 기준이 됨 (True일 때만)

## 구현 화면

**구성도**: `screenshots/project2-flow.png` (추가 예정)

## 실행 결과 (curl로 테스트, n8n Executions에서 확인)

같은 사용자(`user_test_01`)로 2초 간격을 두고 웹훅을 두 번 호출한 결과:

| 호출 | shouldAlert | Slack 알림 | Sheets 기록 |
|---|---|---|---|
| 1번째 (신규) | `true` | 발송됨 | 기록됨 |
| 2번째 (2초 후) | `false` (경과 0.0006시간) | 스킵됨 | 스킵됨 |

두 분기(True/False) 모두 실제로 최소 1회 이상 실행된 것을 확인했다.

- [ ] n8n Executions 상세 — 두 실행(1번째 alert 발송, 2번째 스킵)이 각각 다른 경로를 탄 것이 보이는 캡처
- [ ] 실제 발송된 Slack 메시지 캡처
- [ ] Google Sheets "ohmyvoca-alerts" 탭 — userId/알림시각 1행만 기록되고 2번째 호출은 기록되지 않은 것을 보여주는 캡처

> 실제 서비스 연동 시에는 오마이보카 백엔드에서 사용내역 발생 시 이 Webhook URL로 POST 요청을 보내도록 구현하면 된다. 이번 과제에서는 curl로 웹훅 호출을 재현해 동일한 트리거 조건을 검증했다.
