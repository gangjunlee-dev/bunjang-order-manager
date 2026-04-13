# 번장 주문관리 대시보드 (Bunjang Order Manager)

## Project Overview
- **Name**: Bunjang Order Manager
- **Goal**: 번장 자동결제 후 판매자 사유로 취소된 주문을 수동으로 재결제하고, 전체 주문/상품을 관리하는 대시보드
- **Tech Stack**: Hono + TypeScript + TailwindCSS (Cloudflare Pages)

## 핵심 기능

### 1. 취소/환불 주문 대시보드
- 전체 주문 중 `REFUNDED` / `CANCELLED` 상태 자동 필터링
- 한눈에 취소된 상품 확인 가능
- **재주문 버튼** 클릭으로 즉시 수동 결제 폼으로 이동

### 2. 수동 주문 생성 (결제)
- 번장 Open API `POST /api/v2/orders`를 통한 직접 주문
- 카드/포인트 결제 제한 우회하여 API 기반 결제
- 상품 ID 입력 시 자동으로 가격/배송비 조회
- 기본 배송 정보 저장 (localStorage)

### 3. 상품 검색 & 조회
- 번장 카탈로그 전체 탐색
- 상품 ID(PID) 직접 조회
- 상품 상세 정보 (이미지, 가격, 상태, 옵션 등)
- 상품에서 바로 주문 가능

### 4. 주문 상세 & 구매 확정
- 주문별 상세 정보 (금액, 판매자, 배송, 반품 내역)
- 구매 확정 (Confirm Purchase) 기능

## API 라우트

| Method | Path | 설명 |
|--------|------|------|
| GET | `/` | 대시보드 메인 페이지 |
| GET | `/api/settings/check` | API 키 설정 상태 확인 |
| GET | `/api/orders` | 주문 목록 조회 (?page, ?size) |
| GET | `/api/orders/:orderId` | 주문 상세 조회 |
| POST | `/api/orders` | 수동 주문 생성 (결제) |
| POST | `/api/orders/:orderId/confirm` | 구매 확정 |
| GET | `/api/products/search` | 상품 카탈로그 조회 (?cursor) |
| GET | `/api/products/:pid` | 상품 상세 조회 |
| GET | `/api/categories` | 카테고리 목록 |
| GET | `/api/brands` | 브랜드 목록 |

## 환경변수 설정

### 로컬 개발 (.dev.vars)
```
BUNJANG_ACCESS_KEY=your_access_key
BUNJANG_SECRET_KEY=your_secret_key
BUNJANG_API_URL=https://openapi.bunjang.co.kr
```

### 프로덕션 (Cloudflare)
```bash
npx wrangler pages secret put BUNJANG_ACCESS_KEY --project-name webapp
npx wrangler pages secret put BUNJANG_SECRET_KEY --project-name webapp
npx wrangler pages secret put BUNJANG_API_URL --project-name webapp
```

## 인증 방식
- 번장 Open API는 JWT 기반 인증 사용
- `accessKey` + `secretKey`로 JWT 토큰 생성 (HS256)
- 토큰 유효기간: 30초
- POST/PUT/DELETE 요청 시 `nonce` (UUID v4) 필수

## 개발 & 배포

```bash
# 로컬 개발
npm run build
pm2 start ecosystem.config.cjs

# 배포
npm run build
npx wrangler pages deploy dist --project-name webapp
```

## 주의사항
- `.dev.vars` 파일에 실제 API 키를 입력해야 동작합니다
- 번장 API 키는 파트너 계약 후 발급 (partner_global@bunjang.co.kr)
- Production API: `https://openapi.bunjang.co.kr`
- Sandbox API: `https://openapi.stg-bunjang.co.kr`
