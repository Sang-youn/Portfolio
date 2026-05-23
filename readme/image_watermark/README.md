# Image Upload Service

이미지를 업로드하면 **WebP 변환**, **워터마크 적용**, **AWS S3 저장** 후 **CloudFront URL**을 반환하는 API 서비스입니다.  
Next.js API Routes로 구현되어 있으며, AWS Lambda(Serverless Framework)로도 배포할 수 있습니다.

## 주요 기능

| 기능 | 설명 |
|------|------|
| 이미지 업로드 | JPEG, PNG, GIF, WebP, BMP, TIFF 지원 |
| WebP 변환 | Sharp로 WebP(quality 100) 변환 후 업로드 |
| 워터마크 | 텍스트 또는 S3에 등록된 이미지 워터마크 |
| 멀티 타겟 | 암호화된 `target` 값으로 S3 버킷·CloudFront 도메인 지정 |
| PDF 업로드 | 별도 API(`/api/uploadPdf`)로 PDF 직접 업로드 |
| API 문서 | Swagger UI (`/api-docs`) |

## 처리 흐름

```
클라이언트 (multipart/form-data)
    │
    ▼
POST /api/upload
    │
    ├─ Multer: 임시 디스크에 파일 저장
    ├─ target 복호화 (AES-256-CBC)
    ├─ Sharp: WebP 변환 + 워터마크 합성
    ├─ S3 PutObject (upload/YYYY-MM-DD/{uuid}.webp)
    └─ CloudFront URL 반환
```

GIF 파일은 애니메이션 보존을 위해 WebP 변환·워터마크 없이 원본 그대로 업로드합니다.

## API

### `POST /api/upload`

이미지 1장을 업로드하고 처리된 URL을 받습니다.

**Content-Type:** `multipart/form-data`

| 필드 | 필수 | 설명 |
|------|------|------|
| `file` | O | 업로드할 이미지 파일 |
| `target` | O | AES로 암호화된 업로드 대상 정보 (`버킷명\|CloudFront_프리픽스` 형식의 평문을 암호화) |
| `watermarkType` | - | `image` 또는 `text` |
| `watermarkText` | - | 텍스트 워터마크 내용 (기본: `${기본텍스트}`) |
| `hAlign` | - | 가로 정렬: `-1` 왼쪽, `0` 가운데, `1` 오른쪽 |
| `vAlign` | - | 세로 정렬: `-1` 아래, `0` 가운데, `1` 위 |

**성공 응답 (200)**

```json
{
  "code": "0000",
  "message": "성공",
  "fileUrl": "https://{cloudfront-prefix}.cloudfront.net/upload/2024-11-18/{uuid}.webp"
}
```

**에러 응답**

| HTTP | code | 설명 |
|------|------|------|
| 400 | E400 | 이미지가 아닌 파일, 파일 미첨부 |
| 405 | - | POST 이외 메서드 |
| 500 | E500 | 처리·업로드 실패 |

### `POST /api/uploadPdf`

PDF 파일을 변환 없이 S3에 업로드합니다. (`pages/api/uploadPdf.js`)

### API 문서

로컬 실행 후 [http://localhost:3000/api-docs](http://localhost:3000/api-docs)에서 Swagger UI로 전체 스펙을 확인할 수 있습니다.

## 워터마크

### 텍스트 (`watermarkType=text`)

- `fonts/Bazzi.ttf` 폰트를 SVG에 임베딩해 이미지 중앙에 합성합니다.

### 이미지 (`watermarkType=image`)

- S3 버킷의 `upload/{버킷명}_watermark.png` 파일을 다운로드해 사용합니다.
- 원본 너비의 약 35%로 리사이즈하고, 60% 불투명도를 적용합니다.
- `hAlign`, `vAlign`으로 위치를 조정합니다.

## 프로젝트 구조

```
├── pages/api/
│   ├── upload.js          # 이미지 업로드·변환·워터마크 (핵심)
│   ├── uploadPdf.js       # PDF 업로드
│   └── docs.js            # Swagger 스펙 JSON
├── pages/api-docs.jsx     # Swagger UI 페이지
├── commons/utils/
│   └── crypto.js          # target·환경변수 AES 복호화
├── doc/swagger/
│   └── uploadApiDoc.js    # OpenAPI 정의
├── fonts/
│   └── Bazzi.ttf          # 텍스트 워터마크용 폰트
├── serverless.yml         # Lambda 배포 설정
└── next.config.ts         # ENC() 형식 환경변수 자동 복호화
```

## 환경 변수

`.env` 파일에 아래 값을 설정합니다. `ENC(...)`로 감싼 값은 `next.config.ts`에서 자동 복호화됩니다.

| 변수 | 용도 |
|------|------|
| `AWS_ACCESS_KEY_ID` | AWS 인증 |
| `AWS_SECRET_ACCESS_KEY` | AWS 인증 |
| `AWS_REGION` | S3 리전 (예: `ap-northeast-2`) |
| `S3_BUCKET_NAME` | 기본 S3 버킷 (target 미지정 시) |
| `UIW_ENCRYPTION_KEY` | AES-256 키 (32바이트 hex) |
| `UIW_IV` | AES IV (16바이트 hex) |

`target` 필드는 `버킷명|cloudfront-도메인-프리픽스` 형태의 문자열을 위 키로 암호화한 hex 값입니다.

## 로컬 실행

```bash
npm install
npm run dev
```

- API: `http://localhost:3000/api/upload`
- 문서: `http://localhost:3000/api-docs`
- 헬스체크 페이지: `/healthCheck_bnVC0m0X2foVekm`

## AWS Lambda 배포

[Serverless Framework](https://www.serverless.com/) 설정(`serverless.yml`):

- **서비스명:** `upload-image-to-webp`
- **함수명:** `upload_image_to_webp_wm`
- **엔드포인트:** `POST /api/upload`
- **리전:** `ap-northeast-2`

```bash
# serverless CLI 설치 후
serverless deploy
```

> Lambda 배포 시 `sharp` 등 네이티브 모듈은 Linux(x64)용으로 빌드·패키징해야 합니다. `serverless.yml` 주석의 esbuild/sharp 설정을 참고하세요.

## 기술 스택

- **Runtime:** Node.js 20
- **Framework:** Next.js 15 (Pages Router API)
- **이미지 처리:** Sharp, Multer
- **스토리지:** AWS S3 (`@aws-sdk/client-s3`)
- **CDN:** CloudFront (업로드 URL은 `{prefix}.cloudfront.net` 형식)
- **문서:** Swagger UI (`swagger-ui-react`, `next-swagger-doc`)

## 의존성 요약

| 패키지 | 역할 |
|--------|------|
| `sharp` | WebP 변환, 리사이즈, 워터마크 합성 |
| `multer` | multipart 파일 수신 |
| `moment` | S3 키 경로 날짜 (`YYYY-MM-DD`) |
| `@aws-sdk/client-s3` | S3 업로드·워터마크 이미지 다운로드 |
