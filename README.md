# openpharm-update

> 약국 PC (`openpharm-v2`) 가 약가·수가·본인부담률·공휴일·산정특례 같은 정책 데이터를 받아가는 **public 데이터 리포지터리**.
>
> 와이어 프로토콜: [openpharm-v2/docs/specs/SYNC_PROTOCOL_V1.md](https://github.com/openpharm/openpharm-v2/blob/main/docs/specs/SYNC_PROTOCOL_V1.md)

---

## 이게 뭐고 왜 public?

대한민국 약국이 사용하는 청구·재고 시스템 `openpharm` 의 **정책 데이터 배포 채널**.

- 약가 (HIRA 약제급여목록표)
- 약국 수가 (보건복지부 행위급여 고시)
- 본인부담률 (국민건강보험법 시행령 등)
- 공휴일 (한국천문연구원 + 대체공휴일 규정)
- 퇴장방지의약품 + 가산률
- 산정특례 V-code

모두 **정부 공시 정보**라 비밀이 아니고, 약국·HIRA·복지부·외부 약사 누구나 **변환 결과가 정확한지 audit** 할 수 있어야 한다 — 그래서 public.

운영자 측 변환 코드·서명 키는 별도 private 리포 (`openpharm-admin`) 에 격리.

---

## 구조

```
artifacts/<pathSlug>/                   ← 약국 PC 가 받아가는 immutable 산출물
  ├─ snapshots/<version>.jsonl.zst     ← full-state (cold-start 가능)
  ├─ deltas/<from>__<to>.jsonl.zst     ← 증분 (선택)
  └─ manifest.json                      ← 카탈로그 + operator key 서명

sources/<channelId>/<version>.source.json   ← 어디서 받은 원본인지 (URL + sha256)
revocation.json                              ← root key 가 서명한 trust list + 폐기 목록
```

`channelId` (snake_case) ↔ `pathSlug` (kebab-case) 변환은 SYNC_PROTOCOL_V1 §1.1.

---

## 발행 채널

| channelId | pathSlug | 갱신 빈도 | 원본 |
|---|---|---|---|
| `drug_prices` | `drug-prices` | 매월 1일 | HIRA 약제급여목록표 |
| `fee_schedules` | `fee-schedules` | 매년 1월/7월 | 복지부 행위급여 고시 |
| `copay_rules` | `copay-rules` | 시행령 개정시 | 국민건강보험법 시행령 별표 3 |
| `holidays` | `holidays` | 매년 1회 | 한국천문연구원 특일정보 |
| `escape_prevention` | `escape-prevention` | 매월 1일 | HIRA 약제기준정보 |
| `special_codes` | `special-codes` | 시행령 개정시 | 본인일부부담금 산정특례 고시 |
| `rulesets/billing` | `rulesets/billing` | 분기~연 | 자체 |

---

## 클라이언트 접근 URL

| 역할 | URL |
|---|---|
| Primary | `https://policy.openpharm.kr/sync/<pathSlug>/manifest.json` |
| Mirror | `https://mirror.openpharm.kr/sync/<pathSlug>/manifest.json` (Cloudflare R2) |
| Raw | `https://raw.githubusercontent.com/openpharm/openpharm-update/main/artifacts/<pathSlug>/manifest.json` |

---

## 어떻게 audit 하는지

1. `sources/<channelId>/<version>.source.json` — 어디서 어떤 sha256 의 원본을 받았는지
2. `artifacts/<pathSlug>/snapshots/<version>.jsonl.zst` — 변환 결과 (immutable)
3. PR diff (`<version>.r1` → `<version>.r2`) — 정정본 변경분

검증 절차는 `docs/AUDIT_GUIDE.md`.

---

## 라이선스

데이터: 공공누리 제1유형 (출처표시) — 정부 공시 정보 가공.
스크립트·workflow: MIT.

---

## 발행 phase

- [x] A1 — 첫 채널 (holidays 2026.r1) 수동 발행, manifest UNSIGNED placeholder
- [ ] A2 — JSON Schema + opub CLI verify
- [ ] A3 — operator key 서명
- [ ] A4 — root 서명 revocation.json + Cloudflare R2 mirror
- [ ] A5 — 자동 모니터 PR (admin repo)
- [ ] A6 — custom domain `policy.openpharm.kr`

상세는 [openpharm-v2/docs/specs/OPENPHARM_ADMIN_V1.md §9](https://github.com/openpharm/openpharm-v2/blob/main/docs/specs/OPENPHARM_ADMIN_V1.md).
