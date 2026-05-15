# openpharm-update

> 약국 PC (`openpharm-v2`) 가 약가·수가·본인부담률·공휴일·산정특례 같은 정책 데이터를 받아가는 **public 데이터 리포지터리**.
>
> 와이어 프로토콜: `SYNC_PROTOCOL_V1` (v2 docs/specs/)
> 운영 규칙: [RULES.md](./RULES.md) 필독.

---

## 1. 이게 뭐고 왜 public?

대한민국 약국이 사용하는 청구·재고 시스템 `openpharm` 의 **정책 데이터 배포 채널**.

- 약가 (HIRA 약제급여목록표)
- 약국 수가 (보건복지부 행위급여 고시)
- 본인부담률 (국민건강보험법 시행령 등)
- 공휴일 (한국천문연구원 + 대체공휴일 규정)
- 퇴장방지의약품 + 가산률
- 산정특례 V-code (특정기호)
- 자체 청구 룰셋 (`rulesets/billing`)

모두 **정부 공시 정보**라 비밀이 아니고, 약국·HIRA·복지부·외부 약사 누구나 **변환 결과가 정확한지 audit** 할 수 있어야 한다 — 그래서 public.

운영자 측 변환 코드·서명 키는 별도 private 리포 (`openpharm-admin`) 에 격리.

---

## 2. 현재 발행 상태 (2026-05-15)

| channelId | pathSlug | 최신 version | 행수 | 상태 |
|---|---|---|---|---|
| `drug_prices` | `drug-prices` | `2026-05-01.r2` | 21,726 | ✅ |
| `rulesets/billing` | `rulesets/billing` | `2026-05-01.r1` | 1 | ✅ |
| `fee_schedules` | `fee-schedules` | `2026-05-01.r1` | 45 | ✅ |
| `copay_rules` | `copay-rules` | `2026-05-01.r1` | 25 | ✅ |
| `holidays` | `holidays` | `2026.r1` | (휴일 데이터) | ✅ |
| `escape_prevention` | `escape-prevention` | — | — | ⏳ HIRA 월간 데이터 대기 |
| `special_codes` | `special-codes` | — | — | ⏳ HIRA 특정기호 목록 대기 |

---

## 3. 구조

```
artifacts/<pathSlug>/                   ← 약국 PC 가 받아가는 immutable 산출물
  ├─ snapshots/<version>.jsonl.zst     ← full-state (cold-start 가능)
  ├─ deltas/<from>__<to>.jsonl.zst     ← 증분 (선택)
  └─ manifest.json                      ← 카탈로그 + operator key 서명 (ed25519 + JCS canonical)

revocation.json                          ← root key 가 서명한 trust list + 폐기 목록
docs/                                   ← 운영 문서
RULES.md                                ← 발행 규칙 (immutability/서명/audit) 필독
.github/workflows/verify-and-deploy.yml ← schema/sig/root 검증 → GitHub Pages + (선택) R2 mirror
```

`channelId` (snake_case) ↔ `pathSlug` (kebab-case) 변환은 SYNC_PROTOCOL_V1 §1.1.

---

## 4. 클라이언트 접근 URL

| 역할 | URL |
|---|---|
| **Primary** (GitHub Pages) | `https://guinnessnet.github.io/openpharm-update/sync/<pathSlug>/manifest.json` |
| Mirror | `https://mirror.openpharm.kr/sync/<pathSlug>/manifest.json` (Cloudflare R2, 설정 시) |
| Raw (audit) | `https://raw.githubusercontent.com/guinnessNet/openpharm-update/main/artifacts/<pathSlug>/manifest.json` |

약국 PC env: `OPENPHARM_SYNC_BASE_URL=https://guinnessnet.github.io/openpharm-update`.

---

## 5. 어떻게 audit 하는지

1. `artifacts/<pathSlug>/manifest.json` 의 `signature` (ed25519) 가 `revocation.json` 의 `trustedOperatorKeys` 에 등록된 operator pubkey 와 매칭하는지.
2. `revocation.json` 의 `signature` 가 root pubkey (`openpharm-v2/internal/sync/default_root.pub` 또는 env `OPENPHARM_SYNC_ROOT_PUBKEY`) 와 매칭하는지.
3. `manifest.snapshots[].hash` (sha256) 가 다운로드한 `snapshots/<version>.jsonl.zst` 실제 해시와 일치하는지.
4. 압축 해제 후 첫 라인 `_meta.rowsHash` 가 row 라인들의 sha256 결합과 일치하는지.

검증 도구: `openpharm-admin/tools/opub/opub.exe verify <manifest_path>`.

---

## 6. 라이선스

- **데이터**: 공공누리 제1유형 (출처표시) — 정부 공시 정보 가공.
- **스크립트·workflow**: MIT (`LICENSE`).

---

## 7. 변경 이력

| 날짜 | 변경 |
|---|---|
| 2026-05-15 | 첫 5 채널 발행 (holidays/drug_prices/rulesets/fee/copay) — root+operator key 발급 + JCS canonical 서명 |
