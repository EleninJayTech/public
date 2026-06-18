# Claude Code 셋업 - 따라하기 (통합안 / v8-unified)

> **두 팀 통합안**(`claude-structure-unified-proposal`) 기반 셋업입니다. 근거·비교는 그 문서를 보세요.
> 기존 `claude-setup-followalong.md`(방법 A 단일)와 달리, **동시성 기준 분할 + 솔루션-aware skill**을 포함합니다.
> 환경: Windows + (VS Code 또는 IntelliJ IDEA) + PowerShell. 대상: 세 프로젝트(`sfa-client-main`·`sfa-server`·`sfa-rag`) 개발자 누구나.

---

## 0. 준비 확인

```powershell
claude --version   # v2.1.120 이상이면 PowerShell만으로 동작
git --version
```

- **IDE**: VS Code 터미널에서 `claude`(확장 자동설치) / IntelliJ는 Plugins에서 "Claude Code" 설치 후 재시작.
- **시크릿**: 진짜 비밀번호·키는 작업 폴더에 두지 않는다(`deny`는 Read만 막고 `cat`은 못 막음). `_secure_config` 등 별도 위치.

---

# 【1부】 글로벌 셋업 — PC당 한 번

## STEP 1. `.claude` 폴더

```powershell
cd C:/Users/<내계정>
New-Item -ItemType Directory -Path .claude/skills/resume -Force
New-Item -ItemType Directory -Path .claude/skills/wrap -Force
```

## STEP 2. 글로벌 `CLAUDE.md` (행동 규칙)

```powershell
code C:/Users/<내계정>/.claude/CLAUDE.md
```

```markdown
# CLAUDE.md (Global)
## 1. Think Before Coding — 가정 명시, 불확실하면 질문, 해석 갈리면 제시.
## 2. Simplicity First — 요청 이상 금지, 1회용 코드 추상화 금지.
## 3. Surgical Changes — 범위 외 코드 손대지 않기, 기존 스타일 따르기.
## 4. Goal-Driven Execution — 수정 전 검증 기준 먼저, 다단계는 짧은 계획.
## 5. Output Style — 간결하게, 전체 파일 재출력 금지(변경분만).
## 6. File Reference — 파일은 @경로/파일명으로 참조.
## 7. Compaction Priority — 압축 시 보존: 수정 파일·대기 작업·최근 결정·미해결.
## 8. Secret Safety — .env·키·인증서 읽거나 cat 금지, 시크릿은 환경변수로.
```

## STEP 3. 글로벌 `settings.json`

```powershell
code C:/Users/<내계정>/.claude/settings.json
```

```json
{
  "theme": "dark",
  "autoMemoryEnabled": true,
  "permissions": {
    "deny": [
      "Read(**/.env)", "Read(**/.env.*)", "Read(**/*.pem)", "Read(**/*.key)", "Read(**/*.p12)",
      "Bash(cat *.env)", "Bash(cat *.env.*)", "Bash(cat *.pem)", "Bash(cat *.key)"
    ]
  },
  "hooks": {
    "PreCompact": [ { "matcher": "auto", "hooks": [
      { "type": "command", "command": "echo \"[알림] 곧 컨텍스트 압축 - /wrap 권장\" 1>&2", "async": true } ] } ],
    "SessionStart": [ { "matcher": "compact", "hooks": [
      { "type": "command", "command": "echo '압축 후 복구: docs/PROGRESS.md 최상단과 PROJECT_PLAN.md를 다시 읽어라.'" } ] } ]
  }
}
```

## STEP 4~5. `/resume`·`/wrap` (글로벌 기본형)

```powershell
code C:/Users/<내계정>/.claude/skills/resume/SKILL.md
code C:/Users/<내계정>/.claude/skills/wrap/SKILL.md
```

```markdown
# resume/SKILL.md
---
name: resume
description: 세션 시작 시 CLAUDE.md·docs/PROGRESS.md 최상단·docs/PROJECT_PLAN.md를 읽고 지난 상태·다음 작업을 보고.
---
# /resume — CLAUDE.md, docs/PROGRESS.md(최상단), docs/PROJECT_PLAN.md를 읽고 "지난 X, 다음 Y?" 보고.
```

```markdown
# wrap/SKILL.md
---
name: wrap
description: 세션 종료 시 docs/PROGRESS.md 최상단에 오늘 작업 append, 새 결정은 DECISIONS.md에 추가, 변경 파일 보고.
---
# /wrap — 오늘 작업을 docs/PROGRESS.md 최상단에 append(작업/결정/다음/미해결), 새 결정 DECISIONS.md, 변경 파일 보고.
```

> 💡 **고동시성 모노레포(sfa-client-main)는 이 글로벌 skill을 repo 안에서 "솔루션-aware"로 덮어씁니다 → 【3부】.**

## STEP 6. 동작 확인
`claude` 실행 → `/` → `/resume`·`/wrap` 자동완성되면 성공.

---

# 【2부】 프로젝트 셋업 — 세 프로젝트 각각 한 번

## STEP 1. 프로젝트 `CLAUDE.md` + 통합 규칙 블록 ⭐

프로젝트 사실(스택·DB·제약)을 담고(없으면 Claude에게 분석 요청), **끝에 아래 통합 규칙 블록을 추가**합니다. 이게 Claude 자동 적용의 핵심입니다.

```markdown
## 문서·기록 규칙 (Claude가 자동 적용)

### 작업 기록 구조 (동시성 기준)
- 저동시성 repo: PROGRESS·DECISIONS·PROJECT_PLAN을 한 세트로 두고 항목 앞에 [모듈] 태그.
- 고동시성 모노레포(이 repo가 해당이면): PROGRESS·DECISIONS는 docs/<app>/ 로 분리,
  마스터 PROJECT_PLAN.md만 docs/ 루트에 공유. /resume·/wrap은 대상 앱을 PWD>브랜치>질문 순으로 판별.
- 기록 항목엔 상태 [Done]/[Pending]/[Blocked]를 병기한다.

### 여러 대상 동시 변경 (방법 A)
- 주로 작업한 대상(repo 또는 app) docs에 본문 기록, 함께 바뀐 대상엔 [공통] 교차 한 줄 + 링크.
- 다른 repo는 접근 가능(cwd 또는 .claude/settings.json의 additionalDirectories)하면 자동 기록,
  불가하면 사용자에게 "○○에도 기록 필요"를 알린다.

### 주제·참고 문서 배치
- 평면(docs/*.md, 이름 접두)으로 시작 → 한 모듈/앱 주제 문서 3개+면 docs/<모듈>/ 승격. 빈 폴더 금지. README 색인.

### 길이 관리
- PROGRESS가 약 800줄/분기 경계를 넘으면 가장 오래된 분기를 docs/archive/PROGRESS-<기간>.md로 옮기고
  활성 파일 맨 아래 포인터 한 줄. DECISIONS는 [[DEC-NNN]] 참조 보호 위해 상단 색인 유지 후 본문만 이동.

### 공유 vs 개인 / 시크릿
- 공유(커밋): CLAUDE.md·.claude/skills·.claude/settings.json·docs/.
- 개인(커밋 금지): .claude/settings.local.json·CLAUDE.local.md.
- 시크릿(.env·키·비밀번호)은 읽지도 커밋하지도 않는다.

### 토큰 참고
- docs/는 자동 선로딩되지 않고 필요 시만 읽힌다. 구조는 토큰이 아니라 충돌 차단·타겟 정확도 기준으로 고른다.
```

## STEP 2. `docs/` 구조 — **동시성 기준**으로 결정

| repo | docs 구조 |
|------|-----------|
| **저동시성**(`sfa-server`·`sfa-rag`) | `docs/PROGRESS.md`·`DECISIONS.md`·`PROJECT_PLAN.md` **한 세트** + `[모듈]` 태그 |
| **고동시성 모노레포**(`sfa-client-main`) | 루트 `docs/PROJECT_PLAN.md`(공유) + **앱별** `docs/<app>/PROGRESS.md`·`DECISIONS.md` |

고동시성(③안) 예시:

```text
sfa-client-main/docs/
├── PROJECT_PLAN.md        # [공통] 마스터 + 활성 앱 인덱스
├── README.md
├── work-support/          # 📂 앱별 (충돌 0, 담당자만 append)
│   ├── PROGRESS.md
│   ├── DECISIONS.md
│   └── 게시판-규칙.md
└── approval/
    ├── PROGRESS.md
    └── 전자결제-라인-컨벤션.md
```

## STEP 3. `.gitignore` — 개인 파일 제외

```gitignore
.claude/settings.local.json
CLAUDE.local.md
```

## STEP 4. `.claude/settings.json` — 크로스-repo 연결

자주 함께 고치는 repo를 연결하면 교차기록이 자동화됩니다.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": { "additionalDirectories": ["../sfa-server"] }
}
```

## STEP 5. 공유 구성 커밋·푸시 (팀 표준 양식)

```powershell
git add CLAUDE.md .claude docs .gitignore
git commit -m "chore: 공통 | claude - Claude Code 통합 구성 추가"
git push
```

> 커밋 양식: `<타입>: <스코프> - <설명>` (타입 풀워드 `feature`/`fix`/`hotfix`/`refactor`/`chore`, Conventional Commits 금지).

---

# 【3부】 고동시성 모노레포 — 솔루션-aware skill (분할이 동작하는 핵심)

`sfa-client-main`처럼 앱별로 docs를 쪼갠 repo는 **repo 안 `.claude/skills/`** 에 솔루션-aware skill을 둬서 글로벌 기본형을 덮어씁니다.

```markdown
# sfa-client-main/.claude/skills/resume/SKILL.md
---
name: resume
description: 모노레포 솔루션-aware 재개. 대상 앱을 PWD>브랜치>질문 순으로 판별해 docs/<app>/PROGRESS.md를 읽는다.
---
# /resume (솔루션-aware)
대상 앱 판별: 1) PWD가 apps/<app> 안 → 그 app  2) 브랜치명(feature/<app>)  3) 불명확하면 "작업할 앱 선택" 질문.
판별 후 docs/<app>/PROGRESS.md 최상단 + docs/PROJECT_PLAN.md를 읽고 "이 앱 지난 X, 다음 Y?" 보고.
```

```markdown
# sfa-client-main/.claude/skills/wrap/SKILL.md
---
name: wrap
description: 모노레포 솔루션-aware 마무리. 대상 앱 docs/<app>/PROGRESS.md에 [상태] 포함 기록, 앱간 변경은 [공통] 교차 한 줄.
---
# /wrap (솔루션-aware)
대상 앱 판별 후 docs/<app>/PROGRESS.md 최상단에 [Done]/[Pending]/[Blocked] 병기해 기록.
다른 앱도 바뀌었으면 그 앱 docs/<app>/PROGRESS.md에 [공통] 교차 한 줄. 완료 모호 시 확인 후 기록.
```

> 저동시성(`sfa-server`·`sfa-rag`)은 repo 전용 skill 없이 **글로벌 기본형** 그대로 씁니다.

---

# 【4부】 여러 대상 동시 작업 = 방법 A (두 층 공통)

- **repo 간**: 다른 repo가 `additionalDirectories`로 접근 가능하면 Claude가 자동 교차기록, 아니면 "기록 필요" 알림.
- **앱 간**(모노레포 내): 다른 앱 `docs/<app>/PROGRESS.md`에 `[공통] … → work-support PROGRESS 참고` 한 줄.

> (선택) **repomix** — 3 repo 통신 규약(DTO/Controller)을 `docs/ast-context.xml`로 패킹해 크로스-repo 참조. `/plugin install repomix-mcp@repomix`.

---

## 셋업 완료 체크

- [ ] (PC당 1회) 글로벌 — `.claude`·`CLAUDE.md`·`settings.json`·`/resume`·`/wrap`
- [ ] (프로젝트마다) `CLAUDE.md` + **통합 규칙 블록** 추가 ⭐
- [ ] `docs/` 구조를 **동시성 기준**으로 — 저동시성=한 세트+태그 / 고동시성=앱별 폴더(③안)
- [ ] 고동시성 모노레포: `.claude/skills/`에 **솔루션-aware** resume·wrap
- [ ] `.gitignore` 개인 파일 제외 / `.claude/settings.json` `additionalDirectories`
- [ ] 공유 구성 커밋·푸시 (팀 표준 양식)

> 매일 쓰는 법은 **「Claude Code 매일 운영 루틴 (통합안 / v8-unified)」**.
