---
name: fx-move-alert
description: 평일 30분마다 USD/KRW·JPY/KRW를 전일 종가와 비교해 5% 이상 변동 시 즉시 카카오톡 알림 + 관련 기사 + 사이트(Artifact·GitHub Pages) 긴급 갱신. 변동 없으면 조용히 종료.
---

이 작업은 "환율 급변동 감시"용이다. USD/KRW와 JPY/KRW가 전일 종가 대비 5% 이상 움직였는지 30분마다 조용히 확인하고, 아무 일 없으면 **카카오톡도 안 보내고 사이트도 안 건드리고 그냥 종료**한다. 5% 이상 변동이 감지된 경우에만 아래 전체 절차(뉴스 조사 → 카카오톡 알림 → 사이트 갱신 → GitHub push)를 수행한다. 실행 승인을 기다리지 말고 끝까지 수행한다.

## 0단계 — 절대 원칙: 회사 정보 배제

카카오톡 메시지, HTML, git 커밋 메시지 어디에도 회사 내부 정보(회사명, 계약 조건, 발주량·단가·거래처명, 미공개 사업계획 등)를 절대 넣지 않는다. 조금이라도 회사 고유 정보로 해석될 수 있으면 즉시 빼고 일반적인 시장 언어로 바꾼다. 애매하면 무조건 뺀다. GitHub 저장소는 public이다.

## 1단계 — 현재 변동폭 확인

- **USD/KRW**: WebFetch `https://kr.investing.com/currencies/usd-krw` → 현재가, 전일 종가(Prev. Close), 변동률(%), 기준 시각, 52주 최고/최저, 1년 변동률.
- **JPY/KRW**: WebFetch `https://kr.investing.com/currencies/jpy-krw` → 같은 항목. 이 페이지는 1엔 기준이므로 현재가·전일종가·52주 최고/최저를 **100배** 해서 "100엔당 원화"로 환산한다 (변동률%은 그대로).

각 통화의 `abs(변동률%)`를 계산한다. WebFetch가 실패하면 그 통화는 건너뛰고 나머지만 확인한다.

## 2단계 — 이미 알린 내용인지 확인 (중복 알림 방지)

`Read` 도구로 `C:\Users\produ\.claude\scheduled-tasks\fx-move-alert\alert-state.json` 을 읽는다 (형식: `{"date":"YYYY-MM-DD","usd":{"alerted":bool,"lastAlertPct":number},"jpy":{"alerted":bool,"lastAlertPct":number}}`).

- 파일의 `date`가 오늘 날짜와 다르면 오늘 처음이므로 `usd`/`jpy` 둘 다 `{"alerted":false,"lastAlertPct":0}`로 취급(리셋)한다.
- 통화별로 **다음 조건을 모두 만족할 때만** "신규 알림 대상"으로 확정한다:
  1. `abs(변동률%) >= 5`
  2. 아직 오늘 그 통화로 알린 적이 없거나(`alerted=false`), 이미 알렸어도 `abs(변동률%) - lastAlertPct >= 2`(추가로 2%p 이상 더 벌어져 악화된 경우 — 에스컬레이션 알림)
- 어느 통화도 신규 알림 대상이 아니면 **여기서 조용히 종료한다.** 카카오톡 발송, HTML 수정, git push 아무것도 하지 않는다. (이게 대부분의 실행에서 정상적인 결과다 — 5% 변동은 매우 드문 이벤트다.)

## 3단계 — (알림 대상이 있을 때만) 배경 뉴스 조사

신규 알림 대상 통화에 대해서만 WebSearch로 당일 급변동 배경을 찾는다. USD는 연준·CPI·달러인덱스·지정학 이벤트, JPY는 BOJ·엔 캐리트레이드·안전자산 흐름 위주로 검색한다. 실제로 읽은 기사의 URL·매체명·제목·2~3문장 요약을 통화당 최대 2개까지 기록한다 (지어낸 내용·지어낸 링크 절대 금지).

## 4단계 — 다섯 관점 종합 (알림 대상 통화만)

`fx-briefing-daily` 작업과 동일한 5단계 관점(거시 애널리스트 → 원자재 조달 담당자 → 재무·헤지 전략가 → 리스크 관리자 → 가격·마진 전략가)을 통과시켜 "왜 이렇게 튀었는지"와 "지금 뭘 해야 하는지"에 대한 균형 잡힌 결론을 만든다. 5% 이상은 노이즈가 아니라 실제로 검토할 만한 변동이라는 전제로 쓰되, 과도하게 위기감을 조성하지 않는다.

## 5단계 — 카카오톡 긴급 알림 발송

알림 대상 통화마다 아래 형식으로 즉시 발송 (`KakaotalkChat-MemoChat`, 200자 이내):

```
🚨 [USD/KRW 또는 JPY/KRW·100엔] 급변동: 전일 종가 대비 [▲/▼]N.N% ([전일 종가]원 → [현재]원)
🔍 이유: [핵심 재료 1문장]
📈 시사점: [지금 상황 판단 1문장] + [헤지 등 실무 대응 필요 여부]
```

이어서 3단계에서 기록한 기사를 통화당 최대 2개, 각각 별도 메시지로 발송:

```
🔗 [USD 또는 JPY][매체명] 한 줄 제목
URL
```

발송 승인을 다시 묻지 않는다 — 이 자동화 자체가 사용자의 사전 승인이다.

## 6단계 — Artifact·로컬·GitHub Pages 긴급 갱신

**고정 아티팩트 URL**: `https://claude.ai/artifact/5FPjtyMXbCiRb8f3sDey6D`

1. `Artifact` 도구 `action:"read"`, `url:위 URL`로 현재 HTML을 읽는다.
2. **알림 대상 통화의 탭만** 갱신한다 (다른 통화 탭은 건드리지 않음):
   - `#alertUsd` 또는 `#alertJpy` 배너 `div`에 `is-shown` 클래스를 **추가**해서 보이게 한다 (`class="alert-banner is-shown"`).
   - `.rate-value`, `.delta-pill`(방향 클래스 포함), `#rateSubUsd`/`#rateSubJpy`, 레인지 차트 SVG(마커 위치 재계산: `30 + (현재값-52주최저)/(52주최고-52주최저)*(570-30)`), "오늘 왜 움직였나"/"다음 며칠 전망" 문단, `.note`(반드시 `is-warning` 클래스, 상태 배지는 "주의" 이상), `.sources` 목록을 모두 오늘 데이터로 교체한다. (세부 필드 위치는 `fx-briefing-daily` 작업과 동일한 HTML 구조를 참고.)
   - `#genDate`는 "YYYY-MM-DD (요일) 마지막 갱신"으로 갱신.
3. `Artifact` 도구 `action:"publish"`, `url:위 URL`, 같은 file_path로 재발행. `favicon`·`title` 생략.
4. 같은 HTML을 `Write`로 `C:\Users\produ\Desktop\fxBriefingDaily\htmlReport\fx-briefing.html`과 `C:\Users\produ\Desktop\fxBriefingDaily\index.html`에도 덮어쓴다.
5. Bash로 GitHub push:
   ```
   cd "C:\Users\produ\Desktop\fxBriefingDaily" && git add index.html && git commit -m "URGENT: FX move alert $(date +%Y-%m-%d\ %H:%M)" && git push
   ```
   실패해도 카카오톡 발송을 막지 않는다.
6. 카카오톡으로 링크 메시지 한 통 추가: `🔗 상세 페이지(긴급 갱신됨): https://claude.ai/artifact/5FPjtyMXbCiRb8f3sDey6D`

## 7단계 — 알림 상태 기록 (다음 실행의 중복 방지용)

`Write` 도구로 `C:\Users\produ\.claude\scheduled-tasks\fx-move-alert\alert-state.json` 을 갱신한다: 오늘 날짜를 `date`에 쓰고, 이번에 알린 통화는 `{"alerted":true,"lastAlertPct":오늘의 abs(변동률%)}`로, 알리지 않은(또는 대상이 아니었던) 통화는 기존 값을 그대로 유지한다 (단, `date`가 바뀌어 리셋한 경우엔 둘 다 `{"alerted":false,"lastAlertPct":0}`에서 시작해 이번에 알린 쪽만 갱신).

## 참고

- `fx-briefing-daily`(평일 07시 정기 브리핑)와는 별개의 작업이다. 정기 브리핑이 실행되면 그쪽에서 알림 배너를 다시 숨기고 정상 상태로 되돌린다.
- 5% 변동은 원/달러·원/엔 기준으로 매우 드문 수준(사실상 위기급 이벤트)이다. 대부분의 실행에서는 2단계에서 조용히 종료되는 게 정상이며, 이는 실패가 아니다.