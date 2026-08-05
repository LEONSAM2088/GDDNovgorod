# Unity DOTS и ECS — подробное руководство

Полный разбор: что такое **ECS**, что такое **DOTS**, чем они отличаются от обычных `MonoBehaviour`, когда это нужно, и как это выглядит на примерах.

Связанный контекст обучения: игра с большим числом юнитов и социальной симуляцией (в духе The Guild 2+).

---

## 1. Одной фразой

| Термин | Смысл |
|---|---|
| **ECS** | Архитектурный паттерн: **Entity + Component + System** |
| **DOTS** | Стек технологий Unity для data-oriented подхода (Entities, Jobs, Burst и др.) |

**ECS** — идея «как организовать данные и логику».  
**DOTS** — «как Unity это реализует быстро».

> DOTS ≠ только ECS.  
> ECS — ядро. DOTS = ECS + многопоточность + компилятор скорости + экосистема пакетов.

---

## 2. Зачем это вообще придумали

Классический Unity удобен:

```text
GameObject
 ├─ Transform
 ├─ Rigidbody
 └─ EnemyAI : MonoBehaviour   ← данные + логика в одном объекте
```

Проблема при **тысячах** объектов:

1. Объекты разбросаны по памяти как попало  
2. CPU постоянно «прыгает» за данными (плохо для кэша)  
3. Многопоточно обновлять `Update()` у тысяч скриптов сложно и опасно  
4. Много лишнего (имена, слои, лишние поля), даже если юнит простой  

**Data-Oriented Design** говорит:

> Разложи данные плотно и однотипно.  
> Гоняй одну операцию сразу по тысячам записей.

Это и есть дух ECS/DOTS.

---

## 3. ECS: три кита

### 3.1. Entity (сущность)

Entity — это почти просто **ID**.

Не «умный объект», а ярлык:

```text
Entity #1024
Entity #1025
Entity #1026
```

Сам по себе entity ничего не умеет. Смысл появляется, когда к нему прикреплены компоненты.

Аналогия:

- Excel: номер строки  
- База данных: primary key  
- Классика Unity: пустой GameObject без скриптов — но ещё «тоньше»

### 3.2. Component (компонент) — только данные

В ECS компонент почти никогда не содержит игровую логику. Только поля.

```csharp
using Unity.Entities;
using Unity.Mathematics;

public struct Position : IComponentData
{
    public float3 Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

public struct Health : IComponentData
{
    public float Current;
    public float Max;
}

public struct EnemyTag : IComponentData {} // пустой тег-маркер
```

Примеры сущностей:

```text
Игрок:     Position + Velocity + Health + PlayerTag
Враг:      Position + Velocity + Health + EnemyTag
Пуля:      Position + Velocity + Damage
Декор:     Position                 (не двигается)
```

Состав компонентов = тип объекта.  
Не нужен класс `Enemy` с наследованием — достаточно набора данных.

### 3.3. System (система) — только логика

Система ищет все entity с нужным набором компонентов и обрабатывает их пачкой.

```text
MoveSystem:
  взять всех с Position + Velocity
  Position += Velocity * deltaTime

DamageSystem:
  взять всех с Health + DamageThisFrame
  Health.Current -= Damage
  если Health <= 0 → уничтожить / пометить мёртвым
```

Одна система = одна ответственность.

---

## 4. Сравнение: MonoBehaviour vs ECS

### Классика

```csharp
public class Mover : MonoBehaviour
{
    public Vector3 velocity;

    void Update()
    {
        transform.position += velocity * Time.deltaTime;
    }
}
```

У каждого объекта свой скрипт, свой `Update`, свои данные рядом с логикой.

### ECS (идея)

```csharp
// Данные
public struct Velocity : IComponentData
{
    public float3 Value;
}

// Логика для ВСЕХ сразу
public partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

Разница не в «синтаксисе ради синтаксиса», а в том, что система обрабатывает **массив однотипных данных**, а не прыгает по случайным объектам сцены.

---

## 5. Что такое DOTS подробно

**DOTS = Data-Oriented Technology Stack**

Основные части:

| Технология | Роль |
|---|---|
| **Entities** | Реализация ECS в Unity |
| **C# Job System** | Многопоточные задачи |
| **Burst Compiler** | Компилирует числовой код очень быстро (LLVM) |
| **Collections** | Спец-контейнеры без лишнего GC (`NativeArray` и др.) |
| **Mathematics** | `float3`, `quaternion` и SIMD-friendly математика |
| Часто рядом | Unity Physics (DOTS), Netcode for Entities, Entities Graphics |

Связка силы:

```text
Entities  → удобно разложить данные
Jobs      → посчитать на многих ядрах CPU
Burst     → сделать этот расчёт очень быстрым
```

Можно использовать Jobs+Burst и без полного ECS.  
Но «настоящий DOTS-геймплей» обычно = Entities + Jobs + Burst.

---

## 6. Как данные лежат в памяти (почему быстро)

Упрощённо:

### OOP / GameObject

```text
[EnemyA объект] ....дырка в памяти.... [EnemyB] .... [EnemyC]
 кажый со своими полями вразнобой
```

CPU загружает кэш-линию, а полезных данных мало.

### ECS / Chunks

Unity группирует entity с **одинаковым набором компонентов** в **архетипы** и хранит их в **chunks** (кусках памяти):

```text
Архетип: Position + Velocity + Health

Chunk:
Positions:  [p0 p1 p2 p3 p4 ...]
Velocities: [v0 v1 v2 v3 v4 ...]
Healths:    [h0 h1 h2 h3 h4 ...]
```

Система идёт линейно: взял пачку позиций, пачку скоростей — посчитал.  
Это дружелюбно к CPU cache.

Если добавить новый компонент (например `Stunned`), entity может переехать в другой архетип. Это нормально, но частые структурные изменения дороже, чем просто менять числа.

---

## 7. Базовые примеры

### Пример A — движущиеся юниты

**Компоненты**

```csharp
using Unity.Entities;
using Unity.Mathematics;

public struct UnitVelocity : IComponentData
{
    public float3 Value;
}
```

`LocalTransform` уже есть в Entities (позиция/поворот/масштаб).

**Система**

```csharp
using Unity.Entities;
using Unity.Transforms;

public partial struct UnitMoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<UnitVelocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

Тысяча юнитов? Система та же. Меняется только количество entity.

### Пример B — needs (голод/энергия) для NPC

Подходит для Guild-like симуляции.

```csharp
public struct Needs : IComponentData
{
    public float Hunger;   // 0..100
    public float Energy;   // 0..100
}

public struct AliveTag : IComponentData {}
```

```csharp
public partial struct NeedsDecaySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var needs in SystemAPI.Query<RefRW<Needs>>().WithAll<AliveTag>())
        {
            needs.ValueRW.Hunger = math.min(100f, needs.ValueRW.Hunger + 0.5f * dt);
            needs.ValueRW.Energy = math.max(0f, needs.ValueRW.Energy - 0.2f * dt);
        }
    }
}
```

Отдельная система может искать еду, когда `Hunger > 70`.

### Пример C — теги вместо наследования

```csharp
public struct PlayerTag : IComponentData {}
public struct NpcTag : IComponentData {}
public struct CriminalTag : IComponentData {}
```

```csharp
// Только NPC
foreach (var needs in SystemAPI.Query<RefRW<Needs>>().WithAll<NpcTag>())
{ ... }

// Только преступники
foreach (var (transform, needs) in
         SystemAPI.Query<RefRO<LocalTransform>, RefRO<Needs>>().WithAll<CriminalTag>())
{ ... }
```

Вместо иерархий классов — комбинации тегов/компонентов.

### Пример D — Job + Burst (ускорение)

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

public partial struct UnitMoveSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        // ScheduleParallel — работа на нескольких потоках
        new MoveJob { DeltaTime = dt }.ScheduleParallel();
    }
}

[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform transform, in UnitVelocity velocity)
    {
        transform.Position += velocity.Value * DeltaTime;
    }
}
```

Именно тут DOTS раскрывается: одна логика → тысячи entity → несколько ядер CPU → Burst-код.

---

## 8. Как создают entity (концептуально)

### В рантайме

```csharp
var entity = entityManager.CreateEntity();
entityManager.AddComponentData(entity, new LocalTransform { ... });
entityManager.AddComponentData(entity, new UnitVelocity { Value = new float3(1,0,0) });
entityManager.AddComponentData(entity, new Needs { Hunger = 10, Energy = 100 });
```

### Через Authoring (привычный Unity-workflow)

1. На GameObject вешаешь Authoring-монобех  
2. Baker «запекает» его в entity + components при конвертации сабсцены  

Идея:

```text
В Editor ты всё ещё кликаешь по объектам
В Runtime мир симулируется как Entities
```

Это гибридный вход в DOTS: контент удобно расставлять, симуляция идёт data-oriented.

---

## 9. Queries, командные буферы, жизненный цикл

### Query

«Дай всех, у кого есть X и нет Y»:

```csharp
SystemAPI.Query<RefRW<Needs>>()
    .WithAll<NpcTag>()
    .WithNone<DeadTag>();
```

### Structural changes

Добавить/удалить компонент, создать/уничтожить entity — это **структурное изменение**.  
Его нельзя бездумно делать посреди итерации query.

Обычно используют **EntityCommandBuffer**:

```text
в системе: "после прохода пометь этих как Dead / уничтожь"
потом буфер применяется безопасно
```

### Системы упорядочиваются

Можно задавать порядок: сначала ввод → потом движение → потом коллизии → потом уничтожение.

В DOTS порядок систем — часть дизайна, не «надеюсь Update вызовется раньше».

---

## 10. DOTS ≠ полный отказ от GameObjects

Реальность проектов:

| Задача | Часто делают так |
|---|---|
| 10 000 NPC needs/jobs/path | Entities |
| Диалог с важным персонажем | Обычный UI + обычный код |
| Меню династии / инвентарь | uGUI / UI Toolkit |
| Камера, VFX рядом с игроком | GameObjects / hybrid |
| Массовый рендер толпы | Entities Graphics / инстансинг |

Формула большинства сложных игр:

> **Симуляция мира на ECS, presentation/UI на классике.**

---

## 11. Когда DOTS нужен, а когда нет

### Нужен / очень полезен

- тысячи+ однотипных агентов  
- тяжёлая симуляция каждый кадр/тик  
- толпа, пули, RTS, демография города, трафик  
- хочешь выжать CPU многопотоком  

### Не нужен (пока)

- обучение основам Unity  
- 2 куба по сети  
- небольшой платформер  
- игра, где узкое место — дизайн/контент, а не CPU  
- команда ещё не освоила обычный геймплейный пайплайн  

### Частая ошибка

Переписать весь проект на DOTS «потому что звучит современно».  
Получается дольше, сложнее отладка, а выигрыша нет, если объектов мало.

Правило:

> Сначала измерь. Если CPU умирает на массовой логике — переноси **узкое место** в ECS.

---

## 12. ECS и игра типа The Guild (социалка + много юнитов)

### Что хорошо ложится в ECS

- `Needs` (голод, энергия, настроение-число)  
- работа/расписание как данные  
- движение по городу  
- простые экономические счётчики  
- поиск «все пекари города» / «все голодные»  

### Что плохо пихать «чисто в ECS»

- глубокий граф отношений (A любит B, B ненавидит C)  
- уникальные сюжетные сцены  
- ветвящиеся диалоги  
- редкие ивенты с кучей исключений  

### Рекомендуемый гибрид

```text
Entity NPC
 ├─ LocalTransform
 ├─ Needs
 ├─ JobId / Schedule
 ├─ Profession
 └─ NpcId (ссылка во внешние таблицы)

Вне ECS:
 ├─ OpinionTable[(a,b) → trust/love/fear]
 ├─ FamilyGraph
 ├─ FactionMembership
 ├─ EventLog / слухи
 └─ Quest/Dialogue systems
```

Системы ECS тикают needs и работу.  
Соц. система на обычном C# читает `NpcId` и обновляет граф отношений.  
UI показывает только focus-слой (рядом с игроком / важные персоны).

### Simulation LOD (важнее «магии DOTS»)

| Слой | Глубина |
|---|---|
| Рядом с игроком | полная анимация, детальный AI, диалоги |
| Текущий город | needs, jobs, путь, локальные события |
| Дальний регион | агрегаты: «город голодает», без шага каждого NPC |

Без LOD даже DOTS не спасёт Guild-like мир «на всю страну в полном AI».

---

## 13. Мини-дизайн данных для NPC (пример)

```csharp
public struct NpcId : IComponentData { public int Value; }
public struct Needs : IComponentData { public float Hunger, Energy, Mood; }
public struct Profession : IComponentData { public int ProfessionId; }
public struct Wallet : IComponentData { public int Coins; }
public struct HomeBuilding : IComponentData { public Entity Building; }

public struct WantEatTag : IComponentData {}
public struct WantSleepTag : IComponentData {}
public struct AtWorkTag : IComponentData {}
```

Системы:

1. `NeedsDecaySystem` — меняет числа  
2. `NeedDecisionSystem` — если Hunger высокий → добавить `WantEatTag`  
3. `EatSystem` — у кого WantEat и есть еда → уменьшить Hunger, снять тег  
4. `WorkSystem` — если AtWork → производить товар / давать зарплату  
5. Presentation system — обновить видимые модели только для focus NPC  

Социалка отдельно:

```csharp
// Не обязательно IComponentData
Dictionary<(int a, int b), Opinion> opinions;

struct Opinion
{
    public float Trust;
    public float Friendship;
    public float Fear;
}
```

---

## 14. Онлайн + DOTS

Есть отдельный путь: **Netcode for Entities**.

Но для Guild-like онлайна ключевое не «ECS по сети», а:

- сервер (или host) симулирует мир  
- клиент получает срез (город/радиус интереса)  
- не реплицировать каждого NPC со всей социалкой всем игрокам  

Input System читает локального игрока.  
Симуляция NPC — серверная.  
Клиент видит результат.

Если онлайн не ближайшая цель — учи ECS/DOTS оффлайн на городе, сеть добавишь позже.

---

## 15. Отладка и мышление

В обычном Unity думаешь: «Что делает этот объект?»  
В ECS думаешь: «Какие системы обрабатывают этот набор компонентов?»

Полезные вопросы при баге:

1. Какие компоненты на entity?  
2. Какая система должна их менять?  
3. Включена ли система?  
4. Не в другом ли архетипе entity после Add/Remove?  
5. Не отложены ли изменения в CommandBuffer?

Инструменты: Entities Hierarchical Window / Systems window / Profiler (в актуальной версии пакета Entities).

---

## 16. С чего начать учиться (практический порядок)

1. Обычный Unity: объекты, сцены, простой AI  
2. Понять проблему: 5–10k объектов на MonoBehaviour  
3. Поставить пакеты Entities (и по гайду Unity — зависимости)  
4. Сделать один `IComponentData` + одну `ISystem` (движение)  
5. Добавить needs-систему  
6. Попробовать `IJobEntity` + `[BurstCompile]`  
7. Только потом: baking/subscenes, physics, graphics, netcode  

Не начинай обучение Unity сразу с DOTS.

---

## 17. Частые ошибки новичков

1. Класть логику в компоненты «как в MonoBehaviour»  
2. Делать один гигантский компонент `NpcAllData`  
3. Постоянно Add/Remove компоненты каждый кадр без нужды  
4. Ожидать, что UI/диалоги «сами красиво лягут» на ECS  
5. Переписывать всё подряд, не измерив профилировщиком  
6. Путать Entity с GameObject 1:1 навсегда (иногда да, иногда нет)  
7. Забыть: Burst любит `unmanaged` типы, не любые C# классы  

---

## 18. Шпаргалка терминов

| Термин | Значение |
|---|---|
| Entity | ID сущности |
| Component | Данные на сущности |
| System | Логика над выборкой сущностей |
| Archetype | Набор типов компонентов |
| Chunk | Блок памяти с entity одного архетипа |
| Query | Запрос «все с такими компонентами» |
| Baking | Конвертация GameObject → Entity |
| Job | Задача, часто параллельная |
| Burst | Ускоряющий компилятор для jobs/систем |
| Structural change | Создание/удаление entity или компонентов |
| ECB | EntityCommandBuffer — отложенные структурные команды |
| Hybrid | Смесь Entities + GameObjects |

---

## 19. Сопоставление с вашими темами обучения

| Тема | Связь с DOTS |
|---|---|
| Input System | Читает локального игрока; в DOTS input часто пишут в component, системы читают |
| Два куба по сети | DOTS не нужен |
| Онлайн мультиплеер | Сначала ownership/репликация; DOTS опционален |
| Guild-like + много NPC | DOTS полезен для массы и needs/jobs; социалка гибридом |
| Локальный кооп | К DOTS почти не относится |

---

## 20. Итог

- **ECS** — способ строить геймплей из данных и систем.  
- **DOTS** — Unity-стек, чтобы делать это быстро и масштабируемо.  
- Сила проявляется на **массовых однотипных расчётах**.  
- Для социальной симуляции DOTS — мощный двигатель фона, но не единственный язык всей игры.  
- Лучшая стратегия: **гибрид + simulation LOD**.  

Если кратко для памяти:

> GameObject думает объектами.  
> ECS думает данными и конвейерами систем.  
> DOTS делает этот конвейер быстрым.

---

## 21. Полезные ссылки

- [Unity Entities package docs](https://docs.unity3d.com/Packages/com.unity.entities@latest)  
- [Entities introduction](https://docs.unity3d.com/Packages/com.unity.entities@latest/manual/index.html)  
- [Job System](https://docs.unity3d.com/Manual/JobSystem.html)  
- [Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest)  
- [Netcode for Entities](https://docs.unity3d.com/Packages/com.unity.netcode@latest)  

API Entities менялся между версиями (`SystemBase` → `ISystem`, старый Convert → Baking).  
Ориентируйся на docs пакета той версии, которую ставишь в проект.

---

*Файл для обучения НовгородGDD / UnityОбучение. Можно дополнять своими схемами города, needs и социалки.*
