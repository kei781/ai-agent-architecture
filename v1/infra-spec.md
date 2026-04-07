# 듀얼 맥미니 AI 인프라 설계 문서

> 이 문서는 2대의 Mac Mini를 활용한 자율 AI 에이전트 인프라의 전체 설계를 정리한 것이다.
> 에이전트가 이 문서를 참조하여 시스템 구조, 역할 분담, 제약 조건, 세팅 방법을 이해하고 작업할 수 있도록 작성되었다.

---

## 1. 하드웨어 구성

| 항목 | 16GB M4 Mac Mini | 128GB M4 Max Mac Mini |
|------|-------------------|------------------------|
| 역할 | 게이트웨이 / 워커 / 서버 노드 | 브레인 / LLM 노드 |
| RAM | 16GB 통합 메모리 | 128GB 통합 메모리 |
| 주요 업무 | 에이전트 실행, DB, 개발, 배포, 프로세스 관리, 외부 통신 | 로컬 LLM 서빙, 이미지 생성 |
| 네트워크 | Tailscale VPN (외부 유일한 접점) | 같은 공유기 내 사설 IP. 인터넷 접근 가능하나 외부 인바운드 차단 |

---

## 2. 보안 아키텍처 (핵심)

### 2.1 네트워크 격리 원칙

**128GB M4 Max 맥미니는 외부에서 직접 접근할 수 없다.** 공유기에 포트포워딩을 하지 않으며, Tailscale도 설치하지 않는다. 128GB 맥미니는 인터넷에 나갈 수는 있지만(모델 다운로드, 업데이트 등), 외부에서 128GB로 들어오는 경로는 존재하지 않는다. 모든 외부 접근은 반드시 16GB M4 맥미니(Tailscale)를 경유한다.

```
     외부 (폰/노트북)
           │
      Tailscale VPN
           │
    ┌──────▼──────┐
    │  16GB M4     │  ← 유일한 외부 접점 (게이트웨이)
    │  (게이트웨이) │
    └──────┬──────┘
           │
     같은 공유기 내 사설 네트워크 (192.168.x.x)
     또는 Thunderbolt 직접 연결 (10.0.0.x)
           │
    ┌──────▼──────┐
    │ 128GB Max    │  ← 인터넷 아웃바운드 가능
    │ (브레인)     │     외부 인바운드 차단 (포트포워딩 없음)
    └─────────────┘     Tailscale 없음
```

### 2.2 연결 방식 (택1 또는 병행)

**방법 A: 같은 공유기 사설 네트워크**
- 두 맥미니를 같은 공유기에 연결
- 16GB: `192.168.x.10` / 128GB: `192.168.x.20` (예시)
- 128GB는 공유기를 통해 인터넷 아웃바운드 가능
- 공유기에서 128GB 관련 포트포워딩 절대 하지 않음

**방법 B: Thunderbolt 직접 연결 (추가)**
- 사설 네트워크와 별도로 Thunderbolt 케이블 직결
- 16GB: `10.0.0.1` / 128GB: `10.0.0.2`
- LLM 호출 등 내부 트래픽을 고속 직결로 처리
- 128GB의 인터넷은 공유기 이더넷/Wi-Fi로 별도 유지

### 2.3 128GB 맥미니 방화벽

Ollama, ComfyUI 등 서비스 포트가 공유기 내 다른 기기에 노출되지 않도록 방화벽 설정:

```bash
# /etc/pf.conf 에 추가
# 16GB 맥미니 IP에서만 서비스 포트 접근 허용
pass in on en0 proto tcp from 192.168.x.10 to any port { 11434, 8188 }
block in on en0 proto tcp from any to any port { 11434, 8188 }

# Thunderbolt 직결 사용 시 (bridge0)
pass in on bridge0 from 10.0.0.1 to any
```

```bash
sudo pfctl -ef /etc/pf.conf
```

---

## 3. 전체 아키텍처

### 16GB M4 Mac Mini (게이트웨이 / 워커 / 서버 노드)

```
├─ Hermes Agent          ← 오케스트레이터 + 실행자 (핵심)
├─ Claude Code           ← 코드/개발 작업 (MCP 제어, Opus 모델)
├─ Codex CLI             ← 코드 리뷰 (GPT-5-Codex 모델)
├─ Gemini 3.0 Flash API  ← 길고 복잡한 텍스트 처리 (외부 API 호출)
├─ DB
│   ├─ MySQL
│   ├─ PostgreSQL
│   └─ ChromaDB          ← RAG용 Vector DB
├─ PM2                   ← 프로세스 매니저 (Node.js 앱 등)
├─ cron / launchd        ← 스케줄 트리거
├─ Playwright            ← 브라우저 자동화
├─ Open WebUI            ← 수동 관리/모니터링 UI (128GB Ollama 프록시)
├─ Nginx                 ← 128GB 서비스 리버스 프록시 (선택)
├─ 텔레그램 챗봇          ← 일반 대화용 (128GB Ollama 프록시)
└─ 텔레그램 작업봇        ← 작업 지시용 (Hermes Agent 연결)
```

### 128GB M4 Max Mac Mini (브레인 / LLM 노드)

```
├─ Ollama                ← LLM 서빙 (사설 IP에서만 리슨)
│   ├─ qwen3:30b-a3b     ← 메인 오케스트레이터 모델
│   ├─ qwen3:8b          ← 빠른 잡무/라우팅용
│   └─ llama3.3:70b      ← 헤비 분석 (필요시 로드)
└─ ComfyUI               ← 이미지 생성 (on-demand)
```

**128GB 맥미니는 인터넷 아웃바운드가 가능하므로** 모델 다운로드, ComfyUI 모델/노드 설치 등을 자체적으로 수행할 수 있다. 단, 외부에서 128GB로 들어오는 인바운드는 차단한다.

---

## 4. 역할별 상세

### 4.1 Hermes Agent (16GB 맥미니)

- **역할**: 전체 시스템의 오케스트레이터이자 실행자
- **위치**: 16GB M4 Mac Mini에서 실행
- **기능**:
  - 태스크 수신 (HTTP 엔드포인트, cron 트리거, 텔레그램 작업봇)
  - 작업 판단 시 128GB 맥미니의 Ollama API 호출
  - Claude Code 호출하여 코드 작성/수정/배포
  - Codex CLI 호출하여 코드 리뷰
  - Gemini 3.0 Flash API 호출하여 긴 텍스트 처리
  - Playwright로 웹 브라우징/스크래핑
  - DB 조회/저장
  - 텔레그램으로 결과 전송
- **스케줄링**: cron 또는 launchd로 주기적 실행. n8n 같은 별도 오케스트레이션 레이어는 사용하지 않는다.

### 4.2 코드 개발 파이프라인 (Claude Code + Codex)

#### 구조
```
Claude Code (Opus) — 코드 작성/수정
  → GitHub push
  → Codex (GPT-5-Codex) — 코드 리뷰 + 점수 채점
  → 90점 미만 항목 존재 시 → Claude Code에 피드백 전달 → 재수정
  → 모든 항목 90점 이상 → 테스트 실행
  → 테스트 통과 → 로컬 git pull → PM2 restart
```

#### 왜 이 구조인가
- **서로 다른 모델이 크로스 리뷰하는 것이 핵심 가치.** Opus가 놓치는 패턴을 GPT-5-Codex가 잡고, 그 반대도 성립한다.
- 같은 모델끼리 루프를 돌면 같은 맹점을 공유하여 같은 실수를 반복해서 못 잡을 확률이 높다.
- 로컬 LLM으로 Codex를 대체하는 것은 불가능하다. 리뷰어는 작성자와 같거나 더 높은 수준이어야 의미가 있으며, 로컬 70B 모델은 Opus 수준의 코드를 리뷰할 능력이 부족하다.
- Sonnet 수준도 코딩 품질이 부족하다고 판단하여 Opus를 기본으로 사용 중이며, 로컬 모델은 Sonnet보다도 아래이므로 리뷰어로 부적합하다.

#### 실행 환경
- **Claude Code**: 16GB 맥미니에서 실행, MCP로 로컬 환경 제어. Claude Code Max 구독 ($100/월, Opus 기본).
- **Codex CLI**: 16GB 맥미니에서 실행. ChatGPT Plus 구독 ($20/월) 필요.
- **GitHub**: 에이전트 전용 리포지토리 사용. Claude Code가 push → Codex가 리뷰 → 루프.

#### 리포 구조 (단일 리포, 브랜치 분리)

```
my-project (private 리포)
├─ main              ← 인간용. 완성된 코드만. 배포 대상.
│                       깨끗한 커밋 히스토리 유지.
└─ agent/<task-id>   ← 에이전트 작업 브랜치. 태스크마다 새로 생성.
                        루프 과정의 지저분한 커밋은 전부 여기.
                        PR 머지 후 자동 삭제.
```

#### 워크플로우 상세

```
1. 텔레그램 작업봇 또는 Hermes Agent에서 개발 태스크 수신

2. 작업 브랜치 생성
   git checkout -b agent/<task-id> origin/main

3. Claude Code (Opus)가 코드 작성/수정 → 작업 브랜치에 push

4. Codex CLI로 코드 리뷰
   - 각 항목(코드 품질, 보안, 성능, 가독성 등) 점수 채점

5. 90점 미만 항목이 있으면:
   - Codex 리뷰 피드백을 Claude Code에 전달
   - Claude Code가 해당 부분 수정
   - 작업 브랜치에 다시 push → 리뷰 루프 반복

6. 모든 항목 90점 이상:
   - 테스트 실행 (unit test, integration test)
   - 테스트 실패 시 → 실패 로그를 Claude Code에 전달 → 수정 후 다시 리뷰 루프

7. 테스트 통과 → PR 생성
   - agent/<task-id> → main 으로 PR 생성
   - PR 본문에 포함:
     · 태스크 요약
     · Codex 최종 리뷰 점수
     · 변경 파일 목록
     · 테스트 결과
   - squash merge 설정 (루프 커밋들이 main에서 단일 커밋으로 정리됨)

8. 머지 (두 가지 모드)
   - 자동 머지: 모든 점수 90점 이상 + 테스트 통과 시 자동 머지
   - 수동 머지: 인간이 PR 확인 후 직접 머지 (중요 변경 시)
   → 모드는 태스크 지시 시 선택 가능

9. 머지 완료 후 배포
   - 16GB 맥미니 로컬에서 git pull (main 브랜치)
   - PM2 restart로 프로세스 반영
   - 작업 브랜치(agent/<task-id>) 자동 삭제
   - 텔레그램 봇으로 배포 완료 알림 (PR 링크 포함)
```

#### PR 생성 스크립트 예시

```bash
#!/bin/bash
# create-pr.sh (16GB 맥미니에서 실행)

TASK_ID=$1
TITLE=$2
SCORE=$3
TEST_RESULT=$4

# GitHub CLI로 PR 생성
gh pr create \
  --repo owner/my-project \
  --base main \
  --head "agent/$TASK_ID" \
  --title "[$TASK_ID] $TITLE" \
  --body "## 태스크 요약
$TITLE

## Codex 리뷰 점수
$SCORE

## 테스트 결과
$TEST_RESULT

---
_이 PR은 에이전트에 의해 자동 생성되었습니다._"

# 자동 머지 모드인 경우
if [ "$AUTO_MERGE" = "true" ]; then
  gh pr merge "agent/$TASK_ID" \
    --repo owner/my-project \
    --squash \
    --delete-branch
fi
```

#### 머지 후 배포 스크립트 예시

```bash
#!/bin/bash
# post-merge-deploy.sh (16GB 맥미니에서 실행)

PROJECT=$1

cd ~/projects/$PROJECT
git checkout main
git pull origin main
pm2 restart $PROJECT

# 텔레그램 알림
curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d chat_id="${CHAT_ID}" \
  -d text="✅ [$PROJECT] 배포 완료. PR: $PR_URL"
```

### 4.3 로컬 LLM (128GB 맥미니)

#### Ollama 설정

```bash
# 사설 IP에서만 리슨하도록 설정
# ~/.zshrc 또는 launchd plist
export OLLAMA_HOST=192.168.x.20
# Thunderbolt 직결 사용 시: export OLLAMA_HOST=10.0.0.2
```

16GB 맥미니의 모든 서비스는 해당 사설 IP로 Ollama에 접근한다.

#### 메인 모델: Qwen3-30B-A3B (MoE)
- **용도**: 오케스트레이션 판단, tool calling, function calling, 일반 추론, 일반 대화
- **특징**: 총 30.5B 파라미터, 토큰당 활성 3.3B만 사용 → 빠르고 메모리 효율적
- **메모리 사용**: 약 18GB
- **강점**: tool calling 성능 우수, 에이전트 워크플로우에 최적화

#### 보조 모델: Qwen3-8B
- **용도**: 이메일 분류, 간단한 텍스트 파싱, 라우팅 판단 등 가벼운 작업
- **메모리 사용**: 약 5GB
- **활용**: 빠른 응답이 필요한 잡무에 사용하여 30B 모델의 부하를 줄임

#### 헤비 모델: Llama 3.3 70B (Q6 양자화)
- **용도**: 복잡한 분석, 긴 컨텍스트 필요한 작업
- **메모리 사용**: 약 55GB
- **활용**: 상시 로드하지 않음. 필요할 때만 Ollama가 자동 로드/언로드
- **양자화**: 128GB이면 Q6~Q8까지 가능. Q4 대비 품질 향상 체감됨

#### 모델 라우팅 전략
```
작업 복잡도 판단:
  - 단순 분류/파싱 → qwen3:8b
  - 일반 오케스트레이션/tool calling/대화 → qwen3:30b-a3b
  - 깊은 분석/긴 컨텍스트 → llama3.3:70b
```

### 4.4 텔레그램 봇 (16GB 맥미니)

2개의 봇을 운용한다. 둘 다 16GB 맥미니에서 실행한다.

#### 챗봇 (일반 대화용) — ChatGPT/Gemini 구독 대체
- **용도**: 일반 질문, 텍스트 요약, 번역, 브레인스토밍 등
- **백엔드**: 128GB Ollama 프록시
- **기본 모델**: qwen3:30b-a3b

```python
# chat_bot.py (16GB 맥미니에서 실행)
import telebot
import requests

bot = telebot.TeleBot("챗봇_TOKEN")
OLLAMA_URL = "http://192.168.x.20:11434/api/chat"

@bot.message_handler(func=lambda m: True)
def handle(msg):
    resp = requests.post(OLLAMA_URL, json={
        "model": "qwen3:30b-a3b",
        "messages": [{"role": "user", "content": msg.text}],
        "stream": False
    })
    bot.reply_to(msg, resp.json()["message"]["content"])

bot.polling()
```

#### 작업봇 (작업 지시용)
- **용도**: 스크래핑 지시, 배포 명령, 이메일 확인, 시스템 제어 등
- **백엔드**: Hermes Agent HTTP 엔드포인트

```python
# task_bot.py (16GB 맥미니에서 실행)
import telebot
import requests

bot = telebot.TeleBot("작업봇_TOKEN")
HERMES_URL = "http://localhost:PORT/api/task"

@bot.message_handler(func=lambda m: True)
def handle(msg):
    resp = requests.post(HERMES_URL, json={
        "task": msg.text,
        "user_id": msg.from_user.id
    })
    bot.reply_to(msg, resp.json()["result"])

bot.polling()
```

#### 알림 기능
작업봇은 사용자가 메시지를 보내지 않아도 **자동으로 알림을 보낼 수 있다**:
- 크론 작업 결과 (스크래핑 요약, 이메일 요약)
- 배포 완료 알림 (Claude Code + Codex 루프 완료 후)
- 에러/장애 알림
- 예약된 리포트

### 4.5 DB 구성 (16GB 맥미니)

| DB | 용도 | 예상 메모리 | 관리 방법 |
|----|------|-------------|-----------|
| MySQL | 일반 관계형 데이터 | ~0.5~1GB | brew services |
| PostgreSQL | 일반 관계형 데이터 | ~0.5~1GB | brew services |
| ChromaDB | RAG용 Vector DB | ~0.5~2GB | PM2 또는 별도 프로세스 |

- ChromaDB 선택 이유: 16GB 환경에서 가볍게 운용 가능. 임베딩 수십만 건 수준에서 500MB~1GB.
- 대규모 RAG(수백만 건 이상) 필요 시 Qdrant로 교체하되, 그 경우 128GB 맥미니로 이관 검토.

### 4.6 ComfyUI (128GB 맥미니)

- **실행 방식**: on-demand. 평소에는 꺼져 있고, 필요할 때만 기동.
- **포트**: 8188 (사설 IP에서만 리슨, 16GB에서만 접근)
- **제어 방법**: 16GB 맥미니에서 SSH로 start/stop

```bash
#!/bin/bash
# ~/scripts/comfyui-toggle.sh (128GB 맥미니에 위치)
if pgrep -f "main.py.*comfyui" > /dev/null; then
    pkill -f "main.py.*comfyui"
    echo "ComfyUI stopped"
else
    cd ~/ComfyUI
    nohup python main.py --listen 192.168.x.20 --port 8188 &
    echo "ComfyUI started on 192.168.x.20:8188"
fi
```

16GB에서 호출:
```bash
ssh user@192.168.x.20 '~/scripts/comfyui-toggle.sh'
```

- 128GB가 인터넷 접근 가능하므로 ComfyUI 모델/노드 다운로드를 자체적으로 수행할 수 있음.
- MCP 연동 가능성 있음. 추후 세팅 시 고려.

### 4.7 Playwright 브라우저 자동화 (16GB 맥미니)

- **위치**: 16GB 맥미니
- **호출 주체**: Hermes Agent
- **분석**: 페이지 내용을 가져온 후 128GB Ollama로 분석 요청

```
Hermes Agent
  → Playwright로 웹 페이지 접근 (16GB에서 실행)
  → HTML/텍스트 추출
  → 128GB Ollama(qwen3:30b-a3b)로 분석 요청
  → 분석 결과 수신 후 다음 액션
```

### 4.8 Open WebUI (16GB 맥미니)

- **위치**: 16GB 맥미니 (128GB Ollama를 프록시)
- **용도**: 수동 관리, 모니터링, 필요 시 직접 LLM과 대화
- **포트**: 3000

```bash
docker run -d \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://192.168.x.20:11434 \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

외부에서 Tailscale 경유로 `http://[16GB-tailscale-ip]:3000`으로 접속.

### 4.9 Nginx 리버스 프록시 (16GB 맥미니 — 선택)

128GB의 서비스들을 16GB의 localhost로 매핑하여 코드 수정을 최소화한다.

```nginx
upstream ollama_backend {
    server 192.168.x.20:11434;
}

upstream comfyui_backend {
    server 192.168.x.20:8188;
}

server {
    listen 11434;
    location / {
        proxy_pass http://ollama_backend;
    }
}

server {
    listen 8188;
    location / {
        proxy_pass http://comfyui_backend;
    }
}
```

이렇게 하면 16GB 내부의 모든 서비스가 `localhost:11434`로 Ollama를, `localhost:8188`로 ComfyUI를 호출할 수 있다.

### 4.10 Gemini 3.0 Flash (외부 API)

- **용도**: 길고 복잡한 텍스트 처리
- **호출 위치**: 16GB 맥미니에서 API 호출
- **메모리 영향**: 없음 (외부 API)
- **비용**: 월 3,000~10,000원 수준 (사용량에 따라)

### 4.11 Claude Code (16GB 맥미니)

- **용도**: 코드 작성, 수정, 리팩토링, 배포 관련 작업
- **모델**: Opus (기본 설정). Sonnet은 품질 부족으로 사용하지 않음.
- **구독**: Claude Code Max ($100/월) — 대체 불가
- **제어**: MCP(Model Context Protocol)를 통해 로컬에서 제어
- **연동**: Hermes Agent가 필요 시 Claude Code를 호출. Codex와 크로스 리뷰 루프.

### 4.12 Codex CLI (16GB 맥미니)

- **용도**: 코드 리뷰 전용. Claude Code(Opus)가 작성한 코드를 리뷰.
- **모델**: GPT-5-Codex (코딩 리뷰 특화)
- **구독**: ChatGPT Plus ($20/월) 필요 — Codex 사용을 위해 유지
- **실행**: 16GB 맥미니에서 Codex CLI로 실행

```bash
# 설치
npm i -g @openai/codex
# 또는
brew install --cask codex
```

- **GitHub 연동**: 에이전트 전용 리포에서 PR 리뷰 또는 CLI에서 직접 리뷰

---

## 5. 메모리 배분

### 5.1 128GB M4 Max (브레인 노드)

| 항목 | 예상 사용량 | 비고 |
|------|-------------|------|
| macOS | ~3~4GB | 시스템 |
| Ollama: qwen3:30b-a3b | ~18GB | 상시 로드 |
| Ollama: qwen3:8b | ~5GB | 상시 로드 |
| **상시 합계** | **~26~27GB** | |
| Ollama: llama3.3:70b Q6 | ~55GB | 필요시만 로드 |
| ComfyUI | ~8~15GB | on-demand |
| **여유분** | **~46~97GB** | 상황에 따라 |

Ollama는 모델을 자동으로 메모리에서 로드/언로드한다.

### 5.2 16GB M4 (게이트웨이 노드)

| 항목 | 예상 사용량 | 비고 |
|------|-------------|------|
| macOS | ~3GB | 시스템 |
| MySQL + PostgreSQL | ~1~2GB | brew services |
| ChromaDB | ~0.5~2GB | 데이터 규모에 따라 |
| PM2 프로세스들 | ~1~3GB | 앱 수에 따라 |
| Claude Code | ~0.5GB | |
| Codex CLI | ~0.3GB | |
| Hermes Agent | ~1GB | |
| Playwright | ~0.5~1GB | 작업 시 |
| Open WebUI (Docker) | ~1GB | |
| 텔레그램 봇 2개 | ~0.1GB | |
| Nginx | ~0.05GB | |
| **합계** | **~9~14GB** | |
| **여유분** | **~2~7GB** | |

⚠️ **주의**: 16GB에 서비스가 집중되어 있으므로 메모리 모니터링 필수. swap이 2GB 이상 지속 발생 시:
1. ChromaDB를 128GB 맥미니로 이관 (1순위)
2. Open WebUI를 Docker 대신 pip 설치로 전환하여 오버헤드 감소
3. Playwright를 상시 실행하지 않고 작업 시에만 기동

---

## 6. 통신 패턴

### 6.1 16GB → 128GB (LLM 추론 요청)

```
Hermes Agent 또는 텔레그램 챗봇
  → HTTP POST http://192.168.x.20:11434/api/chat
  → 모델 선택 (qwen3:8b / qwen3:30b-a3b / llama3.3:70b)
  → 응답 수신 후 처리
```

### 6.2 16GB → 128GB (이미지 생성)

```
Hermes Agent
  → SSH: ssh user@192.168.x.20 '~/scripts/comfyui-toggle.sh'  (ComfyUI 시작)
  → HTTP POST http://192.168.x.20:8188/prompt  (이미지 생성 요청)
  → 결과 대기 및 수신
  → SSH: ComfyUI 종료
```

### 6.3 외부 → 16GB (모니터링/수동 제어)

```
폰/노트북
  → Tailscale VPN 경유
  → Open WebUI: http://[16GB-tailscale-ip]:3000
  → 텔레그램: 챗봇/작업봇으로 직접 메시지
```

### 6.4 자동 알림 (16GB → 외부)

```
cron 트리거 → Hermes Agent 실행
  → 128GB Ollama로 분석
  → 텔레그램 봇 API로 사용자에게 알림 전송
```

### 6.5 코드 개발 → 배포 루프

```
Hermes Agent [16GB]
  → 작업 브랜치 생성: agent/<task-id>
  → Claude Code (Opus) 코드 작성 [16GB]
  → 작업 브랜치에 push [16GB → 인터넷]
  → Codex CLI 리뷰 [16GB]
  → 90점 미만 → Claude Code 수정 → 다시 push → 리뷰 루프
  → 90점 이상 → 테스트 실행 [16GB]
  → 통과 → gh pr create (agent/<task-id> → main) [16GB]
  → squash merge (자동 또는 인간 승인)
  → git pull (main) [16GB]
  → PM2 restart [16GB]
  → 작업 브랜치 삭제
  → 텔레그램 알림 + PR 링크 [16GB → 인터넷]
```

---

## 7. 비용 구조

### 7.1 월간 고정 비용

| 항목 | 비용 | 용도 |
|------|------|------|
| 128GB M4 Max 맥미니 할부 (36개월) | 15만원 | 로컬 LLM, ComfyUI |
| Claude Code Max | 10만원 ($100) | 코딩 — Opus |
| ChatGPT Plus | 2.7만원 ($20) | Codex 코드 리뷰 |
| Gemini Flash API | 0.3~1만원 | 긴 텍스트 처리 |
| 전기세 (맥미니 2대 24/7) | 0.3만원 | 무시 가능 |
| **합계** | **~28~29만원** | |

### 7.2 취소한 구독

| 항목 | 절감 | 사유 |
|------|------|------|
| ChatGPT 중 Codex 외 일반 사용 | - | 텔레그램 챗봇(로컬 LLM)으로 대체 |
| Gemini Advanced | 2.7만원/월 | Gemini Flash API로 대체 (월 수천원) |

※ ChatGPT Plus는 Codex 사용을 위해 유지. 일반 대화용도로는 사용하지 않고 로컬 LLM으로 대체.

### 7.3 로컬 LLM이 대체하는 API 비용

오케스트레이션, 브라우징 분석, 잡무 판단, 일반 대화 등을 API로 했을 경우:

| 일일 사용량 | 월 예상 API 비용 (Sonnet 기준) | 절감액 |
|------------|-------------------------------|--------|
| 50회 | ~22만원 | ~22만원 |
| 100회 | ~45만원 | ~45만원 |
| 200회 | ~90만원 | ~90만원 |

추가 절감:
- ComfyUI(로컬)로 이미지 생성 API 대체: 월 5~20만원 절감
- ChatGPT/Gemini 일반 대화 구독 대체: 월 ~2.7만원 절감

**월 28~29만원 지출로 월 40~110만원 상당의 API/구독 비용 대체.**

---

## 8. 설치 순서

### 8.1 128GB M4 Max 세팅

```bash
# 1. 네트워크 설정
# 공유기에 이더넷 또는 Wi-Fi 연결 (인터넷 아웃바운드용)
# Thunderbolt Bridge 설정 (16GB 직결용, 선택): IP 10.0.0.2

# 2. Ollama 설치 및 모델 다운로드
brew install ollama

# 사설 IP에서만 리슨
echo 'export OLLAMA_HOST=192.168.x.20' >> ~/.zshrc
source ~/.zshrc
brew services start ollama

ollama pull qwen3:30b-a3b
ollama pull qwen3:8b
# ollama pull llama3.3:70b-q6_K  # 나중에 필요시

# 3. ComfyUI 설치 (별도 가이드 참고)

# 4. 방화벽 설정 (16GB에서만 서비스 포트 접근 허용)
sudo cat >> /etc/pf.conf << 'EOF'
pass in on en0 proto tcp from 192.168.x.10 to any port { 11434, 8188 }
block in on en0 proto tcp from any to any port { 11434, 8188 }
EOF
sudo pfctl -ef /etc/pf.conf

# 5. SSH 키 기반 인증 설정 (16GB 공개키 등록)
```

### 8.2 16GB M4 세팅

```bash
# 1. 네트워크 설정
# 공유기에 이더넷 또는 Wi-Fi 연결
# Thunderbolt Bridge 설정 (128GB 직결용, 선택): IP 10.0.0.1

# 2. 128GB 연결 확인
ping 192.168.x.20
curl http://192.168.x.20:11434/api/tags  # Ollama 모델 목록 확인

# 3. Tailscale 설치
brew install tailscale

# 4. DB 설치
brew install mysql postgresql@16
brew services start mysql
brew services start postgresql@16

# 5. ChromaDB
pip install chromadb
# 서버 모드: chroma run --host 0.0.0.0 --port 8000

# 6. PM2
npm install -g pm2
pm2 startup
pm2 save

# 7. Playwright
pip install playwright
playwright install chromium

# 8. Open WebUI
docker run -d \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://192.168.x.20:11434 \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main

# 9. Nginx (선택)
brew install nginx
# 설정 후:
brew services start nginx

# 10. 텔레그램 봇 2개 설정
pm2 start chat_bot.py --name "telegram-chat" --interpreter python3
pm2 start task_bot.py --name "telegram-task" --interpreter python3
pm2 save

# 11. Claude Code 설치 (공식 가이드 참고)

# 12. Codex CLI 설치
npm i -g @openai/codex
# 또는: brew install --cask codex

# 13. Hermes Agent 설치 (프로젝트별 가이드 참고)

# 14. SSH 키 생성 및 128GB에 등록
ssh-keygen -t ed25519
ssh-copy-id user@192.168.x.20
```

---

## 9. 자동 시작 설정

### 128GB 맥미니 (브레인)

| 서비스 | 자동 시작 방법 |
|--------|---------------|
| Ollama | `brew services start ollama` |
| ComfyUI | 수동 (on-demand, 16GB에서 SSH로 제어) |

### 16GB 맥미니 (게이트웨이)

| 서비스 | 자동 시작 방법 |
|--------|---------------|
| MySQL | `brew services start mysql` |
| PostgreSQL | `brew services start postgresql@16` |
| ChromaDB | PM2 또는 launchd |
| PM2 프로세스들 | `pm2 startup` + `pm2 save` |
| Hermes Agent | PM2 |
| 텔레그램 봇 (챗/작업) | PM2 |
| Open WebUI | Docker `--restart always` |
| Nginx | `brew services start nginx` |
| Tailscale | 시스템 설정에서 자동 시작 |

---

## 10. 보안 고려사항

1. **128GB 맥미니는 외부 인바운드가 차단된다.** 공유기에서 포트포워딩하지 않으며, Tailscale을 설치하지 않는다. 인터넷 아웃바운드(모델 다운로드 등)만 가능.
2. **Ollama API는 인증이 없다.** 사설 IP에서만 리슨하고, 방화벽으로 16GB IP만 허용.
3. **Open WebUI는 16GB에서 실행.** 첫 접속 시 관리자 계정 생성, open registration 비활성화.
4. **DB는 외부 접근 차단.** bind-address를 localhost 또는 Tailscale IP로 제한.
5. **ComfyUI는 사설 IP에서만 리슨.** 16GB에서만 접근 가능.
6. **텔레그램 봇은 user_id 검증 추가 권장.** 본인 외 다른 사용자의 메시지를 무시하도록 설정.
7. **SSH 키 기반 인증 사용.** 16GB → 128GB SSH 접속 시 패스워드 대신 키 사용.
8. **GitHub 에이전트 리포는 private으로 설정.** 코드 리뷰 루프에 사용되는 리포는 외부 비공개.

---

## 11. 주요 워크플로우 예시

### 11.1 이메일 자동 수신 및 처리
```
cron (매 N분) [16GB]
  → Hermes Agent [16GB]
  → IMAP으로 이메일 수신 [16GB, 인터넷]
  → 128GB Ollama(qwen3:8b)로 분류/요약
  → 중요 메일은 텔레그램 봇으로 알림 [16GB, 인터넷]
  → 필요 시 ChromaDB에 저장 [16GB]
```

### 11.2 주기적 웹 스크래핑 및 분석
```
cron (매 6시간) [16GB]
  → Hermes Agent [16GB]
  → Playwright로 타겟 사이트 접근 [16GB, 인터넷]
  → HTML 파싱 [16GB]
  → 128GB Ollama(qwen3:30b-a3b)로 데이터 분석
  → 결과를 텔레그램 봇으로 전송 [16GB, 인터넷]
  → DB에 저장 [16GB]
```

### 11.3 코드 개발 및 배포 (Claude Code + Codex 루프)
```
텔레그램 작업봇으로 지시 [16GB]
  → Hermes Agent [16GB]
  → 작업 브랜치 생성: agent/<task-id>
  → Claude Code (Opus)로 코드 작성 [16GB]
  → 작업 브랜치에 push [16GB, 인터넷]
  → Codex CLI로 리뷰 [16GB]
  → 90점 미만 → Claude Code 수정 → push → 리뷰 반복
  → 90점 이상 → 테스트 실행 [16GB]
  → 통과 → PR 생성 (agent/<task-id> → main) [16GB, 인터넷]
  → 자동 머지 또는 인간 확인 후 머지
  → git pull (main) + PM2 restart [16GB]
  → 작업 브랜치 삭제
  → 텔레그램 봇으로 배포 완료 알림 + PR 링크 [16GB, 인터넷]
```

### 11.4 이미지 생성
```
텔레그램 작업봇으로 요청 [16GB]
  → Hermes Agent [16GB]
  → SSH로 128GB ComfyUI 시작
  → ComfyUI API로 이미지 생성 요청
  → 생성 완료 대기
  → 결과 이미지 16GB로 전송
  → 텔레그램 봇으로 이미지 전달 [16GB, 인터넷]
  → SSH로 ComfyUI 종료
```

### 11.5 일반 대화 (ChatGPT/Gemini 구독 대체)
```
텔레그램 챗봇으로 질문 [16GB]
  → 128GB Ollama(qwen3:30b-a3b)로 추론
  → 텔레그램 챗봇으로 응답 [16GB]
```

---

## 12. 제약 조건 및 유의사항

1. **128GB 맥미니는 외부 인바운드가 차단된다.** 공유기에서 포트포워딩하지 않고, Tailscale도 설치하지 않는다. 인터넷 아웃바운드만 가능.
2. **모든 외부 접근은 16GB 맥미니(Tailscale)를 경유한다.**
3. **16GB 맥미니에는 로컬 LLM을 돌리지 않는다.** 모든 LLM 추론은 128GB 맥미니의 Ollama API를 통해 처리.
4. **n8n 같은 별도 오케스트레이션 도구는 사용하지 않는다.** Hermes Agent가 오케스트레이터.
5. **ComfyUI는 상시 실행하지 않는다.** on-demand 기동/종료.
6. **16GB 맥미니의 메모리를 모니터링한다.** swap 2GB 이상 지속 시 ChromaDB 이관.
7. **DB(MySQL, PostgreSQL)는 brew services로, Node.js 앱과 커스텀 프로세스는 PM2로 관리.**
8. **Ollama 모델 로드/언로드는 자동.** 상시 로드(qwen3:30b-a3b, qwen3:8b)와 필요시 로드(llama3.3:70b) 구분.
9. **Claude Code는 Opus만 사용.** Sonnet 품질 부족. 코딩을 로컬 LLM으로 대체하지 않음.
10. **Codex는 코드 리뷰 전용.** Claude Code(Opus)와 크로스 리뷰 루프. 로컬 LLM으로 대체 불가.
11. **텔레그램 봇에 user_id 기반 접근 제한 설정.** 본인만 사용.
12. **ChatGPT Plus 구독은 Codex 전용.** 일반 대화는 로컬 LLM 텔레그램 챗봇 사용.
13. **GitHub 리포는 private.** main 브랜치에는 완성된 코드만 squash merge. 에이전트 작업 브랜치는 머지 후 자동 삭제.
