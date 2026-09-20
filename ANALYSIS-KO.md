# club-3090 전수조사 분석 정리 (한국어)

> 이 문서는 club-3090 레포지토리를 전수조사하여 분석한 내용을 한국어로 정리한 것입니다.
> 작성일: 2026-09-20

## 📍 저장소 주소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/club-3090 |
| **업스트림 (원본)** | https://github.com/noonghunna/club-3090 |
| 커뮤니티 프로젝트 | https://github.com/VykosX/club-3090-server |
| Discord | https://discord.gg/gzdfjhj5yN |

---

## 1. 이게 뭐하는 물건인가?

### 한 줄 요약

> **RTX 3090 같은 소비자용 그래픽카드로 집에서 LLM(대규모 언어모델)을 직접 서빙하는 방법을 전부 검증해서 정리해둔 "레시피 창고"**

README 첫 줄: *"Recipes for serving LLMs locally on RTX 3090s."*

### 규모

| 항목 | 수치 |
|---|---|
| 총 파일 수 | 1,056개 (32MB, 코드/문서만) |
| 지원 모델 | 22종 (Qwen, Gemma, DeepSeek, GLM, Nemotron 등) |
| 검증된 compose 설정 | 158개 |
| 지원 엔진 | 7종 (vLLM, llama.cpp, ik-llama, SGLang, beellama 등) |
| 자동화 스크립트 | 44개 |
| 테스트 스크립트 | 175개 |
| 문서 | 39개 (`UPSTREAM.md` 단독 191KB) |
| 부가 서비스 | 7개 (ComfyUI, Open WebUI, LiteLLM, Qdrant, SearXNG 등) |

참고: `CHANGELOG.md` 307KB, `BENCHMARKS.md` 251KB.

### 폴더 구조

```
club-3090/
├── models/          모델별 레시피 (22종)
│   └── <모델>/<엔진>/compose/<GPU수>/<양자화>/<서빙스택>.yml
├── scripts/         자동화 도구 (setup / launch / switch / bench / verify ...)
├── docs/            39개 문서 (입문 → 하드웨어 → 실패분석 → 기여자 가이드)
├── tools/           kv-calc.py (VRAM 계산기), serve-cockpit (터미널 UI `c3`)
└── services/        ComfyUI, Open WebUI, LiteLLM, Qdrant, SearXNG, Studio
```

경로 자체가 의미를 인코딩합니다:
`models/<model>/<engine>/compose/<topology>/<quant>/<serving>.yml`

예) `models/qwen3.6-27b/vllm/compose/dual/autoround-int4/fp8-mtp.yml`

### 이 레포가 특별한 이유

**1) "추측"이 아니라 "측정"**
`CLAUDE.md`에 규칙이 명문화되어 있음:
> *"Don't claim a TPS number you didn't measure."*

벤치 프로토콜: 워밍업 3회 + 측정 5회, 샘플러 값(temperature/top_p/top_k/min_p) 전부 명시 전송.

**2) 실패를 숨기지 않음**
compose 파일 헤더에 알려진 문제를 그대로 기재:
```yaml
#   Status:    ⚠️ Production w/ caveats
#   Caveats:   MTP drafter exposed to OPEN vllm#50021 (GDN spec-decode crash class)
```

**3) 7단계 상태 등급제**
✅ Production / ⚠️ 주의필요 / 🧪 실험중 / 🐣 인큐베이팅 / 👁️ 프리뷰 / ⏸️ 업스트림 대기 / 🗑️ 폐기

**4) 실패 지점 문서화**
`docs/CLIFFS.md` (85KB)가 통째로 "어디서 터지는지" 분석. 예: Cliff 2b — 누적 25K 토큰 부근에서 VRAM 누수.

---

## 2. 쉬운 비유

### 요리책 비유

- **재료** = AI 모델 (HuggingFace에서 무료 다운로드)
- **주방** = 그래픽카드 (RTX 3090)
- **조리도구** = 엔진 (vLLM, llama.cpp)
- **레시피** = 이 레포 (club-3090)

AI 모델은 다운받는다고 바로 돌아가지 않습니다.
270억 파라미터 모델을 24GB 카드에 넣으려면 압축(양자화)이 필요하고,
너무 압축하면 품질이 떨어지고 덜 하면 메모리에 안 들어갑니다.
메모리 배분 · 문맥 길이 · 2장일 때 분할 방식… 이 모든 조합의 정답을
수백 번 실험해서 찾아놓은 것이 이 레포입니다.

### 사용 흐름

```bash
git clone https://github.com/bmshin94/club-3090.git
bash scripts/setup.sh qwen3.6-27b   # 다운로드 + SHA 검증
bash scripts/launch.sh              # 대화형 마법사로 실행
curl http://localhost:8020/v1/chat/completions -d '{...}'
```

### 비용 비교

| | ChatGPT Plus | club-3090 |
|---|---|---|
| 월 비용 | $20 | 0원 (전기세만) |
| 사용량 제한 | 있음 | 없음 |
| 데이터 | 외부 서버 | 내 PC 밖으로 안 나감 |
| 인터넷 | 필수 | 불필요 |
| 초기 비용 | 0원 | 그래픽카드 값 |

### 전제 조건 (중요)

- VRAM 24GB급 GPU 1~2장 (RTX 3090/4090/5090, A6000 등)
- Linux (Ubuntu 22.04+) 또는 Windows + WSL2
- Docker + NVIDIA Container Toolkit
- NVIDIA 드라이버 580.x+
- 모델당 디스크 ~30GB

**GPU가 없으면 실행은 불가능**하며, 문서 학습용으로만 활용 가능합니다.

---

## 3. 주요 질문 답변

### Q1. 설치 및 사용법

```bash
# 준비물 확인
nvidia-smi
docker --version
python3 -c "import yaml"

# 설치
git clone https://github.com/bmshin94/club-3090.git
cd club-3090
bash scripts/setup.sh qwen3.6-27b
bash scripts/launch.sh

# 확인
curl -sf http://localhost:8020/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.6-27b","messages":[{"role":"user","content":"안녕"}],"max_tokens":200}'
```

자주 쓰는 명령어:

| 명령어 | 용도 |
|---|---|
| `scripts/switch.sh --list` | 이 PC에서 실행 가능한 설정 목록 |
| `scripts/switch.sh vllm/dual` | 설정 전환 |
| `scripts/bench.sh` | 속도(TPS) 측정 |
| `scripts/verify-full.sh` | 기능 8가지 검증 |
| `scripts/quality-test.sh` | AI 품질(도구호출/지시따르기) 측정 |
| `scripts/soak-test.sh` | 30~60분 내구성 테스트 |
| `scripts/health.sh` | 실시간 상태 |
| `scripts/report.sh` | 개인정보 마스킹된 진단 리포트 |
| `scripts/update.sh` | 업데이트 |

터미널 UI:
```bash
uv pip install -e tools/serve-cockpit
c3
```

스크립트 없이 Docker만:
```bash
MODEL_DIR=/path/to/models docker compose \
  -f models/qwen3.6-27b/vllm/compose/dual/autoround-int4/fp8-mtp.yml up -d
```

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가?

**셋 다 아닙니다.**

레포 전체 검색 결과: `plugin.json` 없음, `.claude/skills/` 없음, MCP 서버 구현 없음.

| 구분 | club-3090 |
|---|---|
| 플러그인 | ❌ 독립 실행 |
| 스킬 | ❌ (단, `CLAUDE.md`는 AI 에이전트용 가이드로 존재) |
| MCP | ❌ 서버 구현 없음 |
| **실제 정체** | ✅ Docker Compose 설정 + Bash 자동화 + 문서 = **인프라 레시피 레포** |

참고: `CLAUDE.md`는 `AGENTS.md`의 심볼릭 링크로, AI 코딩 에이전트(Claude Code, Cursor, Copilot 등)가
이 레포에서 작업할 때 따라야 할 규칙을 담고 있습니다.

### Q3. API 토큰이 필요한가?

**대부분 불필요합니다.**

- 기본 모델들은 공개 저장소라 HuggingFace 토큰 없이 다운로드 가능
- 게이트 모델(Llama, Gemma 등)만 HF 토큰 필요 — **무료** (동의만 하면 발급)
- **서빙 자체에는 API 키가 전혀 없음** — `localhost:8020`에 인증 없이 접근

```python
from openai import OpenAI
client = OpenAI(
    base_url="http://localhost:8020/v1",
    api_key="dummy"   # 검사하지 않음
)
```

⚠️ 외부 노출 시에는 반드시 인증 레이어(LiteLLM 프록시 등)를 붙여야 합니다.

### Q4. 왜 GitHub에서 유명한가? (추론)

> 주의: 실제 스타 수는 확인하지 못했습니다. 아래는 내용물 기반 추론입니다.

1. **구체적이고 광범위한 니즈** — 중고 3090(24GB)은 가성비 최고지만 정보가 파편화되어 있었음
2. **측정된 숫자만 제시** — 추측 금지 규칙 + 고정된 벤치 프로토콜
3. **실패의 정직한 문서화** — `CLIFFS.md` 85KB가 전부 실패 분석
4. **업스트림 생태계 기여** — vLLM/llama.cpp 본진에 버그 리포트 및 PR (`vllm#40914` 등 머지)
5. **커뮤니티 설계** — Discord + Discussions + Issues 3단 구조, 벤치마크 수집용 이슈 템플릿,
   개인정보 자동 마스킹 리포트, 기여자 개별 크레딧
6. **상용 제품급 문서** — 초보/중급/기여자 3트랙, 39개 문서 1MB+

### Q5. 로컬 에이전트 구축에 도움이 되는가?

**매우 도움이 됩니다.** 이 레포의 핵심 강점 중 하나입니다.

1. **OpenAI 호환 API** — LangChain, LlamaIndex, CrewAI, AutoGen 등에서 `base_url`만 변경
2. **Tool Calling 품질을 실측** — `quality-test.sh`가 ToolCall-15, InstructFollow-15,
   StructOutput-15, DataExtract-15 등으로 측정하고 결과를 compose 헤더에 기록
3. **에이전트 전용 함정 문서화** — Cliff 2b(누적 25K 토큰 VRAM 누수)는 일반 테스트를
   통과하고 실제 에이전트 워크로드에서만 발생. `soak-test.sh`가 이를 검출
4. **에이전트 벤치마크** — `bench-agentic.sh`, `concurrency-probe.sh`,
   `aider-polyglot-30`, `hermesagent-20`, `cli-40`, `bugfind-15` 팩
5. **부가 인프라 완비** — Qdrant(벡터DB/RAG), SearXNG(검색 도구), LiteLLM(프록시),
   Open WebUI(채팅 UI), ComfyUI(이미지/영상 생성)

추천 구성:
```
[에이전트 앱 (React/Node)]
        ↓ OpenAI SDK
[LiteLLM :4000]            ← 라우팅/인증/로깅
        ↓
[club-3090 vLLM :8020]     ← 추론
        ↓
[Qdrant(기억) + SearXNG(검색) + 커스텀 도구]
```

### Q6. React나 PHP로 만들 수 있는가?

**레포 자체의 재구현은 불가능하지만, 위에 얹는 레이어는 완전히 가능합니다.**

재구현이 불가능한 이유:

| 구성요소 | 이유 |
|---|---|
| vLLM / llama.cpp | C++/CUDA/Python 기반 |
| CUDA 커널 | 브라우저/PHP에서 GPU 직접 제어 불가 |
| Docker 오케스트레이션 | 셸/시스템 레벨 작업 |
| 양자화/KV 계산 | Python 수치연산 |

**얹을 수 있는 것 (React):**
- 스트리밍 채팅 UI, GPU 모니터링 대시보드, 모델 관리 패널,
  벤치마크 시각화, 웹 설정 마법사, 에이전트 플레이그라운드

```jsx
const res = await fetch('http://localhost:8020/v1/chat/completions', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    model: 'qwen3.6-27b',
    messages: [{role: 'user', content: input}],
    stream: true
  })
});
```

**얹을 수 있는 것 (PHP):**
- 사내 AI 포털, WordPress 플러그인, 고객센터 자동응답, API 게이트웨이

```php
$ch = curl_init('http://localhost:8020/v1/chat/completions');
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
    'model' => 'qwen3.6-27b',
    'messages' => [['role'=>'user', 'content'=>$userInput]]
]));
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = json_decode(curl_exec($ch), true);
echo $response['choices'][0]['message']['content'];
```

**실제 선례:** README의 Community projects에 `VykosX/club-3090-server`가 있으며,
브라우저 관리 패널(`:8008/admin`), OpenAI 호환 리버스 프록시(`:8009`),
GPU 인식 멀티 인스턴스 오케스트레이션, 사용자별 API 인증/할당량을 제공합니다.
단, "아직 공식 채택되지 않음" 상태입니다.

구조 정리:
```
[club-3090]  ← 인프라 (Bash/Python/Docker/CUDA) — 그대로 사용
     ↕ REST API (OpenAI 호환)
[내 애플리케이션]  ← React / PHP / Next.js / Laravel 자유
```

---

## 4. 수익화 아이디어

> 아래는 아이디어와 일반적 시장 논리이며, 구체적 매출 수치는 측정값이 아닌 가설입니다.

### 아이디어 1. 로컬 AI 구축 컨설팅 / SI

**배경:** 병원(환자정보), 법무법인(사건자료), 제조(설계도면), 금융(고객정보),
공공기관(망분리)은 클라우드 AI를 쓰기 어렵습니다. 한국은 특히 망분리 규제로 온프레미스 수요가 큽니다.

**상품 구성**

| 패키지 | 내용 |
|---|---|
| 진단 | 현황 분석 + 하드웨어 견적 + ROI 보고서 |
| 구축 | 서버 세팅 + 배포 + 모델 튜닝 + 사내 시스템 연동 |
| 운영 | 월 유지보수 + 모델 업데이트 + 장애 대응 |

**차별화:** club-3090의 측정 벤치마크 데이터, `report.sh` 자동 진단 리포트,
`CLIFFS.md` 기반 함정 회피 지식

**진입 난이도:** 낮음 (초기 투자 최소, GPU 1~2장으로 데모 가능)

### 아이디어 2. 관리형 로컬 AI SaaS

**컨셉:** CLI 기반 club-3090을 웹 UI로 감싸 "클릭 몇 번으로" 사용

```
[웹 대시보드 (React)]
  ├── 모델 카탈로그 + 원클릭 설치
  ├── GPU 실시간 모니터링
  ├── 벤치마크 자동 실행 + 비교 그래프
  ├── 사용자/팀 관리 + API 키 발급
  ├── 사용량 통계 + 부서별 과금
  └── 알림 (온도, OOM, 다운)
        ↓
[club-3090 엔진]
```

**수익 모델:** 오픈코어(기본 무료 → 팀기능/SSO/감사로그 유료),
디바이스당 구독, 엔터프라이즈 온프레미스 라이선스

**기회 근거:** 유사 프로젝트가 존재하나 "공식 채택 전" 상태 — 확실한 승자가 없음

**진입 난이도:** 중간

### 아이디어 3. 교육 콘텐츠 / 온라인 강의

**배경:** 로컬 AI 수요는 크지만 양질의 한국어 자료가 희소. club-3090 문서는 전부 영어이며 깊이가 높음.

| 형태 | 가격대 감각 |
|---|---|
| YouTube 튜토리얼 | 무료 (광고 + 유입) |
| 온라인 강의 (인프런/클래스101) | 5~15만원 |
| 전자책/노션 가이드 | 2~5만원 |
| 기업 오프라인 워크샵 | 100~300만원/회 |
| 멤버십 (Q&A + 최신 설정) | 월 1~3만원 |

**커리큘럼 예시:** 개념+하드웨어 → 환경구축 → 첫 모델 → 양자화/VRAM →
벤치마킹 → RAG → 에이전트 → 웹 배포

**진입 난이도:** 매우 낮음

### 아이디어 4. 버티컬 AI SaaS

**컨셉:** club-3090을 엔진으로, 특정 산업 문제를 해결. 인프라 비용이 낮아 가격 경쟁력 확보.

| 타겟 | 서비스 | 로컬이 유리한 이유 |
|---|---|---|
| 병원 | 진료기록 요약/코딩 | 환자정보 외부 전송 불가 |
| 법무 | 판례 검색 + 계약서 검토 | 기밀성 |
| 학원 | 자동 첨삭 + 문제 생성 | 대량 처리 시 API 비용 부담 |
| 이커머스 | 상품설명 대량 생성 | 수만 건 처리 비용 |
| 미디어 | 자막/요약/썸네일 | AI Studio 활용 |
| 중소기업 | 사내 문서 RAG 검색 | 데이터 반출 불가 |

**강점 조합:** `services/`에 ComfyUI(이미지/영상/음악 생성)가 포함되어
텍스트+이미지+영상+음악을 한 서버에서 처리 가능 → 콘텐츠 제작 SaaS

**진입 난이도:** 높음 (도메인 지식 + 영업 필요)

### 아이디어 5. 하드웨어 + 소프트웨어 번들 (AI 박스)

**컨셉:** 전원을 켜면 바로 동작하는 온프레미스 AI 서버 판매

```
[중고 3090 ×2 + 워크스테이션]   하드웨어 마진
        + [club-3090 사전설치/튜닝]   세팅 비용
        + [웹 UI + 1년 지원]          소프트웨어 가치
```

**타겟:** IT 인력이 없는 중소기업, 개인 개발자/스타트업, 연구실

**리스크:** 재고/AS 부담, 중고 GPU 품질 편차, 모델 라이선스 개별 확인 필요

**진입 난이도:** 높음 (자본 필요)

### 아이디어 6. API 대행 서비스

**컨셉:** GPU를 다수 보유하고 클라우드 대비 저렴한 추론 API 제공

**경제성 (가설):**
```
투자: 3090 ×2 중고 약 160만원 + 서버 약 100만원 = 약 260만원
운영: 전기 (700W × 24h × 30일 × 200원/kWh) 약 월 10만원
비교: 클라우드 GPU 렌탈 약 월 50~100만원
→ 약 3~6개월 손익분기 (가정 기반)
```

**차별화:** 무검열 모델, 한국어 특화 튜닝, 고정 요금제, 데이터 미보관 정책

**리스크:** 24/7 SLA 부담, 콘텐츠 책임, 대형 클라우드와의 가격 경쟁

**진입 난이도:** 중간

### 추천 로드맵

```
[1단계] 콘텐츠 (리스크 0)          — 한국어 가이드로 브랜딩 + 잠재고객 확보   1~3개월
[2단계] 컨설팅 수주 (첫 매출)      — 실제 기업 니즈 파악                      3~6개월
[3단계] 도구 제품화 (React)        — 반복 작업을 웹 UI로, 오픈소스 공개       6~12개월
[4단계] SaaS 전환 (스케일)         — 유료 기능 또는 버티컬 특화               12개월+
```

**이 순서의 근거**
1. 1단계가 마케팅이자 학습 과정
2. 2단계에서 상상이 아닌 실제 고객 니즈 확인
3. 3단계는 2단계의 자동화 — 이미 검증된 수요
4. 4단계는 3단계의 수익화 — 사용자 기반 확보 후

### 사전 확인 사항

- **모델 라이선스** — 모델마다 상업적 이용 조건이 다름 (Apache 2.0 vs 커스텀)
- **club-3090 라이선스** — Apache 2.0 (상업적 이용/수정/재배포 자유)
- **개인정보보호법** — 의료/금융 분야는 추가 규제 적용
- **전력 용량** — GPU 2장 구성은 1000W급, 가정용 회로 용량 확인 필요

---

## 5. 핵심 요약

| 질문 | 답변 |
|---|---|
| 뭐하는 건가 | 소비자용 GPU로 LLM을 서빙하는 검증된 레시피 모음 |
| 정체 | 플러그인/스킬/MCP 아님 → Docker Compose + Bash + 문서 |
| API 토큰 | 대부분 불필요 (게이트 모델만 무료 HF 토큰) |
| 로컬 에이전트 | 매우 유용 (OpenAI 호환 + ToolCall 품질 실측 + RAG 인프라) |
| React/PHP | 재구현 불가, UI/서비스 레이어는 완전 가능 |
| 수익화 | 컨설팅 → 교육 → 도구 제품화 → SaaS 순 추천 |
| 전제 조건 | VRAM 24GB급 GPU 필수 |

---

*이 문서는 club-3090 레포지토리 전수조사 결과를 정리한 것입니다.*
*저장소: https://github.com/bmshin94/club-3090*
