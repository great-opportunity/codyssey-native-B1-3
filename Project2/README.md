# 프로젝트 2: 자유 주제 자동화 — 메일 자동분석 → 슬랙 알림

## 반복 업무 정의

메일함에 새 메일이 쌓이면 하나씩 열어서 내용을 읽고, 중요한 건인지 광고성인지 판단하고, 필요하면 후속 조치를 취해야 한다. 바쁠 때는 이 확인 자체를 미루게 되고, 그러다 정말 중요한 메일을 놓치는 경우가 있었다. **새 메일이 오면 AI가 내용을 읽고 요약·카테고리 분류·후속 조치 제안을 자동으로 만들어서, 광고/스팸이 아닌 것만 슬랙으로 바로 알려준다.**

## 도구 선정 및 이유

**n8n**을 선택했다. 이유:
- Project 1에서 이미 n8n 인프라(셀프호스팅, Google Sheets/Slack 연동)를 구축해둔 상태라 재사용 가능
- Gmail Trigger(폴링)로 새 메일 도착을 실시간에 가깝게 감지할 수 있음
- AI 분석은 최초에 로컬 CLI 도구(Hermes)를 Execute Command 노드로 호출할 계획이었으나, **n8n이 Execute Command 노드를 보안 기본값으로 차단**해 둔 것을 확인함(호스트에서 임의 쉘 명령 실행이 가능해 외부 노출 인스턴스에는 위험). 이 기본값을 낮추는 대신 **HTTP Request 노드로 Claude API(Anthropic)를 직접 호출**하는 방식으로 설계를 바꿨다 — 더 안전하고 노드 구성도 단순해짐
- REST API가 공개돼 있어 워크플로우를 스크립트로 만들고 curl로 즉시 테스트할 수 있었음

## 워크플로우 흐름

```
Gmail Trigger (1분 폴링, 안읽은 메일만, 본문 포함)
  → Code 노드 (Action 1: 제목/발신자/본문을 정리해 AI에게 보낼 프롬프트 구성)
  → HTTP Request (Action 2: Claude API 호출, 모델 claude-haiku-4-5)
  → Code 노드 (AI 응답에서 category/summary/actions JSON 추출, 파싱 실패 시 폴백)
  → IF 분기 (category == '광고/스팸'?)
      True  → Set 노드로 상태 기록 + Google Sheets "메일로그" 탭에 행 추가 (Action 3)
      False → Slack #notification 채널로 요약 + 후속 조치 제안 전송 (Action 4)
```

- **Trigger**: Gmail Trigger — 1분마다 안읽은 새 메일을 확인하고, 본문까지 포함해서 가져오도록 `Simplify: false` 설정
- **Action 1**: Code 노드에서 "반드시 JSON 형식으로만 답하라"고 지시하는 프롬프트 구성 (후속 파싱 안정화 목적)
- **Action 2**: `POST https://api.anthropic.com/v1/messages` — 인증은 n8n Header Auth 크리덴션으로 분리 보관하고, `api.anthropic.com` 도메인으로만 쓰이도록 제한
- **분기 로직**: AI가 응답한 `category`(업무/개인/광고·스팸/기타) 값이 `광고/스팸`인지 확인
- **Action 3**: (True일 때만) Google Sheets `메일로그` 탭에 `[receivedAt, subject, from, category, reason]` 행 추가
- **Action 4**: (False일 때만) Slack `#notification` 채널에 제목·발신자·카테고리·요약·후속 조치 제안 전송

## 구현 화면

**구성도**: `screenshots/project2-flow.png` (추가 예정)

## 실행 결과 (n8n Executions에서 확인)

정상 메일 1건 + 광고성 메일 1건을 실제로 발송해 테스트했다 (n8n 실행 #223 → #226 → #229, 아래는 최종 성공 실행 #229 기준).

| 메일 | category | 결과 |
|---|---|---|
| "[테스트] 다음 주 회의 일정 변경 안내" | `업무` | Slack #notification에 요약 + 후속 조치 제안 2건 전송됨 |
| "🎉 지금 가입하면 100만원 무료 쿠폰 즉시 지급!!!" | `광고/스팸` | Slack 생략, 메일로그 시트에 5개 컬럼 전부 기록됨 |

두 분기(True/False) 모두 실제로 최소 1회 이상 실행된 것을 확인했다.

**실패 → 원인 파악 → 수정 과정**은 이 과제에서 실제로 겪은 트러블슈팅이라 그대로 남긴다.
1. 실행 #223: Claude API 크리덴션의 키 값이 비어 있어 `x-api-key header is required` 오류로 전체 실패 → 실제 Anthropic API 키로 크리덴션 재발급 후 해결
2. 실행 #226: 스팸 분기에서 시트에 제목/발신자/카테고리가 빈 값으로 기록됨 → 원인은 Set 노드의 기본 옵션이 기존 필드를 모두 버리는 것이었음 → `includeOtherFields: true` 옵션 추가로 해결. 같은 실행에서 로그 사유(reason) 문구가 표현식이 아니라 코드 텍스트 그대로 저장되는 버그도 발견 → `{{ }}` 누락이 원인, 수정 후 해결
3. 실행 #229: 두 수정 사항 반영 후 재실행 → 정상/스팸 분기 모두 의도대로 동작 확인

- [x] n8n Executions 상세 — 실행 #253에서 두 분기(정상 메일→슬랙 전송, 스팸 메일→시트 기록)가 한 실행 안에서 각각 다른 경로로 체크마크 찍힌 것이 보이는 캡처: `screenshots/n8n-executions-both-branches.jpg`
- [x] 실제 발송된 Slack 메시지 캡처: `screenshots/slack-alert.jpg`
- [x] Google Sheets "메일로그" 탭 — 스팸 메일이 기록된 것을 보여주는 캡처: `screenshots/mail-log-sheet.jpg`

## 보너스 2 — 실패 알림 및 재시도 전략

- **재시도**: 외부 API를 부르는 노드 전부(Claude API, Sheets 기록, Slack 전송)에 `retryOnFail` 설정(2~3회, 1초 간격)으로 일시적 네트워크 오류를 자체 흡수
- **대체 경로**: AI 응답이 기대한 JSON 형식이 아니어도 워크플로우가 죽지 않도록, 파싱 실패 시 카테고리를 `기타`로, 요약은 원본 응답 텍스트로 폴백 처리
- **실패 알림**: Project 1에서 만든 Error Workflow(`Error Trigger → 텔레그램`)를 그대로 재사용하도록 연결해 둠
- 이번 과제는 의도적으로 오류를 주입하지 않고도, 실행 #223(인증 오류)과 #226(데이터 누락 버그)에서 실제 실패를 겪고 n8n 실행 로그를 근거로 원인을 특정해 수정하는 과정을 그대로 거쳤다

## 보안·과금

- Claude API 키는 n8n Header Auth 크리덴션으로 분리 저장하고 `api.anthropic.com` 도메인에만 쓰이도록 제한했다. 워크플로우 JSON/화면에는 노출되지 않는다.
- 애초 계획했던 로컬 쉘 명령 실행(Execute Command 노드) 대신 HTTP API 직접 호출로 설계를 바꿔, n8n 인스턴스 전체의 보안 기본값(임의 명령 실행 차단)을 낮추지 않고도 기능을 구현했다.
- **과금**: n8n 자가호스팅(무료) + Google Sheets 무료 + Slack 무료 + Claude API(Haiku 모델, 메일 1건당 프롬프트 토큰 소액 과금) → 개인/소규모 사용 기준으로는 월 비용이 미미한 수준
