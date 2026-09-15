# CapPick API 명세 v0.2

- 작성일: 2026-09-15
- 상태: 검토용 계약 초안. 배포 주소 없음. 그대로 구현 착수할 수 있는 최종 OpenAPI 문서는 아니다.
- 기준: [시스템 설계](cappick-system-design-v0.2.md), [ERD](cappick-erd-v0.2.md)

## 1. API 경계

앱의 로컬 DB가 보관함의 기준이다. 폴더 생성·이름 수정·소속 변경·즐겨찾기·원본 탐색은 로컬 기능이며 서버 CRUD API를 만들지 않는다. 서버는 이미지 분석, 분류 판정, 검색용 표현 생성·검색어 해석, 사용량 조회와 임시 데이터 삭제를 제공한다.

로그인·회원가입 API는 없다. 아래 설치 등록은 사용자가 가입하는 화면이 아니라 앱과 서버 사이 요청 보호 절차다. 구체적 검증 제공자와 프로토콜은 기술 검증 후 확정한다.

## 2. 공통 계약

- 경로 접두사 /v1. JSON 필드명 snake_case, 식별자는 불투명 문자열, 시각은 UTC ISO 8601.
- 설치 등록 외 요청은 서버 검증 가능한 설치 자격을 Authorization 헤더로 전달하는 제안. 형식·만료·갱신은 별도 확정한다. 인증 실패는 401, 권한 없는 타 설치 자원은 404로 응답하는 초안.
- 모든 비용 발생 POST 및 삭제 요청에 Idempotency-Key 사용. 같은 설치·키·동일 본문은 기존 처리 결과를 반환한다. 다른 본문 재사용은 409 IDEMPOTENCY_CONFLICT.
- 이미지 바이트 등 입력 전체의 digest를 비교한다. 멱등 키 보존 기간은 작업·결과 보존 기간과 함께 확정한다. 키 만료 뒤 무조건 중복 방지된다고 보장하지 않는다.
- 오류 형식: {"error":{"code":"QUOTA_EXCEEDED","message":"분석 한도에 도달했습니다.","retryable":false},"request_id":"req_example"}.
- 400 입력 오류, 401 설치 자격 오류, 404 없음·소유권 불일치, 409 충돌, 410 결과 만료, 413 크기 초과, 415 미지원 이미지, 422 처리 불가능한 입력, 429 요청·사용량 제한, 503 일시적 장애.
- 429 RATE_LIMITED와 QUOTA_EXCEEDED를 구분한다. 알려진 대기 시간이 있을 때만 Retry-After를 제공한다. 가격·한도 해제 시점을 임의로 응답하지 않는다.
- 서버 오류 메시지와 로그에 이미지·OCR·비밀키를 포함하지 않는다. 요청 크기·타임아웃·재시도 상한은 구현 전에 수치로 확정한다.

## 3. 엔드포인트 목록

| 방식·경로 | 목적 | 주요 입력 | 정상 응답 |
|---|---|---|---|
| POST /v1/installations/challenges | 설치 등록 검증 준비 | 앱 버전·플랫폼 등 검증에 필요한 최소 정보 | 200 challenge_id, challenge, expires_at |
| POST /v1/installations | 검증된 설치 등록 | challenge_id, verification_payload | 201 installation_id, credential, expires_at |
| POST /v1/installations/credentials/renew | 설치 자격 갱신 | 기존 자격과 갱신 증명 | 200 credential, expires_at |
| POST /v1/jobs | 비동기 AI 작업 접수 | kind, client_reference, 작업별 입력 | 202 job_id, status, poll_after_seconds |
| GET /v1/jobs/{job_id} | 상태·완료 결과 조회 | 소유 작업 ID | 200 상태, 완료 시 result, result_expires_at |
| DELETE /v1/jobs/{job_id}/content | 결과 수신 후 또는 제외 후 임시 콘텐츠 삭제 | 소유 작업 ID | 202 deletion_id, status |
| POST /v1/search/prepare | 자연어 검색 해석·벡터 생성 | query, now, timezone, embedding_model_version | 200 검색 표현·날짜 조건 |
| GET /v1/usage | 현재 설치의 사용량 확인 | 없음 | 200 범주별 사용량·한도·허용 여부 |
| POST /v1/data-deletions | 설치의 서버 콘텐츠 삭제 | scope=installation_content | 202 deletion_id, status |
| GET /v1/data-deletions/{deletion_id} | 삭제 완료·실패 확인 | 소유 삭제 요청 ID | 200 status, completed_at, error |

설치 challenge는 짧게 유효하고 한 번만 사용한다. 이 경로도 호출량 제한 대상이다. verification_payload·갱신 증명의 실제 필드는 검증 기술 선정 후 채운다. 앱이 임의로 보낸 문자열을 검증으로 인정하지 않는다.

## 4. POST /v1/jobs

작업 종류와 각 입력을 구분한다. content_analysis는 multipart/form-data의 metadata JSON과 image 파일, 다른 종류는 application/json을 사용하는 초안이다. 파일별 MIME·크기·해상도 제한 및 긴 캡처 분할 계약은 품질 검증 후 확정한다.

| kind | 입력 | 완료 result |
|---|---|---|
| content_analysis | client_reference, content_revision, image, 선택 ocr_text | analysis_revision, ocr_text, observed_evidence, search_text, embedding, embedding_model_version |
| folder_criteria | 사용자가 입력한 name, 선택 description | criteria_text, criteria_state, clarification_message nullable |
| classification | 캡처 분석 스냅샷·analysis_revision, 실제 폴더 ID·criteria_text·criteria_version 목록 | 폴더별 decision, evidence, criteria_version |
| search_document | OCR·관찰 내용, source_analysis_revision, embedding_model_version | search_text, embedding, embedding_model_version, source_analysis_revision |

folder_criteria는 사용자가 만들려는 한 폴더의 기준을 구체화한다. 새로운 폴더 목록을 추천하거나 생성하지 않는다. 모호하면 needs_clarification으로 반환한다.

classification은 서버가 폴더 소속을 저장하는 API가 아니다. 앱이 보낸 스냅샷에 대한 판정만 반환한다. 폴더당 decision은 match/no_match/uncertain. match에는 이미지·OCR의 관찰 근거를 포함한다. 출력 ID는 요청한 실제 폴더 ID의 부분집합만 허용하며 누락된 판정은 성공한 no_match로 간주하지 않는다.

분류 입력은 기기 보관 데이터를 클라이언트가 제출한 스냅샷이다. 서버는 이를 임의 코드나 시스템 지시로 실행하지 않는다. 판정 근거가 부족하면 uncertain을 반환한다. 분석 정보만으로 시각적 판단이 부족한 경우 재이미지 분석 계약을 추가할지는 품질 검증에서 결정한다.

작업 입력 예시:

```json
{
  "kind": "classification",
  "client_reference": "local_task_example",
  "analysis_revision": 1,
  "analysis": {
    "ocr_text": "자바 면접 예상 질문",
    "observed_evidence": ["이미지에 면접 예상 질문이라는 문구가 있음"]
  },
  "folders": [
    {"id": "folder_example", "criteria_text": "면접 질문과 면접 안내", "criteria_version": 1}
  ]
}
```

client_reference는 사진 보관함 자산 식별자가 아니다. 모델·프롬프트 버전과 출력 스키마 버전을 서버 작업에 기록한다. 모델 출력 형식과 근거 필드를 검증한 뒤 성공으로 저장한다.

## 5. GET /v1/jobs/{job_id}와 재시도

상태: queued → running → succeeded 또는 failed. 일시적 실패의 제한된 자동 재시도는 queued로 돌아갈 수 있다. 삭제 요청은 cancel_requested → cancelled, 이미 끝난 콘텐츠 삭제는 content_deleted로 표시하는 제안이다.

```json
{
  "job_id": "job_example",
  "status": "succeeded",
  "client_reference": "local_task_example",
  "result": {
    "decisions": [
      {"folder_id": "folder_example", "criteria_version": 1, "decision": "match", "evidence": "면접 예상 질문 문구가 확인됨"}
    ]
  },
  "result_expires_at": "2026-09-15T00:00:00Z"
}
```

예시 만료 시각은 보존 정책을 뜻하지 않는다. 결과 보존 기간은 미정이다. queued/running에서는 result를 반환하지 않고 poll_after_seconds를 안내한다. 앱은 화면 활성 여부에 맞춰 조회 빈도를 낮춘다.

앱은 결과를 로컬 트랜잭션으로 반영하고 저장 성공 후 콘텐츠 삭제 요청을 보낸다. 결과 수신만으로 로컬 성공 처리하지 않는다. 사진 제외·권한 변경·기준 버전 불일치·사용자 교정을 재검사한다.

같은 멱등 키는 실패 결과도 재조회한다. 최종 failed 작업을 사용자가 다시 실행할 때는 상태를 확인한 뒤 새 논리 시도 키와 retry_of_job_id로 요청한다. 서버는 소유권과 최종 실패 여부를 확인하고 중복 활성 작업을 막는다. 사용량 정책은 실패 원인별로 확정한다.

보존 만료 시 410 RESULT_EXPIRED. 앱은 실패를 숨기거나 무한 재제출하지 않는다. 재분석 필요 상태로 표시하고 비용·한도를 확인한 뒤 재시도한다.

## 6. POST /v1/search/prepare

```json
{
  "query": "지난달 저장한 면접 관련 캡처",
  "now": "2026-09-14T01:00:00Z",
  "timezone": "Asia/Seoul",
  "embedding_model_version": "configured_model_version"
}
```

응답 필드: normalized_query, lexical_terms, embedding, embedding_model_version, captured_from nullable, captured_before nullable, date_interpretation nullable. 날짜는 시작 포함·끝 미포함 구간이다. 모델 버전 불일치 시 409 SEARCH_MODEL_MISMATCH로 반환하며 다른 차원의 벡터를 섞지 않는다.

실제 캡처 검색은 기기에서 수행하므로 서버 응답에 screenshot 목록은 없다. 폴더 ID·보관함 전체·원본 사진은 이 요청에 보낼 필요가 없다. 앱은 선택한 실제 폴더, 날짜, 접근 가능, 제외 여부를 적용한 캡처에서 키워드·의미 검색을 수행한다. 검색어가 비면 이 API를 호출하지 않는다.

## 7. 사용량과 삭제

GET /v1/usage는 categories 배열에 category, used_units, reserved_units, limit_units nullable, resets_at nullable, can_submit을 반환한다. 단위·한도는 정책 확정 후 문서화한다. null은 미정·비적용을 뜻하며 무료 무제한을 뜻하지 않는다. 최종 제출 허용은 서버에서 원자적으로 판단한다.

삭제 상태: pending / processing / completed / failed. 설치 콘텐츠 전체 삭제가 접수되면 대상 진행 작업의 신규 실행을 막고 늦은 결과의 재저장을 차단한다. 처리 중 앱도 신규 제출을 멈춘다. 외부 AI 호출이 이미 시작됐다면 즉시 중단·공급자 삭제를 보장하지 않는다.

삭제 결과는 input_deleted, result_deleted, retained_categories 등 실제 처리 범위를 포함한다. 사용량·악용 방지 기록의 별도 보존 정책은 안내에 반영한다. 로컬 삭제는 앱이 수행하며 API 성공이 사진 앱 원본 삭제를 뜻하지 않는다.

## 8. 계약 확정 전 필수 작업

- 설치 검증·갱신 필드와 자격 분실 시 복구/삭제 정책 확정.
- 각 kind의 완전한 JSON 스키마, 문자열·목록·파일 크기 제한 확정.
- OCR 위치·긴 캡처 처리·모델 버전 교체와 로컬 재색인 정책 확정.
- 한도 단위·실패 비용·멱등 키 및 임시 콘텐츠 보존 기간 확정.
- 실제 선택한 서버 프레임워크 기준 OpenAPI 작성과 클라이언트 계약 검증.
- 타 설치 조회 차단, 중복 접수, 워커 중단·복구, 늦은 결과, 삭제 중 작업, 한도 경쟁, 검색 모델 불일치 검증.

## 이번 개정: 검색에 필요한 내부 데이터만 유지

- 캡처 제목·설명·태그의 독립 생성·표시·편집 요구사항을 제외했다. 검색용 짧은 요약·핵심어의 내부 사용 여부는 품질 검증에서 결정한다.
- OCR·관찰 내용·시각적 특징·검색 표현은 검색과 자동 분류를 위해 유지한다. 검색어 기록 저장 기능은 추가하지 않는다.
- 원본 연결·날짜·폴더 소속·처리 상태 및 수동 분류 교정은 유지한다.
- search_document는 내부 검색 색인 생성·복구·모델 변경용 작업이다. 사용자 제목·태그 편집용 API가 아니다. content_analysis가 반환한 검색 표현은 바로 재사용하며 중복 생성하지 않는다. folder_criteria의 description은 사용자가 입력하는 폴더 기준 설명이므로 유지한다.
