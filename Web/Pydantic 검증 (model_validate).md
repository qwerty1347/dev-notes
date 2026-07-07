---
tags:
  - fastapi
  - pydantic
  - validation
created: 2026-04-18
---

# FastAPI와 Pydantic 검증: `validate_model` / `model_validate`

FastAPI 라우트 파라미터로 Pydantic 모델을 선언하면 FastAPI가 자동으로 검증을 수행한다. 따라서 라우트 내부에서는 별도의 `validate_model` 호출이 대부분 불필요하다.

## `validate_model` / `model_validate` 를 쓰는 경우

- 라우트 바깥에서 외부 데이터를 모델로 변환하며 검증할 때
- 배치 스크립트, 서비스 계층, 메시지 소비기 등에서 raw dict를 모델로 검증할 때
- 외부 시스템에서 받은 데이터를 신뢰 경계 안으로 들여올 때
- 일부 필드만 검증하거나 다단계 변환이 필요할 때

```python
# Pydantic v2
user = User.model_validate(raw_dict)
user = User.model_validate_json(raw_json_bytes)
```

## FastAPI 라우트에서는?

FastAPI가 이미 다음을 수행한다:
- 타입 힌트 기반 검증
- 누락 필드, 타입 불일치, 유효성 검사 오류 자동 처리

따라서 라우트 내부에서 다시 `model_validate` 를 호출하면 중복 검증이 된다.

## 언제 안 써도 되나?

- 라우트 파라미터로 Pydantic 모델을 선언했을 때
- 단순 타입 검증이 목적일 때
- 이미 FastAPI가 데이터 구조를 책임지는 경우

## 핵심 요약
- 라우트 안에서는 `FastAPI`가 자동 검증을 담당한다.
- 바깥에서 dict/JSON을 모델로 변환할 때만 `model_validate` 를 명시적으로 쓴다.
