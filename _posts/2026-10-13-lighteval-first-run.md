---
title: "오픈소스 평가 도구 lighteval, 처음 돌려 보고 첫 PR까지"
date: 2026-10-13 19:30:00 +0900
categories: [LLM, Benchmark]
tags: [lighteval, llm-evaluation, open-source, log-likelihood, python-venv, git, pull-request]
math: true
mermaid: false
---

## 들어가며

지난 글에서는 KorMedMCQA라는 벤치마크를 뜯어봤다. 데이터와 논문, 채점 코드를 **읽기만** 했다. 이번에는 평가 도구를 내 컴퓨터에 받아서 **직접 돌려 봤다**. 그리고 처음으로 오픈소스 저장소에 **PR을 올렸다**.

도구는 Hugging Face의 **lighteval**이다. 최종 목표는 여기에 KorMedMCQA 태스크를 추가하는 것이다. 코드를 고치기 전에, 일단 설치하고 기존 태스크 하나를 돌려 보는 게 첫걸음이었다. 두 번째 걸음은 운영 저장소에 자기소개 파일을 올리며 PR 흐름을 익히는 것이었다.

글은 이 순서로 간다.

1. 포크와 클론은 뭐가 다른가
2. 가상환경과 설치 명령을 한 줄씩
3. 첫 실행: 명령과 로그 읽기
4. 결과 파일을 열어 보니: log-likelihood 채점
5. 첫 PR: 운영 저장소에 자기소개 올리기
6. KorMedMCQA로 돌아오면

## 1. 포크와 클론

처음에 헷갈린 것은 "포크를 따는 것"과 "클론하는 것"이 같은 일인가였다. **다른 일**이다.

| 작업 | 무엇인가 | 어디서 일어나나 |
|---|---|---|
| **fork** | 남의 저장소를 내 계정(조직)으로 복사 | GitHub 서버 |
| **clone** | GitHub 저장소를 내 컴퓨터로 내려받기 | 내 컴퓨터 |

이번 프로젝트는 포크가 이미 만들어져 있어서 나는 **클론만** 했다. 저장소는 세 층이다.

```
huggingface/lighteval        ← upstream: 원본. 최종 PR을 보내는 곳
Hugging-Face-KREW/lighteval  ← origin: 포크. 팀 작업을 push하는 곳
  └─ 내 컴퓨터 ./lighteval    ← clone
```

```bash
git lfs install
git clone https://github.com/Hugging-Face-KREW/lighteval.git
cd lighteval
git remote add upstream https://github.com/huggingface/lighteval.git
```

- `git lfs install`: lighteval은 `.json`, `.parquet` 같은 큰 파일을 **Git LFS**로 관리한다. 이게 없으면 클론이 실패할 수 있다.
- `origin`, `upstream`은 **브랜치가 아니라 저장소 주소에 붙인 별명**이다. 클론하면 `origin`은 자동으로 생기고, 원본 주소는 `upstream`이라는 이름으로 직접 추가한다.

### 1-1. 작업할 때마다 최신 상태에서 갈라 나온다

다른 사람의 작업이 머지돼도 내 로컬은 그걸 모른다. 그래서 새 작업을 시작할 때마다 최신 상태를 받아 와서 브랜치를 새로 만든다.

```bash
git fetch origin                                    # 받아 오기만 함, 내 파일은 그대로
git switch -c <내 브랜치> origin/<기여할 브랜치>       # 최신 상태에서 새 브랜치
```

`fetch`는 받아 오기만 하고, `pull`은 받아 와서 현재 브랜치에 바로 합친다. 처음에는 `fetch`를 쓰고 직접 합치는 쪽이 무슨 일이 일어나는지 보기 쉽다.

## 2. 가상환경과 설치

설치 명령은 여섯 줄이다.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -e ".[dev]"
pip install "huggingface_hub<2"
pre-commit install
```

### 2-1. 가상환경이 왜 필요한가

`pip install`은 그냥 쓰면 **컴퓨터 전체가 같이 쓰는 Python**에 패키지를 깐다. 그런데 프로젝트마다 필요한 버전이 다르다. lighteval은 `huggingface_hub` 1.x가 필요한데, 다른 프로젝트가 2.x를 쓴다면 둘을 한곳에 깔 수 없다.

**가상환경(venv)은 프로젝트 전용 Python 상자**다. 실체는 `.venv/`라는 폴더 하나다. 안에 그 프로젝트용 `python`, `pip`, 패키지가 따로 들어간다. 설치가 꼬이면 폴더를 지우고 다시 만들면 된다.

### 2-2. 한 줄씩

| 명령 | 하는 일 | 언제 |
|---|---|---|
| `python3 -m venv .venv` | `.venv/` 상자를 만든다. Python 내장 기능이다 | 처음 한 번 |
| `source .venv/bin/activate` | 상자를 켠다. 이후 `python`, `pip`이 상자 안의 것으로 잡힌다. 프롬프트에 `(.venv)`가 붙는다 | **터미널을 열 때마다** |
| `pip install --upgrade pip` | pip 자체를 최신으로 올린다 | 처음 한 번 |
| `pip install -e ".[dev]"` | lighteval과 개발 도구를 설치한다(아래) | 처음 한 번 |
| `pip install "huggingface_hub<2"` | 2.x를 1.x로 내린다(아래) | 바로 위 줄 다음 |
| `pre-commit install` | `git commit` 직전에 코드 검사가 자동으로 돌게 한다 | 처음 한 번 |

`source`를 붙이는 이유도 있다. 스크립트를 **지금 셸 안에서** 실행하라는 뜻이다. 그냥 실행하면 별도 셸에서 돌고 끝나서 지금 터미널에는 아무 변화가 없다.

### 2-3. `pip install -e ".[dev]"`

가장 중요한 줄이다.

- `.`: 현재 폴더(lighteval 저장소)를 설치한다. pip이 `pyproject.toml`을 읽고 필요한 패키지를 알아낸다.
- `-e`: **편집 모드(editable)**. 코드를 복사하지 않고 `src/lighteval`에 연결만 한다. 그래서 파일을 고치면 다시 설치하지 않아도 바로 반영된다. 새 태스크 파일을 추가하는 작업에서는 사실상 필수다.
- `[dev]`: 개발용 묶음이다. `pyproject.toml`을 보면 `dev = lighteval[quality, tests]`이고, `ruff`, `pre-commit`, `pytest` 등이 들어 있다.
- 따옴표: zsh가 `[ ]`를 파일 이름 패턴으로 해석하지 못하게 막는다.

### 2-4. `huggingface_hub<2`는 왜 넣나

lighteval의 `pyproject.toml`에는 `huggingface_hub>=0.30.2`라고만 적혀 있다. 상한이 없어서 pip이 최신 2.x를 깐다. 그런데 2.x에서는 `HfApi.list_models()`의 `model_name` 인자가 없어져서 실행이 이 오류로 멈춘다.

```
HfApi.list_models() got an unexpected keyword argument 'model_name'
```

그래서 설치 직후에 1.x로 덮어쓴다. 순서가 중요하다. 앞에서 하면 `-e ".[dev]"` 설치가 다시 2.x로 올릴 수 있다. 이 문제는 실제로 실행 로그에서 다시 만난다(3-3).

## 3. 첫 실행

설치 확인용으로 아주 작은 모델에 기존 태스크 하나를 10문항만 돌린다.

```bash
lighteval accelerate \
  "model_name=HuggingFaceTB/SmolLM2-135M-Instruct" \
  "arc:easy|0" \
  --max-samples 10 \
  --output-dir ./results \
  --save-details
```

### 3-1. 명령 읽기

| 조각 | 뜻 |
|---|---|
| `accelerate` | 백엔드. HF `transformers`로 모델을 내 컴퓨터에 직접 올려 돌린다 |
| `model_name=...` | 평가할 모델. **SmolLM2-135M**은 파라미터 1억 3500만 개짜리 아주 작은 모델이다. 성능이 아니라 설치 확인이 목적이다 |
| `"arc:easy\|0"` | `태스크\|few-shot 수`. ARC-Easy(초등 과학 객관식)를 예시 없이(0-shot) |
| 따옴표 | `\|`는 셸에서 파이프라서 따옴표가 없으면 명령이 둘로 쪼개진다 |
| `--max-samples 10` | 10문항만 |
| `--save-details` | 점수 요약뿐 아니라 **문항별 기록**도 저장 |

### 3-2. 로그를 단계별로

실행하고 30초쯤 지났을 때 멈춘 것처럼 보였다. 로그를 다시 읽어 보니 다운로드 중이었다. 전체는 약 **77초** 걸렸다.

| 시각 | 단계 | 무슨 일 |
|---|---|---|
| 19:07:18 | `LOADING TASKS` | 등록된 태스크 674개를 읽고 `arc:easy`를 찾음 |
| 19:07:20~39 | 데이터 다운로드 | ARC-Easy train/test/validation parquet |
| 19:07:39 | gather 테스트 | `device='mps:0'`: **Mac GPU(MPS)**를 잡았다 |
| 19:07:39~08:08 | `LOADING MODEL` | 모델 가중치 268MB 다운로드 후 MPS에 올림 |
| 19:08:08~24 | 배치 크기 탐색 | 메모리가 버티는 최대 배치를 찾음 |
| 19:08:24~34 | `RUNNING MODEL` | **실제 평가는 약 9초** |
| 19:08:35 | `COMPUTING METRICS`, `SAVING` | 점수 계산, 결과 저장 |

77초 중 모델이 실제로 계산한 시간은 9초 정도다. 나머지는 처음에만 하는 다운로드와 준비다. 두 번째부터는 캐시를 써서 훨씬 빠르다.

`RUNNING MODEL` 바로 아래에 이 줄이 있다.

```
Running SamplingMethod.LOGPROBS requests
```

**LOGPROBS**. 모델이 답을 생성하지 않고 확률만 계산한다는 뜻이다. 4장에서 자세히 본다.

### 3-3. 경고 두 개

결과는 나왔지만 중간에 낯선 경고가 두 개 있었다.

**nltk 인증서 오류**

```
[nltk_data] Error loading punkt: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] ...>
```

nltk가 문장 분리용 데이터를 받으려다 SSL 인증서 확인에 실패했다. python.org에서 받은 Python은 macOS에서 인증서를 따로 설치해야 하는 경우가 많다. ARC 채점에는 이 데이터를 쓰지 않아서 결과에는 영향이 없다.

**huggingface_hub 경고**

```
FutureWarning: Deprecated argument(s) used in 'list_models': model_name.
Will not be supported from version '2.0'.
```

2-4에서 본 **바로 그 버그**다. 1.x에서는 "2.0부터 없어진다"는 경고만 남기고 지나간다. 2.x였다면 여기서 멈췄을 것이다. 설치 단계에서 왜 버전을 내렸는지 로그로 확인한 셈이다.

### 3-4. 결과

```
|   Task   |Version|Metric|Value|   |Stderr|
|----------|-------|------|----:|---|-----:|
|all       |       |acc   |  0.4|±  |0.1633|
|arc:easy:0|       |acc   |  0.4|±  |0.1633|
```

10문항 중 4개를 맞혔다. **Stderr 0.1633**은 표준오차다. $\sqrt{0.4 \times 0.6 / 9} \approx 0.163$으로 계산된다. 10문항뿐이라 흔들림이 크니 점수 자체에 의미를 둘 필요는 없다. 이 실행의 목적은 "설치가 됐다"를 확인하는 것이다.

## 4. 결과 파일을 열어 보니: log-likelihood 채점

`--save-details` 덕분에 `results/details/` 아래에 문항별 parquet 파일이 생겼다. 열어 보니 문항마다 선택지 4개에 숫자가 하나씩 붙어 있었다.

### 4-1. 첫 문항

```
Question: Which is the function of the gallbladder?   (정답: store bile)
```

| 선택지 | log 확률 |
|---|---:|
| **" store bile"** | **−20.875** ← 선택 |
| " produce bile" | −21.625 |
| " store digestive enzymes" | −25.750 |
| " produce digestive enzymes" | −26.375 |

가장 큰 값(0에 가장 가까운 값)을 고르고, 그게 정답이라 1점이다. 이 숫자가 무엇인지 기초부터 따라가 봤다.

### 4-2. 언어 모델은 다음 토큰 분류기다

이미지 분류기는 입력을 받아 클래스 10개에 대한 확률을 낸다. 언어 모델도 같다. 다만 **클래스가 어휘 전체(수만 개 토큰)**이고 **입력이 지금까지의 텍스트**다.

선택지 `" store bile"`은 토큰 두 개(`" store"`, `" bile"`)다. 문장 전체 확률은 조건부 확률의 곱이다.

$$
P(\text{store bile} \mid \text{질문}) = P(\text{store} \mid \text{질문}) \times P(\text{bile} \mid \text{질문, store})
$$

두 번째 항에서 `" store"`는 모델이 실제로 고른 게 아니다. 정답 후보를 입력에 **강제로 넣고** 그다음 토큰의 확률을 읽는다. 학습 때의 teacher forcing과 같은 방식이다.

### 4-3. 왜 log인가

`" store bile"`의 실제 확률은 **8.6 × 10⁻¹⁰**이다. 확률을 곱할수록 0에 붙어서 컴퓨터가 다루기 어려워진다. log를 씌우면 곱셈이 덧셈이 되고 적당한 음수가 된다. log는 증가 함수라 대소 관계도 그대로다.

| 확률 | log 확률 |
|---|---:|
| 1 | 0 |
| 0.5 | −0.69 |
| 0.01 | −4.6 |
| 10⁻⁹ | −20.7 |

그래서 값은 **항상 0 이하**이고, 0에 가까울수록 모델이 그럴듯하다고 본 것이다.

익숙한 개념과도 이어진다. 학습 때 쓰는 **cross-entropy loss가 정답 토큰의 −log 확률**이다. 학습은 이 값을 줄이도록 가중치를 바꾸고, 평가는 가중치를 고정한 채 후보들의 값을 비교한다.

확률이 전부 작아 보이지만 비교만 하면 된다. 네 선택지끼리 비율로 다시 나누면 **67% / 32% / 0.5% / 0.3%**다. 모델은 "담즙을 저장하거나 만든다" 쪽으로 기울었다.

### 4-4. 짧은 선택지가 유리하다

log 확률은 토큰마다 음수를 더해 간다. 토큰이 늘면 합은 내려가기만 한다. 그래서 내용과 상관없이 **짧은 선택지가 유리**하다. 10문항 안에서도 보였다.

```
Question: ... this transferred population of lizard lost the ability to climb.
Which mechanism is most likely responsible ...?   (정답: genetic drift)
```

| 선택지 | log 확률 |
|---|---:|
| " gene flow" | −18.375 |
| " natural selection" | −17.500 |
| " genetic drift" (정답) | −17.375 |
| **" mutation"** | **−12.250** ← 선택 |

나머지 셋은 −17~−18로 비슷한데, 한 단어짜리 `" mutation"`만 5 가까이 높다.

이걸 보정하려고 log 확률을 **글자 수로 나누는** 방법(흔히 `acc_norm`)이 있다. details의 숫자로 직접 나눠 보니 결과가 이랬다.

| | 정답 수 | 바뀐 문항 |
|---|---:|---|
| 그대로 (`acc`) | 4 / 10 | |
| 글자 수로 나눔 | 5 / 10 | 3개를 새로 맞히고, 2개는 오히려 틀림 |

오히려 틀린 문항 하나가 바로 4-1의 쓸개 문제다. 나누고 나니 긴 `" produce digestive enzymes"`(−26.375 ÷ 26 ≈ −1.01)가 짧은 정답 `" store bile"`(−20.875 ÷ 11 ≈ −1.90)을 이겼다. **정규화 방식 하나로 같은 모델 점수가 달라진다.** 논문 표의 `acc`와 `acc_norm`을 섞어 비교하면 안 되는 이유다.

참고로 lighteval의 `arc:easy`는 `Metrics.loglikelihood_acc`, 즉 **정규화 없음**을 쓴다.

### 4-5. 확률이 가장 높다 ≠ 그렇게 답한다

details에는 `argmax_logits_eq_gold`라는 칸도 있다. 위치마다 확률 1등 토큰만 이어 붙였을 때(그냥 생성했을 때) 그 선택지와 정확히 같은지를 기록한다. **40개 선택지 모두 False**였다.

이 모델에게 자유롭게 답을 쓰게 했다면 어느 선택지도 그대로 나오지 않았다는 뜻이다. 그래도 log-likelihood 방식은 "넷 중에서는 이게 낫다"를 골라 점수를 준다. 형식을 못 지키는 작은 모델도 잴 수 있다는 장점이 있다. 반대로 실제로 답을 쓰는 능력과는 다른 것을 잰다는 한계도 있다.

## 5. 첫 PR: 운영 저장소에 자기소개 올리기

lighteval 코드에 손대기 전에 연습 과제가 하나 있었다. 운영 저장소 `Hugging-Face-KREW/k-llm-eval-sprint`에 **자기소개 파일을 PR로 올리는 것**이다. 파일 두 개만 고치는 작은 일이다. 대신 브랜치, push, PR, 리뷰, 머지를 한 바퀴 다 돈다.

### 5-1. 한 폴더에 저장소가 둘

운영 저장소도 클론해야 했다. 그런데 공부 노트를 모아 둔 폴더 자체가 이미 git 저장소였고, 그 안에 lighteval도 받아 둔 상태였다. 저장소 안에 저장소를 또 받아도 되는지 헷갈렸다.

```
opensource/                  ← 저장소 1: 공부 노트
├── lighteval/               ← 저장소 2: lighteval 포크
└── k-llm-eval-sprint/       ← 저장소 3: 운영 저장소
```

**된다.** git은 명령을 친 폴더에서 위로 올라가며 **가장 가까운 `.git`**을 찾아 쓴다. `cd`로 들어간 폴더가 곧 대상 저장소다. 안쪽 저장소의 커밋과 브랜치는 바깥과 완전히 따로 논다.

주의할 점은 하나다. 바깥 저장소는 안쪽 폴더를 "모르는 폴더"(`?? lighteval/`)로 본다. 여기서 `git add .`을 하면 **저장소 안의 저장소(embedded repository)**로 등록돼 꼬인다. 그래서 바깥 `.gitignore`에 두 폴더를 넣었다.

```
/lighteval
/k-llm-eval-sprint
```

### 5-2. 포크 없이 바로 push한다고?

내가 알던 오픈소스 기여는 **내 계정으로 포크 → 내 포크에 push → 원본으로 PR**이었다. 그런데 운영 저장소 안내에는 포크 이야기가 없다. 클론하고 브랜치를 만들어 바로 push하라고 한다.

차이는 **쓰기 권한**이었다.

| | 포크 방식 | 공유 저장소 방식 |
|---|---|---|
| 쓰기 권한 | 원본에 없음 | **초대를 수락해서 있음** |
| push 하는 곳 | 내 계정의 복사본 | 원본 저장소에 **브랜치로 직접** |
| PR 방향 | 내 저장소 → 원본 | 같은 저장소 안에서 내 브랜치 → `main` |
| 쓰는 곳 | 모르는 프로젝트에 기여할 때 | 팀원끼리 협업할 때 |

운영 저장소는 멘토가 참가자를 초대한 팀 저장소라 공유 저장소 방식이다. 바로 올려도 남의 것을 망치지 않는다. **`main`은 보호돼 있어서** 아무도 직접 push할 수 없고, PR에서 승인을 1명 이상 받아야 병합된다. 내가 push하는 건 내 브랜치뿐이다.

포크 방식도 이 프로젝트에서 쓴다. 나중에 `huggingface/lighteval` 원본에 PR을 보낼 때다. 원본에는 쓰기 권한이 없어서 `Hugging-Face-KREW/lighteval` 포크를 거친다. 1장의 세 층 구조가 바로 그것이다.

### 5-3. 순서

```bash
git clone https://github.com/Hugging-Face-KREW/k-llm-eval-sprint.git
cd k-llm-eval-sprint
git switch -c people/kangmin04

cp people/_template.md people/kangmin04.md    # 채운다
# teams/team-d-kormedmcqa/README.md 팀원 표에 한 줄 추가

git add people/kangmin04.md teams/
git commit -m "Add kangmin04 profile"
git push -u origin people/kangmin04
```

파일이 어디에 있는지 단계별로 보면 이렇다.

| 단계 | 파일이 있는 곳 | 누가 보나 |
|---|---|---|
| `commit` | 내 컴퓨터 | 나만 |
| `push` | GitHub의 `people/kangmin04` 브랜치 | 모두. 하지만 `main`은 그대로 |
| PR + 승인 + 머지 | `main` | 이제야 본문에 들어감 |

커밋 메시지는 영어 한 줄이 규칙이다. `-u`는 "이 브랜치를 원격 브랜치와 짝지어 둬라"는 뜻이다. 다음부터는 `git push`만 쳐도 된다.

### 5-4. PR 화면에서

push 직후 GitHub 저장소에 들어가면 노란 배너에 **Compare & pull request** 버튼이 뜬다. 누르면 PR 작성 화면이다.

- **방향**: `base: main ← compare: people/kangmin04`. 운영 저장소는 `main`이 맞다. lighteval 포크에서는 기본값 `main`을 **팀 브랜치로 바꿔야** 한다. 세션에서 "가장 흔한 실수"로 꼽힌 부분이다.
- **본문**: 저장소에 PR 템플릿(`.github/pull_request_template.md`)이 있어서 "무엇을 바꿨나요", "리뷰어가 봐 줬으면 하는 점", 체크박스가 미리 채워져 나온다.
- **리뷰어**: 멘토 두 분은 내가 지정하지 않았는데 이미 들어가 있었다. `.github/CODEOWNERS`에 `* @harheem @4n3mone`이 있어서 모든 PR에 자동으로 붙는다. 나는 같은 팀원 한 명만 추가했다.

이렇게 해서 **PR #6**이 열렸다.

### 5-5. push한 뒤에 찾은 실수

올리고 나서 다시 보니 팀 README 표에 넣은 줄이 틀려 있었다.

```
| 이름 | GitHub | 맡은 일 |
|---|---|---|
|김강민|kangmin04|
```

표는 **3칸**인데 내 줄은 2칸이다. 마지막 칸이 비어 있더라도 구분선은 있어야 한다.

```
| 김강민 | [@kangmin04](https://github.com/kangmin04) | |
```

이미 push한 커밋은 `amend`로 고치지 않는다. 그러면 force push를 해야 하는데, 공유 저장소에서는 피하는 게 좋다. **커밋을 하나 더 쌓아서 push**하면 열려 있는 PR에 자동으로 붙는다. PR을 새로 만들 필요도 없다.

```bash
git add teams/team-d-kormedmcqa/README.md
git commit -m "Fix team table row format"
git push
```

같은 방법으로 PR 번호가 생긴 뒤 `people/kangmin04.md`의 기여 기록 표 1주차 칸에 PR 링크를 넣는다.

그리고 충돌 가능성도 하나 있다. 멘토가 올린 다른 PR이 같은 README의 **같은 자리**(팀원 표 바로 아래)에 내용을 추가한다. 그쪽이 먼저 병합되면 내 PR에 충돌 표시가 뜬다. 그때는 GitHub의 **Resolve conflicts**에서 `<<<<<<<`, `=======`, `>>>>>>>` 표시를 지우고 **양쪽 내용을 모두 남기면** 된다.

### 5-6. 머지 뒤 정리

브랜치는 **머지가 끝난 뒤에** 지운다. 머지 전에 지우면 PR이 자동으로 닫힌다.

| 위치 | 지우는 방법 |
|---|---|
| GitHub (원격) | 머지 직후 PR 화면의 **Delete branch** 버튼 |
| 내 컴퓨터 (로컬) | `git switch main` → `git pull` → `git branch -d people/kangmin04` |

`git branch -d`는 머지되지 않은 브랜치는 지우지 않고 거부한다. 실수로 작업을 날릴 일이 없다. 다음 주 기여 기록을 쓸 때는 최신 `main`에서 새 브랜치를 만든다.

## 6. KorMedMCQA로 돌아오면

지난 글에서 본 KorMedMCQA 채점(lm-evaluation-harness)은 방식이 반대다. 모델이 답을 **생성**하고, 정규식으로 A~E를 뽑는다.

| | 오늘 돌린 ARC (lighteval) | KorMedMCQA (harness) |
|---|---|---|
| 모델이 하는 일 | 선택지별 확률 계산 | 답 글자를 직접 씀 |
| 필요한 것 | logits 접근(오픈 웨이트 모델) | 텍스트 출력만(API 모델도 가능) |
| 형식 실패 | 없음 | 있음(정규식이 못 뽑으면 0점) |
| 길이 편향 | 본문을 붙이면 있음 | 없음 |

lighteval에 KorMedMCQA를 넣을 때 정할 것이 여기서 나온다. **어느 방식으로 채점할 것인가.** 원본 점수를 재현하려면 원본과 같은 생성 방식이 기본값이 된다. log-likelihood로 바꾸면 위 표의 차이만큼 점수가 원본과 달라진다.

## 마치며

이번에 한 일을 정리하면 이렇다.

- 포크와 클론의 차이, `origin`과 `upstream`
- 가상환경을 만들고 편집 모드로 설치하기
- 로그를 단계별로 읽고 경고가 무엇인지 알아보기
- 결과 파일의 숫자로 log-likelihood 채점을 끝까지 따라가 보기
- 한 폴더에 저장소 여러 개 두기, 포크 방식과 공유 저장소 방식의 차이
- 브랜치 → push → PR → 리뷰 → 머지를 한 바퀴 돌아 보기

