# Portfolio

간단한 포트폴리오 리포지터리입니다. 이 저장소에는 발표자료와 몇 가지 데모 서비스(이미지 업로드·워터마크, OAuth 예제)가 포함되어 있습니다.

## 프로젝트 구성

- `presentation/` : 발표 자료(PDF)
  - `jasypt/`, `oauth/`, `plsql/` 등
- `readme/` : 데모 프로젝트와 관련 문서
  - [readme/oauth2/README.md](readme/oauth2/README.md) — Daeryun OAuth (Keycloak 기반 인증 서버 + Resource Server 설정 및 로컬 세팅 가이드)
  - [readme/image_watermark/README.md](readme/image_watermark/README.md) — Image Upload Service (WebP 변환, 워터마크, S3 업로드)

## 주요 항목 요약

- Image Upload Service (`readme/image_watermark`)
  - 이미지 업로드 시 WebP 변환, 워터마크 적용 후 S3에 저장
  - 로컬 실행: `npm install` 후 `npm run dev`
  - Swagger UI로 API 문서를 확인할 수 있음 (자세한 내용은 해당 README 참조)

- OAuth2 예제 (`readme/oauth2`)
  - Keycloak을 Authorization Server로 사용하여 토큰 발행/검증 기능 구현
  - Docker(Compose) 기반 로컬 세팅, DB 복원 및 Keycloak 설정 가이드 포함

- Presentation (`presentation/`)
  - 다양한 발표자료(PDF)가 포함되어 있습니다. 필요 시 열어서 참고하세요.


