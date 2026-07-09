# AI 활용 / Claude Code 학습 노트

> 시작: 2026-07-09
> 방식: AI와 대화하며 배운 것을 실시간 기록. 새 내용은 항상 끝에 추가한다.

---

## 1. 왜 시작했나

Andrej Karpathy의 "전문가가 되는 법" 세 가지를 따르기 위해서.

1. 프로젝트를 깊이 있게 끝까지 한다
2. 배운 것을 자기 언어로 정리한다
3. 과거의 나와만 비교한다

**AI 활용과 Claude Code**를 주제로, 배운 것을 그때그때 이 파일에 쌓아간다.

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

---

## 5. 한눈에 보는 큰 그림

```
git push ──> ~/.git-credentials (credential.helper=store)
gh CLI   ──> 자체 로그인 (scope: repo + read:org 필요)
REST API ──> 토큰만 있으면 됨 (scope는 엔드포인트별)
```

---

## 6. 다음 파볼 주제

- [ ] Claude Code의 skill / agent / hook 구조
- [ ] CLAUDE.md와 메모리가 실제로 언제 로드되는지
- [ ] MCP 서버 연결 구조
- [ ] gh 토큰을 fine-grained token으로 재발급해서 스코프 정리
