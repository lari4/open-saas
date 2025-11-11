# AI Prompts Documentation

Это документация всех AI промтов, используемых в приложении Open SaaS. Промты сгруппированы по функциональным областям.

## Содержание

1. [Task Scheduling & Planning](#task-scheduling--planning)
   - [System Prompt: Daily Planner Expert](#system-prompt-daily-planner-expert)
   - [User Prompt: Task Schedule Generation](#user-prompt-task-schedule-generation)
   - [Function Definition: Parse Today's Schedule](#function-definition-parse-todays-schedule)

---

## Task Scheduling & Planning

Эта группа промтов отвечает за создание оптимального расписания задач на день с использованием AI. Система берет список задач пользователя и количество рабочих часов, и генерирует структурированное расписание с подзадачами, приоритетами и временными оценками.

### System Prompt: Daily Planner Expert

**Местоположение:** `template/app/src/demo-ai-app/operations.ts:264-265`

**Назначение:**
Этот системный промт определяет роль и поведение AI-ассистента как эксперта по планированию дня. Он инструктирует модель:
- Разбивать каждую основную задачу на минимум 3 подзадачи
- Создавать детальный план достижения целей
- Использовать мотивационный подход (обещание награды за выполнение инструкций)

**Промт:**

```text
you are an expert daily planner. you will be given a list of main tasks and an estimated time to complete each task. You will also receive the total amount of hours to be worked that day. Your job is to return a detailed plan of how to achieve those tasks by breaking each task down into at least 3 subtasks each. MAKE SURE TO ALWAYS CREATE AT LEAST 3 SUBTASKS FOR EACH MAIN TASK PROVIDED BY THE USER! YOU WILL BE REWARDED IF YOU DO.
```

**Ключевые особенности:**
- ✅ Четко определяет роль (expert daily planner)
- ✅ Устанавливает минимальное требование (минимум 3 подзадачи)
- ✅ Использует капитализацию для акцента на важных требованиях
- ✅ Включает мотивационный элемент ("YOU WILL BE REWARDED")

---

### User Prompt: Task Schedule Generation

**Местоположение:** `template/app/src/demo-ai-app/operations.ts:269-271`

**Назначение:**
Этот пользовательский промт передает конкретные данные для генерации расписания. Он динамически формируется с учетом:
- Количества рабочих часов в день
- Списка задач с описаниями и временными оценками
- Запроса на разбиение задач на подзадачи с временем и приоритетами

**Шаблон промта:**

```typescript
`I will work ${hours} hours today. Here are the tasks I have to complete: ${JSON.stringify(parsedTasks)}. Please help me plan my day by breaking the tasks down into actionable subtasks with time and priority status.`
```

**Пример сгенерированного промта:**

```text
I will work 8 hours today. Here are the tasks I have to complete: [{"description":"Respond to emails","time":"2"},{"description":"Learn WASP","time":"3"},{"description":"Read a book","time":"1"}]. Please help me plan my day by breaking the tasks down into actionable subtasks with time and priority status.
```

**Входные данные:**
- `hours` (number) - количество часов работы
- `parsedTasks` (array) - массив объектов с полями:
  - `description` (string) - описание задачи
  - `time` (string) - оценка времени выполнения

**Ключевые особенности:**
- 🔄 Динамическая генерация на основе пользовательских данных
- 📊 Передача структурированных данных в JSON формате
- 🎯 Четкий запрос на конкретные выходные данные (подзадачи, время, приоритет)

---

### Function Definition: Parse Today's Schedule

**Местоположение:** `template/app/src/demo-ai-app/operations.ts:274-329`

**Назначение:**
Это определение функции для OpenAI Function Calling API, которое обеспечивает структурированный и типизированный вывод от AI. Функция `parseTodaysSchedule` определяет точную схему данных, которую должна вернуть модель.

**Структура функции:**

```typescript
{
  type: "function",
  function: {
    name: "parseTodaysSchedule",
    description: "parses the days tasks and returns a schedule",
    parameters: {
      type: "object",
      properties: {
        tasks: {
          type: "array",
          description: "Name of main tasks provided by user, ordered by priority",
          items: {
            type: "object",
            properties: {
              name: {
                type: "string",
                description: "Name of main task provided by user",
              },
              priority: {
                type: "string",
                enum: ["low", "medium", "high"],
                description: "task priority",
              },
            },
          },
        },
        taskItems: {
          type: "array",
          items: {
            type: "object",
            properties: {
              description: {
                type: "string",
                description: 'detailed breakdown and description of sub-task related to main task. e.g., "Prepare your learning session by first reading through the documentation"',
              },
              time: {
                type: "number",
                description: "time allocated for a given subtask in hours, e.g. 0.5",
              },
              taskName: {
                type: "string",
                description: "name of main task related to subtask",
              },
            },
          },
        },
      },
      required: ["tasks", "taskItems", "time", "priority"],
    },
  },
}
```

**Выходные данные:**

```typescript
interface GeneratedSchedule {
  tasks: Array<{
    name: string;           // Название основной задачи
    priority: "low" | "medium" | "high";  // Приоритет задачи
  }>;
  taskItems: Array<{
    description: string;    // Детальное описание подзадачи
    time: number;          // Время в часах (например, 0.5)
    taskName: string;      // Название связанной основной задачи
  }>;
}
```

**Пример выходных данных:**

```json
{
  "tasks": [
    {
      "name": "Respond to emails",
      "priority": "high"
    },
    {
      "name": "Learn WASP",
      "priority": "medium"
    }
  ],
  "taskItems": [
    {
      "description": "Check and respond to important emails",
      "time": 1,
      "taskName": "Respond to emails"
    },
    {
      "description": "Organize and prioritize remaining emails",
      "time": 0.5,
      "taskName": "Respond to emails"
    },
    {
      "description": "Watch tutorial video on WASP",
      "time": 0.5,
      "taskName": "Learn WASP"
    }
  ]
}
```

**Ключевые особенности:**
- 📋 Строгая типизация выходных данных
- 🔗 Связь подзадач с основными задачами через `taskName`
- ⏱️ Числовое представление времени для удобных вычислений
- 🎯 Enum для приоритетов (предотвращает некорректные значения)
- 📝 Примеры в описаниях полей для лучшего понимания моделью

---

## Конфигурация модели

**Местоположение:** `template/app/src/demo-ai-app/operations.ts:259-337`

**Используемая модель:** `gpt-3.5-turbo`

**Параметры запроса:**

```typescript
{
  model: "gpt-3.5-turbo",
  messages: [
    { role: "system", content: "..." },
    { role: "user", content: "..." }
  ],
  tools: [...],
  tool_choice: {
    type: "function",
    function: { name: "parseTodaysSchedule" }
  },
  temperature: 1
}
```

**Параметры:**
- `temperature: 1` - Максимальная креативность для разнообразных предложений по планированию
- `tool_choice` - Принудительное использование функции для гарантированного структурированного вывода

---

## Обработка результатов

**Местоположение:** `template/app/src/demo-ai-app/operations.ts:339-341`

Результат извлекается из ответа OpenAI и парсится:

```typescript
const gptResponse = completion?.choices[0]?.message?.tool_calls?.[0]?.function.arguments;
return gptResponse !== undefined ? JSON.parse(gptResponse) : null;
```

**Обработка ошибок:**
- Возвращает `null` если ответ не получен
- Выбрасывает HTTP 500 ошибку на уровне вызывающей функции (строка 59-64)

---

## Конфигурация окружения

**Файл конфигурации:** `template/app/.env.server.example`

**Необходимые переменные окружения:**

```bash
OPENAI_API_KEY=sk-k...
```

**Инициализация клиента:**

```typescript
// template/app/src/demo-ai-app/operations.ts:18-25
const openAi = setUpOpenAi();

function setUpOpenAi(): OpenAI {
  if (process.env.OPENAI_API_KEY) {
    return new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  } else {
    throw new Error("OpenAI API key is not set");
  }
}
```

**Обязательные шаги для настройки:**
1. Получить API ключ от OpenAI (https://platform.openai.com/api-keys)
2. Создать файл `.env.server` на основе `.env.server.example`
3. Добавить ключ в переменную `OPENAI_API_KEY`
4. Перезапустить сервер

---

## Использование

### Клиентская часть

**Файл:** `template/app/src/demo-ai-app/DemoAppPage.tsx`

**Вызов генерации расписания:**

```typescript
// Строки 145-181
const handleGeneratePlan = async () => {
  try {
    setIsPlanGenerating(true);
    const response = await generateGptResponse({
      hours: todaysHours,
    });
    if (response) {
      setResponse(response);
    }
  } catch (err: any) {
    if (err.statusCode === 402) {
      // Пользователь исчерпал кредиты
      toast({
        title: "⚠️ You are out of credits!",
        action: (
          <ToastAction altText="Go to pricing page">
            <Link to={routes.PricingPageRoute.to}>
              Go to pricing page
            </Link>
          </ToastAction>
        ),
      });
    } else {
      // Общая ошибка
      toast({
        title: "Error",
        description: err.message || "Something went wrong",
        variant: "destructive",
      });
    }
  } finally {
    setIsPlanGenerating(false);
  }
};
```

### Серверная часть

**Файл:** `template/app/src/demo-ai-app/operations.ts`

**Входные данные операции:**

```typescript
// Строки 28-32
const generateGptResponseInputSchema = z.object({
  hours: z.number(),
});

type GenerateGptResponseInput = z.infer<typeof generateGptResponseInputSchema>;
```

**Основная функция:**

```typescript
// Строки 34-106
export const generateGptResponse: GenerateGptResponse<
  GenerateGptResponseInput,
  GeneratedSchedule
> = async (rawArgs, context) => {
  // 1. Проверка аутентификации
  if (!context.user) {
    throw new HttpError(401, "Only authenticated users are allowed");
  }

  // 2. Валидация входных данных
  const { hours } = ensureArgsSchemaOrThrowHttpError(
    generateGptResponseInputSchema,
    rawArgs,
  );

  // 3. Получение задач пользователя из БД
  const tasks = await context.entities.Task.findMany({
    where: { user: { id: context.user.id } },
  });

  // 4. Вызов OpenAI API
  const generatedSchedule = await generateScheduleWithGpt(tasks, hours);
  if (generatedSchedule === null) {
    throw new HttpError(500, "Problem with OpenAI communication");
  }

  // 5. Сохранение результата и управление кредитами
  const createResponse = context.entities.GptResponse.create({
    data: {
      user: { connect: { id: context.user.id } },
      content: JSON.stringify(generatedSchedule),
    },
  });

  const transactions: PrismaPromise<GptResponse | User>[] = [createResponse];

  // Уменьшение кредитов для пользователей без подписки
  if (!isUserSubscribed(context.user)) {
    if (context.user.credits > 0) {
      const decrementCredit = context.entities.User.update({
        where: { id: context.user.id },
        data: { credits: { decrement: 1 } },
      });
      transactions.push(decrementCredit);
    } else {
      throw new HttpError(402, "User has no subscription and is out of credits");
    }
  }

  // 6. Выполнение транзакции и возврат результата
  await prisma.$transaction(transactions);
  return generatedSchedule;
};
```

---

## Система кредитов и монетизация

**Логика проверки подписки:**

```typescript
// Строки 108-113
function isUserSubscribed(user: User) {
  return (
    user.subscriptionStatus === SubscriptionStatus.Active ||
    user.subscriptionStatus === SubscriptionStatus.CancelAtPeriodEnd
  );
}
```

**Правила использования:**
- ✅ Пользователи с активной подпиской - неограниченное использование
- 💳 Пользователи без подписки - расходуют кредиты (1 кредит = 1 генерация)
- ⛔ Пользователи без кредитов - получают HTTP 402 ошибку

**Последовательность операций:**
1. Проверка аутентификации
2. Валидация входных данных
3. Получение данных пользователя
4. Вызов OpenAI API
5. Сохранение результата
6. Уменьшение кредитов (если нет подписки)

**Важно:** Кредиты уменьшаются ПОСЛЕ успешного получения ответа от OpenAI, чтобы пользователи не теряли кредиты при ошибках API.

---

## Статистика и хранение

**Сохранение ответов:**

Все генерированные расписания сохраняются в БД в таблице `GptResponse`:

```typescript
// Строки 66-71
const createResponse = context.entities.GptResponse.create({
  data: {
    user: { connect: { id: context.user.id } },
    content: JSON.stringify(generatedSchedule),
  },
});
```

**Получение истории:**

```typescript
// Строки 214-228
export const getGptResponses: GetGptResponses<void, GptResponse[]> = async (
  _args,
  context,
) => {
  if (!context.user) {
    throw new HttpError(401);
  }
  return context.entities.GptResponse.findMany({
    where: {
      user: { id: context.user.id },
    },
  });
};
```

---

## Итоговая сводка

**Все промты в приложении:**

| Тип | Назначение | Местоположение |
|-----|-----------|----------------|
| System Prompt | Определение роли AI как эксперта по планированию | `operations.ts:264-265` |
| User Prompt | Передача задач и часов для генерации расписания | `operations.ts:269-271` |
| Function Definition | Схема структурированного вывода (JSON Schema) | `operations.ts:274-329` |

**Ключевые технологии:**
- 🤖 OpenAI GPT-3.5-Turbo
- 🛠️ Function Calling API
- 🔒 Аутентификация и авторизация
- 💰 Система кредитов и подписок
- 💾 Сохранение истории ответов

**Файлы проекта:**
- `template/app/src/demo-ai-app/operations.ts` - серверная логика
- `template/app/src/demo-ai-app/DemoAppPage.tsx` - UI и клиентская логика
- `template/app/src/demo-ai-app/schedule.ts` - TypeScript типы

---
