---
title: "가중평균이냐 단순평균이냐: KorMedMCQA를 lighteval로 옮기며 한 고민"
date: 2026-10-31 15:30:00 +0900
categories: [LLM, Benchmark]
tags: [kormedmcqa, lighteval, lm-evaluation-harness, llm-evaluation, open-source, weighted-average]
math: false
mermaid: false
---

## 들어가며

KorMedMCQA의 lm-evaluation-harness 구현을 lighteval 태스크로 옮기고 있다. 프롬프트, 정답 추출, few-shot은 코드를 옮기면 맞출 수 있었다. [지난 글](/posts/kormedmcqa-three-ways/)에서 의사 435문항의 출력과 채점이 문항 단위로 모두 일치하는 것도 확인했다.

끝까지 남은 문제는 코드 한 줄짜리처럼 보이는 **전체 평균**이었다. harness는 문항 수로 가중하고, lighteval은 단순 평균만 낸다.

고민은 이것이었다.

> 논문과 같은 가중평균을 내려고 코드를 더 넣을 것인가, lighteval의 다른 벤치마크처럼 단순 평균을 따를 것인가.

결론부터 적는다.

- **첫 PR은 단순 평균(`_average`)으로 낸다.** 직종별 점수만 논문과 비교한다.
- **차이는 숨기지 않고 문서화한다.** 점수가 낮은 모델은 차이가 1점을 넘는다.
- **가중 집계는 KorMedMCQA 전용 코드가 아니라 일반 기능으로 따로 제안한다.** 조사해 보니 비교한 5개 중 집계 방식을 고를 수 없다고 확인된 도구는 lighteval뿐이었다(HELM은 고를 수 있는지 확인하지 못했다).

이 결론에 오기까지 판단이 두 번 바뀌었다. 이 글은 그 과정을 기록한다.

## 1. harness를 lighteval로 옮길 때 달라지는 것

평균 이야기 전에 전체 그림부터 정리한다. 두 도구의 코드를 나란히 놓고 보니 차이는 세 종류였다.

| 종류 | 항목 |
|---|---|
| 구조가 달라서 새로 짠다 | 직종 묶기, 전체 평균, 답 추출 |
| 이름과 위치만 바뀐다 | 정답 표현, few-shot, 스플릿, 생성 설정 |
| 같은 설정인데 결과가 달라질 수 있다 | limit 문항 선택, 평균 방식, chat template |

### 새로 짜는 것

| 항목 | harness | lighteval |
|---|---|---|
| 직종 묶기 | group yaml 하나 | 태스크 이름의 `:` 앞부분이 같으면 묶임 |
| 답 추출 | `filter_list`로 가공한 뒤 `exact_match` | filter 단계가 없어서 메트릭 클래스 안에서 추출 |
| 전체 평균 | `weight_by_size: true` | `_average`에 단순 평균 고정 |

답 추출은 위치만 옮겼다. 정규식과 `extract_choice`는 그대로 쓰고, 호출 위치를 메트릭의 `compute()` 안으로 바꿨다.

```python
class ExtractChoiceAcc(SampleLevelComputation):
    def compute(self, doc: Doc, model_response: ModelResponse, **kwargs) -> float:
        pred = extract_choice(model_response.final_text[0])
        gold = doc.get_golds()[0].strip()
        return float(pred == gold)
```

### 이름만 바뀌는 것

| harness | lighteval |
|---|---|
| `sampler: first_n` | `few_shots_select="sequential"` |
| `--num_fewshot 5` | 태스크 이름의 `\|5` |
| `test_split`, `fewshot_split` | `evaluation_splits`, `few_shots_split` |
| `until` | `stop_sequence` |
| `max_gen_toks` | `generation_size` |

이름만 바뀌어도 **기본값은 다르다.** 안 적으면 조용히 다른 실험이 된다.

| 안 적었을 때 lighteval 기본값 | 결과 |
|---|---|
| `evaluation_splits` → `validation` | 평가 split이 달라진다 |
| `few_shots_split` → `train`이 먼저 선택됨 | harness의 `fewshot` split을 안 쓴다 |
| `few_shots_select` → `balanced` | 같은 5문항이라도 순서가 바뀐다 |
| `override_chat_template` → 토크나이저를 보고 자동 판단 | harness 기본과 달리 chat template이 켜질 수 있다 |

프롬프트 코드는 달라 보였다. harness는 few-shot 템플릿에 정답을 넣고, lighteval은 `query + " " + 정답`으로 붙인다. 그런데 디코딩해서 비교하니 5-shot 접두부 1,914자가 완전히 같았다. 여기까지는 **맞추면 맞춰지는 문제**였다.

평균은 달랐다. 맞출 설정 자체가 없었다.

## 2. 문제: 직종 점수는 같은데 전체 점수는 다르다

harness는 group yaml에서 이렇게 합친다.

```yaml
aggregate_metric_list:
  - metric: exact_match
    aggregation: mean
    weight_by_size: true
    filter_list: extract_answer
```

lighteval은 `info_loggers.py`에 계산이 고정돼 있다(커밋 `c2af9e5f`).

```python
self.metric_aggregated[average_task] = {
    metric: sum(self.metric_aggregated[k][metric] for k in list_of_subtasks) / len(list_of_subtasks)
    for metric in metrics
}
```

KorMedMCQA는 직종별 test 문항 수가 다르다. 그래서 두 평균은 비중부터 다르다.

| 직종 | test 문항 | 가중평균 비중 | 단순평균 비중 |
|---|---:|---:|---:|
| 의사 | 435 | 14.5% | 25% |
| 간호사 | 878 | 29.2% | 25% |
| 약사 | 885 | 29.4% | 25% |
| 치과의사 | 811 | 27.0% | 25% |
| 합계 | 3,009 | 100% | 100% |

의사만 비중이 크게 다르다. 단순 평균은 의사 점수를 가중평균보다 10.5포인트 더 반영한다.

논문의 평균이 정말 가중평균인지도 검산했다. 내 노트에 옮겨 둔 논문 Table 1의 10개 행(모델 9개와 응시자 평균)을 문항 수로 가중하니 **10행 모두 논문 값과 일치**했다.

## 3. 따져 본 선택지 네 개

처음 떠올린 방법은 네 가지였다.

| 방법 | 장점 | 문제 |
|---|---|---|
| A. 단순 평균을 그대로 쓴다 | 추가 코드가 없다 | 전체 평균이 논문과 다르다 |
| B. 사후 스크립트로 계산한다 | 5-shot에서도 harness와 같은 계산이다 | 태스크 밖의 별도 도구라 PR에 넣기 어렵다 |
| C. 네 직종을 이어 붙인 통합 태스크를 둔다 | lighteval 결과에 바로 나온다 | 몽키패치가 필요하고 5-shot에서 값이 달라진다 |
| D. `_average` 계산을 가중으로 바꾼다 | 근본적으로 고친다 | 다른 벤치마크의 점수까지 전부 바뀐다 |

### B. 사후 스크립트

결과 파일에서 직종별 점수와 문항 수를 읽어 직접 계산한다.

```python
total_n = sum(n for _, _, n in rows)
weighted = sum(score * n for _, score, n in rows) / total_n
simple = sum(score for _, score, _ in rows) / len(rows)
```

계산은 정확하다. 그러나 lighteval을 실행한 사람은 이 스크립트를 따로 알아야 한다. 개인 검증용으로는 충분하지만 태스크 PR에 넣을 수는 없다.

### C. 통합 태스크

문항을 전부 이어 붙이면 평균이 곧 문항 가중이다. 깔끔해 보이지만 함정이 두 개 있었다.

첫째, lighteval은 subset 하나만 읽는다. 네 직종을 합치려면 `LightevalTask.download_dataset_worker`를 가로채야 한다.

```python
_orig_download_dataset_worker = LightevalTask.download_dataset_worker

def _download_dataset_worker(task: LightevalTask) -> DatasetDict:
    if task.dataset_config_name != ALL:
        return _orig_download_dataset_worker(task)
    parts = [load_dataset(path=task.dataset_path, name=s, ...) for s in SUBSETS]
    dataset = DatasetDict({split: concatenate_datasets([p[split] for p in parts]) for split in parts[0]})
    ...

LightevalTask.download_dataset_worker = staticmethod(_download_dataset_worker)
```

실험 코드로는 괜찮다. 프레임워크 메서드를 바꿔치기하는 코드를 벤치마크 PR에 넣기는 어렵다.

둘째, **이 방식은 5-shot에서 논문과 다른 값을 낸다.** `fewshot` split도 같이 이어 붙기 때문이다. 풀은 doctor 5개가 맨 앞에 오는 20개가 되고, `sequential`은 앞의 5개만 쓴다. 그러면 간호사, 약사, 치과의사 문항도 전부 의사 예시를 받는다. harness는 직종마다 자기 예시를 쓴다.

평균을 맞추려고 만든 태스크가 프롬프트를 틀어 버리는 셈이다. 0-shot에서만 harness의 가중평균과 같다. 이 판단은 코드를 읽어서 한 것이고, 통합 태스크를 실제로 돌려 보지는 않았다.

이름도 조심해야 한다. `kormedmcqa:all`처럼 `:`를 쓰면 `kormedmcqa:_average` 묶음에 섞인다. 그래서 `kormedmcqa_all`로 지었다.

### D. `_average` 자체를 바꾸기

`mmlu`도 과목별 문항 수가 다른데 `mmlu:_average`는 단순 평균이다. `_average`를 바꾸면 기존 벤치마크의 숫자가 모두 달라진다. 새 벤치마크를 추가하는 첫 PR의 범위가 아니다.

이 시점의 첫 판단은 **A(단순 평균)** 였다. 이유는 네 가지였다.

1. 업스트림 관례와 맞는다. KorMedMCQA만 다르면 리뷰어가 왜 다르냐고 물을 것이다.
2. 가중평균을 넣으려면 프레임워크를 건드려야 한다.
3. 직종별 점수는 평균 방식과 무관하게 나온다.
4. 일반 기능은 나중에 따로 제안할 수 있다.

그런데 "관례"가 진짜 근거인지는 아직 몰랐다. lighteval이 **일부러** 단순 평균을 택한 것이라면 제안할 필요가 없다. 다른 도구들이 다 단순 평균이라면 더더욱 그렇다. 그래서 조사했다.

## 4. 다른 평가 도구는 어떻게 풀었나

같은 상황(서브태스크별 문항 수가 다른 벤치마크)을 다른 도구는 어떻게 처리하는지 MMLU를 기준으로 확인했다.

| 도구 | 기본 | 다른 쪽을 고를 수 있나 | 확인한 곳 |
|---|---|---|---|
| lm-evaluation-harness | 가중 | `weight_by_size: false` | `lm_eval/config/group.py` |
| HELM | 가중 | 확인 못 함 | `summarize.py`, `statistic.py` |
| Inspect (`inspect_evals`) | 가중 | `grouped(all="groups")` | `mmlu.py`, `grouped.py` |
| OpenCompass | 단순 | 별도 그룹 `mmlu-weighted` | `summarizers/default.py`, `groups/mmlu.py` |
| lighteval | 단순 | 없음 | `info_loggers.py` |

도구마다 푸는 방식이 달랐다.

**harness는 설정 한 줄로 푼다.** `weight_by_size`의 기본값은 `true`다. 문서는 이것을 micro-average라고 부르고, `false`면 macro-average라고 설명한다. KorMedMCQA yaml의 `true`는 기본값과 같다. 그래서 논문 때문에 일부러 넣었다고 단정할 근거는 없다.

**HELM은 구조상 가중이다.** 여러 실행을 한 칸으로 합칠 때 `Stat.merge`를 쓴다. 이 메서드는 평균끼리 평균 내지 않고 `sum`과 `count`를 각각 더한다.

```python
def merge(self, other: "Stat") -> "Stat":
    self.sum += other.sum
    ...
    self.count += other.count
```

전체 문항을 한 덩어리로 평균 내는 것과 같다.

**Inspect는 메트릭 인자로 고른다.** `inspect_evals`의 MMLU는 `scorer=choice()`만 쓴다. `choice()`의 메트릭은 `accuracy()`이고 전체 샘플 평균이다. 과목별로 나누고 싶으면 `grouped()`를 쓰는데, 인자가 그대로 macro와 micro의 선택이다.

```python
all: Literal["samples", "groups"] | Literal[False] = "samples"
```

`"samples"`는 전체 문항 평균이고 `"groups"`는 그룹 점수의 평균이다.

**OpenCompass는 이름을 두 개 둔다.** 이 도구가 지금 고민에 가장 가까웠다. 헤드라인 `mmlu`는 단순 평균이고, 문항 수를 `weights`로 준 `mmlu-weighted`를 옆에 둔다.

```python
mmlu_summary_groups.append({'name': 'mmlu', 'subsets': _mmlu_all})
mmlu_summary_groups.append({'name': 'mmlu-weighted', 'subsets': _mmlu_all, 'weights': _mmlu_weights})
```

### 첫 조사에서 판단을 잘못했다

처음에는 표가 지금과 달랐다. HELM과 Inspect 칸이 비어 있었다. 읽은 파일이 집계 코드가 아니라 실행 설정과 스키마였기 때문이다.

그 상태로 harness는 가중, OpenCompass와 lighteval은 단순이라고 보고 **"lighteval만 특이한 건 아니다"** 라고 판단했다. 그런데 비어 있던 칸은 "모른다"는 뜻이었다. 집계 코드(`summarize.py`, `statistic.py`, `grouped.py`)를 직접 내려받아 읽자 결론이 바뀌었다.

| | 첫 판단 | 소스를 읽은 뒤 |
|---|---|---|
| 기본값 | 도구마다 갈린다 | 5개 중 3개가 가중 |
| 고를 수 있는가 | 일부 도구만 | 3개는 고를 수 있고 HELM은 미확인 |
| lighteval | 특이하지 않다 | 고를 방법이 없다고 확인된 유일한 도구 |

단순 평균이 기본인 것은 특이하지 않다. OpenCompass도 그렇다. **고를 수 없다는 것**이 특이하다. 질문을 "lighteval의 기본값이 이상한가"에서 "lighteval에 선택지가 있는가"로 바꾸자 답이 분명해졌다.

## 5. lighteval은 왜 단순 평균만 있을까

다음 질문은 이것이었다. 이유가 있어서 막아 둔 것이라면 제안해도 소용이 없다.

| 기록 | 내용 |
|---|---|
| [PR #41](https://github.com/huggingface/lighteval/pull/41) (2024-02 머지) | `_average` 자동 계산을 추가했다. PR 본문이 비어 있다 |
| 이후 변경 | 내가 찾은 변경 중 평균 계산 방식을 바꾼 것은 없다 |
| "weighted average" 이슈 검색 | [#1240](https://github.com/huggingface/lighteval/issues/1240) 하나. 벤치마크 간 IRT 점수화 제안이라 서브태스크 집계와 결이 다르다 |
| "macro average" 이슈 검색 | [#1424](https://github.com/huggingface/lighteval/issues/1424) 하나. 같은 스프린트의 KMMLU 제안으로, micro와 macro 평균을 함께 보고하겠다는 문장이 있다 |
| 문항 수를 결과에 싣는 작업 | 이슈 [#804](https://github.com/huggingface/lighteval/issues/804), PR [#851](https://github.com/huggingface/lighteval/pull/851), [#1280](https://github.com/huggingface/lighteval/pull/1280)은 열려 있고 [#1285](https://github.com/huggingface/lighteval/pull/1285)는 닫혔다 |

**단순 평균을 택한 이유를 적은 기록은 찾지 못했다. 가중평균을 제안했다가 거부된 기록도 찾지 못했다.** 그래서 "일부러 막았다"기보다 "처음 구현한 형태로 남아 있다"에 가깝다고 본다. 다만 이것은 기록이 없다는 데서 한 추론이다. 키워드 검색이라 놓친 이슈가 있을 수 있고, 메인테이너의 내부 논의는 볼 수 없다.

### 합친 태스크를 두는 선례

전부 `_average` 하나로만 처리하는 것도 아니었다. 벤치마크 3개는 서브태스크 옆에 전체를 합친 태스크를 함께 둔다.

| 벤치마크 | 합친 태스크 | 서브태스크 예시 |
|---|---|---|
| `bbq` | `name="bbq"`, `hf_subset="all"` | `bbq:Age` |
| `lsat_qa` | `name="lsat_qa"`, `hf_subset="all"` | `lsat_qa:assignment` |
| `bold` | `name="bold"`, `hf_subset="all"` | `bold:gender` |

"합친 태스크를 둘 수 있다"는 근거로는 쓸 수 있다. 다만 그대로 따라 하기는 어렵다.

- 세 벤치마크는 데이터셋에 `all` config가 있다. KorMedMCQA의 config는 `doctor`, `nurse`, `pharm`, `dentist` 네 개뿐이다. 그래서 몽키패치의 선례는 되지 않는다.
- 세 벤치마크는 이름에 콜론이 없다. 코드를 읽은 바로는 `bbq`와 `bbq:Age`가 같은 `bbq:_average` 묶음에 들어간다. 함께 돌리면 합친 태스크가 평균에 한 번 더 섞일 수 있다. 실행해서 확인하지는 않았다.

## 6. 그래서 차이는 얼마나 나나

처음 정리할 때는 숫자가 GPT-4o 하나뿐이었다. 가중 85.61, 단순 85.53. 0.08점 차이다. 이 숫자만 보면 고민할 가치가 없어 보인다.

그래서 처음에 이런 기준을 세워 뒀다. **차이가 1점 이상이면 단순 평균 쪽 근거가 약해진다.** 모델을 새로 돌릴 필요 없이 논문 표의 직종별 점수로 계산할 수 있어서 전부 계산해 봤다.

| 대상 | 의사 | 간호사 | 약사 | 치과의사 | 가중(논문) | 단순 | 차이 |
|---|---:|---:|---:|---:|---:|---:|---:|
| o1-preview | 92.41 | 95.10 | 94.24 | 88.66 | 92.72 | 92.60 | +0.12 |
| Claude-3.5-Sonnet | 88.05 | 92.14 | 91.41 | 74.23 | 86.51 | 86.46 | +0.05 |
| GPT-4o | 85.98 | 91.46 | 85.99 | 78.67 | 85.61 | 85.53 | +0.08 |
| Gemini-1.5-pro | 79.31 | 88.27 | 80.34 | 70.28 | 79.79 | 79.55 | +0.24 |
| Qwen2.5-72B | 77.24 | 87.47 | 82.71 | 66.21 | 78.86 | 78.41 | +0.45 |
| Llama-3.1-70B | 72.18 | 82.23 | 77.74 | 63.13 | 74.31 | 73.82 | +0.49 |
| Meditron3-70B | 70.57 | 82.69 | 75.82 | 62.64 | 73.51 | 72.93 | +0.58 |
| EXAONE-3.0-7.8B | 48.28 | 67.65 | 56.61 | 45.87 | 55.73 | 54.60 | **+1.13** |
| EEVE-10.8B | 48.97 | 66.17 | 58.19 | 42.42 | 54.94 | 53.94 | **+1.00** |
| 응시자 평균(사람) | 79.66 | 79.90 | 71.86 | 78.23 | 77.05 | 77.41 | **-0.36** |

GPT-4o는 표에서 차이가 작은 쪽이었다. 처음에 본 숫자가 대표값이 아니었다.

차이가 왜 이렇게 나는지는 비중 차이로 풀면 보인다.

```
가중 - 단순 = -0.105 × 의사 + 0.042 × 간호사 + 0.044 × 약사 + 0.020 × 치과의사
```

**의사 점수가 간호사, 약사 점수보다 많이 낮을수록 차이가 커진다.** EXAONE은 의사가 48.28, 간호사가 67.65다. 상위 모델은 네 직종 점수가 상대적으로 고르거나 플러스와 마이너스가 상쇄돼 차이가 작다.

사람은 부호가 반대다. 약사 점수만 낮고 의사 점수가 높기 때문이다. 그래서 **모델과 사람의 격차는 평균 방식에 더 민감하다.**

| Qwen2.5-72B와 사람의 격차 | 값 |
|---|---:|
| 가중평균 기준 | 78.86 - 77.05 = **+1.81** |
| 단순평균 기준 | 78.41 - 77.41 = **+1.00** |

모델 점수가 0.45 낮아지고 사람 점수가 0.36 높아지면서 격차가 거의 절반이 됐다. 표의 10개 행 안에서 순위는 그대로였다. 나머지 모델까지 계산한 것은 아니다.

이 표는 논문의 반올림 값으로 계산했다. 내가 돌린 결과가 아니다. 내 실행은 아직 의사 서브셋뿐이라서 `weighted_average.py`로 실제 네 직종을 합쳐 본 적은 없다.

## 7. 최종 결정

세운 기준을 넘긴 모델이 나왔다. 그래서 "차이가 작으니 괜찮다"는 근거는 버렸다. 그렇다고 결론을 뒤집지는 않았다. 처음 적어 둔 예외도 "차이가 커도 코드를 추가하기보다 업스트림에 제안한다"였다. **KorMedMCQA 전용 코드가 답이 아니라는 판단은 차이의 크기와 상관이 없기 때문이다.**

| 질문 | 결정 | 이유 |
|---|---|---|
| 이번 PR의 전체 평균 | `_average` 단순 평균 | 몽키패치 없이 태스크만 추가한다 |
| 논문과 비교하는 기준 | 직종별 점수만 | 평균 방식과 무관하게 같은 계산이다 |
| PR 본문 | 차이를 문서화 | `_average`를 논문 평균과 바로 비교하면 틀린다 |
| 가중 집계 | 별도의 일반 제안 | 한 벤치마크의 문제가 아니라 도구의 빈칸이다 |
| 통합 태스크 | PR에 넣지 않음 | 몽키패치가 필요하고 5-shot에서 프롬프트가 달라진다 |
| `weighted_average.py` | 개인 검증용 | 단순과 가중을 같이 찍는 도구로 쓴다 |

PR 본문에는 이 내용을 적을 것이다.

- harness와 논문은 문항 수 가중, lighteval의 `kormedmcqa:_average`는 단순 평균이다.
- 직종별 점수는 비교할 수 있지만 전체 평균은 그대로 비교하면 안 된다.
- 차이는 모델에 따라 다르다. 논문 표 기준으로 +0.05에서 +1.13이고, 사람 평균은 -0.36이다.
- 논문과 같은 전체 평균은 직종별 점수와 문항 수(435, 878, 885, 811)로 계산한다.

일반 제안의 근거도 조사 덕분에 생겼다. harness의 `weight_by_size`, OpenCompass의 `weights`, Inspect의 `grouped(all=...)`가 같은 문제에 대한 세 가지 해법이다. KMMLU 제안도 micro와 macro를 함께 다루므로 같이 논의할 수 있다.

걸리는 점도 있다. 가중 집계에는 태스크별 문항 수가 필요한데, 샘플 수를 결과에 싣는 PR들이 아직 머지되지 않았다. 일반 제안은 이 작업과 함께 논의해야 할 수도 있다. 아직 메인테이너와 멘토에게 묻기 전이라 이 결정은 내 판단이다.

## 8. 아직 확인하지 않은 것

| 항목 | 상태 |
|---|---|
| nurse, pharm, dentist parity | 의사만 비교했다 |
| 내 실행의 가중과 단순 차이 | 네 직종 전체를 돌린 적이 없다 |
| `kormedmcqa_all` 실제 점수 | 코드만 읽었고 실행하지 않았다 |
| HELM에서 집계 방식 선택 | 확인하지 못했다 |
| lighteval이 단순 평균을 택한 이유 | 공개 기록에서 찾지 못했다 |

## 마치며

세 가지가 남았다.

**1. 평균 방식은 구현 세부가 아니라 벤치마크 정의다.** 같은 모델도 사람과의 격차가 1.81에서 1.00으로 달라진다. 전체 점수를 볼 때는 어떻게 합쳤는지부터 확인해야 한다.

**2. 빈칸을 판단 근거로 쓰면 안 된다.** HELM과 Inspect 칸이 비어 있는 채로 결론을 내렸다가 뒤집혔다. "확인 못 함"을 "해당 없음"처럼 읽은 탓이다. 답은 집계 코드를 직접 여는 데 있었다.

**3. 숫자 하나로 크기를 판단하면 안 된다.** GPT-4o의 0.08점만 보면 사소한 문제였다. 같은 계산을 전체 표로 넓히자 1점을 넘는 모델이 나왔다.

처음 질문은 "코드를 더 넣을까, 관례를 따를까"였다. 조사가 끝나고 나니 질문이 바뀌었다. 이번 PR에서는 관례를 따르고, **관례에 빠진 선택지는 따로 제안한다.** 다음은 네 직종을 모두 돌려 내 실행의 차이를 재는 것이다.

## 참고

- [lm-evaluation-harness groups 문서](https://lm-evaluation-harness.readthedocs.io/writing_tasks/groups_and_benchmarks/)
- [HELM summarize.py](https://github.com/stanford-crfm/helm/blob/main/src/helm/benchmark/presentation/summarize.py), [statistic.py](https://github.com/stanford-crfm/helm/blob/main/src/helm/benchmark/metrics/statistic.py)
- [inspect_evals mmlu.py](https://github.com/UKGovernmentBEIS/inspect_evals/blob/main/src/inspect_evals/mmlu/mmlu.py), [inspect_ai grouped.py](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/scorer/_metrics/grouped.py)
- [OpenCompass default.py](https://github.com/open-compass/opencompass/blob/main/opencompass/summarizers/default.py), [groups/mmlu.py](https://github.com/open-compass/opencompass/blob/main/opencompass/configs/summarizers/groups/mmlu.py)
- [KorMedMCQA 데이터셋](https://huggingface.co/datasets/sean0042/KorMedMCQA), 논문 [arXiv 2403.01469v3](https://arxiv.org/abs/2403.01469v3)
