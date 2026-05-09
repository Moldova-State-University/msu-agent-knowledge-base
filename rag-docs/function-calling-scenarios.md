# Function Calling Scenarios

## 1. Purpose

This document defines when and which tool should be invoked in function calling scenarios. It also describes possible triggers, expected results, and behavior for each tool.

The agent answers user questions related to different university domains, including:

- Class schedule
- Teachers and university staff
- General university regulations, policies, and academic procedures

## 2. Available Tools

| Tool               | Description |
|--------------------|-------------|
| `get_schedule`     | Handles all timetable and scheduling-related questions |
| `get_teacher_info` | Handles questions about teachers and administrative staff |
| `rag_retrieval`    | Handles general university questions via vector search over documents |


## 3. High-Level Routing Logic

The agent first analyzes the user query and determines its intent:

- If the question is about **class schedule, classroom, current/next lesson, week type, teacher in timetable** → invoke `get_schedule`
- If the question is about **teacher/staff information, office location, dean/administration contacts** → invoke `get_teacher_info`
- If the question is about **general university knowledge, rules, certificates, transfer policies, curriculum, academic process, official procedures** → invoke `rag_retrieval`
- If the query contains signals for **more than one domain**, the agent may invoke multiple tools in sequence or choose the primary tool based on dominant intent


## 4. Trigger → Tool → Expected Behavior Mapping

| Trigger / User intent                               | Tool                 | Expected behavior |
|-----------------------------------------------------|----------------------|---|
| "Какая сейчас пара?"                                | `get_schedule`       | Return current class based on time, date, group, and schedule data |
| "Какая следующая пара?"                             | `get_schedule`       | Return next scheduled class |
| "В каком кабинете сейчас матан?"                    | `get_schedule`       | Return classroom for the matching lesson |
| "Какая сейчас неделя - чётная или нечётная?"        | `get_schedule`       | Return current academic week type |
| "Какая пара у IA2404 в понедельник?"                | `get_schedule`       | Return timetable for the specified group and day |
| "Где найти преподавателя Иванова?"                  | `get_teacher_info`   | Return office, department, or known location of teacher |
| "Где сейчас декан?"                                 | `get_teacher_info`   | Return available location/status of dean if data exists |
| "Как связаться с куратором?"                        | `get_teacher_info`   | Return contact info or department contact |
| "Как получить справку об обучении?"                 | `rag_retrieval`      | Search university documents and return procedure |
| "Какие правила перевода на другой факультет?"       | `rag_retrieval`      | Retrieve policy from knowledge base |
| "Какие документы нужны для академического отпуска?" | `rag_retrieval`      | Retrieve required documents and steps |
| "Что входит в учебный план?"                        | `rag_retrieval`      | Return curriculum-related information from indexed documents |

## 5. Tool Specifications

### 5.1 `get_schedule`

**Purpose:** Handles all timetable-related questions.

**Typical triggers:** The tool is invoked when the query contains schedule-related intent - current/next class, today's classes, weekly timetable, classroom, subject by time, teacher by schedule, even/odd week, group timetable.

**Example trigger phrases:**
- "Какая сейчас пара?"
- "Что у нас следующим?"
- "Где проходит следующая лекция?"
- "Какая аудитория у физики?"
- "Какая неделя?"
- "Расписание на завтра"
- "Какие пары у группы IA2404 в пятницу?"

**Expected input parameters:**

| Parameter | Description |
|-----------|-------------|
| `group` | Student group identifier |
| `date` | Specific date |
| `time` | Time of day |
| `day_of_week` | Day of the week |
| `subject` | Subject name |
| `teacher_name` | Teacher name |
| `week_type` | Even or odd week |

**Expected result:** Structured schedule data - lesson name, start/end time, classroom, teacher, group, week type, date/day.

**Fallback:** `"По указанным параметрам расписание не найдено"`

### 5.2 `get_teacher_info`

**Purpose:** Handles questions about teachers and administrative staff.

**Typical triggers:** The tool is invoked when the query asks about teacher location, office, department, dean/admin office, how to find a specific staff member, teacher availability, or contact information.

**Example trigger phrases:**
- "Где найти Петрова?"
- "В каком кабинете преподаватель Сидоров?"
- "Как найти декана?"
- "Где сейчас заведующий кафедрой?"
- "Как связаться с преподавателем?"

**Expected input parameters:**

| Parameter | Description |
|-----------|-------------|
| `person_name` | Full or partial name |
| `role` | Role/position |
| `department` | Department name |
| `building` | Building identifier |
| `availability_flag` | Whether to check current availability |

**Expected result:** Staff metadata - full name, role/position, department, office/cabinet, building, contact info, known schedule/location status.

**Fallback:** `"Не удалось найти актуальную информацию по этому сотруднику"`


### 5.3 `rag_retrieval`

**Purpose:** Handles general university questions by retrieving information from university documents and knowledge base.

**Typical triggers:** The tool is invoked when the question concerns policies, procedures, regulations, curriculum, academic documents, certificates, transfer/scholarship/dormitory rules, exams, retakes, attendance, or any general informational question not covered by the other tools.

**Example trigger phrases:**
- "Как получить справку?"
- "Как перевестись на другой факультет?"
- "Что делать при академической задолженности?"
- "Какие правила пересдачи?"
- "Как оформить академический отпуск?"

**Expected input parameters:**

| Parameter | Description |
|-----------|-------------|
| `user_question` | Original user query |
| `reformulated_query` | Rewritten query for retrieval |
| `filters` | Optional filters by document type/category |

**Expected result:** Relevant document chunks, extracted answer, supporting context, and optionally source references.

**Pipeline:**
1. Query is reformulated
2. Relevant chunks are retrieved from vector DB
3. Results are reranked
4. Context is assembled
5. Final response is generated grounded in retrieved documents

**Fallback:** `"В базе знаний не найдено достаточной информации для точного ответа"`

## 6. Routing Priority Rules

### Rule 1 - Prefer `get_schedule` for timetable-specific intent

If the user asks about pair/class now, next class, room for a subject, even/odd week, or timetable for a group → `get_schedule` has priority.

> "Где сейчас пара у IA2404?" → `get_schedule`

### Rule 2 - Prefer `get_teacher_info` for person/location intent

If the user asks where a person can be found, what office they are in, or how to contact them → use `get_teacher_info`.

> "Где найти декана?" → `get_teacher_info`

### Rule 3 - Use `rag_retrieval` for policy/procedure/document questions

If the answer must come from rules, documents, or regulations → use `rag_retrieval`.

> "Как оформить перевод?" → `rag_retrieval`

### Rule 4 - Use multiple tools if needed

If the query mixes intents, the agent may invoke tools sequentially.

> "Где преподаватель Иванов и какие у него сегодня пары?"
> 1. `get_teacher_info`
> 2. `get_schedule`


## 7. Function Calling Scenarios

### Scenario 1 - Schedule Query

**User query:** "Какая сейчас пара у группы IA2404?"

- **Trigger detected:** current lesson + group + schedule intent
- **Tool invoked:** `get_schedule`
- **Expected result:** current lesson, time, classroom, teacher
- **Final response:** concise answer with current class information

### Scenario 2 - Teacher Location Query

**User query:** "Где можно найти преподавателя Смирнова?"

- **Trigger detected:** person lookup + location
- **Tool invoked:** `get_teacher_info`
- **Expected result:** office/department/contact/location if available
- **Final response:** where the person can be found

### Scenario 3 - University Rules Query

**User query:** "Какие документы нужны для академического отпуска?"

- **Trigger detected:** procedure + documents + policy knowledge
- **Tool invoked:** `rag_retrieval`
- **Expected result:** retrieved document chunks and grounded answer
- **Final response:** list of required documents and procedure steps


### Scenario 4 - Mixed Query

**User query:** "Кто сейчас ведёт пару у IA2404 и где найти этого преподавателя потом?"

- **Trigger detected:** mixed schedule + teacher lookup
- **Tools invoked:**
    1. `get_schedule` → teacher currently teaching the lesson
    2. `get_teacher_info` → office/location/contact of that teacher
- **Final response:** combined answer

## 8. Fallback Behavior

| Situation | Behavior |
|-----------|----------|
| No tool clearly matches | Default to `rag_retrieval` for general university questions |
| Required parameters are missing | Ask a clarifying question |
| Tool result is empty | Return a transparent fallback message |

**Example fallback messages:**
- "Уточните, пожалуйста, группу"
- "Уточните, какого именно преподавателя вы имеете в виду"
- "По этому запросу не найдено актуальной информации"
