# AI 활용 / Claude Code 학습 노트

> 시작: 2026-07-09
> 방식: AI와 대화하며 배운 것을 실시간 기록. 새 내용은 항상 끝에 추가한다.

---

## 1. 왜 시작했나

Andrej Karpathy의 ["전문가가 되는 법"](https://x.com/karpathy/status/1325154823856033793) 세 가지를 따르기 위해서.

1. 구체적인 프로젝트를 반복해서 맡아 깊이(depth-wise), 끝까지 해낸다 — 필요한 것을 그때그때(on demand) 배우고, 기초부터 넓게(bottom-up breadth-wise) 배우지 않는다
2. 배운 것을 전부 내 언어로, 남을 가르치듯(teach) 정리한다
3. 과거의 나와만 비교하고, 남과는 비교하지 않는다

**AI 활용과 Claude Code**를 주제로, 세 단계로 돈다:
실전에서 걸린 문제를 파고들어 직접 확인하고(1) → 이 파일에 노트로 적립한 뒤 쌓이면 블로그에 가르치듯 발행하고(2) → 정리 세션마다 "예전의 나는 뭘 몰랐나"를 회고한다(3).

---

## 2. 깨달은 것 (누적 요약)

> 노트가 쌓이면 여기에 핵심만 3~5줄로 계속 갱신한다.

- AI에게 일을 시킬 때는 "결과"가 아니라 **"확인 가능한 상태"**를 요구해야 한다.
- 자동화가 실패하는 지점은 대부분 코드가 아니라 **인증·권한·환경(PATH)** 이다.

---

## 3. 질문 목록

> 세션마다 던진 질문을 번호로 쌓는다. 상세는 아래 노트 번호와 매칭.

| # | 질문 | 노트 |
|---|------|------|
| 1 | GitHub 토큰은 어디에 저장돼 있고, 어떤 게 진짜로 쓰이나? | 노트 1 |
| 2 | gh CLI 로그인이 토큰으로 안 될 때 무엇이 문제인가? | 노트 2 |
| 3 | 커스텀 스킬은 어떻게 만들고, 언제 어떻게 발동되나? | 노트 3 |

---

## 4. 노트

### 노트 1 — git 인증 정보는 세 군데에 흩어질 수 있다 (2026-07-09)

git 계정을 확인해 보니 인증 정보가 세 곳에 있었다.

1. `git config`의 `user.password` — **잘못된 위치**. git은 이 값을 쓰지 않고, 평문 노출만 된다. (만료된 토큰이었고 삭제함)
2. `~/.git-credentials` — `credential.helper=store`일 때 실제로 쓰이는 곳. `https://<계정>:<토큰>@github.com` 형식.
3. `gh` CLI 자체 로그인 — git과 별개로 자기만의 인증을 가진다.

⚠️ `git push`가 되는 것과 `gh`가 되는 것은 **별개의 인증**이다.

확인 방법:

```bash
git config --global --list          # user.* 와 credential.helper 확인
cat ~/.git-credentials              # 실제 저장된 계정 확인 (토큰 노출 주의)
gh auth status                      # gh 로그인 상태
```

토큰이 살아있는지, 어느 계정인지는 API로 바로 확인할 수 있다:

```bash
curl -s -H "Authorization: token <토큰>" https://api.github.com/user
# → "login" 필드가 계정명. 만료면 "Bad credentials"
```

### 노트 2 — gh 로그인은 토큰 스코프가 발목을 잡는다 (2026-07-09)

`gh auth login --with-token`은 토큰에 `read:org` 스코프가 없으면 거부한다.
하지만 **REST API 직접 호출은 그 스코프 없이도 된다.**

```bash
# gh 없이 레포 생성
curl -X POST -H "Authorization: token <토큰>" \
  https://api.github.com/user/repos \
  -d '{"name":"my-repo","private":false}'
```

⚠️ "gh가 안 됨" ≠ "토큰이 죽음". 스코프 문제일 수 있으니 API로 토큰 자체를 먼저 검증할 것.

### 노트 3 — 스킬은 SKILL.md 파일 하나로 시작하고, description이 곧 트리거다 (2026-07-09)

학습 노트를 자동으로 적립하는 `/study-log` 스킬을 직접 만들면서 확인한 것.

스킬의 실체는 마크다운 파일 하나다:

```
~/.claude/skills/study-log/SKILL.md    # 개인(전역) 스킬 — 모든 프로젝트에서 사용
<프로젝트>/.claude/skills/<이름>/       # 프로젝트 스킬 — 해당 저장소에서만
플러그인 제공 스킬                       # plugin:skill 형태로 네임스페이스가 붙음
```

구조는 frontmatter + 본문 절차서:

```markdown
---
name: study-log
description: ...학습 노트에 추가하고 커밋한다. "노트에 추가해줘"라고 하면 사용...
---
# 절차
1. 파일 읽고 마지막 노트 번호 확인
2. ...
```

핵심 깨달음 두 가지:

1. **description이 코드가 아니라 트리거다.** 모델이 사용자의 말과 description을 대조해서 스스로 호출을 결정한다. 그래서 description에는 기능 설명보다 **"사용자가 어떤 말을 하면 써라"**를 적어야 잘 발동된다.
2. **본문은 그냥 지시문이다.** 프로그램이 아니라 모델이 읽고 따르는 절차서라서, 사람에게 인수인계 문서 쓰듯 금지사항("append-only, 과거 노트 수정 금지")까지 적어두면 그대로 지켜진다.

⚠️ "새 스킬은 세션을 다시 시작해야 뜬다"고 예상했는데 **틀렸다.** SKILL.md를 저장하자마자 같은 세션의 스킬 목록에 바로 나타났다 — 스킬 목록은 세션 중에도 갱신된다. 예상과 실제가 다를 수 있으니 항상 직접 확인할 것.

---

## 5. 한눈에 보는 큰 그림

```
git push ──> ~/.git-credentials (credential.helper=store)
gh CLI   ──> 자체 로그인 (scope: repo + read:org 필요)
REST API ──> 토큰만 있으면 됨 (scope는 엔드포인트별)
```

---

## 6. 다음 파볼 주제

- [x] Claude Code의 skill 구조 (노트 3)
- [ ] agent(서브에이전트) 구조 — 컨텍스트가 어떻게 격리되나
- [ ] hook 구조 — 어떤 시점에 무엇이 실행되나
- [ ] CLAUDE.md와 메모리가 실제로 언제 로드되는지
- [ ] MCP 서버 연결 구조
- [ ] gh 토큰을 fine-grained token으로 재발급해서 스코프 정리
