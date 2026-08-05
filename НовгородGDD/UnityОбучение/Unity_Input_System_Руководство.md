# Unity Input System — полное практическое руководство

Руководство по пакету **Input System** (`com.unity.inputsystem`) для Unity 6.  
Цель: понять, **какой способ выбрать**, чем отличаются «корневые» Actions от нескольких `.inputactions`, и как читать ввод через polling / events / PlayerInput.

---

## 1. Две системы ввода в Unity (кратко)

| | **Input Manager** (legacy) | **Input System** (пакет) |
|---|---|---|
| API | `UnityEngine.Input` (`Input.GetKey` и т.д.) | `UnityEngine.InputSystem` |
| Где настраивается | *Project Settings → Input Manager* | *Project Settings → Input System Package* + ассеты `.inputactions` |
| Статус | Устаревший | **Рекомендуется для новых проектов** |

`if (Input.GetKey(...))` — это **не отдельная система**, а legacy API старого Input Manager.

В Unity 6 пакет Input System обычно уже есть. Если нет: **Window → Package Manager → Input System**.

После установки проверь:

**Edit → Project Settings → Player → Active Input Handling**

- **Input System Package** — только новый
- **Both** — и старый `Input.`, и новый (удобно при миграции)
- **Input Manager (Old)** — только legacy

---

## 2. Главные понятия (словарь)

### Action (действие)

«Что игрок хочет сделать», а не «какая клавиша»: `Jump`, `Move`, `Attack`, `Pause`.

### Binding (привязка)

Конкретная кнопка/ось устройства, которая вызывает Action:  
`Space`, `Gamepad South (A)`, `W/A/S/D`, стик и т.д.

Одно действие может иметь **много** привязок (клавиатура + геймпад + тач).

### Action Map (карта действий)

Группа Actions для контекста:

- `Player` — ходьба, прыжок, атака  
- `UI` — навигация меню  
- `Vehicle` — управление машиной  
- `Dialogue` — пропуск реплики  

Обычно **включаешь одну карту** (или нужный набор), остальные выключаешь.

### Control Scheme

Набор устройств под сценарий: `Keyboard&Mouse`, `Gamepad`, `Touch`.  
Нужен для переключения схем и локального мультиплеера.

### Input Action Asset (`.inputactions`)

Файл-ассет со всеми Maps / Actions / Bindings / Schemes.  
Редактируется в **Actions Editor**.

### Project-wide Actions («в корне» / в Project Settings)

Это **тот же тип данных** (Action Asset), но назначенный как **проектный по умолчанию**.  
Доступен через:

```csharp
InputSystem.actions
```

Именно его Unity показывает в:

**Edit → Project Settings → Input System Package → Input Actions**

---

## 3. Project-wide Actions vs несколько `.inputactions` — в чём разница?

Это самый частый источник путаницы.

### Вариант A — Project-wide (рекомендуется для большинства игр)

```
Project Settings → Input System Package → Input Actions
         ↓
один «главный» Action Asset на весь проект
         ↓
код: InputSystem.actions.FindAction("Jump")
```

**Плюсы**

- Один источник правды для всей игры  
- Быстрый старт (есть дефолтные `Move`, `Jump`, …)  
- Меньше ассетов и путаницы  

**Когда использовать**

- Один игрок  
- Обычная игра / учебный проект  
- Хочешь «просто заработало»  

### Вариант B — отдельные `.inputactions` файлы в Project

Создание: **ПКМ в Project → Create → Input Actions**

```
Assets/
  Input/
    PlayerControls.inputactions
    MenuControls.inputactions   // опционально
```

**Плюсы**

- Разделение по системам/модулям (персонаж, меню, транспорт)  
- Можно генерировать C# класс на каждый ассет  
- Удобно в больших проектах / командах  

**Когда использовать**

- Большая игра с разными режимами ввода  
- Несколько независимых систем (player / UI / vehicles)  
- Нужен `Generate C# Class`  

### Вариант C — и то, и другое

Технически можно. Но **не смешивай без причины**.

Правило:

> Для одной игры обычно достаточно **одного** Action Asset.  
> Либо он project-wide, либо лежит как `.inputactions` в Project и на него ссылаются скрипты / `PlayerInput`.

### Что НЕ является «корнем»

- **Action Map** (`Player`, `UI`) — это папки **внутри** одного ассета, не отдельные системы.  
- **Несколько карт внутри одного `.inputactions`** — норма и рекомендуется.  
- **Несколько `.inputactions`** — уже отдельные ассеты (второй уровень организации).

```
Один .inputactions (или project-wide)
├── Action Map: Player
│   ├── Move
│   ├── Jump
│   └── Attack
├── Action Map: UI
│   ├── Navigate
│   └── Submit
└── Action Map: Vehicle
    ├── Steer
    └── Brake
```

Переключение карт:

```csharp
playerMap.Disable();
uiMap.Enable();
```

или через `PlayerInput.SwitchCurrentActionMap("UI")`.

---

## 4. Все основные способы работы (выбор)

Unity официально выделяет 3 больших workflow + несколько подвариантов чтения.

| № | Способ | Сложность | Когда брать |
|---|---|---|---|
| 1 | **Polling Actions** (`ReadValue` в `Update`) | ★☆☆ | Движение, камера, удержание кнопки — **рекомендуется чаще всего** |
| 2 | **Callbacks на Action** (`performed += ...`) | ★★☆ | Разовые события: прыжок, выстрел, пауза |
| 3 | **PlayerInput + Unity Events** | ★★☆ | Без кода привязки: в Inspector указываешь метод объекта |
| 4 | **PlayerInput + Send/Broadcast Message** | ★★☆ | Методы `OnJump`, `OnMove` по имени |
| 5 | **PlayerInput + C# Events** | ★★★ | То же, но подписка из кода |
| 6 | **Generate C# Class** | ★★☆ | Типобезопасный код без строк `"Jump"` |
| 7 | **Прямое чтение устройств** | ★☆☆ | Быстрый прототип (`Keyboard.current`) |
| 8 | **Создание Actions в коде** | ★★★ | Динамический/временный ввод |

Ниже — каждый с примером.

---

## 5. Способ 1 — Polling (рекомендуемый базовый)

Идея: в `Update` **спрашиваешь** текущее состояние действия.

Подходит для: движение, look, sprint (удержание), aim.

### Пример (project-wide Actions)

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerMovePolling : MonoBehaviour
{
    InputAction moveAction;
    InputAction jumpAction;
    InputAction sprintAction;

    [SerializeField] float speed = 5f;
    [SerializeField] float sprintMultiplier = 1.6f;

    void Start()
    {
        // FindAction — по имени Action (не клавиши!)
        moveAction = InputSystem.actions.FindAction("Move");
        jumpAction = InputSystem.actions.FindAction("Jump");
        sprintAction = InputSystem.actions.FindAction("Sprint");

        // Важно: не вызывай FindAction каждый кадр
    }

    void Update()
    {
        Vector2 move = moveAction.ReadValue<Vector2>();
        float sprint = sprintAction.IsPressed() ? sprintMultiplier : 1f;

        transform.Translate(new Vector3(move.x, 0f, move.y) * speed * sprint * Time.deltaTime);

        if (jumpAction.WasPressedThisFrame())
        {
            Debug.Log("Jump!");
        }
    }
}
```

### Полезные методы polling

| Метод | Смысл |
|---|---|
| `ReadValue<T>()` | Текущее значение (`float`, `Vector2`, …) |
| `IsPressed()` | Кнопка сейчас зажата |
| `WasPressedThisFrame()` | Нажата **в этом кадре** |
| `WasReleasedThisFrame()` | Отпущена в этом кадре |
| `WasPerformedThisFrame()` | Interaction дошла до `Performed` (важно для Hold и т.п.) |
| `WasCompletedThisFrame()` | Interaction закончилась |

**Совет:** для непрерывного движения — polling.  
Для «выстрелил один раз» — `WasPressedThisFrame()` или event `performed`.

---

## 6. Способ 2 — Callbacks на Action (events из кода)

Идея: система **сама вызывает** твой метод, когда действие сработало.

Фазы Action:

1. `started` — ввод начался  
2. `performed` — действие выполнено (главное для кнопок)  
3. `canceled` — отмена / отпускание  

### Пример

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerJumpCallback : MonoBehaviour
{
    InputAction jumpAction;

    void OnEnable()
    {
        jumpAction = InputSystem.actions.FindAction("Jump");
        jumpAction.performed += OnJump;
        jumpAction.canceled += OnJumpCanceled;

        // Если action не включён глобально — включи:
        // jumpAction.Enable();
    }

    void OnDisable()
    {
        jumpAction.performed -= OnJump;
        jumpAction.canceled -= OnJumpCanceled;
    }

    void OnJump(InputAction.CallbackContext ctx)
    {
        // ctx валиден ТОЛЬКО внутри колбэка — не сохраняй его в поле!
        Debug.Log("Jump performed");
    }

    void OnJumpCanceled(InputAction.CallbackContext ctx)
    {
        Debug.Log("Jump canceled / released");
    }
}
```

### Чтение значения в колбэке

```csharp
void OnMove(InputAction.CallbackContext ctx)
{
    Vector2 v = ctx.ReadValue<Vector2>();
}
```

### Подписка на всю карту

```csharp
InputActionMap map = InputSystem.actions.FindActionMap("Player");
map.actionTriggered += ctx =>
{
    Debug.Log($"{ctx.action.name} → {ctx.phase}");
};
```

**Совет:** events удобны для редких действий; для стика/движения polling обычно проще и предсказуемее.

---

## 7. Способ 3 — PlayerInput + Unity Events (то, про что ты говорил)

Идея: на объекте висит компонент **Player Input**.  
В Inspector у каждого Action указываешь: **какой объект → какой метод**.

Это и есть «скрипт вызывает указанную функцию у объекта».

### Настройка

1. На игрока добавь компонент **Player Input**  
2. **Actions** → project-wide или свой `.inputactions`  
3. **Default Map** → например `Player`  
4. **Behavior** → **Invoke Unity Events**  
5. Раскрой **Events → Player → Jump / Move / …**  
6. Нажми `+` и перетащи объект со скриптом, выбери метод  

### Методы в скрипте

Сигнатура для Unity Events у PlayerInput:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerViaUnityEvents : MonoBehaviour
{
    Vector2 move;

    // Привязывается в Inspector к Action "Move"
    public void OnMove(InputAction.CallbackContext ctx)
    {
        if (ctx.performed || ctx.started)
            move = ctx.ReadValue<Vector2>();
        if (ctx.canceled)
            move = Vector2.zero;
    }

    // Привязывается к Action "Jump"
    public void OnJump(InputAction.CallbackContext ctx)
    {
        if (!ctx.performed) return;
        Debug.Log("Jump from Unity Event");
    }

    void Update()
    {
        transform.Translate(new Vector3(move.x, 0f, move.y) * 5f * Time.deltaTime);
    }
}
```

**Плюсы**

- Видно связи в Inspector  
- Дизайнер/геймдизайнер может перекинуть вызов без правки кода  

**Минусы**

- Легко «сломать» ссылки при рефакторинге  
- Для движения всё равно часто хранят значение и применяют в `Update`  

---

## 8. Способ 4 — PlayerInput + Send Messages / Broadcast Messages

**Behavior**:

- **Send Messages** — ищет методы на **том же** GameObject  
- **Broadcast Messages** — на объекте **и детях**  

Имена методов строятся так: Action `Jump` → метод `OnJump`.

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerViaMessages : MonoBehaviour
{
    // Action "Jump" → OnJump
    public void OnJump()
    {
        Debug.Log("Jump");
    }

    // Если нужно значение — параметр InputValue
    public void OnMove(InputValue value)
    {
        Vector2 v = value.Get<Vector2>();
        // InputValue валиден только во время этого вызова!
    }
}
```

Быстро, но хрупко (строковые имена, нет проверки компилятором).

---

## 9. Способ 5 — PlayerInput + C# Events

**Behavior → Invoke CSharp Events**

В Inspector ничего не вешаешь — подписываешься в коде:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerViaCsharpEvents : MonoBehaviour
{
    PlayerInput playerInput;

    void OnEnable()
    {
        playerInput = GetComponent<PlayerInput>();
        playerInput.onActionTriggered += OnAction;
    }

    void OnDisable()
    {
        playerInput.onActionTriggered -= OnAction;
    }

    void OnAction(InputAction.CallbackContext ctx)
    {
        if (ctx.action.name == "Jump" && ctx.performed)
            Debug.Log("Jump");
    }
}
```

Есть также события потери устройства: `onDeviceLost`, `onDeviceRegained`.

---

## 10. Способ 6 — Generate C# Class (типобезопасно)

На `.inputactions` в Inspector:

1. ✅ **Generate C# Class**  
2. Apply  

Unity создаёт класс вроде `PlayerControls` / `InputSystem_Actions`.

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerGenerated : MonoBehaviour
{
    InputSystem_Actions actions; // имя зависит от ассета
    // или: PlayerControls actions;

    void OnEnable()
    {
        actions = new InputSystem_Actions();
        actions.Player.Enable();

        actions.Player.Jump.performed += OnJump;
    }

    void OnDisable()
    {
        actions.Player.Jump.performed -= OnJump;
        actions.Player.Disable();
        actions.Dispose();
    }

    void Update()
    {
        Vector2 move = actions.Player.Move.ReadValue<Vector2>();
        // ...
    }

    void OnJump(InputAction.CallbackContext ctx)
    {
        Debug.Log("Jump");
    }
}
```

Ещё удобнее — интерфейсы (`IPlayerActions`) + `SetCallbacks(this)`:

```csharp
public class PlayerWithInterface : MonoBehaviour, InputSystem_Actions.IPlayerActions
{
    InputSystem_Actions actions;

    void OnEnable()
    {
        actions = new InputSystem_Actions();
        actions.Player.AddCallbacks(this); // или SetCallbacks
        actions.Player.Enable();
    }

    void OnDisable()
    {
        actions.Player.RemoveCallbacks(this);
        actions.Player.Disable();
        actions.Dispose();
    }

    public void OnMove(InputAction.CallbackContext context) { /* ... */ }
    public void OnJump(InputAction.CallbackContext context) { /* ... */ }
    public void OnLook(InputAction.CallbackContext context) { /* ... */ }
    public void OnAttack(InputAction.CallbackContext context) { /* ... */ }
    // остальные методы интерфейса...
}
```

**Плюс:** нет магических строк `"Jump"`, рефакторинг безопаснее.

---

## 11. Способ 7 — Прямое чтение устройств (без Actions)

Для прототипа:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class DirectDeviceInput : MonoBehaviour
{
    void Update()
    {
        Keyboard kb = Keyboard.current;
        Gamepad pad = Gamepad.current;
        Mouse mouse = Mouse.current;

        if (kb != null && kb.spaceKey.wasPressedThisFrame)
            Debug.Log("Space");

        Vector2 stick = pad != null ? pad.leftStick.ReadValue() : Vector2.zero;
        Vector2 wasd = Vector2.zero;

        if (kb != null)
        {
            if (kb.wKey.isPressed) wasd.y += 1;
            if (kb.sKey.isPressed) wasd.y -= 1;
            if (kb.aKey.isPressed) wasd.x -= 1;
            if (kb.dKey.isPressed) wasd.x += 1;
        }

        Vector2 move = stick.sqrMagnitude > 0.01f ? stick : wasd;
        transform.Translate(new Vector3(move.x, 0f, move.y) * 5f * Time.deltaTime);

        if (mouse != null)
        {
            Vector2 delta = mouse.delta.ReadValue();
            // look / aim
        }
    }
}
```

**Минус:** нет перепривязки, схем, удобного мультиплеера. Для прод-кода лучше Actions.

---

## 12. Способ 8 — Actions полностью из кода

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class RuntimeActions : MonoBehaviour
{
    InputAction move;
    InputAction jump;

    void OnEnable()
    {
        move = new InputAction("Move", InputActionType.Value);
        move.AddCompositeBinding("2DVector")
            .With("Up", "<Keyboard>/w")
            .With("Down", "<Keyboard>/s")
            .With("Left", "<Keyboard>/a")
            .With("Right", "<Keyboard>/d");
        move.AddBinding("<Gamepad>/leftStick");

        jump = new InputAction("Jump", InputActionType.Button);
        jump.AddBinding("<Keyboard>/space");
        jump.AddBinding("<Gamepad>/buttonSouth");

        move.Enable();
        jump.Enable();
        jump.performed += _ => Debug.Log("Jump");
    }

    void OnDisable()
    {
        move.Disable();
        jump.Disable();
        move.Dispose();
        jump.Dispose();
    }

    void Update()
    {
        Vector2 v = move.ReadValue<Vector2>();
    }
}
```

Гибко, но настройки не видны дизайнеру в Editor — обычно для спецслучаев.

---

## 13. Типы Action: Value / Button / PassThrough

В редакторе у Action есть **Action Type**:

| Тип | Для чего |
|---|---|
| **Value** | Непрерывные значения: стик, мышь delta, триггер |
| **Button** | Нажатия: прыжок, атака, пауза |
| **PassThrough** | Пропускать ввод со **всех** привязок без «выбора победителя» (редко нужно) |

Правило большого пальца:

- движение / look → **Value**  
- одноразовые действия → **Button**

---

## 14. Interactions и Processors (мощь системы)

### Interactions (когда action считается выполненным)

Примеры:

- **Press** — нажатие  
- **Hold** — удержание N секунд (взаимодействие с объектом)  
- **Tap** — быстрый тап  
- **SlowTap**  

Пример: `Interact` с Hold 0.4s → в коде `WasPerformedThisFrame()` сработает только после удержания.

### Processors (как обработать значение)

- **Normalize**  
- **Scale**  
- **Invert**  
- **Deadzone** (для стиков)  

Настраиваются на Binding в Actions Editor — часто лучше, чем править в коде.

---

## 15. Action Maps на практике (меню / игра)

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class InputContextSwitcher : MonoBehaviour
{
    InputActionMap playerMap;
    InputActionMap uiMap;

    void Start()
    {
        playerMap = InputSystem.actions.FindActionMap("Player");
        uiMap = InputSystem.actions.FindActionMap("UI");
        EnterGameplay();
    }

    public void EnterGameplay()
    {
        uiMap.Disable();
        playerMap.Enable();
    }

    public void EnterUI()
    {
        playerMap.Disable();
        uiMap.Enable();
        // Курсор, пауза и т.д.
    }
}
```

С `PlayerInput`:

```csharp
GetComponent<PlayerInput>().SwitchCurrentActionMap("UI");
```

**Совет:** не держи `Player` и `UI` одновременно активными без нужды — получишь двойной ввод (и ходить, и листать меню).

---

## 16. Локальный мультиплеер

Компоненты:

1. **Player Input** — на префабе игрока  
2. **Player Input Manager** — спавнит игроков и раздаёт устройства  

Каждый `PlayerInput` получает **свою копию** Actions и своё устройство (геймпад 1 / геймпад 2).

Важно:

> При `PlayerInput` **не читай** `InputSystem.actions` напрямую для игрока.  
> Бери actions через этот экземпляр `PlayerInput` (у него своя отфильтрованная копия).

```csharp
PlayerInput pi = GetComponent<PlayerInput>();
InputAction jump = pi.actions["Jump"];
```

---

## 17. UI Toolkit / uGUI и Input System

Для UI обычно нужен:

- **Input System UI Input Module** (вместо старого Standalone Input Module)

Его можно связать с тем же Action Asset / с `PlayerInput` (поле UI Input Module), чтобы один игрок управлял и персонажем, и своим UI.

Карта `UI` в дефолтных actions как раз для этого (`Navigate`, `Submit`, `Cancel`, `Point`, `Click`, …).

---

## 18. Что выбрать? Практическая шпаргалка

### Учебный проект / соло-игра

1. Project-wide Actions  
2. Polling в `Update` для Move/Look  
3. `WasPressedThisFrame` или `performed` для Jump/Attack  

### Хочешь кликать методы в Inspector

→ **PlayerInput + Invoke Unity Events**

### Большой проект / строгая архитектура

→ отдельный `.inputactions` + **Generate C# Class**  
→ polling + точечные callbacks  

### Прототип за 5 минут

→ `Keyboard.current` / `Gamepad.current`  
→ потом перенеси на Actions  

### Локальный кооп

→ **PlayerInput + PlayerInput Manager**

---

## 19. Частые ошибки

1. **`FindAction` каждый кадр** — медленно; кэшируй в `Start`/`OnEnable`.  
2. **Сохранил `CallbackContext` / `InputValue` в поле** — после колбэка они невалидны. Читай значение сразу.  
3. **Action Map не Enable** — «ничего не работает».  
4. **Active Input Handling = Old** — новый API молчит.  
5. **Смешал `Input.` и Input System** без режима Both — путаница.  
6. **Несколько project-wide логик + несколько ассетов** без схемы — дубли ввода.  
7. **С PlayerInput читаешь `InputSystem.actions`** в мультиплеере — ломается раздача устройств.  
8. **Ожидаешь `GetKeyDown` поведение**, но Action Type/Interaction другие — смотри фазы в Input Debugger.

---

## 20. Отладка

1. **Window → Analysis → Input Debugger**  
   - какие устройства видны  
   - какие Actions enabled  
   - что реально нажимается  

2. В Play Mode у `PlayerInput` есть Debug-секция: user id, scheme, devices.

3. Временный лог:

```csharp
InputSystem.onActionChange += (obj, change) =>
{
    if (obj is InputAction action)
        Debug.Log($"{action.name}: {change}");
};
```

---

## 21. Минимальный «правильный» старт (чеклист)

1. Пакет Input System установлен  
2. Active Input Handling = **Input System Package** (или Both)  
3. Открыл Project Settings → Input Actions (project-wide)  
4. Есть карта `Player` с `Move` (Value, Vector2) и `Jump` (Button)  
5. Привязки: WASD + left stick для Move; Space + South для Jump  
6. Скрипт polling как в разделе 5  
7. Проверил в Input Debugger, что Actions зелёные/активные  

---

## 22. Сравнение «одной фразой»

| Вопрос | Ответ |
|---|---|
| `Input.GetKey` vs Input System? | Старое API vs новый пакет |
| Polling vs Events? | Спрашиваешь каждый кадр vs тебя вызывают сами |
| Unity Events на PlayerInput? | В Inspector указал метод объекта — система его вызовет |
| Project-wide vs `.inputactions`? | Один и тот же формат; project-wide — «главный» для `InputSystem.actions` |
| Несколько Action Maps? | Контексты внутри одного ассета (Player/UI/…) |
| Несколько `.inputactions`? | Несколько отдельных ассетов (обычно не нужно на старте) |

---

## 23. Рекомендуемая схема для вашего обучения

Для курса/практики в этой папке лучше так:

1. **Один** project-wide Action Asset  
2. Две карты: `Player` и `UI`  
3. Скрипт персонажа — **polling**  
4. Отдельно один урок на **PlayerInput + Unity Events**, чтобы понять Inspector-привязки  
5. Не усложнять Generate C# Class, пока не понадобится  

Так ты поймёшь 90% реальных задач без хаоса из «десяти способов сразу».

---

## 24. Полезные ссылки (документация Unity)

- [Input System Package](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest)  
- [Workflows](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest/manual/Workflows.html)  
- [Player Input](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest/manual/PlayerInput.html)  
- [Responding to Actions](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest/manual/RespondingToActions.html)  
- [Action Assets](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest/manual/ActionAssets.html)  

Официальный видеоряд Unity 6 (Input System): серия из 7 роликов в Manual → Input.

---

*Файл для обучения: Unity Input System. Можно дополнять своими заметками по проекту НовгородGDD.*
