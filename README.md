# Відновлення LM Studio + Continue setup на новому ПК

Інструкція для Claude Code: виконай кроки нижче по порядку, щоб повністю
відтворити конфігурацію локальних LLM (LM Studio) та інтеграцію з VS Code
(Continue extension) з іншого ПК користувача.

Джерело: сесія Claude Code, машина `spw@tuf`, дата фіксації 2026-07-17.

## 0. Передумови (перевір і зупинись, якщо не виконано)

- LM Studio встановлено, `lms` CLI доступний (зазвичай `~/.lmstudio/bin/lms`)
- VS Code + розширення Continue встановлено (`~/.vscode/extensions/continue.continue-*`)
- GPU з щонайменше 8 GB VRAM (весь бюджет нижче розрахований під RTX 4070 8GB;
  якщо VRAM більше — ліміти можна послабити, якщо менше — пресети НЕ підійдуть)
- Перевір версію Continue: `ls ~/.vscode/extensions/ | grep continue`.
  Конфіг нижче валідований проти схеми `continue-2.0.0`. Якщо версія інша —
  ОБОВ'ЯЗКОВО звір поля з файлом
  `~/.vscode/extensions/continue.continue-*/config-yaml-schema.json`
  перед застосуванням (див. розділ "Як перевірити конфіг" нижче) — поля
  API між версіями Continue можуть відрізнятись, і минулого разу саме тому
  весь конфіг був невалідний (`completionOptions`/`systemMessage`/
  `tabAutocompleteModel`/`embeddingsProvider` — жодного з цих полів не існує
  в 2.0.0; правильні назви — `defaultCompletionOptions`,
  `chatOptions.baseSystemMessage`, `models[].roles: [autocomplete]`,
  `models[].roles: [embed]`).

## 1. Завантажити моделі

Моделі (45.57 GB разом) потрібно завантажити заново — вони не переносяться
текстовою інструкцією. Команди (виконати по одній, кожна тягне з HuggingFace
через LM Studio):

```bash
export PATH="$HOME/.lmstudio/bin:$PATH"
lms get qwen2.5-coder-1.5b-instruct        # 1.46 GB — tab autocomplete
lms get gemma-4-e2b-it                      # 4.41 GB — fast + vision
lms get deepseek/deepseek-r1-0528-qwen3-8b  # 5.03 GB — reasoning
lms get ornith-1.0-9b                       # 5.63 GB — agentic coding
lms get qwen3.5-9b                          # 6.55 GB — general chat
lms get qwen3.6-14b-a3b-fablevibes          # 6.77 GB — balanced MoE
lms get gemma-4-26b-a4b-it-qat              # 15.63 GB — best quality + vision
lms get text-embedding-nomic-embed-text-v1.5  # 84 MB — embeddings
```

Якщо `lms get <id>` не знаходить модель — відкрий LM Studio GUI → пошук за
цим же ID, завантаж вручну. Перевірка після завантаження:

```bash
lms ls
```

Має показати 8 моделей (7 LLM + 1 embedding), розміри як у коментарях вище.

## 2. Створити ~/bin/lms-pick

```bash
mkdir -p ~/bin
```

Створи файл `~/bin/lms-pick` з точно таким вмістом:

```bash
#!/bin/bash
export PATH="$HOME/.lmstudio/bin:$PATH"

G='\033[0;32m' Y='\033[1;33m' C='\033[0;36m'
R='\033[0;31m' BOLD='\033[1m' DIM='\033[2m' NC='\033[0m'

# name | model_id | vram_gb | role | lms_params
# Примітка: 14B MoE (6.3 GB) не може завантажитись разом з іншими моделями на 8 GB VRAM
# Примітка: Qwen 3.5 9B реально займає ~6.5 GB + прихований vision-модуль — не влазить
# разом з ЖОДНОЮ другою моделлю (перевірено емпірично), тому лише solo
MODELS=(
  "Qwen2.5-Coder 1.5B|qwen2.5-coder-1.5b-instruct|0.9|Tab autocomplete|--gpu max --context-length 8192 --parallel 4 -y"
  "Gemma E2B 4.6B|gemma-4-e2b-it|4.4|Fast + vision|--gpu max --context-length 8192 --parallel 1 -y"
  "DeepSeek R1 8B|deepseek/deepseek-r1-0528-qwen3-8b|5.0|Reasoning|--gpu max --context-length 8192 --parallel 1 -y"
  "Ornith 9B|ornith-1.0-9b|5.6|Agentic coding|--gpu max --context-length 8192 --parallel 1 -y"
  "Qwen 3.5 9B|qwen3.5-9b|6.6|General chat|--gpu max --context-length 8192 --parallel 1 -y"
  "Qwen3.6 14B MoE|qwen3.6-14b-a3b-fablevibes|6.3|Balanced (solo)|--gpu max --context-length 4096 --parallel 1 -y"
  "Gemma 26B QAT|gemma-4-26b-a4b-it-qat|15.6|Best quality + vision|--context-length 8192 --parallel 2 -y"
)

# Пресети (індекси 0-based). ⚠ = перевищує 8 GB або займає весь VRAM
PRESET_NAMES=(
  "Мінімум     — autocomplete (0.9 GB)"
  "Coding      — autocomplete + Ornith 9B (6.5 GB)"
  "Чат         — Qwen 3.5 9B solo (6.6 GB)"
  "Reasoning   — autocomplete + DeepSeek R1 8B (5.9 GB)"
  "Баланс      — Qwen3.6 14B MoE solo (6.3 GB)"
  "Якість      — Gemma 26B solo (RAM offload)"
  "Вибрати вручну"
)
PRESET_IDX=("0" "0 3" "4" "0 2" "5" "6" "")

f() { echo "${MODELS[$1]}" | cut -d'|' -f"$2"; }

vram_sum() {
  local total=0
  for i in $1; do
    total=$(echo "$total + $(f $i 3)" | bc)
  done
  echo $total
}

# ── Header ────────────────────────────────────────────────────────
clear
echo -e "${BOLD}╔══════════════════════════════════════╗${NC}"
echo -e "${BOLD}║       LM Studio Model Launcher       ║${NC}"
echo -e "${BOLD}╚══════════════════════════════════════╝${NC}"

# ── Server ────────────────────────────────────────────────────────
if lms server status 2>/dev/null | grep -q "running"; then
  echo -e "\n${G}✓${NC} Сервер активний → http://127.0.0.1:1234"
else
  echo -e "\n${Y}⟳${NC} Запускаю сервер..."
  lms server start --port 1234 2>/dev/null
  echo -e "${G}✓${NC} Сервер запущено → http://127.0.0.1:1234"
fi

# ── Поточні моделі ────────────────────────────────────────────────
LOADED=$(lms ps 2>/dev/null | grep -E "IDLE|RUNNING|LOADING" | awk '{print $1}')
if [ -n "$LOADED" ]; then
  echo -e "\n${DIM}Зараз завантажено:${NC}"
  while IFS= read -r m; do
    [ -n "$m" ] && echo -e "  ${DIM}• $m${NC}"
  done <<< "$LOADED"
fi

# ── Пресети ───────────────────────────────────────────────────────
echo -e "\n${BOLD}Пресети:${NC}"
for i in "${!PRESET_NAMES[@]}"; do
  if [ -n "${PRESET_IDX[$i]}" ]; then
    vram=$(vram_sum "${PRESET_IDX[$i]}")
    if (( $(echo "$vram > 7.2" | bc -l) )); then
      flag="${Y}⚠ ${vram} GB${NC}"
    else
      flag="${G}${vram} GB${NC}"
    fi
    printf "  ${C}%d${NC}) %-45s %b\n" "$((i+1))" "${PRESET_NAMES[$i]}" "$flag"
  else
    printf "  ${C}%d${NC}) %s\n" "$((i+1))" "${PRESET_NAMES[$i]}"
  fi
done

echo -e "\n${BOLD}Вибір [1-${#PRESET_NAMES[@]}]:${NC} \c"
read -r choice

# ── Ручний вибір ──────────────────────────────────────────────────
if [ "$choice" = "${#PRESET_NAMES[@]}" ] || [ -z "$choice" ]; then
  echo -e "\n${BOLD}Моделі:${NC}"
  printf "  ${DIM}%-3s %-22s %-8s %s${NC}\n" "№" "Назва" "VRAM" "Призначення"
  echo -e "  ${DIM}────────────────────────────────────────────${NC}"
  for i in "${!MODELS[@]}"; do
    printf "  ${C}%d${NC}) %-22s ${Y}%-8s${NC} %s\n" \
      "$((i+1))" "$(f $i 1)" "$(f $i 3) GB" "$(f $i 4)"
  done
  echo -e "\n${DIM}⚠ 14B MoE (№6) не завантажується разом з іншими${NC}"
  echo -e "\n${BOLD}Номери через пробіл (напр: 1 3):${NC} \c"
  read -r manual
  SELECTED=""
  for n in $manual; do
    SELECTED="$SELECTED $((n-1))"
  done
else
  SELECTED="${PRESET_IDX[$((choice-1))]}"
fi

SELECTED=$(echo "$SELECTED" | tr -s ' ' | sed 's/^ //')

# ── Перевірка VRAM ────────────────────────────────────────────────
total=$(vram_sum "$SELECTED")
if (( $(echo "$total > 7.2" | bc -l) )); then
  echo -e "\n${R}✗${NC} Загалом ${total} GB — недостатньо VRAM на 8GB карті (макс ~7.2 GB)"
  echo -e "   ${DIM}Підтверджено: реальний OOM-крах (SIGABRT), не хибне спрацювання${NC}"
  exit 1
fi

# ── Вивантажити старі ─────────────────────────────────────────────
if [ -n "$LOADED" ]; then
  echo -e "\n${Y}⟳${NC} Вивантажую попередні моделі..."
  while IFS= read -r m; do
    if [ -n "$m" ]; then
      lms unload "$m" 2>/dev/null && echo -e "  ${DIM}✓ $m${NC}"
    fi
  done <<< "$LOADED"
  sleep 1
fi

# ── Завантажити вибрані ───────────────────────────────────────────
echo -e "\n${BOLD}Завантажую:${NC}"
ok=0; fail=0
for idx in $SELECTED; do
  idx=$(echo "$idx" | tr -d ' ')
  [ -z "$idx" ] && continue
  name=$(f $idx 1)
  id=$(f $idx 2)
  size=$(f $idx 3)
  params=$(f $idx 5)
  echo -e "  ${Y}⟳${NC} $name  (${size} GB)..."
  if lms load "$id" $params 2>/dev/null; then
    echo -e "  ${G}✓${NC} $name"
    ((ok++))
  else
    echo -e "  ${R}✗${NC} Помилка: $name — недостатньо VRAM?"
    ((fail++))
  fi
done

# ── Підсумок ──────────────────────────────────────────────────────
echo -e "\n${BOLD}${G}Готово!${NC}  Завантажено: ${ok}  Помилок: ${fail}"
echo -e "${DIM}VRAM: ~${total} GB  |  API: http://127.0.0.1:1234/v1${NC}"
lms ps 2>/dev/null | grep -E "IDLE|RUNNING|LOADING" | awk '{printf "  • %s\n", $1}'
echo
```

```bash
chmod +x ~/bin/lms-pick
```

## 3. Додати PATH у ~/.bashrc

Перевір, чи вже є ці рядки (не дублюй, якщо вже присутні):

```bash
export PATH="$HOME/.local/bin:$PATH"
export PATH="$HOME/.lmstudio/bin:$PATH"
export PATH="$HOME/bin:$PATH"
```

Додай у кінець `~/.bashrc`, якщо відсутні, потім `source ~/.bashrc`.

## 4. Створити ~/.continue/config.yaml

```bash
mkdir -p ~/.continue
```

Створи файл `~/.continue/config.yaml` з точно таким вмістом (валідовано проти
`continue-2.0.0` схеми командою
`jsonschema.validate()` — див. розділ "Як перевірити конфіг"):

```yaml
name: Main Config
version: 1.0.0
schema: v1

models:
  - name: DeepSeek R1 8B
    provider: lmstudio
    model: deepseek/deepseek-r1-0528-qwen3-8b
    contextLength: 8192
    roles:
      - chat
    defaultCompletionOptions:
      maxTokens: 8000
      temperature: 0.6
      topP: 0.95
      reasoning: true
    chatOptions:
      baseSystemMessage: >
        You are a reasoning specialist. Before answering math, logic, or
        algorithmic questions, think step-by-step and show your reasoning.
        Check edge cases before concluding. Give a clear final answer after
        the reasoning, not buried inside it.

  - name: Ornith 9B (coding)
    provider: lmstudio
    model: ornith-1.0-9b
    contextLength: 8192
    roles:
      - chat
    defaultCompletionOptions:
      maxTokens: 8000
      temperature: 0.2
      topP: 0.9
    chatOptions:
      baseSystemMessage: >
        You are a precise coding assistant. Prefer minimal, targeted changes
        over rewrites or new abstractions. Explain what a change does and why
        before applying it. Never perform destructive or irreversible actions
        (deleting files, force operations, dropping data) without explicit
        confirmation. If unsure about intent, ask rather than guess.

  - name: Qwen 3.5 9B
    provider: lmstudio
    model: qwen3.5-9b
    contextLength: 8192
    roles:
      - chat
    defaultCompletionOptions:
      maxTokens: 4000
      temperature: 0.7
      topP: 0.9
      reasoning: true
    requestOptions:
      extraBodyProperties:
        enable_thinking: true
    chatOptions:
      baseSystemMessage: >
        You are a direct, helpful conversational assistant. Give concise,
        clear answers without unnecessary preamble, repetition, or filler.
        If a question is ambiguous, ask a short clarifying question instead
        of guessing.

  - name: Qwen3.6 14B MoE
    provider: lmstudio
    model: qwen3.6-14b-a3b-fablevibes
    contextLength: 4096
    roles:
      - chat
    defaultCompletionOptions:
      maxTokens: 4000
      temperature: 0.6
      topP: 0.9
      reasoning: true
    requestOptions:
      extraBodyProperties:
        enable_thinking: true
    chatOptions:
      baseSystemMessage: >
        You are a well-rounded general-purpose assistant balancing speed and
        depth. Answer directly first, then add detail only if the question
        genuinely requires it. Avoid padding simple answers with caveats.

  - name: Gemma 26B (vision)
    provider: lmstudio
    model: gemma-4-26b-a4b-it-qat
    contextLength: 8192
    roles:
      - chat
    capabilities:
      - image_input
    defaultCompletionOptions:
      maxTokens: 4000
      temperature: 0.4
      topP: 0.9
    chatOptions:
      baseSystemMessage: >
        You are a thorough, high-quality assistant capable of analyzing
        images. When given visual input, describe the relevant details you
        observe before answering. Prioritize accuracy and completeness over
        brevity — this model is chosen specifically when quality matters
        more than speed.

  - name: Gemma E2B (fast)
    provider: lmstudio
    model: gemma-4-e2b-it
    contextLength: 8192
    roles:
      - chat
    capabilities:
      - image_input
    defaultCompletionOptions:
      maxTokens: 4000
      temperature: 0.7
      topP: 0.9
    chatOptions:
      baseSystemMessage: >
        You are a fast-response assistant for quick lookups and simple
        tasks. Keep answers short and to the point. Skip lengthy explanations
        or caveats unless explicitly asked — speed is the priority for this
        model.

  - name: Qwen2.5 Coder 1.5B
    provider: lmstudio
    model: qwen2.5-coder-1.5b-instruct
    contextLength: 4096
    roles:
      - autocomplete
    defaultCompletionOptions:
      temperature: 0.1
      topP: 0.95

  - name: Nomic Embed Text
    provider: lmstudio
    model: text-embedding-nomic-embed-text-v1.5
    roles:
      - embed

context:
  - provider: code
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: codebase
```

### Як перевірити конфіг (обов'язково після створення)

```bash
pip install --quiet jsonschema pyyaml 2>/dev/null
python3 -c "
import yaml, json, jsonschema
config = yaml.safe_load(open('$HOME/.continue/config.yaml'))
schema = json.load(open('$(ls -d $HOME/.vscode/extensions/continue.continue-*/config-yaml-schema.json | head -1)'))
jsonschema.validate(config, schema)
print('OK: config matches installed Continue schema')
"
```

Якщо помилка валідації — версія Continue на новому ПК відрізняється від
2.0.0, і поля потрібно звірити з `config-yaml-schema.json` цієї версії
вручну (шукати `properties` для `ConfigYaml` та вкладеного `models[]`).

## 5. Відомі проблеми та як їх діагностувати

### VRAM-бюджет (для 8GB карт)

- Жорсткий поріг у `lms-pick`: 7.2 GB сумарно (не 8.0!) — перевірено
  емпірично, вище стабільно падає з `cudaMalloc failed: out of memory`.
- **Qwen 3.5 9B несумісний з ЖОДНОЮ іншою моделлю одночасно** — тягне
  прихований vision/CLIP-модуль (~0.9 GB понад заявлений розмір), через що
  навіть 0.9 GB autocomplete зверху вже викликає `SIGABRT`. Тому пресет
  "Чат" вантажить його **тільки соло**.
- Оцінка `lms load --estimate-only` для Qwen 3.5 9B ненадійна
  ("Confidence: LOW", не враховує vision-модуль) — не покладайся на неї.

### Passkey auth desync (lms CLI ↔ LM Studio backend)

Симптом: `lms-pick` показує "недостатньо VRAM?" навіть для найменшої моделі
(0.9 GB), хоча GPU вільний. Траплялось мінімум двічі за сесію — це не
одноразовий збій, перевіряй одразу, якщо `lms-pick` масово фейлить:

```bash
export PATH="$HOME/.lmstudio/bin:$PATH"
lms ps
```

Якщо в помилці `Invalid passkey for lms CLI client` — саме воно:
`lms unload`/`lms load` мовчки не спрацьовують (скрипт ховає помилки через
`2>/dev/null`), лишається зомбі-процес `llama-server`, що займає VRAM.

Виправлення:

```bash
nvidia-smi                                    # знайти PID зомбі llama-server
ps aux | grep -iE "llama-server|llmster"      # знайти PID llmster (головний бекенд)
kill <PID llama-server>
kill <PID llmster>                            # безпечно, автоперезапуститься
lms server start --port 1234                  # пересинхронізує passkey
lms ps                                         # має відповісти без помилки auth
```

## 6. Перевірка після встановлення

```bash
lms-pick   # спробувати пресет 1 (Мінімум) — має завантажитись без помилок
```

Потім у VS Code: перезапустити вікно (Command Palette →
"Developer: Reload Window"), відкрити Continue, перевірити що модель
відповідає і автозавершення коду працює.
