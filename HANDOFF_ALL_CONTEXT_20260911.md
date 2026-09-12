# StockBot 전체 인수인계 문서

작성 기준: 2026-09-11
목적: 다른 에이전트가 현재 데이터 구조, 구현 방향, 실패 원인, 안전 정책, 다음 작업을 한 문서로 이어받기 위한 handoff.

---

## 1. 핵심 결론

현재 연구의 Data Gate와 Session/O​​vernight 연구는 완료되었지만, Full constrained candidate evolution은 완료되지 않았다.

최근 Evolution 실행은 다음 상태에서 정체 후 종료되었다.

```text
phase=TRAIN_VALIDATION
completed=1/603
current_symbol=000040
last_completed_symbol=000020
workers=1
health=STALE
last progress 약 46분 전
ETA 약 1,522시간, 약 63일
```

따라서 현재 Evolution 결과를 최종 결과로 사용하지 않는다. Execution Policy와 Final Decision도 실행하지 않는다.

문제는 현재 증거상 데이터 전체가 깨진 것보다, 400개 후보를 분봉 데이터에 종목별로 반복 계산하는 Evolution 구조가 지나치게 무거운 것일 가능성이 높다.

---

## 2. 사용자 데이터 구조

### 원본 입력 데이터

```text
E:\StockData\ByDate\
```

- 사용자 원본 CSV 위치
- 수정·삭제·재작성하지 않는다.
- 데이터 재수집이 필요하더라도 기존 원본을 먼저 삭제하지 않고 별도 버전으로 만든다.

### 현재 canonical DB 및 report

```text
D:\StockBotWork_20260909_safe\data\
D:\StockBotWork_20260909_safe\reports\
```

주요 파일:

```text
D:\StockBotWork_20260909_safe\data\quant_platform.sqlite3
D:\StockBotWork_20260909_safe\reports\ONE_SHOT_DATA_GATE_20260910.json
D:\StockBotWork_20260909_safe\reports\SESSION_AND_OVERNIGHT_RESEARCH_20260910.json
D:\StockBotWork_20260909_safe\reports\post_import_coverage_20260909.json
D:\StockBotWork_20260909_safe\reports\post_import_validation_v11.json
D:\StockBotWork_20260909_safe\reports\minute_csv_import_auto_daily_v11.json
```

### 확인된 Data Gate 수치

```text
status=PASS_WITH_REVIEW
failed_checks=[]
db_rows=162,636,177
symbols=2,947
dates=410
stable_symbols=603
partial_symbols=2,344
```

해석:

- 현재 stable universe는 603종목이다.
- Data Gate는 실패가 아니다.
- `PASS_WITH_REVIEW`는 검토가 필요한 통과 상태다.
- DB와 원본 CSV를 지우고 다시 받을 근거는 아직 없다.

### Session/O​​vernight 결과

```text
status=PASS_WITH_REVIEW
symbols_run=603
orders_submitted=0
```

Session 연구는 603/603 종목까지 진행되었으므로, 현재 데이터가 전체적으로 읽히지 않는 상태는 아니다.

---

## 3. 사용자가 별도로 구현한 TypeScript 시스템

사용자는 TypeScript로 별도의 연구/백테스트 시스템을 구현하고 있다.

현재 구상:

```text
일봉 + 분봉 데이터 세트
연구 종목 약 500개
랜덤 조건식 조합 생성
수익률 후보 선별
추가 검증 통과 후 paper/mock deployment 검토
차트에서 좋은 자리를 역으로 찾아 수치 조건식으로 변환
수동 지정 socket/execution universe 약 100개
```

중요한 구분:

- 연구 universe: 약 500개 또는 stable 603개
- 실행/socket universe: 수동 지정 100개
- 실행 100개에서만 잘 되는 전략을 전체 universe 일반화로 판단하지 않는다.
- 수동 100개는 선정일, 선정 기준, 유동성, 거래량, 종목군을 버전으로 기록한다.

현재 workspace에 제공된 `STOCKBOT_LATEST_20260911.zip`은 Python 기반 read-only 연구 runner와 UI/worker 패키지다. 사용자의 TypeScript 시스템과 자동으로 동일한 source of truth라고 가정하지 않는다. 다른 agent는 두 시스템의 역할과 실행 경계를 먼저 합의해야 한다.

---

## 4. 현재 제공 package

현재 workspace의 전달 파일:

```text
/home/user/stock-bot/STOCKBOT_LATEST_20260911.zip
```

아이디어 문서:

```text
/home/user/stock-bot/IDEAS_20260911/
```

package 내부 구조:

```text
STOCKBOT_LATEST_20260911/
├─ README_FIRST.md
├─ quant_platform/                       Python 연구 소스
├─ tests/                                regression/integration tests
├─ data/                                 package 생성 시점 snapshot
├─ reports/                              audit/status snapshot
├─ run/                                  bat, worker entry, UI, retry tools
├─ docs/                                 package guide/research run policy
├─ requirements.txt
├─ requirements-platform.txt
├─ env.example
└─ PACKAGE_MANIFEST_20260911.json
```

package에는 다음을 넣지 않았다.

```text
ARCHIVE
DELIVERABLES
nested package
ZIP 속 ZIP
아이디어 별도 문서
```

검증 결과:

```text
73 passed, 1 skipped
```

---

## 5. 개선된 UI/worker 기능

현재 최신 package에는 다음 기능이 들어 있다.

### 상태 JSON

worker는 다음 파일에 heartbeat와 진행 상태를 기록한다.

```text
D:\StockBotWork_20260909_safe\reports\FULL_AUTO_RESEARCH_STATUS_20260910.json
```

기록 항목:

```text
phase
status
started_at
elapsed_seconds
completed
total
percent
rate_per_minute
eta_seconds
current_symbol
last_completed_symbol
in_flight
workers
health
last_progress_at
last_progress_age_seconds
stalled_reason
error
orders_submitted
read_only
```

### 상태판

- UI polling: 약 3초
- worker heartbeat: 약 10초
- CSV/SQLite 전체 스캔을 UI polling마다 수행하지 않는다.
- 진행률, ETA, 현재/마지막 symbol, worker 수, stale 경고를 표시한다.
- 10분 이상 완료 종목이 없으면 `STALE`과 가능한 원인을 표시한다.
- 원인을 확인하지 못한 경우 임의로 확정하지 않고 `느린 종목·I/O·worker 대기`로 표시한다.

### Checkpoint

- 진행 상태는 heartbeat로 더 자주 기록한다.
- full checkpoint는 현재 기본 5종목 단위다.
- 동일한 DB, seed, 기간, 비용, candidate 설정과 동일 output path를 사용하면 checkpoint resume이 가능하다.
- 설정이 바뀌는 새 연구는 새 output/run ID를 사용한다.

### 자동 정리

자동 정리 범위는 다음으로 제한한다.

```text
*.tmp
*.partial
정상 종료 후 lock
```

자동 삭제 금지:

```text
data
canonical DB
원본 CSV
Data Gate report
Session/O​​vernight report
Evolution checkpoint/report
Policy report
Final report
```

현재 worker에는 hard process kill timeout보다 stale 감지와 실패 상태 기록이 우선 구현되어 있다. 종목별 강제 timeout을 추가할 때는 결과 왜곡을 막기 위해 timeout 종목을 조용히 건너뛰지 말고 `TIMEOUT`으로 기록한 뒤 해당 run을 실패 처리한다.

---

## 6. 현재 실행 실패 원인

최근 Full Auto log:

```text
[1/5] Read-only data gate       PASS_WITH_REVIEW
[2/5] Session/O​​vernight       PASS_WITH_REVIEW, 603/603
[3/5] Candidate evolution       exit_code=-1
[4/5] Execution Policy          BLOCKED
[5/5] Final Decision             실행되지 않음
```

Evolution 결과:

```text
APPROVED_EVOLUTION_RESEARCH_20260910.json 완성되지 않음
checkpoint 없음 또는 유효한 resume checkpoint 없음
```

따라서 Policy 단계의 다음 메시지는 정상적인 차단이다.

```text
[BLOCKED] Run RUN_APPROVED_EVOLUTION_20260910.bat first.
```

`exit_code=-1`은 정상 완료가 아니다. read-only worker가 종료되었거나 강제 종료되었거나 예외가 표면화되지 않은 것이다.

최근 개선 package에서 확인된 실행:

```text
started_at=2026-09-11T15:29:35+09:00
completed=0/603
current_symbol=000020
health=STALE
```

이후 상태판에는:

```text
completed=1/603
current_symbol=000040
last_completed_symbol=000020
rate_per_minute=0.0066
eta 약 63일
health=STALE
```

이 표시되었다. 이는 전체 연구를 계속할 수 있는 속도가 아니므로 해당 run은 중단 대상이다.

---

## 7. 지금 해야 할 일

### 즉시 조치

1. 현재 Evolution worker가 있으면 종료한다.
2. 두 번째 Full Auto를 동시에 실행하지 않는다.
3. 기존 `data`와 Data Gate/Session report를 삭제하지 않는다.
4. retry log와 stale status를 보존한다.
5. 다음 worker 개선 전에는 603종목 전체를 재실행하지 않는다.

프로세스 확인:

```powershell
Get-Process StockBotEvolutionWorker -ErrorAction SilentlyContinue | Select-Object Id,CPU,StartTime,Responding
```

Python fallback 확인:

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -match 'evolution_runner|APPROVED_EVOLUTION_RESEARCH_20260911' } | Select-Object ProcessId,Name,CommandLine
```

### 개선 후 진단 순서

새 package에서 먼저:

```text
run\BUILD_EVOLUTION_VALIDATION_UI_20260910.bat
run\RUN_RESEARCH_STATUS_20260910.bat
run\RUN_DIAGNOSTIC_FIRST_SYMBOL_20260911.bat
```

진단은:

```text
symbol 1개
candidate 3개
workers=1
read-only
```

결과:

```text
EVOLUTION_DIAGNOSTIC_FIRST_SYMBOL_20260911.json
evolution_diagnostic_20260911.log
```

진단이 빠르게 통과하면 데이터보다 계산량 문제가 크다.

그 다음 연구는 바로 603/400으로 가지 않는다.

```text
1~3 후보 × 1~2 종목 진단
10~30 후보 × 20~50 종목 선별
상위 후보 × 500~603 종목
상위 3 후보 심층 validation
봉인 OOS
```

현재 retry wrapper:

```text
run\RUN_EVOLUTION_RETRY_20260911.bat
```

첫 실행은 workers=1이다. heartbeat와 처리 속도가 확인된 뒤에만 workers=2 또는 4로 올린다.

retry output:

```text
APPROVED_EVOLUTION_RESEARCH_20260911.json
evolution_retry_console_20260911.log
```

후속 단계:

```text
run\RUN_EXECUTION_POLICY_RETRY_20260911.bat
run\RUN_FINAL_RESEARCH_RETRY_20260911.bat
```

Evolution `exit_code=0`과 report 생성 전에는 후속 단계를 실행하지 않는다.

---

## 8. 데이터 재수집 여부

현재 데이터 전체를 삭제하고 다시 받는 것은 1순위가 아니다.

근거:

```text
Data Gate failed_checks=[]
Session 603/603 완료
DB 162,636,177 rows
stable 603 symbols
```

우선 계산 구조를 줄인다.

필요할 때만 다음처럼 새 DB를 별도로 만든다.

```text
기존:
D:\StockBotWork_20260909_safe\data\quant_platform.sqlite3

새 검증용:
D:\StockBotWork_20260909_safe\data\quant_platform_v2.sqlite3
```

기존 DB와 새 DB를 Data Gate, symbol count, date coverage, Session 결과로 비교한 뒤 교체 여부를 결정한다. 원본 `E:\StockData\ByDate`는 삭제하지 않는다.

---

## 9. 최적 조건 탐색 원칙

영상의 Markov 국면 모델은 정답이 아니라 후보 아이디어로만 사용한다.

후보 생성기:

```text
랜덤 조건식 조합
기술적 지표 조합
일봉 setup + 분봉 trigger
Opening Edge
차트에서 발견한 패턴의 수치화
Markov regime filter
HMM은 후순위 후보
```

최적 후보는 raw 수익률 1등이 아니다.

선택 기준:

```text
비용 후 평균/중앙 수익률
MDD
Profit Factor
거래 수
손익비
조건 복잡도
기간 안정성
종목군 안정성
시장 국면 안정성
slippage 민감도
```

차트에서 조건을 역으로 찾은 경우:

```text
차트 아이디어
→ 숫자 조건식/DSL 변환
→ 500/603종목 적용
→ 종목군별 분포
→ 기간별 분포
→ 국면별 분포
→ 비용 후 성과
→ Validation
→ 봉인 OOS
```

반드시 표시할 분포:

```text
조건 적용 종목 수 / 전체
종목군 coverage
기간 coverage
최대 종목군 수익 기여도
최대 기간 수익 기여도
한 종목의 전체 수익 기여도
```

특정 한 종목군이나 기간에 수익이 집중되면 수익률이 높아도 REVIEW 또는 BLOCK이다.

---

## 10. Markov/HMM 활용 원칙

사용 가능한 아이디어:

```text
Bull/Bear/Sideways 국면
rolling transition matrix
국면 지속성/stickiness
국면별 전략 성과
P(up) - P(down) confidence
short-horizon matrix power
walk-forward 재계산
```

고정된 `20일 수익률 ±5%` 기준은 하나의 가설이다. 다음을 학습 구간에서 비교할 수 있다.

```text
lookback 10/20/40일
threshold ±3/±5/±7%
state 2/3/4개
```

HMM은 나중에 비교 모델로 넣고, 처음부터 기본 전략으로 사용하지 않는다. 국면 모델은 단독 전략보다 기본 조건의 필터 또는 position sizing 보정으로 먼저 시험한다.

OOS 결과를 보고 국면 파라미터를 다시 튜닝하지 않는다.

---

## 11. 배포 기준

모투/paper로 보내기 전 단계:

```text
후보 발견
→ Validation
→ 종목군 일반화
→ 기간 일반화
→ 비용/슬리피지 스트레스
→ 봉인 OOS
→ Shadow
→ Paper
→ 수동 승인
```

상위 3개는 raw return 상위 3개가 아니라 서로 다른 역할과 낮은 상관을 고려한다.

예:

```text
추세형
Opening Edge형
평균회귀/방어형
```

자동 live 주문과 자동 전략 승격은 하지 않는다.

항상:

```text
orders_submitted=0
read_only=true
```

을 유지한다.

---

## 12. 다른 agent가 지켜야 할 금지사항

- 현재 Full Auto가 있으면 두 번째 Full Auto를 실행하지 않는다.
- `FINAL_SUMMARY_NOT_FOUND`는 실행 중이면 실패로 단정하지 않는다.
- 단, `exit_code=-1`, worker 없음, evolution report 없음이면 정상 완료가 아니다.
- CPU만 보고 진행을 판단하지 않는다. heartbeat, checkpoint, completed count를 함께 본다.
- Data Gate와 Session 결과를 Evolution 실패 때문에 삭제하지 않는다.
- 원본 CSV와 canonical DB를 수정하지 않는다.
- 이전 OOS 결과로 조건을 재튜닝하지 않는다.
- 결과가 없는 상태에서 Policy/Final을 실행하지 않는다.
- AI나 Z Code가 만든 조건을 자동 배포하지 않는다.
- 실적·뉴스·수급 데이터가 실제로 연결되지 않으면 추정하지 않고 확인 불가로 표시한다.

---

## 13. 연구의 장기 반복 구조

연구는 한 번으로 끝내지 않는다.

```text
RUN-001: 랜덤 후보 탐색
RUN-002: 차트에서 찾은 패턴 수치화
RUN-003: Opening Edge 비교
RUN-004: 국면 필터 비교
RUN-005: 비용/유동성 개선
```

각 run은 새 ID와 새 report를 사용한다. 이전 결과는 baseline과 재현성 비교에 사용하지만, 봉인 OOS를 다시 최적화하는 데 사용하지 않는다.

최종 목표는 “과거 수익률 1등”이 아니라:

```text
여러 종목에서 재현
여러 기간에서 유지
여러 국면에서 설명 가능
비용 후에도 생존
특정 종목·기간 의존도 낮음
실제 체결 가능한 후보
```

를 만족하는 현재까지의 최선 후보를 찾는 것이다.
