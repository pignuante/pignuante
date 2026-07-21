# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 이 레포의 정체

`pignuante/pignuante` — GitHub **프로필 README 레포다.** `README.md`가 <https://github.com/pignuante> 최상단에 그대로 렌더된다. 애플리케이션 코드·빌드·린트·테스트가 없고, 유일한 "빌드"는 GitHub Actions가 생성하는 위젯 SVG다.

**master 푸시 = 공개 프로필 즉시 반영이다.** 깨진 이미지, 죽은 링크, 근거 없는 경력 주장이 바로 노출된다. 커밋 전에 아래 검증을 돌린다.

## 검증

### 1. 렌더 미리보기 (GitHub과 동일한 마크업)

README는 거의 전부 raw HTML이라 일반 마크다운 뷰어로는 실제 모습을 알 수 없다. GitHub 렌더러를 직접 쓴다:

```bash
gh api -X POST /markdown -f mode=gfm -f context=pignuante/pignuante \
  -f text="$(cat README.md)" > /tmp/rendered.html
```

### 2. 모든 URL + 배지 로고 검사 (필수)

가장 중요한 검사다. **shields.io는 로고 슬러그가 존재하지 않아도 HTTP 200을 반환한다** — 로고만 조용히 빠진 배지가 만들어진다. 실제로 simple-icons가 상표 정책으로 `openai`, `amazonwebservices`, `microsoftazure`를 제거해 기존 배지 3개의 로고가 안 나오고 있었다. 200만 보면 못 잡는다:

```bash
python3 - <<'PY'
import re, subprocess
live = re.sub(r'<!--.*?-->', '', open('README.md').read(), flags=re.S)  # 주석 블록 제외
urls = list(dict.fromkeys(re.findall(r'(?:src|srcset|href)="([^"]+)"', live)))
for u in urls:
    if u.startswith('mailto:'): continue
    r = subprocess.run(['curl','-s','-w','\n%{http_code}','-m','30','-L',u], capture_output=True, text=True)
    body, _, code = r.stdout.rpartition('\n')
    logo_missing = 'img.shields.io/badge' in u and 'logo=' in u and 'data:image' not in body
    if code != '200' or logo_missing:
        print(f"{code} {'LOGO-MISSING' if logo_missing else ''}  {u}")
PY
```

알려진 정상 예외:
- `solved.ac` → **403** (Cloudflare가 curl 차단, 브라우저는 정상)
- `streak-stats.demolab.com` → 콜드 스타트가 25초를 넘겨 타임아웃할 수 있다. 재시도하면 200.

### 3. 라이브 프로필 최종 확인 (푸시 후)

서드파티 이미지는 GitHub camo 프록시를 거치고, `raw.githubusercontent.com` 에셋은 GitHub 소유 도메인이라 프록시를 안 거친다. camo URL은 `.../<hash>/<hex>` 형태로 **hex 경로까지 포함해야** 한다(해시만 자르면 전부 404가 나와 오진한다):

```bash
curl -s -L -H "User-Agent: Mozilla/5.0" https://github.com/pignuante > /tmp/profile.html
grep -o 'https://camo.githubusercontent.com/[a-z0-9]*/[0-9a-f]*' /tmp/profile.html | sort -u | \
  while read -r u; do echo "$(curl -s -o /dev/null -w '%{http_code}' -L "$u")  $u"; done
```

## 위젯 아키텍처

핵심 설계 원칙: **서드파티 공개 인스턴스에 얹지 말고 자기 Actions에서 생성한다.** 이 레포는 실제로 당해봤다 — `github-readme-stats.vercel.app`은 503 `DEPLOYMENT_PAUSED`, `github-profile-trophy.vercel.app`은 402 `DEPLOYMENT_DISABLED`, streak의 heroku 호스트는 무료 dyno 종료로 사망. 레포 자체는 멀쩡한데 남의 무료 배포가 죽은 것이다.

| 워크플로우 | 트리거 | 산출물 | 주의 |
|---|---|---|---|
| `snake.yml` | master push + 매일 + 수동 | `output` 브랜치의 `github-snake{,-dark}.svg` | 푸시 시 자동 자가 치유 |
| `profile-3d.yml` | 매일 + **수동만** | master의 `profile-3d-contrib/*.svg` 10개 | push 트리거를 추가하지 말 것 |
| `metrics.yml` | **수동만** | master의 `github-metrics.svg` | `METRICS_TOKEN` 필요 |

지켜야 할 제약:

- **`profile-3d.yml`에 push 트리거를 붙이면 무한 루프다.** 이 워크플로우는 master에 SVG를 커밋하고, 그 커밋이 다시 자신을 트리거한다. `snake.yml`은 `paths-ignore`로 생성물 커밋을 걸러 루프를 막고 있으니 그 목록을 지우지 말 것.
- **`metrics.yml`은 PAT이 필요하다.** `secrets.GITHUB_TOKEN`으로는 안 된다(유저 단위 데이터 접근 불가). 시크릿이 없는 상태에서 schedule을 켜면 매일 실패 알림이 쌓이므로 `workflow_dispatch`만 남겨뒀다. 시크릿 등록 후에야 schedule과 README의 주석 처리된 `<img>` 블록을 되살린다.
- `profile-3d-contrib/*.svg`와 `output` 브랜치는 **생성물이다.** 직접 편집하지 말 것.
- 워크플로우가 green이어도 이미지가 뜨는 것은 아니다. 항상 raw URL을 다시 호출해 200과 `content-type: image/svg+xml`을 확인한다.

```bash
gh workflow run "GitHub Profile 3D Contrib"   # 또는 "Metrics", "Generate snake animation"
gh run watch <run-id> --exit-status
```

## README 편집 규칙

- **주석 처리된 블록은 죽은 코드가 아니라 복구 지점이다.** 트로피 섹션(호스트 402)과 metrics `<img>`(토큰 대기)는 사유를 적어 보존해뒀다. 정리한다고 지우지 말 것.
- **공개 프로필에 나가는 주장은 근거가 있어야 한다.** 스택 배지는 실제 레포/의존성에서 확인하고 넣는다(예: TypeScript·Vite·Tailwind는 `pignuante.github.io`의 `package.json` 근거, zk-SNARK은 `hy-zkp-snark` 근거). **포크를 본인 작업처럼 전시하지 말 것** — `qllm-infer`, `openhwp`, `my-skills`, `liberate-fhe`는 포크다(liberate-fhe는 "Open Source Contributions"에 별도 표기).
- 소속·직함처럼 확인이 필요한 정보는 GitHub 프로필 필드에서 추론해 넣지 말고 사용자에게 물어본다. (CERN/Genève 표기는 사용자가 명시적으로 거절했다.)
- 연락처 현황: 웹사이트는 **ai-scream.ai** (라이브). 이메일 `hanyul.ryu@hanyul.xyz`는 도메인에 A 레코드가 없어 웹은 죽었지만 **iCloud MX·SPF가 살아 있어 메일은 정상이므로 유지한다.** `blog.pignu.kr`은 `pignu.kr` 자체가 미등록이라 추가하면 안 된다.
- 프로젝트 목록의 출처는 <https://ai-scream.ai> 와 `github.com/ai-screams` 조직이다(HwpForge, scoop-uv, Howl, Azimuth, Mara). 사이트 링크는 `/howl`이지만 정규 레포명은 `Howl`이다.

## 기타

- `.gitignore`가 `.claude/`(기존 "AI Tools" 섹션)와 `.omc/`를 무시한다. 이 레포에 프로젝트 전용 Claude 설정을 커밋하려면 먼저 해당 라인을 걷어내야 한다.
- 커밋 메시지는 Conventional Commits를 쓴다.
