# STATUS

## 2026-05-15

- [x] P2 R2 mirror push workflow step 추가
  - 파일: `.github/workflows/verify-and-deploy.yml`
  - 조건: `vars.R2_ENABLED == 'true'`
  - 동기화 범위: `_site/sync/` → `s3://openpharm-update-mirror/sync`
  - 가드: `R2_ENDPOINT_URL` 미설정 시 실패 처리
  - IAM 준비 전에는 skip step 실행

## 다음 작업

- [ ] R2 IAM 키 발급 후 `R2_ENABLED=true` 전환 및 실운영 검증
