# thought-flow-extractor v3.3

**Personal Bidirectional Interaction-Flow Extractor** — một Claude skill phân tích vòng lặp tương tác user ↔ AI từ transcript chat. Không phải tóm tắt — đây là **bản đồ nhân-quả của một cuộc hội thoại**.

> *English short summary*: A Claude skill that reverse-engineers chat transcripts to extract bidirectional interaction patterns. Analyzes both why each user prompt exists AND why each AI response chose its specific framing. Outputs structured journal in Markdown + JSON.

---

## Skill này làm gì

Cho 1 transcript chat (Claude / ChatGPT / markdown / paste text), skill output:

- **Per-turn analysis**: substance + causality + forward signal
- **Bidirectional flow**: user prompt → AI interpretation → AI response strategy → user reaction
- **Constraint tracking**: AI preserve / drop / vi phạm gì qua các turn
- **Artifact evolution**: artifact qua versions, dependency graph, quality delta
- **Pattern detection** (4 nhóm): user / AI / interaction loop / artifact
- **AI tự tạo vấn đề gì cho user**: track khi AI gây ra correction loop
- **Alignment / misalignment map**: AI hiểu sai chỗ nào, repair ở đâu, unresolved gì
- **Reusable lessons**: bài học prompt + bài học AI collaboration

## Triết lý cốt lõi

```
Không chỉ phân tích user nghĩ gì.
Không chỉ phân tích AI trả lời gì.
Mà phân tích toàn bộ hệ tương tác:

user intent → prompt design → AI interpretation → response strategy
  → artifact output → user correction → context accumulation → next interaction
```

**Primary analysis unit is NOT the user, NOT the AI, but the INTERACTION LOOP.**

---

## Cài đặt

1. Download file `thought-flow-extractor-v3.3.skill`
2. Mở Claude.ai → Settings → Capabilities → Skills
3. Upload file `.skill`
4. Skill sẽ tự trigger khi mày dùng các phrase trigger (xem dưới)

Hoặc cài qua Claude Code: `~/.claude/skills/thought-flow-extractor.skill`.

## Cách dùng

### Trigger phrases

Skill tự bật khi user message chứa các cụm:
- "phân tích cuộc hội thoại"
- "luồng tư duy" / "thought flow"
- "interaction flow"
- "reverse engineer chat"
- "prompt-response causality"
- Paste transcript + "phân tích" / "AI hiểu sai chỗ nào" / "alignment"

### Không trigger trên

- "Tóm tắt file" generic
- "Dịch file"
- "Sửa câu"
- File không phải transcript chat

### Flags (3 tùy chọn)

| Flag | Values | Default |
|---|---|---|
| `--privacy` | `raw` \| `redacted` \| `anonymized` | `raw` |
| `--output` | `<prefix>` (filename prefix) | `journal` |
| `--lang` | `vi` \| `en` \| `auto` | `auto` |

Skill **tự quyết** depth/schema strictness/focus — không user-configurable, không pick mode. Auto-adapt theo transcript length.

### Output format

Trước khi xuất file, skill **bắt buộc hỏi user** chọn format:
1. `md` — chỉ `journal.md` (prose cho người đọc)
2. `json` — chỉ `journal.json` (schema cho LLM batch / programmatic)
3. `both` — cả 2 file

Không default. Không skip câu hỏi.

---

## Ví dụ workflow

```
User: Phân tích đoạn chat này: <paste transcript 8 turn>

AI [Claude với skill]:
[Đọc safety-rules + input-format-detection]
[Detect: chatgpt_export, 8 turns]
[Build messages → turns → interaction_flow (user_to_ai + ai_to_user)]
[Detect feedback_loops + alignment_map]
[Build ai_response_strategy_layer + ai_created_next_problem]
[Detect patterns (4 nhóm) — LAST, không sớm]
[Run 10 evaluator checks R1-R10]
[Hỏi: "Mày muốn xuất file format nào? md / json / both"]

User: both

AI: [Xuất journal.md (prose) + journal.json (schema)]
```

## Cấu trúc package

```
thought-flow-extractor/
├── SKILL.md                              ← Entry point, quick reference, workflow
├── CHANGELOG.md                          ← v3.0 → v3.3 history
├── MIGRATION.md                          ← v3.2 → v3.3 breaking changes
├── references/                           (lazy-load theo workflow step)
│   ├── safety-rules.md                   ← 9 safety rules (injection, PII, AI internal state, ...)
│   ├── input-format-detection.md         ← Parse 6 formats (Claude/ChatGPT/markdown/...)
│   ├── causality-framework.md            ← DAG, 17 edge relations, 5 directions
│   ├── schema.md                         ← Schema overview + 3 lens payload mẫu
│   ├── schema.json                       ← JSON Schema Draft 2020-12
│   ├── pattern-catalog.md                ← 4-subject model + 8 forbidden archetype labels
│   ├── language-rules.md                 ← VN informal verbatim, anti-flattery
│   └── evaluator-rubric.md               ← 10 critical self-checks (R1-R10)
└── assets/
    ├── template.md                       ← 1 template duy nhất (no variants)
    ├── example.md                        ← Good example (data pipeline session)
    └── bad-example.md                    ← 6 categorized bad patterns
```

14 files, ~26K tokens khi load đầy đủ. **Lazy-load**: SKILL.md có schedule load reference theo workflow step (Claude không cần load all 14 file cùng lúc).

## Output schema overview

`journal.md` (prose) — văn xuôi cho người đọc. Section names VN tự nhiên ("Cốt truyện hội thoại", "Hai bên trao đổi qua mỗi turn", "AI tự tạo vấn đề gì cho user", "Khi mày prompt lần sau", v.v.).

`journal.json` (schema-strict) — cho LLM batch / programmatic query.

**Required blocks** (9 top-level):
- `document_meta` (source, language, privacy, safety_flags, ...)
- `conversation_core` (one_sentence_summary, primary_user_goal, outcome)
- `turns[]` (state_delta.constraint_delta 5 sub-fields, evidence_refs)
- `interaction_flow.user_to_ai[]` + `interaction_flow.ai_to_user[]`
- `pattern_layer` (4 nhóm: user / ai / interaction_loop / artifact)
- `evidence_layer` (claim_ledger, evidence_quotes, unknowns)
- `extensions`

**Optional blocks** (fill nếu transcript có evidence):
- `task_facets`, `constraint_layer`, `causality_graph`, `artifact_layer`
- `ai_response_strategy_layer` (default REQUIRED nếu ≥ 1 AI response)
- `ai_errors`, `counterfactual_notes`, `constraint_memory_check`
- `interaction_outcome`, `reusable_prompting_lessons`, `ai_collaboration_lessons`
- `session_compression`, `queryable_summary`, `domain_lenses`

Schema chi tiết: `references/schema.md` + `references/schema.json`.

---

## Critical rules (10 self-checks)

Mỗi journal output phải pass:

| Rule | Mục đích |
|---|---|
| R1 | Evidence coverage — mọi claim có ≥ 1 quote_id |
| R2 | No archetype labels — 8 forbidden labels (polymath, high-agency, ...) |
| R3 | No fluff — 4 tests: xóa / đảo cảm xúc / "ai cũng nói" / thông tin |
| R4 | Anti-forced-fit pattern — ≥ 2 evidence + 4 required fields |
| R5 | Causal traceability — mọi turn từ T2 có ≥ 1 edge |
| R6 | Quote fidelity VN informal — tao/mày/ko verbatim |
| R7 | Interaction flow populated — user_to_ai + ai_to_user đầy đủ |
| R8 | No AI internal state — observable response strategy only |
| R9 | Bidirectional balance — ≥ 1 pattern ai / interaction_loop / artifact |
| R10 | Constraint memory check — preserved / dropped / repeated / violated |

Chi tiết: `references/evaluator-rubric.md`.

---

## Anti-patterns (KHÔNG bao giờ làm)

- ❌ Archetype labels: "polymath", "high-agency learner", "critical thinker", "special user", "brilliant", "genius", "rare", "tư duy đặc biệt"
- ❌ AI mind-reading: "AI đã nghĩ X", "AI thực sự hiểu Y", "AI cố tình"
- ❌ User psychology diagnosis: "User có vẻ depressed / stress / burnout"
- ❌ Force-fit pattern: gán pattern mà chỉ 1 evidence
- ❌ Flattery: "câu hỏi rất sâu sắc", "hành trình tư duy", "ở tầng sâu hơn"
- ❌ Identity from local behavior: "User là Y" (chỉ "Trong session này, user làm X")
- ❌ Sanitize informal VN: "tao muốn" → "tôi muốn" (NEVER)

Xem `assets/bad-example.md` cho 6 lỗi categorized với SAI/ĐÚNG side-by-side.

---

## Versioning

- **v3.3** (current) — Single-purpose, lean: 1 mode, 14 files, ~26K tokens, R1-R10 numbering
- **v3.2** — Bidirectional + 4 pattern groups + constraint causal driver
- **v3.1** — interaction_flow + ai_response_strategy_layer + AI response shaping user
- **v3.0** — Adaptive modular schema + safety + causality DAG
- **v2.0** — Recursive causality, pattern catalog không hardcode

Migration: `MIGRATION.md`. Changelog: `CHANGELOG.md`.

## Performance

So với v3.2:
- File count: 27 → 14 (−48%)
- Total content: 186 KB → 100 KB (−46%)
- Estimated load tokens: ~46K → ~26K (−43%)
- Mode variants: 4 depth × 3 schema → 1 mode duy nhất
- Flags: 8 → 3

Lazy-load: SKILL.md có schedule load reference theo workflow step để giảm context footprint.

---

## Triết lý thiết kế

**Personal use, không enterprise.** Tối ưu cho:
- 1 nhiệm vụ duy nhất: extract thought flow
- 1 yêu cầu = 1 skill (không multi-mode)
- Load nhanh + ít context room
- Adapt được lịch sử chat đa dạng (Amazon support / image prompt / coding / fantasy / ...)

**Không** thêm:
- CI/CD pipeline
- Test infrastructure nặng
- Pydantic models / custom validators
- Schema mode if/then enforcement
- Production-grade tooling

**Inspired by** corrections từ chính tác giả trong session tạo skill: "pattern không được hardcode", "phân tích interaction loop chứ không chỉ user-centered", "constraint là causal driver chính".

## Use cases

- Phân tích lịch sử chat ChatGPT/Claude của mình để rút pattern prompting
- Build personal memory/knowledge base về cách mình tương tác AI
- Track AI behavior patterns qua nhiều session (cross-session candidates)
- Reverse-engineer chat của người khác (với consent) để học prompting
- Debug interaction misalignment khi AI hiểu sai intent
- Cross-validate AI response strategy với user expectation

**Không phải tool cho**:
- Tóm tắt nhanh file
- Dịch transcript
- Sửa lỗi syntax
- Phân tích content không phải chat (article, code, doc)

---

## Output language

Mặc định **TIẾNG VIỆT** (kể cả khi transcript EN, trừ khi user yêu cầu EN explicit).

Quote verbatim VN informal preserve (tao/mày/ko/đm/typo). KHÔNG sanitize.

Mixed transcript (VI + EN) → output VN, giữ EN technical terms.

---

## Contributing

Đây là personal skill, nhưng PR / issue welcome nếu:
- Fix concrete bug (numbering, broken reference, schema inconsistency)
- Add bad-example mới với rule cụ thể bị vi phạm
- Improve language rules (forbidden phrases mới)
- Refine causality framework edge relations

**Không accept**:
- Add CI/test infrastructure
- Add Pydantic / TypeScript bindings (out of scope)
- Add mode variants (compact / deep / forensic — đã bỏ ở v3.3)
- Add domain taxonomy hard-coded (custom_fields đã đủ)
- Add features không match "1 task = 1 skill" philosophy

## Credits

Skill v1-v3.3 được build qua nhiều iteration với:
- **Claude Sonnet/Opus** — implementation
- **ChatGPT** — analysis + design review
- **Author**: personal use, không sponsor

Triết lý "interaction loop as primary unit" và "constraint as causal driver" emerge từ chính sessions design skill — meta-recursion thú vị.

## License

MIT (hoặc whatever fits — personal use không sponsor).

---

## File hierarchy quick reference (cho Claude khi load skill)

```
Bước workflow → File load
─────────────────────────
Bước 1 (Safety)        → safety-rules.md
Bước 3 (Format detect) → input-format-detection.md
Bước 6 (Bidirectional) → causality-framework.md
Bước 9 (Pattern)       → pattern-catalog.md
Bước 10 (Self-check)   → evaluator-rubric.md
Output viết MD         → language-rules.md
Schema validate        → schema.md + schema.json
```

LLM không cần load tất cả 14 file cùng lúc. SKILL.md đủ entry point.
