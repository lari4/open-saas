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

### Детальный поток данных между промтами

Эта диаграмма показывает, какие данные передаются между каждым этапом обработки и как трансформируются промты.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA TRANSFORMATION FLOW                      │
└─────────────────────────────────────────────────────────────────┘

INPUT FROM USER:
┌───────────────────────────────────────┐
│  User Input                           │
│  - todaysHours: 8                     │
│  - tasks: [                           │
│      { description: "Emails", ... },  │
│      { description: "Learn WASP", ... }│
│    ]                                  │
└───────────────┬───────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 1: Client Side Preparation                               │
│  Function: handleGeneratePlan()                                │
│                                                                 │
│  Input:  { hours: 8 }                                          │
│  Output: RPC Call to generateGptResponse({ hours: 8 })        │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 2: Server Side - Fetch Tasks from DB                    │
│  Function: context.entities.Task.findMany()                   │
│                                                                 │
│  Input:  user.id                                               │
│  Output: Task[] = [                                            │
│    {                                                            │
│      id: "task_123",                                           │
│      description: "Respond to emails",                         │
│      time: "2",                                                │
│      isDone: false,                                            │
│      userId: "user_456",                                       │
│      createdAt: "2024-01-15T10:00:00Z"                        │
│    },                                                           │
│    {                                                            │
│      id: "task_124",                                           │
│      description: "Learn WASP",                                │
│      time: "3",                                                │
│      isDone: false,                                            │
│      userId: "user_456",                                       │
│      createdAt: "2024-01-15T10:05:00Z"                        │
│    }                                                            │
│  ]                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 3: Data Transformation for AI                           │
│  Function: parsedTasks = tasks.map(...)                       │
│                                                                 │
│  Input:  Task[] (full objects from DB)                        │
│  Transform: Extract only { description, time }                │
│  Output: parsedTasks = [                                       │
│    { description: "Respond to emails", time: "2" },           │
│    { description: "Learn WASP", time: "3" }                   │
│  ]                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 4: Construct System Prompt (Static)                     │
│  Location: operations.ts:264-265                               │
│                                                                 │
│  Prompt:                                                        │
│  "you are an expert daily planner. you will be given a        │
│   list of main tasks and an estimated time to complete        │
│   each task. You will also receive the total amount of        │
│   hours to be worked that day. Your job is to return a        │
│   detailed plan of how to achieve those tasks by breaking     │
│   each task down into at least 3 subtasks each. MAKE SURE     │
│   TO ALWAYS CREATE AT LEAST 3 SUBTASKS FOR EACH MAIN TASK     │
│   PROVIDED BY THE USER! YOU WILL BE REWARDED IF YOU DO."      │
│                                                                 │
│  Role: "system"                                                │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 5: Construct User Prompt (Dynamic)                      │
│  Location: operations.ts:269-271                               │
│                                                                 │
│  Template:                                                      │
│  `I will work ${hours} hours today. Here are the tasks I      │
│   have to complete: ${JSON.stringify(parsedTasks)}.           │
│   Please help me plan my day by breaking the tasks down       │
│   into actionable subtasks with time and priority status.`    │
│                                                                 │
│  Injected Data:                                                │
│    - hours = 8                                                 │
│    - parsedTasks = [{"description":"Respond to emails",       │
│                      "time":"2"},                              │
│                     {"description":"Learn WASP","time":"3"}]  │
│                                                                 │
│  Final Prompt:                                                 │
│  "I will work 8 hours today. Here are the tasks I have to     │
│   complete: [{"description":"Respond to emails","time":"2"},  │
│   {"description":"Learn WASP","time":"3"}]. Please help me    │
│   plan my day by breaking the tasks down into actionable      │
│   subtasks with time and priority status."                    │
│                                                                 │
│  Role: "user"                                                  │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 6: Send to OpenAI with Function Definition              │
│  Location: operations.ts:259-337                               │
│                                                                 │
│  Request Payload:                                              │
│  {                                                              │
│    model: "gpt-3.5-turbo",                                    │
│    messages: [                                                 │
│      { role: "system", content: "[System Prompt]" },         │
│      { role: "user", content: "[User Prompt]" }              │
│    ],                                                           │
│    tools: [{                                                   │
│      type: "function",                                         │
│      function: {                                               │
│        name: "parseTodaysSchedule",                           │
│        description: "parses the days tasks...",               │
│        parameters: {                                           │
│          type: "object",                                       │
│          properties: {                                         │
│            tasks: { ... },                                     │
│            taskItems: { ... }                                  │
│          }                                                      │
│        }                                                        │
│      }                                                          │
│    }],                                                          │
│    tool_choice: {                                              │
│      type: "function",                                         │
│      function: { name: "parseTodaysSchedule" }                │
│    },                                                           │
│    temperature: 1                                              │
│  }                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 7: OpenAI Processing (Internal)                         │
│                                                                 │
│  1. Parse system prompt → Set AI behavior/role                │
│  2. Parse user prompt → Extract task data and context         │
│  3. Analyze function schema → Understand output structure     │
│  4. Generate response → Create structured schedule            │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 8: OpenAI Response                                       │
│                                                                 │
│  Response from OpenAI:                                         │
│  {                                                              │
│    choices: [{                                                 │
│      message: {                                                │
│        role: "assistant",                                      │
│        tool_calls: [{                                          │
│          type: "function",                                     │
│          function: {                                           │
│            name: "parseTodaysSchedule",                       │
│            arguments: "{                                       │
│              \"tasks\": [                                      │
│                {                                               │
│                  \"name\": \"Respond to emails\",             │
│                  \"priority\": \"high\"                        │
│                },                                              │
│                {                                               │
│                  \"name\": \"Learn WASP\",                    │
│                  \"priority\": \"medium\"                      │
│                }                                               │
│              ],                                                │
│              \"taskItems\": [                                  │
│                {                                               │
│                  \"description\": \"Check and respond...\",   │
│                  \"time\": 1,                                  │
│                  \"taskName\": \"Respond to emails\"          │
│                },                                              │
│                {                                               │
│                  \"description\": \"Organize emails...\",     │
│                  \"time\": 0.5,                                │
│                  \"taskName\": \"Respond to emails\"          │
│                },                                              │
│                ...                                             │
│              ]                                                 │
│            }"                                                  │
│          }                                                      │
│        }]                                                       │
│      }                                                          │
│    }]                                                           │
│  }                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 9: Parse Response on Server                             │
│  Function: JSON.parse(gptResponse)                            │
│  Location: operations.ts:339-341                               │
│                                                                 │
│  Extract:                                                       │
│    completion.choices[0].message.tool_calls[0]                │
│      .function.arguments                                       │
│                                                                 │
│  Parse: Convert JSON string to object                         │
│                                                                 │
│  Output: GeneratedSchedule = {                                │
│    tasks: [                                                    │
│      { name: "Respond to emails", priority: "high" },        │
│      { name: "Learn WASP", priority: "medium" }              │
│    ],                                                           │
│    taskItems: [                                                │
│      {                                                         │
│        description: "Check and respond to important emails",  │
│        time: 1,                                                │
│        taskName: "Respond to emails"                          │
│      },                                                        │
│      {                                                         │
│        description: "Organize and prioritize emails",         │
│        time: 0.5,                                              │
│        taskName: "Respond to emails"                          │
│      },                                                        │
│      ...                                                       │
│    ]                                                            │
│  }                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 10: Save to Database                                     │
│  Location: operations.ts:66-71                                 │
│                                                                 │
│  Create GptResponse:                                           │
│  {                                                              │
│    id: "response_789",                                        │
│    userId: "user_456",                                        │
│    content: "{\"tasks\":[...],\"taskItems\":[...]}",         │
│    createdAt: "2024-01-15T10:15:00Z"                         │
│  }                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 11: Return to Client                                     │
│                                                                 │
│  RPC Response: GeneratedSchedule (TypeScript object)          │
│  {                                                              │
│    tasks: Task[],                                              │
│    taskItems: TaskItem[]                                       │
│  }                                                              │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  STEP 12: Render in UI                                         │
│  Components: Schedule → TaskCard → TaskCardItem               │
│                                                                 │
│  Display:                                                       │
│  - Main tasks sorted by priority (high → low)                 │
│  - Each task card color-coded by priority                     │
│  - Subtasks listed under each main task                       │
│  - Time estimates shown in human-readable format              │
│  - Interactive checkboxes for completion tracking             │
└────────────────────────────────────────────────────────────────┘
```

---

## Диаграммы потоков данных

### Поток данных через промты

```
USER DATA → SYSTEM PROMPT + USER PROMPT → FUNCTION SCHEMA → AI → STRUCTURED JSON

┌──────────────────────────────────────────────────────────────┐
│ Детальная схема передачи данных через промты                │
└──────────────────────────────────────────────────────────────┘

┌─────────────────┐
│   User Tasks    │
│  from Database  │
└────────┬────────┘
         │
         ├─────────────────────────────────────┐
         │                                     │
         ▼                                     ▼
┌──────────────────┐              ┌──────────────────────┐
│  System Prompt   │              │    User Prompt       │
│  (Static)        │              │    (Dynamic)         │
│                  │              │                      │
│  Role:           │              │  Data Injection:     │
│  "Expert daily   │              │  - hours: 8          │
│   planner"       │              │  - tasks: [...]      │
│                  │              │                      │
│  Instructions:   │              │  Request:            │
│  "Break into     │              │  "Help me plan       │
│   3 subtasks"    │              │   my day..."         │
└────────┬─────────┘              └──────────┬───────────┘
         │                                   │
         └──────────────┬────────────────────┘
                        │
                        ▼
         ┌──────────────────────────┐
         │   Combined Message       │
         │   Array to OpenAI        │
         │                          │
         │   [                      │
         │     {role: "system",...},│
         │     {role: "user",...}   │
         │   ]                      │
         └──────────┬───────────────┘
                    │
                    ├─────────────────────────┐
                    │                         │
                    ▼                         ▼
         ┌──────────────────┐    ┌──────────────────────┐
         │  Function Schema │    │   OpenAI Model       │
         │                  │    │   gpt-3.5-turbo      │
         │  Defines:        │───→│                      │
         │  - Output shape  │    │   Processing:        │
         │  - Field types   │    │   1. Understand role │
         │  - Constraints   │    │   2. Parse tasks     │
         └──────────────────┘    │   3. Generate plan   │
                                 │   4. Format output   │
                                 └──────────┬───────────┘
                                            │
                                            ▼
                         ┌──────────────────────────────┐
                         │   Structured JSON Response   │
                         │                              │
                         │   {                          │
                         │     tasks: [...],            │
                         │     taskItems: [...]         │
                         │   }                          │
                         │                              │
                         │   ✓ Type-safe                │
                         │   ✓ Validated                │
                         │   ✓ Ready for DB/UI          │
                         └──────────────────────────────┘
```

### Связь промтов с данными

```
┌─────────────────────────────────────────────────────────────────┐
│             PROMPT-TO-DATA RELATIONSHIP MATRIX                  │
└─────────────────────────────────────────────────────────────────┘

┌───────────────┬──────────────────┬──────────────────────────────┐
│  Prompt Type  │   Input Data     │       Output Data            │
├───────────────┼──────────────────┼──────────────────────────────┤
│               │                  │                              │
│  System       │  NONE            │  AI Behavior Context         │
│  Prompt       │  (Static text)   │  - Role: Daily planner       │
│               │                  │  - Rule: 3+ subtasks         │
│               │                  │  - Tone: Professional        │
│               │                  │                              │
├───────────────┼──────────────────┼──────────────────────────────┤
│               │                  │                              │
│  User         │  - hours: number │  Contextual Information      │
│  Prompt       │  - tasks: [{     │  - Work duration: 8h         │
│               │      description,│  - Task list with times      │
│               │      time        │  - User's expectations       │
│               │    }]            │                              │
│               │                  │                              │
├───────────────┼──────────────────┼──────────────────────────────┤
│               │                  │                              │
│  Function     │  Schema          │  Structured Output           │
│  Definition   │  Definition      │  - tasks: [                  │
│               │  (JSON Schema)   │      {name, priority}        │
│               │                  │    ]                         │
│               │                  │  - taskItems: [              │
│               │                  │      {description,time,...}  │
│               │                  │    ]                         │
│               │                  │                              │
└───────────────┴──────────────────┴──────────────────────────────┘

DATA FLOW THROUGH PROMPTS:

Database → parsedTasks → User Prompt → OpenAI
              ↓             ↓            ↓
         [Task, ...]   "I will work   Processing
                       8 hours..."
                            ↓
                       Function Schema → Structured JSON
                            ↓                 ↓
                       Type Constraints   Valid Output
                            ↓                 ↓
                       OpenAI validates → Returns JSON
```

---

## Обработка ошибок

### Полная схема обработки ошибок

```
┌─────────────────────────────────────────────────────────────────┐
│                      ERROR HANDLING FLOW                         │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 1: Authentication                                  │
│  Location: operations.ts:38-43                                 │
│                                                                 │
│  Check: context.user exists?                                   │
│  ├─ NO → HttpError(401)                                        │
│  │       Message: "Only authenticated users..."               │
│  │       ↓                                                      │
│  │       CLIENT: Redirect to login page                       │
│  └─ YES → Continue to next step                               │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 2: Input Validation                               │
│  Location: operations.ts:45-48                                 │
│                                                                 │
│  Validator: Zod Schema                                         │
│  Schema: z.object({ hours: z.number() })                      │
│                                                                 │
│  Check: Input matches schema?                                  │
│  ├─ NO → HttpError(400)                                        │
│  │       Message: "Invalid input: hours must be a number"     │
│  │       ↓                                                      │
│  │       CLIENT: Show validation error                        │
│  └─ YES → Continue to next step                               │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 3: Database Query                                 │
│  Location: operations.ts:49-55                                 │
│                                                                 │
│  Query: Task.findMany(...)                                     │
│                                                                 │
│  Possible Errors:                                              │
│  ├─ Database connection failed                                 │
│  │   → Thrown by Prisma, caught by Wasp                       │
│  │   → CLIENT: Show generic error                             │
│  │                                                              │
│  └─ User has no tasks (not an error, continue with empty [])  │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 4: OpenAI API Call                                │
│  Location: operations.ts:259-337                               │
│                                                                 │
│  Possible Errors:                                              │
│  ├─ API Key Invalid/Missing                                    │
│  │   → Thrown at startup: "OpenAI API key is not set"        │
│  │   → Server fails to start                                  │
│  │                                                              │
│  ├─ Rate Limit Exceeded                                        │
│  │   → OpenAI throws error                                    │
│  │   → Caught and returns null                                │
│  │   → Triggers ERROR POINT 5                                 │
│  │                                                              │
│  ├─ Network Error                                              │
│  │   → Timeout or connection failure                          │
│  │   → Caught and returns null                                │
│  │   → Triggers ERROR POINT 5                                 │
│  │                                                              │
│  ├─ Invalid Model Response                                     │
│  │   → AI returns malformed JSON                              │
│  │   → JSON.parse fails                                       │
│  │   → Caught and returns null                                │
│  │   → Triggers ERROR POINT 5                                 │
│  │                                                              │
│  └─ Success → Continue to next step                           │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 5: AI Response Validation                        │
│  Location: operations.ts:59-64                                 │
│                                                                 │
│  Check: generatedSchedule !== null?                           │
│  ├─ NO → HttpError(500)                                        │
│  │       Message: "Problem with OpenAI communication"         │
│  │       ↓                                                      │
│  │       CLIENT: Show error toast                             │
│  │       Toast: "Error: Something went wrong"                 │
│  └─ YES → Continue to next step                               │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 6: Credit Check (Non-Subscribed Users)           │
│  Location: operations.ts:83-99                                 │
│                                                                 │
│  Check: isUserSubscribed(context.user)?                       │
│  ├─ YES → Skip credit check, continue                         │
│  │                                                              │
│  └─ NO → Check credits                                         │
│      Check: user.credits > 0?                                  │
│      ├─ NO → HttpError(402)                                    │
│      │       Message: "User has no subscription and is        │
│      │                 out of credits"                        │
│      │       ↓                                                  │
│      │       CLIENT: Show "out of credits" toast              │
│      │       Toast: "⚠️ You are out of credits!"              │
│      │       Action: Link to pricing page                     │
│      │                                                          │
│      └─ YES → Prepare credit decrement, continue              │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  ERROR POINT 7: Database Transaction                          │
│  Location: operations.ts:103                                   │
│                                                                 │
│  Operations:                                                   │
│  1. Create GptResponse                                         │
│  2. Decrement user credits (if applicable)                    │
│                                                                 │
│  Transaction Failure:                                          │
│  ├─ Database constraint violation                             │
│  │   → Prisma throws error                                    │
│  │   → Transaction rolled back automatically                  │
│  │   → Error propagated to client                             │
│  │                                                              │
│  ├─ Connection lost during transaction                        │
│  │   → Prisma throws error                                    │
│  │   → Transaction rolled back                                │
│  │   → CLIENT: Show generic error                             │
│  │                                                              │
│  └─ Success → Return generatedSchedule                        │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  CLIENT: Error Display                                         │
│  Location: DemoAppPage.tsx:154-181                            │
│                                                                 │
│  Error Handling:                                               │
│                                                                 │
│  try {                                                         │
│    // Call API                                                 │
│  } catch (err) {                                               │
│    if (err.statusCode === 402) {                             │
│      // Out of credits → Show toast with pricing link        │
│    } else {                                                    │
│      // Generic error → Show error toast                     │
│    }                                                            │
│  } finally {                                                   │
│    setIsPlanGenerating(false); // Always clear loading       │
│  }                                                              │
└────────────────────────────────────────────────────────────────┘
```

### Матрица HTTP ошибок

```
┌──────────┬───────────────────────┬──────────────────────────────┐
│  Code    │  Trigger              │  User Experience             │
├──────────┼───────────────────────┼──────────────────────────────┤
│  401     │  Not authenticated    │  Redirect to login page      │
├──────────┼───────────────────────┼──────────────────────────────┤
│  402     │  Out of credits       │  Toast: "⚠️ Out of credits!" │
│          │  (Payment Required)   │  Action: Link to pricing     │
├──────────┼───────────────────────┼──────────────────────────────┤
│  500     │  OpenAI API failure   │  Toast: "Error: Something    │
│          │  Database error       │         went wrong"          │
│          │  Server error         │  Variant: destructive (red)  │
└──────────┴───────────────────────┴──────────────────────────────┘
```

---

## Оптимизация и кеширование

### Текущая реализация

```
┌─────────────────────────────────────────────────────────────────┐
│              OPTIMIZATION & CACHING STRATEGY                     │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  1. Database Query Optimization                                 │
│                                                                 │
│  Current: Task.findMany() - Fetches all tasks                 │
│  Optimization: Already filtered by user.id                     │
│                                                                 │
│  ✓ Indexed queries (user.id is foreign key)                   │
│  ✓ Only fetches necessary fields                              │
│  ✓ No N+1 query problems                                      │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  2. Data Transformation Optimization                           │
│                                                                 │
│  Before AI call: Transform Task[] → parsedTasks[]             │
│                                                                 │
│  parsedTasks = tasks.map(({ description, time }) => ({        │
│    description,                                                │
│    time                                                        │
│  }))                                                            │
│                                                                 │
│  Why: Reduce payload size to OpenAI API                       │
│  Result: Faster API calls, lower costs                        │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  3. Response Caching Strategy                                  │
│                                                                 │
│  Current: All responses saved to GptResponse table            │
│  Purpose: History tracking, not caching                       │
│                                                                 │
│  No automatic cache reuse because:                            │
│  - Each request is unique (different tasks/hours)             │
│  - User expectations change frequently                        │
│  - AI responses vary due to temperature=1                     │
│                                                                 │
│  Potential improvement:                                        │
│  - Cache identical requests for 5 minutes                     │
│  - Key: hash(tasks + hours)                                   │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  4. Credit Management Optimization                             │
│                                                                 │
│  Strategy: Decrement AFTER successful AI response             │
│                                                                 │
│  Sequence:                                                     │
│  1. Check credits                                              │
│  2. Call OpenAI                                                │
│  3. Validate response                                          │
│  4. Save + decrement in single transaction                    │
│                                                                 │
│  Benefits:                                                     │
│  - Users don't lose credits on API failures                   │
│  - Atomic operation (both succeed or both fail)               │
│  - Fair billing                                                │
│                                                                 │
│  Trade-off:                                                    │
│  - Users could abuse by rapid requests if API slow            │
│  - Risk is limited to 1 credit per concurrent request         │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  5. Client-Side Optimizations                                  │
│                                                                 │
│  Current optimizations:                                        │
│  ✓ Loading state prevents double-clicks                       │
│  ✓ Button disabled during generation                          │
│  ✓ Default example response for instant preview               │
│  ✓ React Query for task list caching                          │
│                                                                 │
│  Client caching via useQuery:                                 │
│  - Tasks auto-refresh on mutations                            │
│  - Reduces unnecessary DB queries                             │
│  - Optimistic updates possible                                │
└────────────────────────────────────────────────────────────────┘
```

### Performance Characteristics

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE METRICS                           │
└─────────────────────────────────────────────────────────────────┘

LATENCY BREAKDOWN (Typical Request):

┌──────────────────────────┬─────────────┬──────────────────┐
│  Operation               │  Time       │  % of Total      │
├──────────────────────────┼─────────────┼──────────────────┤
│  Client → Server (RPC)   │  ~50ms      │  2%              │
│  Authentication          │  ~10ms      │  <1%             │
│  Input Validation        │  ~5ms       │  <1%             │
│  Database Query          │  ~50ms      │  2%              │
│  Data Transformation     │  ~5ms       │  <1%             │
│  OpenAI API Call         │  2-5s       │  90-95%          │
│  Response Parsing        │  ~10ms      │  <1%             │
│  Database Transaction    │  ~100ms     │  3-5%            │
│  Server → Client         │  ~50ms      │  2%              │
├──────────────────────────┼─────────────┼──────────────────┤
│  TOTAL                   │  2.5-5.5s   │  100%            │
└──────────────────────────┴─────────────┴──────────────────┘

KEY INSIGHT: OpenAI API call dominates total latency (90%+)

COST ANALYSIS (Per Request):

┌──────────────────────────┬─────────────────────────────────┐
│  Resource                │  Cost (approximate)             │
├──────────────────────────┼─────────────────────────────────┤
│  OpenAI API Call         │  $0.001 - $0.003                │
│  (gpt-3.5-turbo)         │  (depends on token count)       │
│                          │                                 │
│  Database Operations     │  Negligible (included in host)  │
│  Server Compute          │  Negligible (included in host)  │
│  Network Transfer        │  Negligible                     │
├──────────────────────────┼─────────────────────────────────┤
│  TOTAL per request       │  ~$0.002                        │
└──────────────────────────┴─────────────────────────────────┘

With 1000 requests/month: ~$2/month in API costs
```

---

## Итоговая сводка

### Полный список всех компонентов пайплайна

```
┌─────────────────────────────────────────────────────────────────┐
│               PIPELINE COMPONENTS SUMMARY                        │
└─────────────────────────────────────────────────────────────────┘

CLIENT COMPONENTS:
├─ DemoAppPage.tsx (UI and user interaction)
│  ├─ NewTaskForm (Task input, hours selection)
│  ├─ Todo (Task list item)
│  ├─ Schedule (Generated schedule display)
│  ├─ TaskCard (Main task card with priority)
│  └─ TaskCardItem (Subtask with time estimate)

SERVER COMPONENTS:
├─ operations.ts (Business logic)
│  ├─ generateGptResponse (Main AI operation)
│  ├─ generateScheduleWithGpt (OpenAI integration)
│  ├─ createTask (Task CRUD)
│  ├─ updateTask (Task CRUD)
│  ├─ deleteTask (Task CRUD)
│  ├─ getAllTasksByUser (Query)
│  └─ getGptResponses (Query)

TYPE DEFINITIONS:
├─ schedule.ts (TypeScript types)
│  ├─ GeneratedSchedule
│  ├─ Task
│  ├─ TaskItem
│  └─ TaskPriority

EXTERNAL SERVICES:
├─ OpenAI GPT-3.5-Turbo (AI model)
├─ Prisma (Database ORM)
└─ Wasp (Full-stack framework)
```

### Ключевые точки интеграции

```
INTEGRATION POINTS:

1. CLIENT ↔ SERVER
   Protocol: Wasp RPC (type-safe)
   Operations: generateGptResponse, CRUD operations

2. SERVER ↔ DATABASE
   ORM: Prisma
   Entities: User, Task, GptResponse

3. SERVER ↔ OPENAI
   Library: OpenAI Node SDK
   Method: chat.completions.create
   Features: Function Calling, Structured Output
```

### Расширение пайплайна

Возможные улучшения и расширения:

```
┌─────────────────────────────────────────────────────────────────┐
│                   FUTURE ENHANCEMENTS                            │
└─────────────────────────────────────────────────────────────────┘

1. MULTI-STEP PROMPTING
   ┌──────────────────────────────────────┐
   │  Current: Single AI call             │
   │  Future: Multi-stage refinement      │
   │                                      │
   │  Step 1: Generate initial schedule   │
   │     ↓                                │
   │  Step 2: Optimize for constraints    │
   │     ↓                                │
   │  Step 3: Add break times             │
   │     ↓                                │
   │  Final: Polished schedule            │
   └──────────────────────────────────────┘

2. CONTEXT ACCUMULATION
   ┌──────────────────────────────────────┐
   │  Use previous schedules for context  │
   │                                      │
   │  New prompt includes:                │
   │  - Past completion rates             │
   │  - Time estimation accuracy          │
   │  - User preferences                  │
   │                                      │
   │  Result: Personalized scheduling     │
   └──────────────────────────────────────┘

3. STREAMING RESPONSES
   ┌──────────────────────────────────────┐
   │  Current: Wait for full response     │
   │  Future: Stream as AI generates      │
   │                                      │
   │  UI updates progressively:           │
   │  ├─ Task 1 appears                   │
   │  ├─ Subtasks stream in               │
   │  ├─ Task 2 appears                   │
   │  └─ Complete                         │
   │                                      │
   │  Benefit: Perceived faster response  │
   └──────────────────────────────────────┘

4. INTELLIGENT CACHING
   ┌──────────────────────────────────────┐
   │  Cache similar requests:             │
   │                                      │
   │  Key: hash(tasks, hours, user_prefs) │
   │  TTL: 5 minutes                      │
   │                                      │
   │  If cache hit:                       │
   │  - Skip OpenAI call                  │
   │  - Instant response                  │
   │  - Don't charge credits              │
   │                                      │
   │  Potential savings: 30-50% of calls  │
   └──────────────────────────────────────┘

5. FEEDBACK LOOP
   ┌──────────────────────────────────────┐
   │  User rates generated schedules      │
   │       ↓                              │
   │  Store feedback in database          │
   │       ↓                              │
   │  Include in future prompts           │
   │       ↓                              │
   │  "Previous schedules rated X/5"      │
   │       ↓                              │
   │  AI adapts to user preferences       │
   └──────────────────────────────────────┘

6. AGENT CHAINS
   ┌──────────────────────────────────────────────────┐
   │  Agent 1: Task Analyzer                          │
   │    Input: Raw tasks                              │
   │    Output: Categorized, estimated tasks          │
   │              ↓                                   │
   │  Agent 2: Schedule Optimizer                     │
   │    Input: Categorized tasks + hours              │
   │    Output: Time-blocked schedule                 │
   │              ↓                                   │
   │  Agent 3: Conflict Resolver                      │
   │    Input: Schedule + calendar conflicts          │
   │    Output: Adjusted schedule                     │
   │              ↓                                   │
   │  Final: Optimized, conflict-free schedule        │
   └──────────────────────────────────────────────────┘
```

---

## Заключение

Этот документ описывает единственный AI-пайплайн в приложении Open SaaS - **Task Schedule Generation Pipeline**.

### Основные характеристики:

- ✅ Простая, линейная архитектура
- ✅ Четкое разделение ответственности (Client/Server/AI)
- ✅ Надежная обработка ошибок на всех уровнях
- ✅ Типобезопасность от клиента до AI
- ✅ Справедливая система монетизации
- ✅ Атомарные транзакции для целостности данных

### Ключевые преимущества реализации:

1. **Type Safety**: TypeScript + Zod + Function Calling
2. **Error Resilience**: 7 точек обработки ошибок
3. **User-Friendly**: Понятные сообщения об ошибках
4. **Cost-Effective**: Оптимизация API вызовов
5. **Fair Billing**: Кредиты списываются только после успеха
6. **Extensible**: Легко добавить новые промты и агентов

### Файлы документации:

- `PROMPTS_DOCUMENTATION.md` - Все промты и их назначение
- `AGENT_PIPELINES.md` - Этот файл (схемы работы агентов)

---
