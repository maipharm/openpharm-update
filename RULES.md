# RULES.md — openpharm-update 발행 규칙

> 본 repo 에 commit 되는 모든 `artifacts/` / `revocation.json` 가 따라야 하는 **절대 규칙**.
> 한 줄이라도 어기면 약국 PC 가 sync 거부 또는 트러스트 깨짐.

---

## 1. Immutability — 이미 발행된 version 은 절대 수정 금지

| 파일 | 변경 가능 시점 |
|---|---|
| `artifacts/<slug>/snapshots/<version>.jsonl.zst` | **commit 후 영원히 금지** |
| `artifacts/<slug>/deltas/<from>__<to>.jsonl.zst` | **commit 후 영원히 금지** |
| `artifacts/<slug>/manifest.json` | 새 snapshot 추가 시 (기존 entry 수정 금지) |
| `revocation.json` | publishedAt/expiresAt/keys 갱신 시 (root 재서명 필수) |

오타·버그 발견 시 **새 revision** 발행 (`2026-05-01.r1` → `2026-05-01.r2`). 이전 파일은 그대로 둠.

위반 시: 과거 처방 재계산 결과가 달라져 청구 audit trail 깨짐.

---

## 2. 서명 체인 — 절대 placeholder commit 금지

```
root key (private, 운영자 HSM)
   ↓ root 서명
revocation.json
   ├─ trustedOperatorKeys[{keyId, publicKey, validFrom}]
   └─ revokedOperatorKeys[{keyId, revokeManifestsAfter}]
       ↓ operator 서명 (trustedOperatorKeys 중 하나)
artifacts/<slug>/manifest.json
   ├─ keyId: "operator-2026-05"
   └─ signature: "ed25519:..."
```

| 규칙 | 위반 시 |
|---|---|
| `manifest.signature` 가 `"UNSIGNED"` 이거나 빈 문자열인 채로 commit 금지 | verify-and-deploy.yml 실패 + 약국 PC `data_corrupt.signature` |
| `revocation.signature` 가 `"UNSIGNED"` 이거나 빈 문자열인 채로 commit 금지 | 약국 PC `sync.precondition.revocation_signature` |
| `manifest.keyId` 가 `revocation.trustedOperatorKeys[].keyId` 에 없는 채로 commit 금지 | 약국 PC `sync.precondition.key_revoked` |
| `revocation.json` 의 root 서명은 **operator key 가 아니라 root key** 로 해야 함 | 클라이언트 검증 실패 |

서명 알고리즘: **ed25519 + JCS canonical JSON** (RFC8785 핵심 — 키 알파벳 정렬 + 공백 제거).

---

## 3. 인코딩 — 절대 UTF-16 / BOM 금지

JSON 파일은 모두 **UTF-8 no-BOM**.

함정:
- PowerShell `>` 리다이렉트 → UTF-16 LE BOM (잘못된 출력).
- Notepad 한국어 환경 기본 저장 → UTF-8 with BOM 또는 UTF-16.

권장: Bash redirect 또는 `[System.IO.File]::WriteAllText("file", $content, [System.Text.UTF8Encoding]::new($false))`.

위반 시: 약국 PC 에서 `invalid character 'ÿ'` 또는 `invalid character 'ï'`.

---

## 4. JCS canonical JSON 서명 호환성

`opub` publisher 와 v2 client `internal/envelope.CanonicalJSON` 가 **동일 알고리즘** 이어야 함:
- 객체 키 알파벳 순 정렬 (재귀).
- 공백 제거.
- `signature` 필드는 서명 계산 시 제외 (빈 문자열 또는 절차).

만약 `manifest.json` 또는 `revocation.json` 을 손으로 편집했다면 — `signature` 필드를 비우고 `opub sign` 또는 `opub sign-root` 로 재서명할 것. **절대 손으로 base64 서명 만지지 말 것**.

---

## 5. 채널 일관성 규칙

### 5.1 channelId ↔ pathSlug

| channelId (manifest 내부) | pathSlug (URL/디렉토리) |
|---|---|
| `drug_prices` | `drug-prices` |
| `rulesets/billing` | `rulesets/billing` |
| `fee_schedules` | `fee-schedules` |
| `copay_rules` | `copay-rules` |
| `holidays` | `holidays` |
| `escape_prevention` | `escape-prevention` |
| `special_codes` | `special-codes` |

snake_case ↔ kebab-case 변환은 `internal/sync/path.go:ChannelPathSlug`. 새 채널 추가 시 양쪽 모두 등록.

### 5.2 version 형식

| 채널 | 권장 version 형식 | 예 |
|---|---|---|
| 연 1회 (holidays) | `<YYYY>.r<n>` | `2026.r1` |
| 월 1회 이상 (drug_prices, escape_prevention) | `<YYYY-MM-DD>.r<n>` | `2026-05-01.r2` |
| 비정기 (rulesets/billing, copay_rules) | `<YYYY-MM-DD>.r<n>` | `2026-05-01.r1` |

`r<n>` 은 같은 발행 사이클 내 정정본 카운터.

### 5.3 effectiveFrom

- 각 snapshot 은 `effectiveFrom` (YYYY-MM-DD) 가 있어야 함 — 룰셋이 발효되는 날짜.
- `effective_to` 는 **다음 version 의 `effectiveFrom`** 으로 자동 결정 (open upper bound). 즉 wire 에는 effectiveTo 없음.

---

## 6. 발행 (push) 절차

상세는 `openpharm-admin/RULES.md` 참조. 본 repo 측 규칙 요약:

1. **수동 commit 금지** — 반드시 `opub publish + opub sign` 으로 생성된 파일만 commit.
2. **single channel per commit** 권장 — debug 용이성. 두 채널 동시 발행 시 commit 분리.
3. **commit message prefix**: `[sync]` 또는 `[<channelId>]`.
4. **push 후 5-30초 내 GitHub Pages 배포**. `gh run watch -R guinnessNet/openpharm-update` 로 verify-and-deploy 통과 확인.
5. 실패 시 — revert 하지 말고 새 commit 으로 픽스 (immutability §1).

---

## 7. 키 회전 (operator key rotation)

새 operator key 발급 시:

1. `opub keygen --operator --keyId operator-<YYYY-MM> --out ~/.openpharm/operator-<YYYY-MM>.key` (admin PC HSM).
2. `revocation.json` 의 `trustedOperatorKeys` 에 새 항목 추가 (validFrom 명시).
3. 이전 키 폐기 시 `revokedOperatorKeys` 에 `revokeManifestsAfter` 와 함께 추가 (이미 발행된 manifest 는 정상, 이후만 거부).
4. `opub sign-root` 로 revocation.json 재서명.
5. commit + push.
6. 새 manifest 부터 새 keyId 사용.

root key 회전은 client binary 재배포 필요 (`internal/sync/default_root.pub` 교체). 가능한 한 회피.

---

## 8. CI 게이트 (verify-and-deploy.yml)

다음을 모두 통과해야 GitHub Pages 배포:

- `revocation.json` spec/signature non-UNSIGNED 검증.
- 모든 `artifacts/*/manifest.json` spec/signature/keyId non-UNSIGNED + snapshots 최소 1건.
- 각 manifest 의 keyId 가 revocation trustedOperatorKeys 에 있는지.

위반 시 workflow fail → Pages 배포 안 됨 → 약국 PC 는 직전 cache 유지.

---

## 9. 비상 시 — revocation

operator key 유출 의심:

1. `revocation.json` 의 `revokedOperatorKeys` 에 항목 추가 (`revokeManifestsAfter`: 유출 시점 timestamp).
2. `opub sign-root` 재서명.
3. commit + push (CI 가 schema 만 검증, 재서명 detect 못함 — 운영자 책임).
4. 약국 PC 는 7일 grace period 내에 새 revocation 수신 + 해당 시점 이후 manifest 모두 거부.

---

## 10. 위반 적발 시 — 외부 audit 권장 행동

본 repo 는 public — 누구나 PR/issue 로 audit 결과 제기 가능. 일반적인 적발 패턴:

- "manifest 가 published 일자보다 미래 effectiveFrom" → 정상 (사전 공지).
- "snapshot rowsHash mismatch" → 버그/유출, 즉시 새 revision 발행 + revocation.
- "동일 version 의 snapshot 이 PR diff 로 바뀜" → §1 violation, 운영자 책임.
