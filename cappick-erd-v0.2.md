# CapPick ERD·데이터 설계 v0.2

- 작성일: 2026-09-15
- 상태: 논리 ERD 초안. DB 제품·물리 타입·인덱스와 마이그레이션은 미확정.
- 기준: [시스템 설계](cappick-system-design-v0.2.md), [기능명세서](cappick-plan-functional-spec-v0.11.md)
- 로컬 DB와 서버 DB는 서로 다른 저장소다. 두 저장소 사이에 DB 외래키를 만들지 않는다.

## 1. 로컬 관계

| 부모 | 관계 | 자식 | 의미 |
|---|---|---|---|
| Capture | 1 : 0..1 | CaptureAnalysis | 생성된 AI 분석 결과 |
| Capture | 1 : 0..1 | SearchDocument | 현재 검색 표현·색인 버전 |
| Capture | 1 : N | FolderCapture | 캡처의 폴더 소속·수동 제거 기록 |
| Folder | 1 : N | FolderCapture | 폴더별 캡처 관계 |
| Capture | 1 : N | LocalTask | 캡처의 단계별 처리·복구 기록 |
| Folder | 1 : N | LocalTask | 폴더별 판정 작업; 폴더 없는 작업은 참조 없음 |
| Folder | 1 : N | ReclassificationRun | 기존 캡처 적용·재분류 실행 |
| ReclassificationRun | 1 : N | ReclassificationItem | 확인할 추가·제거 예정 항목 |
| Capture | 1 : N | ReclassificationItem | 재분류의 대상 캡처 |

Capture와 Folder는 FolderCapture를 통해 다대다로 연결된다. User 테이블은 없다. 전체·미분류·즐겨찾기는 조회 조건이며 Folder 행을 만들지 않는다.

## 2. 로컬 엔터티

모든 id는 앱이 생성하는 불변 식별자 제안이다. created_at·updated_at은 UTC 시각으로 저장한다. 아래 PK는 기본키, FK는 같은 로컬 DB의 외래키, UQ는 중복 금지다.

| 엔터티 | 주요 필드 | 제약·용도 |
|---|---|---|
| Capture | id PK, photo_asset_id UQ, captured_at, width, height, availability, excluded_at, favorite, content_revision, search_state, classification_state | 사진 자산 참조는 해당 기기 보관함 범위. 제외 시 최소 자산 식별 기록 유지 |
| CaptureAnalysis | capture_id PK/FK, content_revision, analysis_revision, ocr_text, observed_evidence, model_version | 검색·직접 근거 기반 분류에 필요한 관찰 내용·시각적 특징. 내부 정보로 사용 |
| SearchDocument | capture_id PK/FK, source_analysis_revision, search_text, embedding, embedding_model_version, state | 실제 색인은 파생 데이터. 서로 다른 모델의 벡터는 비교하지 않음 |
| Folder | id PK, display_name, criteria_text, criteria_version, criteria_state, membership_revision | 이름 변경은 criteria_version을 올리지 않음. 모호한 기준은 needs_clarification |
| FolderCapture | folder_id PK/FK, capture_id PK/FK, state, source, evidence, criteria_version, revision | 복합 PK로 중복 방지. state=present/blocked; source=manual/auto |
| LocalTask | id PK, capture_id FK nullable, folder_id FK nullable, kind, input_revision, criteria_version nullable, server_job_id nullable, idempotency_key UQ, state, attempts, error_code, next_retry_at | 서버 제출 전부터 기록. 생성한 논리 작업을 재개할 때 같은 키 유지 |
| ReclassificationRun | id PK, folder_id FK, criteria_version, membership_revision, mode, state | mode=initial_apply/reclassify. 미리보기 기준과 소속 버전 고정 |
| ReclassificationItem | run_id PK/FK, capture_id PK/FK, action, evidence, expected_membership_revision nullable, state | action=add/remove. 확인 후 항목별 멱등 적용 |
| AppState | key PK, value | 분석 일시정지·안내 버전·삭제 대기 작업 등 앱 설정. 비밀 토큰은 이 테이블에 평문 저장하지 않음 |

search_text는 OCR·관찰 내용에서 만든 검색용 표현이며 제목·설명·태그를 각각 필수로 저장하지 않는다. 짧은 내부 요약·핵심어를 사용할지는 품질 검증으로 결정한다. 사용자 편집 필드와 CaptureOverride 테이블은 없다. 수동 폴더 소속·제거 기록은 FolderCapture로 보존한다.

## 3. 상태와 소속 규칙

- availability: accessible / unavailable. unavailable만으로 원본이 영구 삭제되었다고 판단하지 않는다.
- search_state: pending / processing / ready / failed. 분석 및 유효 검색 문서가 준비된 경우 ready.
- classification_state: pending / processing / complete / failed / blocked. 판단 완료 후 실제 소속이 없을 때만 미분류에 포함한다.
- LocalTask.state: pending / submitted / processing / succeeded / failed / waiting_limit / discarded. 앱 일시정지는 신규 제출 제어이며 진행 작업의 성공을 취소하지 않는다.
- 서버 상태와 로컬 반영 완료는 다르다. 서버 succeeded여도 로컬 저장에 실패했다면 같은 결과를 다시 가져와 반영한다.
- 분류 상태는 해당 캡처에 적용하기로 한 작업들로 계산한다. 기준 변경만으로 과거 캡처 전체를 pending으로 돌리지 않는다. 실제로 요청한 기존 적용·재분류만 대상에 추가한다.
- 판정 결과가 불명확하거나 기준이 모호하면 자동 추가하지 않는다. 미완료 상태를 확실한 미분류로 표시하지 않는다.

| 행동 | FolderCapture 변경 |
|---|---|
| 직접 추가 | present + manual; 기존 blocked 해제 |
| 직접 제거 | blocked + manual; 소속은 없지만 자동 재추가 방지 기록 유지 |
| AI 일치 | blocked·manual present는 보존. 나머지를 present + auto로 반영 |
| 확인된 재분류의 불일치 | auto present만 제거; manual 관계는 보존 |
| 폴더 삭제 | 해당 폴더의 소속·차단·관련 재분류 기록 삭제; Capture 유지 |

미분류 조회: 접근 가능 AND 제외 아님 AND 검색 준비 완료 AND 분류 complete AND present 관계 수=0. 즐겨찾기는 폴더 소속 수에 포함하지 않는다.

재분류 적용 전 run의 기준 버전과 관련 수정 여부를 검사한다. 변경되면 새 미리보기를 요구한다. 적용 중에는 항목별로 최신 수동 관계·접근 상태를 재검사하며 사용자 변경을 덮어쓰지 않는다.

## 4. 서버 관계와 엔터티

| 부모 | 관계 | 자식 |
|---|---|---|
| Installation | 1 : N | Job |
| Installation | 1 : N | UsageEntry |
| Installation | 1 : N | DeletionRequest |
| Job | 1 : 0..N | UsageEntry |

Installation은 회원이 아니라 검증된 앱 설치의 서버 요청 주체다. 재설치 뒤 동일인 식별을 보장하지 않는다.

| 엔터티 | 주요 필드 | 제약·용도 |
|---|---|---|
| Installation | id PK, credential_reference, status, created_at, last_seen_at | 서버 검증된 자격과 연결. 비밀 원문 로깅 금지 |
| Job | id PK, installation_id FK, idempotency_key, request_digest, kind, status, input_reference, input_expires_at, result_reference, result_expires_at, attempts, lease_until, error_code, created_at, completed_at | UQ(installation_id,idempotency_key). 입력·결과는 임시 저장. 이미지 바이트를 큐 메시지에 넣지 않는 구현도 가능 |
| UsageEntry | id PK, installation_id FK, job_id FK nullable, operation_key UQ, category, units, state, created_at | 예약·확정·해제는 원자적 전이. 작업 중복 실행으로 정산 중복 금지. 자연어 검색 등 작업 외 요청도 포함 |
| DeletionRequest | id PK, installation_id FK, idempotency_key, scope, status, requested_at, completed_at, error_code | 콘텐츠 삭제 재시도 추적. 사용량 보호 기록과 별도 정책 |

서버 Job의 client_reference는 불투명한 앱 작업 식별자일 뿐 로컬 FK가 아니다. 사진 보관함 자산 ID는 서버에 전송할 필요가 없다. 콘텐츠 삭제 후 멱등 처리용 최소 기록 보존 여부·기간은 정책으로 정한다.

## 5. 무결성과 삭제

- 검색 결과의 범위 제한은 표시 직전에도 확인한다. 삭제·권한 변경 직전 생성한 결과가 노출되지 않게 한다.
- CaptureAnalysis·SearchDocument는 Capture에 종속한다. 영구 로컬 정리 시 종속 삭제하되 사용자가 제외한 캡처의 최소 식별 기록은 재등록 방지 목적에 맞게 유지한다.
- FolderCapture FK는 로컬에만 적용한다. 폴더 삭제 후 도착한 서버 판정 결과로 폴더를 재생성하지 않는다.
- 서버는 모든 Job·Usage·Deletion 조회 시 요청 설치와 소유 설치가 같은지 검증한다.
- 작업 접수와 사용량 예약, 결과 저장과 상태 완료를 각각 원자적으로 처리한다. 외부 AI 호출은 DB 트랜잭션 밖에서 실행한다.
- 인덱스 후보: captured_at, search_state, FolderCapture(folder_id,state), Job(status,lease_until), Job(installation_id,created_at). 실제 실행 계획과 기기 성능으로 확정한다.

## 6. 확정 전 항목

물리 DB·벡터 저장 방식, OCR 및 evidence 구조, 필드 길이·업로드 제한, 결과 보존 기간, 사용량 단위와 한도, 설치 검증과 자격 갱신 프로토콜은 미정이다. ERD에 이 항목이 있다고 구현 또는 출시 정책이 확정된 것은 아니다.

## 이번 개정: 검색에 필요한 내부 데이터만 유지

- 캡처 제목·설명·태그의 독립 생성·표시·편집 요구사항을 제외했다. 검색용 짧은 요약·핵심어의 내부 사용 여부는 품질 검증에서 결정한다.
- OCR·관찰 내용·시각적 특징·검색 표현은 검색과 자동 분류를 위해 유지한다. 검색어 기록 저장 기능은 추가하지 않는다.
- 원본 연결·날짜·폴더 소속·처리 상태 및 수동 분류 교정은 유지한다.
