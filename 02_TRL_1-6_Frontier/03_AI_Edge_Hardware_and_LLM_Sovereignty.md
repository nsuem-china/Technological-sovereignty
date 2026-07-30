---
title: "Концепция открытого суверенного стека ИИ: RISC-V и альтернатива CUDA"
tags: [tech-sovereignty, risck-v, cuda-alternative, hardware, alignment, trl]
trl_level: "TRL 3-6"
---

{% hint style="info" %}
**Стратегический мандат**
Проект фундаментальной трансформации экосистемы ИИ-вычислений: переход от закрытой монополии NVIDIA (CUDA/cuDNN/NCCL) и запатентованных литографических процессоров к **открытому, суверенному микрокомпонентному и программному стеку**. В основе архитектуры — открытый стандарт ISA [RISC-V](../lib-1/RISC-V.md), гибридная модульная компоновка [Chiplet Architecture](../lib-1/Chiplet%20Architecture.md), вычислители в памяти [Compute-in-Memory](../lib-1/Compute-in-Memory.md) и сквозная компиляторная цепочка на базе [MLIR Framework](../lib-1/MLIR%20Framework.md) и [Triton Compiler](../lib-1/Triton%20Compiler.md).
{% endhint %}

---

## 1. Архитектурный манифест: Декомпозиция программного барьера

Главный фактор технологического зависания (Vendor Lock-in) — не физическое отсутствие ускорителей, а **программный слой CUDA**, де-факто ставший стандартом индустрии. Преодоление зависимости требует полной горизонтальной декомпозиции всех пяти уровней абстракции ИИ-инфраструктуры.

> **1. Прикладной уровень (LLM, VLA, Generative AI)**

### │

> **2. Фреймворки обучения: [PyTorch](../lib-1/PyTorch.md) / [MindSpore](../lib-1/MindSpore.md) / JAX**

### │

> **3. Слой DSL и графовых компиляторов: [Triton Compiler](../lib-1/Triton%20Compiler.md) / [MLIR Framework](../lib-1/MLIR%20Framework.md)**

### │

> **4. Runtime, HAL & Comm: [Open-RoCEv2](../lib-1/Open-RoCEv2.md) / Custom Interconnect / [Open HAL](../lib-1/Open%20HAL.md)**

### │

> **5. Аппаратный стек: [RISC-V](../lib-1/RISC-V.md) Multi-Core + Systolic Array + [ReRAM CiM](../lib-1/ReRAM%20CiM.md) Упаковка: [UCIe Interconnect](../lib-1/UCIe%20Interconnect.md) 2.5D SiP + SRAM-Heavy Architecture**


{% hint style="info" %}
**Фундаментальный принцип открытого стека**
Перенос логики аппаратной оптимизации вычислений из проприетарных драйверов и скрытых библиотек (cuBLAS, cuDNN) в **открытые декларативные языки написания ядер ([Triton Compiler](../lib-1/Triton%20Compiler.md))** и **промежуточные представления ([MLIR Framework](../lib-1/MLIR%20Framework.md))**, транслируемые прямо в векторно-матричные инструкции открытой ISA.
{% endhint %}

---

## 2. Аппаратный фундамент: [RISC-V](../lib-1/RISC-V.md), чиплеты и CiM

### 2.1. Доменная архитектура (DSA) на базе RISC-V
Базовый вычислительный блок включает открытое ядро управления (Control Core) и специализированый тензорный процессор:

* **Инструкции векторизации:** [RISC-V Vector Extensions](../lib-1/RISC-V%20Vector%20Extensions.md) (RVV 1.0) с расширенной длиной векторов (`VLEN` $\ge 512\text{ bit}$, `LMUL` до $8$) для параллельной обработки последовательностей.
* **Матричные расширения (Custom ISA Matrix Extensions - `Xmatrix`):** Проприетарно-свободные ISA-расширения для прямых команд управления 2D-систолическими массивами (Systolic Arrays). Выполняют матричное умножение $C += A \times B$ за фиксированное число тактов для форматов `FP16`, `BF16`, `INT8`, `INT4` и `FP8` (E4M3 / E5M2).
* **Конфигурация регистрового файла:** Dynamic Vector Length (`vsetvli`) позволяет исключить лишние операции заполнения/очистки (padding/tail processing) на краевых элементах матриц.

> **RISC-V Core (Control & scalar)**

│
                     ┌────────────────┴────────────────┐
### ▼                                 ▼

| RVV 1.0 Vector Engine (VLEN=1024, LMUL=4) |  | Matrix Engine (Systolic Array 64x64 MACs) |
| --- | --- | --- |

│                                 │
                     └────────────────┬────────────────┘
│
                         ┌────────────┴────────────┐
### ▼                         ▼

| L2 SRAM Scratchpad (Ultra-high Bandwidth) |  | [ReRAM CiM](../lib-1/ReRAM%20CiM.md) Crossbar (Analog Memory Compute) |
| --- | --- | --- |


### 2.2. Аналоговый вычислитель в памяти: [ReRAM CiM](../lib-1/ReRAM%20CiM.md)
Для решения проблемы энергопотребления на операциях умножения вектора на матрицу ($GEMV$) на фазе инференса в стек интегрируется гибридный блок **Compute-in-Memory**:
* **Технология:** Металлические окисные мемристоры ($HfO_x$ / $TaO_x$), организованные в матричные кроссбары (Crossbar Arrays).
* **Механика вычислений:** Входные активации подаются в виде напряжений на строки, веса модели закодированы проводимостью $G_{ij}$ ячеек ($1T1R$). Выходной ток на столбцах равен $I_j = \sum V_i \cdot G_{ij}$ согласно законам Кирхгофа.
* **Ограничения и TRL:** Ограничено погрешностями ЦАП/АЦП (ADC/DAC) и температурным дрейфом проводимости. Текущий уровень — **TRL 3–4** (лабораторные и опытно-промышленные прототипы).

{% hint style="info" %}
**Операционная интенсивность и Bottleneck**
Вычисление в памяти устраняет перенос весов из DRAM, снижая энергозатраты на одну операцию MAC с $\sim 20\text{ pJ}$ (DRAM read) до $\sim 0.1\text{ pJ}$ (Analog CiM), что переводит систему из режима Memory-Bound в Compute-Bound.
{% endhint %}

### 2.3. Межсоединения, чиплеты и преодоление Lithography Gap
При отсутствии доступа к литографии $< 5\text{ nm}$ и технологиям упаковки TSMC CoWoS, стек использует стратегия модульной сборки на суверенных мощностях:

* **Стандарт интерфейса:** [UCIe Interconnect](../lib-1/UCIe%20Interconnect.md) (Universal Chiplet Interconnect Express) на уровнях Streaming Protocol и Physical Layer (Die-to-Die PHY).
* **Технология упаковки:** 2.5D SiP на базе высокоплотных органических интерпозеров (Organic Substrate with Fine Pitch) или кремниевых мостов (Silicon Bridges), обеспечивающих ПСП $> 2\text{ TB/s/mm}$.
* **Преодоление отсутствия HBM3e (SRAM-Heavy NPU):** Взамен дорогостоящих стеков HBM3e применяется архитектура с распределенным сверхбольшим объемом накристального SRAM-кэша (до $1\text{--}2\text{ GB}$ SRAM на чиплет, соединенного через mesh-сеть) в комбинации с внешними каналами LPDDR5X (8–16 каналов).

---

## 3. Программный стек: Разрыв зависимости от CUDA

Создание альтернативы CUDA основано на исключении закрытого слоя кода NVIDIA NVCC и библиотек cuDNN/cuBLAS из производственного цикла.

                          [ High-Level Code: PyTorch / JAX ]
                                          │
                                          ▼
                          [ [Triton Compiler](../lib-1/Triton%20Compiler.md) Python DSL ]
                                          │
                                          ▼
                          [ [MLIR Framework](../lib-1/MLIR%20Framework.md) Infra ]
│
    ┌─────────────────────────────────────┼─────────────────────────────────────┐
    ▼                                     ▼                                     ▼
 Dialect: `tosa`                  Dialect: `linalg`                     Dialect: `vector`
 (Tensor Ops)                     (Structured Loops)                    (SIMD / Hardware)
│                                     │                                     │
    └─────────────────────────────────────┼─────────────────────────────────────┘
                                          │
                                          ▼
                              [ Custom RISC-V Pass ]
│
               ┌──────────────────────────┴──────────────────────────┐
               ▼                                                     ▼
   [ LLVM RISC-V Vector Backend ]                        [ Custom Matrix Codegen ]
               │                                                     │
               ▼                                                     ▼
   [ RVV 1.0 Assembly (`vle`, `vfma`) ]                 [ Systolic Array Stream Commands ]

### 3.1. Ключевые компоненты Anti-CUDA компиляции

1. **Kernel DSL ([Triton Compiler](../lib-1/Triton%20Compiler.md)):** Разработчик пишет высокопроизводительные ядра (FlashAttention, SwiGLU, Rotary Embeddings) на Python-подобном Triton DSL. Компилятор не обращается к CUDA C++, а генерирует промежуточный IR Triton.
2. **Многоуровневая сквозная трансляция ([MLIR Framework](../lib-1/MLIR%20Framework.md)):**
   * High-Level Dialect (`tosa`, `stablehlo`): Высокоуровневая оптимизация графа вычислений (fusion, dead-code elimination).
   * Mid-Level Dialect (`linalg`): Разбиение матриц на плитки (tiling) и параллелизация циклов.
   * Low-Level Dialect (`vector`, `memref`): Прямое отображение на векторные регистры RISC-V и явное управление локальным SRAM (Scratchpad Memory).
3. **Сетевой транспорт альтернативы NCCL ([Open Collective Comm](../lib-1/Open%20Collective%20Comm.md) / MSCL):**
   * Открытый аналог NCCL, работающий поверх стандартов Ethernet (RoCEv2) и внутренних фабрик чиплетов.
   * Релизует кольцевые и древовидные алгоритмы коллективного взаимодействия (All-Reduce, All-To-All) напрямую через DMA-контроллеры процессоров RISC-V, минуя центральный CPU.

---

## 4. Сравнительный анализ: CUDA vs. Sovereign Open Stack

| Уровень стека :--- **ISA Architecture** **Kernel DSL Layer** **Compiler Pipeline** **Primitive Libs** **Interconnect** **Memory System** | Западная экосистема (NVIDIA Stack) :--- Проприетарная CUDA PTX / SASS CUDA C/C++ / PTX Assembly NVCC / NVVM (Closed Source) cuBLAS, cuDNN, TensorRT NVLink 4 / NVSwitch HBM3e (Up to 4.8 TB/s) | Суверенный открытый стек (RISC-V Sovereign) :--- **[RISC-V](../lib-1/RISC-V.md)** (RVV 1.0 + Custom `Xmatrix`) **[Triton Compiler](../lib-1/Triton%20Compiler.md)** / Halide **[MLIR Framework](../lib-1/MLIR%20Framework.md)** + LLVM Target Passes **Open-Source C++ / MLIR Vectorized Libs** **[UCIe Interconnect](../lib-1/UCIe%20Interconnect.md)** / Open-RoCEv2 **SRAM-Heavy (1-2GB) + LPDDR5X (Interleaved)** | TRL :--- **TRL 5–6** **TRL 5** **TRL 4–5** **TRL 4** **TRL 3–5** **TRL 4–5** | Ключевые технологические риски :--- Фрагментация нестандартных расширений ISA Неполное покрытие нелинейных слоев Сложность генерации кода под неоднородные ядра Отсутствие авто-тюнинга под специфический SRAM Высокие задержки при масштабировании на $> 1000$ узлов Ограничение максимального размера контекста LLM |
| --- | --- | --- | --- | --- |


---

## 5. Интеграция с Alignments, Security & Edge AI

### 5.1. Аппаратное ускорение выравнивания ([Direct Preference Optimization](../lib-1/Direct%20Preference%20Optimization.md))
Процессы выравнивания языковых моделей требуют интенсивного вычисления логитов и регуляризации Divergence ($KL$).
* **Оптимизация на уровне ISA:** Вычисление выравнивания DPO переводятся в формат FP8 с динамическим масштабированием графического диапазона на уровне RVV-векторных регистров.
* **Результат:** Исключается требование хранения вызовов второй модели-награды (Reward Model) в VRAM, снижая требования к оперативной памяти на $40\%$.

### 5.2. Автономные системы: [Vision-Language-Action Models](../lib-1/Vision-Language-Action%20Models.md) (VLA)
В робототехнике критична минимальная задержка реакций ($< 5\text{ ms}$) на внешние раздражители.
* **Гибридный режим исполнение:** 
  1. Сенсорный поток обрабатывается блоком [Spiking Neural Networks](../lib-1/Spiking%20Neural%20Networks.md) (SNN) на базе [ReRAM CiM](../lib-1/ReRAM%20CiM.md) с энергопотреблением $< 500\text{ mW}$ (режим ожидания).
  2. При обнаружении аномалии или команды векторный блок RISC-V просыпается (Wake-up call) за несколько наносекунд и вычисляет `Action Token` VLA-модели через систолический массив.

{% hint style="danger" %}
**Безопасность аппаратного обеспечения (Hardware Trojans)**
Из-за открытости стандарта RISC-V и использования сторонних фабрик (Foundries) существует риск внедрения аппаратных закладок на этапе RTL-синтеза. Необходима интеграция **Hardware Root of Trust (HRoT)** с модулями криптографической верификации конфигурации (PUF — Physically Unclonable Functions).
{% endhint %}

---

## 6. Инженерные кейсы и решения (Self-Assessment)

### Кейс 1: Портирование и оптимизация ядра FlashAttention-2 на RISC-V RVV 1.0

{% hint style="info" %}
**Исходная проблема**
Команда адаптации производит запуск LLM (контекст 64k) на RISC-V ускорителе. Прямая трансляция стандартного C-кода Attention дает падение скорости в $8.5$ раз по сравнению с NVIDIA A100 из-за постоянных сбросов данных в LPDDR5X.

**Задача:** Спроектировать последовательность компиляции и оптимизировать распределение SRAM для достижения потерянной производительности.
{% endhint %}

{% hint style="info" %}
**Инженерное решение**
1. **Рефакторинг на [Triton Compiler](../lib-1/Triton%20Compiler.md):** Код алгоритма переписывается с явным разбиением блочной матрицы $Q, K, V$ на плитки размера `BLOCK_M = 64`, `BLOCK_N = 64`.
2. **Управление памяти через MLIR Dialect `memref`:** Производится принудительное выделение буферов $Q_b, K_b, V_b$ во внутреннем SRAM-Scratchpad вместо обращения к глобальной памяти.
3. **Векторизация через RVV 1.0:**
   ```llvm
>    ;; Использование векторных инструкций RISC-V для Softmax
>    vsetvli t0, a0, e32, m4, ta, ma   ; Настройка: 32-bit floats, LMUL=4
>    vle32.v v4, (a1)                 ; Загрузка вектора QK^T
>    vfmax.vv v8, v4, v8              ; Поиск максимума для Online Softmax
>    vfsub.vv v4, v4, v8              ; Нормализация
>    vfexp.v v4, v4                   ; Экспонента
>    ```
4. **Результат:** За счет исключения Memory Bottleneck скорость вычислений увеличивается в $7.2$ раза относительно наивной реализации.
{% endhint %}

---

### Кейс 2: Устранение «Memory Wall» в робототехнических VLA-системах

{% hint style="info" %}
**Исходная проблема**
Бортовой компьютер робота с бюджетом $30\text{ Вт}$ должен исполнять VLA-модель с $7\text{B}$ параметров. Пропускная способность памяти LPDDR5X ($102\text{ GB/s}$) ограничивает генерацию до $4\text{ токенов/сек}$, что делает управление невозможным.

**Задача:** Снизить задержку и энергопотребление с помощью гибридной архитектуры RISC-V + ReRAM.
{% endhint %}

{% hint style="info" %}
**Инженерное решение**
1. **Квантование моделей:** Модель квантуется до гибридного формата: линейные слои проекций — `INT4`, слои Attention — `FP8`.
2. **Маппинг на [ReRAM CiM](../lib-1/ReRAM%20CiM.md):** Матрицы весов проекций $W_q, W_k, W_v, W_o$ загружаются в проводимости ReRAM-кроссбаров. Операция $GEMV$ выполняется за 1 такт подачей напряжения.
3. **Оптимизация контроллера RISC-V:** Ядро RISC-V освобождается от генерации адресов памяти и выполняет только нелинейные функции активации (GeLU, LayerNorm) с использованием инструкций RVV 1.0.
4. **Метрики системы:** Пропускная способность возрастает до $38\text{ токенов/сек}$, энергопотребление падаёт до $19.5\text{ Вт}$.
{% endhint %}

---

### Кейс 3: Масштабирование All-Reduce вычислений без NCCL через [UCIe Interconnect](../lib-1/UCIe%20Interconnect.md)

{% hint style="info" %}
**Исходная проблема**
При объединении 4 чиплетов на одном органическом интерпозере классический программный обмен сообщениями через системный bus CPU создает задержку All-Reduce в $120\text{ \mu s}$, вызывая простоя систолических массивов.

**Задача:** Организовать аппаратную поддержку коллективных операций без использования проприетарного NVLink.
{% endhint %}

{% hint style="info" %}
**Инженерное решение**
1. **Внедрение Direct-Stream пакетизатора:** В каждый чиплет интегрируется аппаратный агент [UCIe Interconnect](../lib-1/UCIe%20Interconnect.md) (UCIe-S Streaming Protocol).
2. **Аппаратный Ring All-Reduce:**
   * Создается кольцевой маршрут между 4 чиплетами на физическом уровне интерпозера.
   * Операция сложения градиентов (Reduce-Scatter) выполняется "на лету" (On-the-fly) в векторном ALU промежуточного чиплета до записи данных в SRAM.
3. **Результат:** Задержка All-Reduce снижается до $4.2\text{ \mu s}$, эффективность масштабирования при параллелизме тензоров (Tensor Parallelism = 4) достигает $91\%$.
{% endhint %}

---

## 7. Связанные понятия и граф знаний

* [RISC-V Vector Extensions](../lib-1/RISC-V%20Vector%20Extensions.md) — спецификация векторной обработки данных RVV 1.0.
* [MLIR Framework](../lib-1/MLIR%20Framework.md) — инфраструктурный слой для построения модульных компиляторов.
* [Triton Compiler](../lib-1/Triton%20Compiler.md) — открытый с++ / python компилятор и DSL для высокопроизводительных ИИ-ядер.
* [Compute-in-Memory](../lib-1/Compute-in-Memory.md) — концепция вычислений внутри массивов памяти для обхода барьера Фон-Неймана.
* [ReRAM CiM](../lib-1/ReRAM%20CiM.md) — физическая реализация аналоговых вычислителей на базе сопротивления мемристоров.
* [UCIe Interconnect](../lib-1/UCIe%20Interconnect.md) — открытый межчиповый стандарт связи 2.5D/3D микросхем.
* [Chiplet Architecture](../lib-1/Chiplet%20Architecture.md) — концепция построения процессоров из отдельных функциональных полупроводниковых кристаллов.
* [Direct Preference Optimization](../lib-1/Direct%20Preference%20Optimization.md) — безытерационный алгоритм прямого выравнивания больших моделей.
* [Vision-Language-Action Models](../lib-1/Vision-Language-Action%20Models.md) — мультимодальные ИИ-системы прямой генерации физических действий.
* [Open Collective Comm](../lib-1/Open%20Collective%20Comm.md) — библиотека открытых алгоритмов распределенного обучения ИИ.