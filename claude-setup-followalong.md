# Claude Code 셋업 - 따라하기 (v8)

> **위에서 아래로 복사-붙여넣기**만 하면 됩니다. 아무것도 안 되어 있다고 가정하고 처음부터 진행합니다.
> 환경: Windows + (VS Code 또는 IntelliJ IDEA) + PowerShell.
> 대상: 세 프로젝트(`sfa-client-main` · `sfa-server` · `sfa-rag`)를 개발하는 누구나.

---

## 0. 준비 확인

```powershell
claude --version   # v2.1.120 이상이면 PowerShell만으로 동작
git --version      # git은 반드시 필요
```

### IDE에 Claude Code 붙이기 (PC당 1회)

- **VS Code**: 통합 터미널(`` Ctrl+` ``)에서 `claude` 입력 → 확장 자동 설치.
- **IntelliJ IDEA**: `Settings > Plugins > Marketplace`에서 **"Claude Code"** 설치 → **IDE 재시작**. `Ctrl+Esc`로 열립니다.

### 시크릿 보안 — 가장 먼저 알아둘 것

진짜 비밀번호·API 키·인증서는 **Claude가 작업하는 폴더에 두지 않습니다.** (`deny` 설정은 Read만 막고 터미널 `cat`은 못 막습니다.) 우리 프로젝트는 시크릿을 별도 위치(`_secure_config` 등)에 둡니다 — 코드/문서에 평문으로 남기지 않습니다.

---

# 【1부】 글로벌 셋업 — PC당 한 번

## STEP 1. `.claude` 폴더 만들기

```powershell
cd C:/Users/<내계정>
New-Item -ItemType Directory -Path .claude/skills/resume -Force
New-Item -ItemType Directory -Path .claude/skills/wrap -Force
```

## STEP 2. 글로벌 행동 규칙 `CLAUDE.md`

```powershell
code C:/Users/<내계정>/.claude/CLAUDE.md
```

아래 내용을 붙여넣습니다. (모든 프로젝트에 공통 적용되는 "Claude의 행동 규칙")

```markdown
# CLAUDE.md (Global)

## 1. Think Before Coding
- 가정은 명시한다. 불확실하면 묻는다.
- 해석이 여러 갈래면 임의로 고르지 말고 제시한다.
- 더 단순한 방법이 있으면 말한다.

## 2. Simplicity First
- 요청한 것 이상으로 만들지 않는다. 1회용 코드에 추상화 금지.

## 3. Surgical Changes
- 주변 코드·주석·포맷을 멋대로 "개선"하지 않는다. 기존 스타일을 따른다.

## 4. Goal-Driven Execution
- 수정 전에 검증 기준을 먼저 정한다. 다단계 작업은 짧은 계획을 말한다.

## 5. Output Style
- 요청 없으면 긴 설명 없이 간결하게. 전체 파일 재출력 금지, 변경 부분만.

## 6. File Reference
- 파일은 `@경로/파일명`으로 참조한다.

## 7. Compaction Priority
- 컨텍스트 압축 시 보존: 수정한 파일 목록 / 대기 중 작업 / 최근 결정 / 미해결 이슈.

## 8. Secret Safety
- .env·인증서·키 파일을 읽거나 cat 하지 않는다. 시크릿은 환경변수로 받는다.
```

## STEP 3. 글로벌 `settings.json` (권한 · 안전벨트)

```powershell
code C:/Users/<내계정>/.claude/settings.json
```

```json
{
  "theme": "dark",
  "autoMemoryEnabled": true,
  "permissions": {
    "deny": [
      "Read(**/.env)", "Read(**/.env.*)",
      "Read(**/*.pem)", "Read(**/*.key)", "Read(**/*.p12)",
      "Bash(cat *.env)", "Bash(cat *.env.*)",
      "Bash(cat *.pem)", "Bash(cat *.key)"
    ]
  },
  "hooks": {
    "PreCompact": [
      { "matcher": "auto", "hooks": [
        { "type": "command", "command": "echo \"[알림] 곧 컨텍스트 압축 - /wrap 권장\" 1>&2", "async": true }
      ]}
    ],
    "SessionStart": [
      { "matcher": "compact", "hooks": [
        { "type": "command", "command": "echo '압축 후 복구: docs/PROGRESS.md 최상단과 PROJECT_PLAN.md를 다시 읽어라.'" }
      ]}
    ]
  }
}
```

## STEP 4. `/resume` 명령 만들기

```powershell
code C:/Users/<내계정>/.claude/skills/resume/SKILL.md
```

```markdown
---
name: resume
description: 세션 시작 시 사용. CLAUDE.md와 docs/PROGRESS.md 최상단, docs/PROJECT_PLAN.md를 읽고 지난 상태와 다음 작업을 확인한다.
---

# /resume — 세션 재개
CLAUDE.md, docs/PROGRESS.md(최상단), docs/PROJECT_PLAN.md를 읽고
"지난 작업은 X, 다음은 Y로 진행할까요?" 형태로 보고하라.
```

## STEP 5. `/wrap` 명령 만들기

```powershell
code C:/Users/<내계정>/.claude/skills/wrap/SKILL.md
```

```markdown
---
name: wrap
description: 세션 종료 시 사용. docs/PROGRESS.md 최상단에 오늘 작업을 append하고, 새 결정은 docs/DECISIONS.md에 추가하며, 변경 파일을 보고한다.
---

# /wrap — 세션 마무리
오늘 작업을 docs/PROGRESS.md 최상단에 append(작업/결정/다음/미해결),
새 결정은 docs/DECISIONS.md에 추가, 마지막에 변경 파일 목록을 보고하라.
```

## STEP 6. 동작 확인

프로젝트 폴더에서 `claude` 실행 → `/` 입력 → `/resume`·`/wrap`이 자동완성되면 성공.
(안 뜨면 IDE 재시작 / 폴더·파일명 / `name:` 확인)

**👉 글로벌 셋업 끝. 아래는 프로젝트마다 합니다.**

---

# 【2부】 프로젝트 셋업 — 세 프로젝트 각각 한 번

> ⚠️ 세 프로젝트는 **각각 독립된 git**입니다. 아래를 **프로젝트마다 한 번씩** 합니다.

## STEP 1. 프로젝트 `CLAUDE.md`

그 프로젝트의 사실(기술 스택·DB·규칙·건드리면 안 되는 영역)을 담습니다. 없으면 Claude에게:

```
이 프로젝트 코드베이스를 분석해서 루트에 CLAUDE.md를 만들어줘.
개요/기술 스택/실행 환경/아키텍처/DB/코딩 컨벤션/제약사항 포함. 200줄 넘기지 마.
행동 규칙은 글로벌에 있으니 넣지 마.
```

### 이어서 — 팀 작업 규칙을 `CLAUDE.md`에 박아둔다 ⭐ (자동화 핵심)

아래 블록을 프로젝트 `CLAUDE.md` **끝에 추가**합니다. **이렇게 규칙을 적어두면, 사람이 매번 기억하지 않아도 Claude가 작업·기록 때 알아서 따릅니다.**

```markdown
## 문서·기록 규칙 (Claude가 자동 적용)

### 작업 기록
- `/wrap` 시 `docs/PROGRESS.md` 최상단에 **`[모듈]` 태그**를 붙여 기록한다.
- `PROGRESS`·`DECISIONS`·`PROJECT_PLAN`은 이 프로젝트에 **한 세트만** 둔다(모듈별 파일 분리 금지).
  `PROJECT_PLAN`은 한 파일 안에서 모듈별 섹션으로 구분한다.

### 기록 길이 관리
- `/wrap` 시 `docs/PROGRESS.md`가 **약 800줄(또는 분기 경계)** 을 넘으면, 가장 오래된 분기를
  `docs/archive/PROGRESS-<기간>.md`로 옮기고 활성 파일 맨 아래에 **포인터 한 줄**을 남긴다.
  (최신 항목은 항상 활성 파일에 유지 — `/resume`는 최상단만 읽으므로 길이 부담이 줄어든다)
- `DECISIONS.md`는 번호·`[[DEC-NNN]]` 참조가 깨지지 않게, 본문을 아카이브로 옮길 때 **상단 색인(번호·제목)** 을 유지한다.

### 주제·참고 문서 배치
- 새 주제 문서는 `docs/` 바로 아래 **평면**으로 만든다(파일명 앞에 모듈명 접두).
- 한 모듈의 주제 문서가 **3개 이상**이면 `docs/<모듈>/` 폴더로 옮긴다. 빈 폴더는 미리 만들지 않는다.
- `docs/README.md` 색인을 갱신한다.

### 크로스-프로젝트 변경
- 이 작업이 다른 프로젝트(프론트/백엔드/RAG)에도 영향을 주면:
  - 이 프로젝트 `docs/`에 **자세히** 기록하고,
  - 다른 프로젝트가 현재 **접근 가능하면**(cwd이거나 `.claude/settings.json`의 `additionalDirectories`에 포함) 그 `docs/PROGRESS.md`에 **`[공통]` 교차 한 줄**을 남긴다.
  - **접근 불가하면**(그 프로젝트를 열지 않았으면), 사용자에게 "○○ 프로젝트 docs에도 기록이 필요하다"고 **알린다**.

### 공유 vs 개인 / 시크릿
- 공유(커밋): `CLAUDE.md`·`.claude/skills`·`.claude/settings.json`·`docs/`.
- 개인(커밋 금지): `.claude/settings.local.json`·`CLAUDE.local.md`.
- 시크릿(`.env`·키·비밀번호)은 읽지도 커밋하지도 않는다.
```

> 💡 이 블록이 곧 **"사람이 직접 하던 일을 Claude의 자동 규칙으로 바꾼 것"** 입니다. 단, CLAUDE.md는 강한 지침이지 100% 보장은 아니므로 `/wrap` 결과를 한 번 훑어보세요. 아래 【3부】는 이 규칙들의 배경 설명입니다.

## STEP 2. `docs/` 3종 만들기

```
docs/ 구조를 만들어줘:
- docs/PROJECT_PLAN.md  (목표·범위·제약·Phase 체크박스)
- docs/PROGRESS.md      (세션 종료 시 최신을 맨 위에 append: 작업/결정/다음/미해결)
- docs/DECISIONS.md     (DEC-NNN: 제목·날짜·맥락·결정·대안)
- docs/README.md        (어떤 문서가 어디 있는지 색인)
```

## STEP 3. `.gitignore` — 개인 파일 제외

프로젝트 루트 `.gitignore`에 추가합니다. (개인 구성은 공유하지 않습니다)

```gitignore
# Claude 개인 구성 — 커밋하지 않음
.claude/settings.local.json
CLAUDE.local.md
```

## STEP 4. `.claude/settings.json` — 자주 함께 고치는 프로젝트 연결 ⭐ (크로스-프로젝트 자동화)

평소 **같이 수정하는 프로젝트**를 `additionalDirectories`에 적어두면, 이 프로젝트에서 Claude를 열어도 **그 프로젝트까지 읽고/기록**할 수 있습니다 → 크로스-프로젝트 기록이 **사람 손 없이 자동**으로 됩니다.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "additionalDirectories": ["../sfa-server"]
  }
}
```

- 예시는 **프론트(`sfa-client-main`)에서 백엔드(`sfa-server`)를 함께** 다루는 경우. 경로는 상대경로(`../<프로젝트>`) — 세 프로젝트가 **한 상위 폴더 아래 나란히** 있어야 동작합니다.
- 자주 함께 고치는 쌍을 넣으세요(프론트↔백엔드 등). 안 넣은 프로젝트는 접근 불가라, 그때는 Claude가 "기록 필요"만 알려줍니다(【3부】).
- (선택) 권한 프롬프트를 줄이려면 자주 쓰는 명령을 `permissions.allow`에(예: `Bash(git status:*)`·`Bash(./gradlew:*)`·`Bash(npm run:*)`), 파괴적 명령은 `ask`에(예: `Bash(git reset --hard:*)`) 둘 수 있습니다.

## STEP 5. 공유 구성 커밋·푸시

```powershell
git add CLAUDE.md .claude docs .gitignore
git commit -m "chore: 공통 | claude - Claude Code 구성 추가"
git push
```

> 💡 **커밋 메시지는 팀 표준 양식**: `<타입>: <스코프> - <설명>`
> 타입은 풀워드 `feature`·`fix`·`hotfix`·`refactor`·`chore` (Conventional Commits `feat(...)` **금지**), 스코프는 `모듈 | 구분`(단일 모듈이면 모듈명만). 예: `feature: rag | 색인관리 - 다음 실행시각 KST 변환`.

✅ 이제 동료가 `git pull` 하면 같은 구성·기록을 공유합니다.

---

# 【3부】 배경 설명 — 위 규칙은 왜 이렇게 정했나

> 아래 내용은 STEP 1의 **"문서·기록 규칙" 블록**으로 이미 자동화됩니다. 여기서는 그 배경과, **Claude 자동 / 사람 몫**의 경계만 정리합니다.

| 일 | 누가 |
|----|------|
| `[모듈]` 태그·기록 3종 1세트·주제문서 평면→폴더 승격·공유/개인 경계 | 🤖 Claude 자동(CLAUDE.md 규칙) |
| 크로스-프로젝트 기록 — 다른 프로젝트가 열려 있어 접근 가능할 때 | 🤖 Claude 자동 |
| 크로스-프로젝트 기록 — 다른 프로젝트를 안 열었을 때 | 🙋 Claude가 알려주면 사람이 그 프로젝트에서 기록 |
| 글로벌·IDE 1회 셋업, `git commit`/`push` 트리거·승인 | 🙋 사람 |

우리는 한 작업이 *백엔드+프론트*를 같이 고치거나 *RAG만* 고치기도 합니다. 핵심 두 가지의 배경:

## (1) 공유할 것 vs 내 PC에만 둘 것

| 종류 | 예시 | git |
|------|------|-----|
| 프로젝트 설명 | `CLAUDE.md` | 🟢 커밋 |
| 작업 명령 | `.claude/skills`, `.claude/settings.json` | 🟢 커밋 |
| 작업 기록 | `docs/PROGRESS.md` 등 | 🟢 커밋 |
| 개인 구성 | `settings.local.json`, `CLAUDE.local.md` | 🔴 커밋 안 함 (위 `.gitignore`) |

## (2) 크로스-프로젝트 기록 — **각 프로젝트에 남긴다**

> 한 작업이 여러 프로젝트를 고치면, **건드린 프로젝트마다** 기록을 남긴다.
> 자세한 본문은 **주로 작업한 프로젝트** 한 곳에, 다른 프로젝트엔 **짧은 한 줄 + 가리키는 링크**.

예) RAG에서 작업했고 백엔드도 고쳤다면:

- `sfa-rag/docs/PROGRESS.md` ← 자세한 본문
- `sfa-server/docs/PROGRESS.md` ← `"[공통] RAG 색인 연동으로 함께 변경됨 → sfa-rag PROGRESS 참고"` 한 줄

이러면 어느 프로젝트 동료든 **자기 프로젝트 기록만 봐도** 맥락을 따라갈 수 있습니다. 커밋은 **프로젝트마다 따로** 합니다.

## (3) docs 폴더 규칙

- **작업 기록 3종**(`PROGRESS`·`DECISIONS`·`PROJECT_PLAN`): 프로젝트당 **한 세트**. 모듈은 항목 앞 **`[모듈]` 태그**로 구분 (모듈별 파일 분리 금지).
- **주제·참고 문서**: 평면(`docs/*.md`) 시작 → 한 모듈 문서가 **3~5개 이상** 쌓이면 `docs/<모듈>/`로 승격. 빈 폴더 미리 만들지 않기.
- `docs/README.md` **색인 유지**.

```text
docs/
├── PROGRESS.md          # 한 세트 — 모듈은 [library] 태그
├── DECISIONS.md
├── PROJECT_PLAN.md      # 한 파일 안에 모듈별 섹션
├── README.md            # 색인
├── library-카드정렬.md      # 적을 땐 평면 + 이름 접두
└── work-support/            # 3개 이상 쌓이면 승격
    ├── 게시판-규칙.md
    └── 카테고리-트리.md
```

## (선택) repomix — 크로스-프로젝트 통신 규약 한 파일로

여러 프로젝트를 함께 다룰 때, 각 프로젝트의 **DTO·Controller·Config**만 추려 한 파일(`docs/ast-context.xml`)로 묶어두면 Claude가 크로스-프로젝트 API/통신 규약을 빠르게 참조합니다.

```text
/plugin marketplace add yamadashy/repomix
/plugin install repomix-mcp@repomix
```

- 루트에 `repomix.config.json`(패킹 대상 glob + `output.filePath: docs/ast-context.xml`)을 둡니다.
- `ast-context.xml`은 **자동 생성물**이라 직접 편집하지 않습니다. 통신 규약(DTO/Controller)이 바뀌면 다시 패킹해 갱신합니다.

---

## 셋업 완료 체크

- [ ] (PC당 1회) 글로벌 — `.claude` 폴더 / `CLAUDE.md` / `settings.json` / `/resume`·`/wrap`
- [ ] `/` → `/resume`·`/wrap` 자동완성 확인
- [ ] (프로젝트마다) `CLAUDE.md` + `docs/` 3종 + `.gitignore`
- [ ] **(프로젝트마다) `CLAUDE.md`에 "문서·기록 규칙" 블록 추가** ⭐ — 이게 있어야 Claude가 자동 적용
- [ ] (선택) `.claude/settings.json`에 `additionalDirectories`로 자주 함께 고치는 프로젝트 연결 — 크로스-프로젝트 자동화
- [ ] 세 프로젝트 공유 구성 커밋·푸시 (팀 표준 커밋 양식)
- [ ] 크로스-프로젝트는 **건드린 프로젝트마다 기록** 약속 합의
- [ ] 기록 3종 1세트 + `[모듈]` 태그 / 주제 문서 평면→승격 / `README.md` 색인

> 매일 쓰는 법은 **「Claude Code 매일 운영 루틴 (v8)」** 참고.
