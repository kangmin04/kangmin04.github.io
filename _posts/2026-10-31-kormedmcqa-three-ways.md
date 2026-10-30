---
title: "같은 벤치마크를 세 번 돌렸더니 점수가 달랐다: harness, lighteval, 커스텀 태스크"
date: 2026-10-31 02:20:00 +0900
categories: [LLM, Benchmark]
tags: [kormedmcqa, lm-evaluation-harness, lighteval, llm-evaluation, reproducibility, open-source]
math: false
mermaid: false
---

## 들어가며

같은 벤치마크를 도구만 바꿔서 세 번 돌렸다. 숫자가 다르게 나왔다. 어느 쪽이 틀렸는지부터 따져 봤다.

결론부터 적는다. **틀린 쪽은 없었다.** 점수가 달랐던 이유는 두 가지였다. 하나는 채점 방식이 달랐기 때문이고, 다른 하나는 **서로 다른 문항을 풀었기 때문**이다. 문항을 맞추고 435문항 전체를 돌리자 차이가 0이 됐다.

돌린 것은 이 세 가지다.

| # | 무엇을 돌렸나 | 채점 방식 | 모델이 하는 일 |
|---|---|---|---|
| 1 | lm-evaluation-harness의 `kormedmcqa` 태스크 | 생성 + 정규식 추출 | 답 글자를 직접 씀 |
| 2 | lighteval 첫 설치 후 `arc:easy` | log-likelihood | 선택지별 확률 계산 |
| 3 | 우리가 만든 lighteval 커스텀 태스크 `kormedmcqa:doctor` | 생성 + 정규식 추출 | 답 글자를 직접 씀 |

2번의 설치와 첫 실행은 지난 글(*오픈소스 평가 도구 lighteval, 처음 돌려 보고 첫 PR까지*)에서 자세히 다뤘다. 이 글에서는 반복하지 않고, **3번과 1번을 어떻게 비교했는지**에 초점을 둔다.

글은 이 순서로 간다.

1. 세 가지를 한 표로 놓고 보기
2. 1번: harness로 KorMedMCQA 돌리기
3. 2번: lighteval은 채점 방식부터 달랐다
4. 3번: 커스텀 태스크와 비교하기
5. 점수가 왜 달랐나: 100문항의 함정
6. 435문항 전체 결과
7. 이 결과로 말할 수 있는 것과 없는 것

## 1. 세 가지를 한 표로 놓고 보기

점수 비교가 헷갈리는 이유의 절반은 **세 실행이 서로 다른 조건**이었기 때문이다. 먼저 조건부터 정리한다.

| | 1. harness | 2. lighteval 첫 실행 | 3. 커스텀 태스크 |
|---|---|---|---|
| 태스크 | `kormedmcqa_doctor` | `arc:easy\|0` | `kormedmcqa:doctor\|5` |
| 데이터 | KorMedMCQA 의사 | ARC-Easy (영어 과학 문제) | KorMedMCQA 의사 |
| 모델 | Qwen2.5-1.5B-Instruct | SmolLM2-135M-Instruct | Qwen2.5-1.5B-Instruct |
| 채점 | 생성 후 정규식, `exact_match` | log-likelihood, `acc` | 생성 후 정규식, `extract_choice_acc` |
| 문항 수 | 100, 435 | 10 | 100, 435 |

2번은 **설치가 되는지 확인하려는 실행**이었다. KorMedMCQA도 아니고, 모델도 다르고, 채점 방식도 다르다. 2번의 점수(0.4)를 1번, 3번의 점수와 비교하는 것은 의미가 없다. 비교 대상은 1번과 3번이다.

## 2. 1번: harness로 KorMedMCQA 돌리기

기준은 lm-evaluation-harness다. 논문 점수도 이 도구로 냈다. 클론한 저장소(커밋 `97a5e2c`)에서 아래처럼 돌렸다.

```bash
cd lm-evaluation-harness && source .venv/bin/activate
lm_eval --model hf \
  --model_args pretrained=Qwen/Qwen2.5-1.5B-Instruct,dtype=float32 \
  --tasks kormedmcqa_doctor --num_fewshot 5 \
  --batch_size 1 --device cpu --log_samples \
  --output_path ../lmeval-runs/qwen15_doctor_full_fs5
```

| 조각 | 뜻 |
|---|---|
| `--tasks kormedmcqa_doctor` | KorMedMCQA 의사 서브셋 |
| `--num_fewshot 5` | 예시 5개. harness yaml에는 이 값이 없어서 직접 줘야 논문 설정과 같다 |
| `--device cpu`, `dtype=float32` | 비교 대상과 조건을 맞추려고 고정 |
| `--log_samples` | 문항별 입력과 출력을 `samples_*.jsonl`로 저장 |

채점은 이렇게 흐른다.

```
5-shot 프롬프트 → 모델이 텍스트 생성 → 정규식으로 A~E 추출 → 정답과 비교
```

모델은 `정답：` 뒤를 이어서 쓴다. 그 출력에서 정규식 네 개를 차례로 시도해 A~E 중 하나를 뽑는다. 못 뽑으면 오답이다. 자세한 원리는 KorMedMCQA 정리 글에서 다뤘다.

첫 문항의 `samples_*.jsonl`을 열어 보면 이런 모양이다.

| 필드 | 값 |
|---|---|
| `target` | `C` |
| `resps` | `[[' C']]` (모델이 쓴 원문) |
| `filtered_resps` | `['C']` (정규식으로 뽑은 답) |
| `exact_match` | `1.0` |

**원문(`resps`)과 뽑은 답(`filtered_resps`)이 따로 저장된다.** 점수가 이상할 때 어디서 틀어졌는지 이 둘을 비교하면 된다.

## 3. 2번: lighteval은 채점 방식부터 달랐다

lighteval을 처음 돌렸을 때는 KorMedMCQA가 아니라 `arc:easy`였다. 이 실행에서 알게 된 것이 이후 비교의 출발점이다.

```
Running SamplingMethod.LOGPROBS requests
```

로그에 이 줄이 있었다. **log-likelihood 채점**이다. 모델이 답을 쓰지 않는다. 선택지마다 "이 문장이 이어질 확률"을 계산하고, 가장 높은 것을 답으로 본다.

| | 2번 (arc, log-likelihood) | 1번, 3번 (KorMedMCQA, 생성) |
|---|---|---|
| 모델이 하는 일 | 선택지별 확률 계산 | 답 글자를 직접 씀 |
| 형식 실패 | 없음 | 있음(정규식이 못 뽑으면 0점) |
| 길이 편향 | 본문을 붙이면 있음 | 없음 |

그래서 KorMedMCQA를 lighteval에 넣을 때 정할 것이 생긴다. 원본 점수를 재현하려면 원본과 같은 **생성 방식**이어야 한다. log-likelihood로 바꾸면 위 표의 차이만큼 점수가 달라진다. 이 이유로 커스텀 태스크는 생성 + 정규식을 기본으로 했다.

참고로 log-likelihood 방식에서도 정규화 하나로 점수가 달라진다. 지난 글에서 같은 10문항을 글자 수로 나눠 다시 채점했더니 4/10이 5/10이 됐다. 채점 방식 하나가 점수를 흔든다는 감각이 이때 생겼다.

## 4. 3번: 커스텀 태스크와 비교하기

lighteval에 `kormedmcqa` 태스크를 넣기 전에, 실험용 커스텀 태스크(`kormedmcqa_task.py`)를 먼저 만들었다. 목적은 하나다. **harness와 같은 점수가 나오는가.**

커스텀 태스크는 harness의 설정을 그대로 옮겼다.

| 항목 | 값 |
|---|---|
| 프롬프트 | `질문 + A~E 줄 + 정답：` 형식 (전각 콜론) |
| few-shot | `fewshot` split의 앞 5개 고정 (`few_shots_select="sequential"`) |
| 정답 추출 | harness의 정규식 네 개 그대로 |
| 최대 생성 길이 | 1024토큰 |
| stop 문자열 | `["Q:", "</s>", "<\|im_end\|>", ".", "\n\n"]` |

실행 명령은 이렇다.

```bash
cd lighteval && source .venv/bin/activate
lighteval accelerate \
  "model_name=Qwen/Qwen2.5-1.5B-Instruct,override_chat_template=False,device=cpu,batch_size=1,dtype=float32" \
  "kormedmcqa:doctor|5" \
  --custom-tasks ../project/experiments/kormedmcqa_task.py \
  --output-dir ../lmeval-runs/qwen15_doctor_lighteval_full_fs5 \
  --save-details
```

**조건을 같게 맞추는 데 쓴 인자**가 중요하다.

| 인자 | 이유 |
|---|---|
| `override_chat_template=False` | harness는 기본이 chat template 미적용이다. lighteval은 토크나이저에 template이 있으면 자동으로 켜고, 켜면 stop 문자열을 버린다. 끄지 않으면 조건이 달라진다 |
| `device=cpu` | lighteval은 기본으로 맥 GPU(MPS)를 쓰는데 메모리 부족(OOM)이 났다 |
| `dtype=float32`, `batch_size=1` | harness 실행과 같게 맞춤 |

두 도구는 별도 venv를 쓴다. 한쪽을 끝낸 뒤 `deactivate`하고 다른 쪽을 켠다. 각각 CPU로 약 25~30분 걸렸다.

## 5. 점수가 왜 달랐나: 100문항의 함정

435문항 전체를 돌리기 전에 100문항으로 먼저 비교했다. 이때 점수가 서로 달랐다.

| 설정 | lm-eval | lighteval |
|---|---|---|
| 5-shot, 100문항 | 0.30 | 0.37 |
| 0-shot, 100문항 | 0.35 | 0.32 |

5-shot에서는 lighteval이 앞서고, 0-shot에서는 lm-eval이 앞섰다. 방향도 일정하지 않다. 구현이 달라서 한쪽이 체계적으로 유리한 것이라면 이렇게 뒤집히지 않는다.

### 5-1. 먼저 의심한 것: few-shot 연결 방식

처음에는 few-shot 예시를 붙이는 방식이 다를 가능성을 의심했다. harness는 `doc_to_target`이 빈 문자열이고, lighteval은 `query + 공백 + 정답`으로 붙이기 때문이다.

그래서 두 도구가 실제로 모델에 넣은 입력을 디코딩해서 비교했다.

| 비교 항목 | 결과 |
|---|---|
| few-shot 5개 접두부 (1,914자) | 완전 동일 |
| 예시 끝 모양 | `정답： B\n\n` 동일 |
| 마지막 질문 끝 | `\n정답：` 동일 (뒤에 공백 없음) |

코드로 읽으면 달라 보여도 **결과 문자열은 같았다.** 프롬프트는 원인이 아니었다.

### 5-2. 진짜 원인: 서로 다른 100문항

원인은 문항 선택이었다.

| | 어떻게 100문항을 고르나 |
|---|---|
| lm-eval `--limit 100` | 앞에서부터 100개 |
| lighteval `--max-samples 100` | 435문항에서 섞어 뽑기 |

두 실행의 100문항을 문항 텍스트로 대조하니 **겹치는 문항이 26개뿐**이었다. 그 26문항만 따로 비교하면 이렇다.

| 설정 | 겹치는 문항 | 예측 일치 | 정확도 (맞힌 수) |
|---|---|---|---|
| 5-shot | 26 | 26 / 26 | 두 도구 모두 8 / 26 |
| 0-shot | 26 | 26 / 26 | 두 도구 모두 8 / 26 |

같은 문제를 풀게 하면 두 도구는 **같은 답을 냈다.** 점수 차이는 구현 차이가 아니라 표본 차이였다. 100문항이면 표준오차가 0.05 안팎이다. 0.30과 0.37 정도는 표본만 달라도 충분히 나온다.

> **교훈:** 두 도구를 비교할 때는 문항 집합부터 맞춘다. `--limit`과 `--max-samples`는 같은 문항을 고르지 않는다.

### 5-3. 모델 크기도 영향을 줬다

처음에는 SmolLM2-135M-Instruct로 10문항씩 돌려 봤다. 배관(명령이 돌아가는지)을 확인하는 데는 충분했다. 그러나 비교에는 쓸 수 없었다.

| | 모델이 낸 답 (10문항) | 점수 |
|---|---|---|
| lm-eval | `D` 10개 | 0.1 |
| lighteval | `D` 9개, `C` 1개 | 0.2 |

모델이 거의 항상 `D`만 답한다. 이 상태에서는 0.1과 0.2의 차이가 **채점 차이인지 우연인지** 구분할 수 없다. Qwen2.5-1.5B는 답이 A~E로 분산돼서 비교에 쓸 수 있었다.

| 예측 | A | B | C | D | E |
|---|---:|---:|---:|---:|---:|
| 개수 (435문항) | 126 | 78 | 106 | 73 | 52 |

## 6. 435문항 전체 결과

문항 집합을 맞추려고 `--limit`, `--max-samples`를 빼고 **의사 서브셋 435문항 전체**를 돌렸다.

| 항목 | lm-eval | lighteval |
|---|---|---|
| 정확도 | 0.3310 | 0.3310 |
| 표준오차 | 0.0226 | 0.0226 |
| 문항 수 | 435 | 435 |

점수가 같은 것만으로는 부족하다. 맞고 틀린 문항이 서로 상쇄해서 우연히 같을 수도 있다. 그래서 문항별로 대조했다.

| 대조 항목 | 결과 |
|---|---|
| 같은 문항 집합 | 435 / 435 |
| 정답(gold) 일치 | 435 / 435 |
| 모델 출력 문자열 일치 | 435 / 435 |
| 둘 다 맞음 | 144 |
| 한쪽만 맞음 | 0 |
| 둘 다 틀림 | 291 |

**435문항 모두 같은 출력, 같은 채점 결과**였다. 144 / 435 = 0.3310이다.

결과를 읽는 요령도 정리해 둔다.

| 보고 싶은 것 | lm-eval | lighteval |
|---|---|---|
| 최종 점수 | `results_*.json` | `results/<모델>/results_*.json` |
| 문항별 프롬프트와 답 | `samples_*.jsonl` | `details/.../details_*.parquet` |

parquet은 텍스트 에디터로 열리지 않는다. lighteval의 `.venv` 파이썬에서 pandas로 읽는다. `doc`(프롬프트, 정답), `model_response`(입력 토큰, 출력 텍스트), `metric`(문항별 점수) 열이 있다. 두 도구는 문항 순서가 달라서, 비교할 때는 행 번호가 아니라 **질문 텍스트**로 짝을 지었다.

## 7. 이 결과로 말할 수 있는 것과 없는 것

**말할 수 있다.** 이 설정(Qwen2.5-1.5B, doctor, 5-shot, 생성 + 추출 채점, CPU float32)에서 커스텀 태스크는 harness 구현과 문항 단위로 일치한다.

**말할 수 없다.** 아래는 아직 검증하지 않았다.

| 미검증 | 이유 |
|---|---|
| nurse, pharm, dentist 서브셋 | doctor만 돌렸다 |
| 0-shot 전체 | 0-shot 출력에는 `C\n해설: ...`처럼 답 뒤에 해설이 붙는 경우와 `C, D, E` 같은 복수 답이 보였다. 이런 출력의 채점 경로가 같은지는 겹친 26문항만 봤다 |
| 다른 모델, chat template을 켠 경우 | 조건을 하나로 고정했다 |
| log-likelihood 방식 채점 | 비교 대상이 아니었다 |

점수 자체에 대한 메모도 하나 남긴다. 0.33은 5지선다 우연 수준(0.2)보다 높다. 그러나 예측이 A와 C로 쏠려 있어서 모델 성능을 논할 단계는 아니다. 성능은 더 큰 모델로 다시 본다.

## 마치며

이번에 한 일을 정리하면 이렇다.

- 같은 벤치마크를 **생성 + 정규식**(harness, 커스텀 태스크)과 **log-likelihood**(lighteval 첫 실행)로 돌려 봤다
- 100문항 비교에서 난 점수 차이는 **구현 차이가 아니라 문항 차이**였다
- 문항 집합을 맞추고 435문항 전체를 돌리자 **출력과 채점이 문항 단위로 모두 일치**했다
- 조건(chat template, 장치, dtype, 디코딩)을 하나씩 맞춰야 비교가 성립한다

점수가 다르면 구현을 의심하기 쉽다. 하지만 먼저 확인할 것은 **같은 문제를 풀었는지**다.

남은 일은 나머지 서브셋(nurse, pharm, dentist)과 0-shot 전체로 같은 비교를 넓히는 것이다.

