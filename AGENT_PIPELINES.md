# Agent Pipelines Documentation

Эта документация описывает все возможные схемы работы AI-агента в приложении Open SaaS, включая последовательность вызовов промтов, передачу данных между этапами, и обработку результатов.

## Содержание

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Pipeline: Task Schedule Generation](#pipeline-task-schedule-generation)
3. [Диаграммы потоков данных](#диаграммы-потоков-данных)
4. [Обработка ошибок](#обработка-ошибок)
5. [Оптимизация и кеширование](#оптимизация-и-кеширование)

---

## Обзор архитектуры

Приложение использует архитектуру с четким разделением на клиент и сервер:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT SIDE                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   DemoAppPage.tsx                        │   │
│  │  - UI Components (Input, Button, Cards)                 │   │
│  │  - State Management (useState, useQuery)                │   │
│  │  - Error Handling (Toast notifications)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
                         RPC (Wasp)
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                         SERVER SIDE                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   operations.ts                          │   │
│  │  - Authentication & Authorization                        │   │
│  │  - Data Validation (Zod)                                │   │
│  │  - Database Operations (Prisma)                         │   │
│  │  - Credit Management                                     │   │
│  │  - AI Integration (OpenAI)                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
                         OpenAI API
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                         OPENAI GPT-3.5                           │
│  - System Prompt Processing                                      │
│  - User Prompt Processing                                        │
│  - Function Calling Execution                                    │
│  - Structured JSON Response                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pipeline: Task Schedule Generation

Это основной и единственный AI-пайплайн в приложении. Он отвечает за генерацию оптимального расписания задач на день.

### Общая схема потока

```
┌──────────────┐
│    START     │
│  User Click  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│         CLIENT: Handle Button Click              │
│  File: DemoAppPage.tsx:145-181                   │
│  Function: handleGeneratePlan()                  │
│                                                   │
│  Actions:                                         │
│  1. Set loading state (isPlanGenerating = true)  │
│  2. Call RPC: generateGptResponse()              │
│  3. Handle response or errors                    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│         RPC CALL: generateGptResponse            │
│  Input: { hours: number }                        │
│  Type: GenerateGptResponse                       │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Authentication Check                  │
│  File: operations.ts:38-43                       │
│                                                   │
│  Check: context.user exists?                     │
│    ├─ NO  → Throw HttpError(401)                │
│    └─ YES → Continue                             │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Input Validation                      │
│  File: operations.ts:45-48                       │
│                                                   │
│  Schema: generateGptResponseInputSchema          │
│  Validator: Zod                                  │
│  Required: hours (number)                        │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Fetch User Tasks                      │
│  File: operations.ts:49-55                       │
│                                                   │
│  Query: Task.findMany()                          │
│  Filter: WHERE user.id = context.user.id         │
│  Returns: Task[]                                 │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Prepare Data for AI                   │
│  File: operations.ts:254-257                     │
│  Function: generateScheduleWithGpt()             │
│                                                   │
│  Transform:                                       │
│    Task[] → parsedTasks[]                        │
│    Extract: { description, time }                │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    OPENAI API: Create Completion                 │
│  File: operations.ts:259-337                     │
│  Model: gpt-3.5-turbo                            │
│  Temperature: 1                                  │
│                                                   │
│  Messages:                                        │
│    1. System Prompt (role: "system")            │
│    2. User Prompt (role: "user")                │
│                                                   │
│  Tools: parseTodaysSchedule function             │
│  Tool Choice: Force function call                │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    OPENAI: Process Prompts                       │
│                                                   │
│  Step 1: System Prompt Processing                │
│    "you are an expert daily planner..."          │
│    → Sets AI role and behavior                   │
│                                                   │
│  Step 2: User Prompt Processing                  │
│    "I will work X hours today..."                │
│    → Provides context and task data              │
│                                                   │
│  Step 3: Function Schema Analysis                │
│    → Understands output structure                │
│                                                   │
│  Step 4: Generate Response                       │
│    → Creates structured JSON matching schema     │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Parse AI Response                     │
│  File: operations.ts:339-341                     │
│                                                   │
│  Extract:                                         │
│    completion.choices[0].message                 │
│      .tool_calls[0].function.arguments          │
│                                                   │
│  Parse: JSON.parse(gptResponse)                  │
│  Returns: GeneratedSchedule | null               │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Validate AI Response                  │
│  File: operations.ts:59-64                       │
│                                                   │
│  Check: generatedSchedule !== null?              │
│    ├─ NO  → Throw HttpError(500)                │
│    └─ YES → Continue                             │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Check Subscription Status             │
│  File: operations.ts:108-113                     │
│  Function: isUserSubscribed()                    │
│                                                   │
│  Check: user.subscriptionStatus in               │
│    [Active, CancelAtPeriodEnd]?                  │
│                                                   │
│  ├─ YES → Skip credit deduction                 │
│  │                                                │
│  └─ NO  → Check credits                         │
│      Check: user.credits > 0?                    │
│      ├─ YES → Prepare credit decrement          │
│      └─ NO  → Throw HttpError(402)              │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Database Transaction                  │
│  File: operations.ts:73-103                      │
│                                                   │
│  Transaction operations (atomic):                 │
│    1. Create GptResponse record                  │
│       - Save generated schedule                  │
│       - Link to user                             │
│                                                   │
│    2. Update User credits (if applicable)        │
│       - Decrement by 1                           │
│       - Only for non-subscribed users            │
│                                                   │
│  Execute: prisma.$transaction()                  │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    SERVER: Return Response                       │
│  File: operations.ts:105                         │
│                                                   │
│  Return: GeneratedSchedule                       │
│  Type: { tasks[], taskItems[] }                  │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    CLIENT: Handle Success                        │
│  File: DemoAppPage.tsx:151-153                   │
│                                                   │
│  Actions:                                         │
│    1. Set response state                         │
│    2. Clear loading state                        │
│    3. Render schedule UI                         │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│    CLIENT: Render Schedule                       │
│  File: DemoAppPage.tsx:275-283, 363-395          │
│  Components: Schedule, TaskCard, TaskCardItem    │
│                                                   │
│  Display:                                         │
│    - Group tasks by priority (high/medium/low)   │
│    - Show main tasks with color coding           │
│    - List subtasks with time estimates           │
│    - Enable checkbox interactions                │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
                  ┌────────┐
                  │  END   │
                  └────────┘
```

---

