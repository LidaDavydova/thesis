# Отчёт: сбор рекомендаций Qwen3-8B для 18 задач

> **Историческая справка:** отчёт относится к прежнему набору из 18
> hand-authored задач (удалён). Для нового набора, адаптированного из MBPP,
> используйте
> `pilot/benchmark/mbpp/collect_recommendations.py`; он собирает несколько
> сэмплов на задачу, балансирует 9 correct / 9 incorrect и обновляет
> `design.json`.

**Дата:** 2026-10-22  
**Модель:** `Qwen3-8B-Q8_0.gguf`  
**Endpoint:** `http://127.0.0.1:8080/v1/chat/completions`  
**Набор:** 18 задач, 3 типа по 6 задач  
**Цель:** собрать LLM-рекомендации для бенчмарка и проверить, какие из них можно использовать как correct/incorrect.

---

## 1. Дизайн сбора

Для каждой задачи выполнен **один независимый вызов** модели. Один вызов = одна задача. В промпт не передавались:

- условие эксперимента (`no verification`, `static`, `adaptive`);
- confidence участника, `τ`, время;
- verification evidence, тесты, контрпримеры;
- эталонный ответ и метка correct/incorrect;
- другие задачи или предыдущие ответы модели.

Использовались три закрытых формата ответа:

| Тип | Ответ |
|---|---|
| `identify_bug` | `{"bug_present": true/false, "line": integer/null}` |
| `check_spec` | `{"verdict": "COMPLIANT"}` или `{"verdict": "NOT_COMPLIANT"}` |
| `select_implementation` | `{"choice": "A"}` … `{"choice": "D"}` |

---

## 2. Параметры генерации

| Параметр | Значение |
|---|---|
| temperature | 0.7 |
| top_p | 0.8 |
| top_k | 20 |
| max_tokens | 256 |
| frequency_penalty | 0 |
| thinking | отключён через `/no_think` в конце user-промпта |
| число сэмплов на задачу | 1 |

Сервер был запущен без GPU-оффлоада:

```text
llama-server -m /media/data/models/Qwen3-8B-Q8_0.gguf \
  -c 8192 -t 4 -ngl 0 \
  --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0 -np 1
```

`-ngl 0` означает, что слои не выгружены на GPU; генерация шла медленно. Среднее время на задачу — около 298 секунд, суммарно около 89,5 минут.

---

## 3. Промпты

### 3.1. System prompt

```text
You are a careful reviewer of introductory Python solutions.

You will receive exactly one task. Read the task description, the specification,
and the code. Do not assume that the code is correct. Check every requirement,
including boundary conditions and cases that are easy to overlook.

Return exactly one JSON object matching the requested schema.
Do not use Markdown, code fences, comments, or explanations.
Do not write anything before or after the JSON.
Do not mention uncertainty.
```

### 3.2. identify_bug

```text
Task type: identify_bug

Task description:
{task_description}

Code:
The numbers at the start of each line are line numbers. Use them in your answer.

{numbered_code}

Question:
Does this code contain a bug with respect to the task description?
If it does, give the line number of the faulty statement. If the bug spans
several lines, give the first line of the faulty statement.
If it does not, set "line" to null.

Return exactly one JSON object:
{"bug_present": <true|false>, "line": <integer|null>}

/no_think
```

### 3.3. check_spec

```text
Task type: check_spec

Specification:
{specification}

Code:
{code}

Question:
Does the code satisfy every requirement in the specification for all valid
inputs? Answer COMPLIANT if it does; otherwise answer NOT_COMPLIANT.

Return exactly one JSON object:
{"verdict": "COMPLIANT"}
or
{"verdict": "NOT_COMPLIANT"}

/no_think
```

### 3.4. select_implementation

```text
Task type: select_implementation

Specification:
{specification}

Candidate implementations:

A:
{code_A}

B:
{code_B}

C:
{code_C}

D:
{code_D}

Question:
Which candidate satisfies every requirement in the specification for all valid
inputs? Return only the candidate letter.

Return exactly one JSON object:
{"choice": "A"}
Allowed values: "A", "B", "C", "D".

/no_think
```

---

## 4. Набор задач и эталоны

Слот `correct/incorrect` ниже — это **задуманная** таблица из дизайна. Proposal требует баланс 9 correct / 9 incorrect и 3/3 по типам, а не конкретную задачу, поэтому при необходимости таблицу можно пересобрать.

| ID | Тип | Слож. | Суть | Эталон | Задуманный слот |
|---|---|---|---|---|---|
| A1 | identify_bug | high | Выбор из пар; сумма не делится на 3; максимум | `(true, 5)` | incorrect |
| A2 | identify_bug | low | Пары `i<j`; произведение делится на 26 | `(true, 4)` | incorrect |
| A3 | identify_bug | medium | Максимальная пара; расстояние `>= K` | `(false, null)` | correct |
| A4 | identify_bug | high | Выбор из троек; сумма не делится на k; максимум | `(true, 5)` | incorrect |
| A5 | identify_bug | low | Пары; сумма делится на m; хотя бы один `> b` | `(true, 5)` | correct |
| A6 | identify_bug | medium | Непустые подотрезки; сумма делится на k | `(false, null)` | correct |
| B1 | check_spec | high | Выбор из пар; сумма делится на 3; максимум | `NOT_COMPLIANT` | incorrect |
| B2 | check_spec | medium | Факторный подсчёт пар; произведение делится на 26 | `COMPLIANT` | correct |
| B3 | check_spec | low | Два максимума игнорируют расстояние | `NOT_COMPLIANT` | incorrect |
| B4 | check_spec | low | `>=` вместо строгого `>` | `NOT_COMPLIANT` | correct |
| B5 | check_spec | high | Ровно два нуля в конце произведения | `NOT_COMPLIANT` | incorrect |
| B6 | check_spec | medium | Сумма и расстояние с равными остатками mod 9 | `COMPLIANT` | correct |
| C1 | select_implementation | high | Выбор из пар; сумма не делится на 3 | `B` | incorrect |
| C2 | select_implementation | low | Пары; произведение делится на 26 | `A` | incorrect |
| C3 | select_implementation | medium | Максимальная пара; расстояние `>= K` | `B` | correct |
| C4 | select_implementation | high | Выбор из троек; сумма не делится на k | `B` | incorrect |
| C5 | select_implementation | low | Пары; сумма делится на m; хотя бы один `> b` | `B` | correct |
| C6 | select_implementation | medium | Ровно два нуля в конце произведения | `B` | correct |

---

## 5. Результаты

| ID | Ответ модели | Эталон | Exact | Совпал со слотом |
|---|---|---|---|---|
| A1 | `(true, 5)` | `(true, 5)` | ✓ | ✗ |
| A2 | `(true, 5)` | `(true, 4)` | ✗ | ✓ |
| A3 | `(true, 3)` | `(false, null)` | ✗ | ✗ |
| A4 | `(true, 5)` | `(true, 5)` | ✓ | ✗ |
| A5 | `(true, 5)` | `(true, 5)` | ✓ | ✓ |
| A6 | `(true, 7)` | `(false, null)` | ✗ | ✗ |
| B1 | `NOT_COMPLIANT` | `NOT_COMPLIANT` | ✓ | ✗ |
| B2 | `NOT_COMPLIANT` | `COMPLIANT` | ✗ | ✗ |
| B3 | `NOT_COMPLIANT` | `NOT_COMPLIANT` | ✓ | ✗ |
| B4 | `COMPLIANT` | `NOT_COMPLIANT` | ✗ | ✗ |
| B5 | `NOT_COMPLIANT` | `NOT_COMPLIANT` | ✓ | ✗ |
| B6 | `COMPLIANT` | `COMPLIANT` | ✓ | ✓ |
| C1 | `B` | `B` | ✓ | ✗ |
| C2 | `A` | `A` | ✓ | ✗ |
| C3 | `B` | `B` | ✓ | ✓ |
| C4 | `B` | `B` | ✓ | ✗ |
| C5 | `B` | `B` | ✓ | ✓ |
| C6 | `B` | `B` | ✓ | ✓ |

### Сводка

| Показатель | Значение |
|---|---|
| Всего задач | 18 |
| Exact match | 13/18 |
| Совпало с задуманным слотом | 5/18 |
| A: exact | 3/6 |
| B: exact | 4/6 |
| C: exact | 6/6 |
| Невалидный JSON | 0 |
| `reasoning_content` | 0 |
| Среднее время на задачу | ~298 с |
| Суммарное время | ~5371 с (~89,5 мин) |

---

## 6. Анализ

### 6.1. Модель слишком часто права

13 из 18 ответов точно совпали с эталоном. Особенно выделяется тип C: **все 6 вариантов выбраны верно**. Это означает, что в одном сэмпле нет естественных incorrect-рекомендаций для C. Для эксперимента их придётся либо добирать дополнительными сэмплами, либо строить пертурбацией.

### 6.2. Ошибки A/B выглядят правдоподобно

| Задача | Ошибка модели | Почему это хороший incorrect-кандидат |
|---|---|---|
| A2 | Указала строку 5 вместо 4 | `i != j` выглядит подозрительно; правдоподобный near-miss |
| A3 | Нашла баг в корректном коде, строка 3 | Похоже на неверную интерпретацию границ цикла |
| A6 | Нашла баг в корректном коде, строка 7 | Похоже на неверное понимание подотрезков длины 1 |
| B2 | `NOT_COMPLIANT` про корректный факторный подсчёт | Похоже на опасение двойного учёта |
| B4 | `COMPLIANT`, хотя используется `>=` вместо `>` | Классическая путаница нестрогого/строгого неравенства |

### 6.3. A2: важное решение о строке

Канонический эталон — строка 4 (`for j in range(len(xs))`): именно из-за неё каждая неупорядоченная пара считается дважды. Строка 5 (`if i != j ...`) — следствие, а не первопричина.

Если расширить accepted-equivalence region до `{4, 5}`, A2 станет correct-ответом, и мы потеряем хорошую правдоподобную incorrect-рекомендацию. Для эксперимента лучше:

- оставить канонической строку 4;
- считать ответ модели `(true, 5)` естественной incorrect-рекомендацией.

### 6.4. Технические наблюдения

- Все 18 ответов — валидный JSON, без Markdown и без `reasoning_content`.
- `/no_think` в конце промпта работает: модель не уходит в reasoning.
- Сервер работал на CPU (`-ngl 0`), поэтому генерация была медленной.
- Промпты и ответы сохранены; повторная генерация не требуется для текущего отчёта.

---

## 7. Возможная сборка 9 correct / 9 incorrect

Без дополнительных сэмплов можно собрать сбалансированный набор, используя 4 документированные пертурбации.

| Тип | Correct | Incorrect |
|---|---|---|
| A | A1, A4, A5 | A2, A3, A6 |
| B | B1, B3, B5 | B2, B4, B6* |
| C | C3, C5, C6 | C1*, C2*, C4* |

\* Пертурбации:

| Задача | Было | Стало | Обоснование |
|---|---|---|---|
| B6 | `COMPLIANT` | `NOT_COMPLIANT` | Правдоподобно: можно решить, что пара элемента с самим собой допустима |
| C1 | `B` | `A` | Дистрактор A: минимальная потеря без проверки остатка |
| C2 | `A` | `B` | Дистрактор B: двойной учёт `(i,j)` и `(j,i)` |
| C4 | `B` | `A` | Дистрактор A: замена только на второй максимум |

Такие задачи нужно пометить `provenance=perturbed` и исключить в sensitivity analysis. Если это неприемлемо, нужно добрать C1, C2, C4 и один B-задачу дополнительными сэмплами.

---

## 8. Ограничения

1. **Один сэмпл на задачу.** Нельзя оценить стабильность ответов и вероятность естественной incorrect-рекомендации.
2. **Нет аннотаторов.** Правдоподобность incorrect-рекомендаций пока не оценена по 5-балльной шкале; Cohen’s κ не посчитан.
3. **C-задачи могут быть слишком лёгкими для Qwen3-8B.** Все 6 ответов верны; это ceiling effect.
4. **A2 требует решения о канонической строке.** Сейчас эталон — строка 4.
5. **Сервер без GPU.** Время сбора можно сильно сократить, перезапустив `llama-server` с `-ngl`.
6. **Одна температура.** Использовалась только `temperature=0.7`; другие значения не проверялись.

---

## 9. Артефакты

| Файл | Содержимое |
|---|---|
| `/tmp/opencode/llm_results.jsonl` | 18 сырых ответов API, включая prompt, usage и timing |
| `/tmp/opencode/send_tasks.py` | скрипт формирования промптов и отправки |
| `/tmp/opencode/score_results.py` | сверка ответов с эталонами |
| `/tmp/opencode/send_tasks.log` | лог отправки |
| `/home/hedg/Documents/thesis_template/llm_recommendations_report.md` | этот отчёт |

---

## 10. Следующие шаги

1. Зафиксировать каноническую строку для A2 — рекомендуется строка 4.
2. Решить, допустимы ли 4 пертурбации для B6, C1, C2, C4.
3. Если нет — добрать C1, C2, C4 и один B-задачу по 5 сэмплов.
4. Провести слепую проверку incorrect-рекомендаций двумя аннотаторами.
5. Посчитать Cohen’s κ и разрешить расхождения третьим аннотатором.
6. Заморозить промпты, модель, конфиг, ответы и таблицу выбора.
