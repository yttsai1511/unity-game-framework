# 事件驅動遊戲架構設計

## 目錄

1. [架構概述](#架構概述)
2. [核心組件](#核心組件)
3. [事件系統詳解](#事件系統詳解)
4. [遊戲狀態流程](#遊戲狀態流程)
5. [系統擴展指南](#系統擴展指南)
6. [設計模式](#設計模式)

## 架構概述

本架構是一個基於事件驅動設計理念的遊戲開發框架，採用分層設計原則，解決遊戲開發過程中系統耦合度高、擴展性差、維護困難等常見問題。通過清晰的責任劃分和鬆散的事件通訊機制，實現系統間低耦合、高內聚的理想結構。

> **設計理念**：本架構設計的核心是「關注點分離」和「鬆耦合依賴」，通過事件驅動模式允許系統獨立演進，同時提供可預測的狀態轉換機制，確保無論專案規模如何增長，架構仍然保持高內聚低耦合的特性。選擇事件驅動模式而非傳統的直接呼叫方式，是為了同時平衡開發效率和系統穩定性，使系統更易於測試、擴展和重構，特別適合多人協作的中大型專案。

### 架構層次

| 層級 | 組件範例 | 職責 |
|------|---------|------|
| 遊戲模式層 | CooperativeMode, SoloMode, VersusMode | 實現特定遊戲規則和玩法機制 |
| 遊戲系統層 | BattleSystem, LoginSystem, RoomSystem | 處理核心遊戲邏輯和功能實現 |
| 管理器層 | GameController, UIManager, SceneLoader | 協調系統間交互，管理共享資源 |
| 核心框架層 | EventBus, SingletonObject | 提供基礎通訊和共享機制 |

> **層級設計考量**：採用分層架構可以明確各組件的職責範圍，上層依賴下層，但下層不感知上層，這確保了系統可以從底層被替換或重構而不影響上層邏輯。管理器層作為框架與遊戲邏輯的橋樑，屏蔽了底層實現細節。在事件驅動設計中，各層級通過事件而非直接引用進行通訊，增強了系統的模組化程度。

### 關鍵特性

- **事件驅動通訊**：通過 EventBus 實現組件間鬆散耦合，避免直接依賴
- **自動系統發現**：使用反射自動初始化遊戲系統，簡化系統註冊流程
- **狀態機流程**：清晰定義狀態轉換與生命週期，統一系統行為模式
- **非同步操作鏈**：統一的資源載入和操作排序機制，簡化異步流程
- **可擴展性設計**：易於添加新系統、新 UI 和新遊戲模式，不需修改既有代碼

### 選擇事件驅動架構的優勢

與其他架構模式相比，事件驅動架構在遊戲開發中具有以下優勢：

1. **解耦系統依賴**：系統間通過事件而非直接引用通訊，降低耦合度
2. **簡化擴展流程**：新功能只需訂閱相關事件，無需修改現有系統
3. **提高可測試性**：可輕易模擬事件，使單元測試更加簡單
4. **支持異步操作**：事件機制天然適合處理異步任務和回呼鏈
5. **更好的並行開發**：團隊成員可專注於各自系統，減少衝突

## 核心組件

### EventBus

> 注意：完整的 EventBus 說明請參考 [EventBus 文件](./EventBus.md)

事件系統核心，實現發布-訂閱模式的事件傳遞機制。框架中通過 `EventBus.Instance` 全域實例存取事件系統：

```csharp
// 訂閱事件
EventBus.Instance.Subscribe<Action<string>>(GameEvents.Scene.LoadCompleted, OnSceneLoadCompleted);

// 發布事件
EventBus.Instance.Publish(GameEvents.UI.Show, MenuName, settings);

// 帶回呼的事件發布
EventBus.Instance.Publish(GameEvents.State.Enter, newState, (Action)(() => {
    // 事件處理完成後執行
}));
```

> **設計考量**：使用單一全域事件總線而非多個域特定事件管理器，是為了簡化系統設計和降低記憶體佔用。全域總線雖然有潛在的事件風暴風險，但通過明確的事件命名和參數型別，以及系統邊界的嚴格定義，可以有效規避這些問題。事件系統的效能調校也考慮了高頻事件處理，在必要時可以實現事件批次處理。

### SingletonObject

單例基類，為管理器類提供單例實現。

```csharp
public abstract class SingletonObject<T> : MonoBehaviour where T : Object
{
    protected static T instance;

    public static T Instance
    {
        get
        {
            if (instance == null)
            {
                instance = FindFirstObjectByType<T>();
            }
            return instance;
        }
    }

    protected virtual void Awake()
    {
        if (instance == null)
        {
            instance = this as T;
        }
        else if (instance != this)
        {
            Destroy(gameObject);
        }
    }
}

// 使用方式
public sealed class UIManager : SingletonObject<UIManager>
{
    // 實現...
}
```

> **設計考量**：採用 MonoBehaviour 單例而非純靜態類，是為了充分利用 Unity 的生命週期和序列化功能，同時保持訪問便捷性。所有管理器都繼承自 SingletonObject，確保它們有一致的初始化和存取方式，簡化系統間的互操作。在 Unity 場景切換時，框架透過特殊處理保證單例的持久性和正確的初始化順序。

### GameController

遊戲主控制器，管理狀態轉換和系統初始化。

```csharp
public sealed partial class GameController : SingletonObject<GameController>
{
    private Dictionary<string, ISystem> gameSystems = new Dictionary<string, ISystem>();
    private string currentState = States.Boot;

    // 狀態定義
    public static class States
    {
        public const string Boot = "Boot";         // 啟動階段，初始化基礎系統
        public const string Login = "Login";       // 登入階段，處理使用者身份驗證
        public const string Lobby = "Lobby";       // 大廳階段，選擇房間或建立房間
        public const string Room = "Room";         // 房間階段，等待開始遊戲
        public const string Gameplay = "Gameplay"; // 遊戲階段，實際遊戲內容
    }

    // 變更遊戲狀態
    public void ChangeState(string newState)
    {
        // 狀態轉換邏輯
    }
}
```

> **設計考量**：GameController 作為遊戲核心控制器，負責系統生命週期和狀態管理，但不直接處理具體業務邏輯。採用自動系統發現機制使系統擴展變得簡單，只需實現相應接口即可被框架自動識別和初始化。狀態機設計使遊戲流程清晰可控，且便於在不同狀態間切換而不丟失資料。GameController 只管理狀態流轉，具體狀態邏輯由各個系統自行處理，實現了關注點分離。

## 事件系統詳解

> 注意：詳細的事件系統實現與使用範例請參考 [EventBus 文件](./EventBus.md)

### 事件設計原則

1. **使用常數定義事件名稱**，避免字串錯誤

```csharp
public static class GameEvents
{
    public static class Scene
    {
        public const string LoadStart = "Scene.LoadStart";
        public const string LoadCompleted = "Scene.LoadCompleted";
    }

    public static class UI
    {
        public const string Show = "UI.Show";
        public const string Hide = "UI.Hide";
        public const string Action = "UI.Action";
        public const string Update = "UI.Update";
    }

    public static class State
    {
        public const string Enter = "State.Enter";
        public const string Exit = "State.Exit";
    }

    public static class Gameplay
    {
        public const string Start = "Gameplay.Start";
        public const string End = "Gameplay.End";
    }
}
```

> **命名空間設計**：事件命名採用「域.行為」格式，清晰標明事件的來源和目的。使用常數而非字串直接量可在編譯期捕獲錯誤，提高系統穩定性。將相關事件分組到對應的靜態類中，不僅便於管理，也形成了自然的文檔結構。這種設計支持以後事件的自然擴展，同時保持命名和使用的一致性。

2. **事件參數型別安全**，利用泛型確保型別正確

```csharp
// 安全的事件訂閱和發布
EventBus.Instance.Subscribe<Action<ISystem.ISettings, Action>>(GameEvents.Gameplay.Start, StartGame);
EventBus.Instance.Publish(GameEvents.Gameplay.Start, settings, callback);
```

### EventBus 核心概念

EventBus 是本框架的通訊核心，實現了發布-訂閱模式，允許組件間的鬆散耦合。

在程式碼中，我們通過 `EventBus.Instance` 存取全域事件總線：

```csharp
// 基本用法
EventBus.Instance.Subscribe<Action<string>>(GameEvents.UI.Action, OnUIAction);
EventBus.Instance.Publish(GameEvents.UI.Action, "ClickButton");
EventBus.Instance.Unsubscribe<Action<string>>(GameEvents.UI.Action, OnUIAction);
```

關於 EventBus 的詳細實現與更多使用範例，請參考：
- [EventBus 文件](./EventBus.md#使用範例)
- [EventBus API 說明](./EventBus.md#文件說明)

## 遊戲狀態流程

### 狀態定義

基本流程：`Boot → Login → Lobby → Room → Gameplay`

> **狀態設計原則**：遊戲狀態劃分基於「單一職責」和「狀態內聚」原則，每個狀態只負責特定的遊戲階段，並擁有明確的進入和退出條件。選擇這五個核心狀態是經過多輪迭代和實踐驗證的結果，既能滿足多數網絡遊戲需求，又保持了系統的簡潔性。狀態可視為遊戲的「大場景」，而每個狀態內部可以有自己的子流程和子狀態。

### 狀態轉換機制

```csharp
// 在 GameController 中
public void ChangeState(string newState)
{
    if (currentState == newState)
    {
        LogError($"狀態變更失敗: {newState} (已處於此狀態)");
        return;
    }

    string oldState = currentState;
    LogMessage($"正在從 {oldState} 切換至 {newState}");

    // 發布狀態退出事件
    EventBus.Instance.Publish(GameEvents.State.Exit, oldState, (Action)(() =>
    {
        LogMessage($"已退出狀態: {oldState}");

        // 發布狀態進入事件
        EventBus.Instance.Publish(GameEvents.State.Enter, newState, (Action)(() =>
        {
            currentState = newState;
            LogMessage($"已進入狀態: {newState}");
        }));
    }));
}
```

### 各狀態職責

1. **Boot 狀態**：
   - 初始化基礎系統和管理器
   - 載入使用者設定和資源
   - 準備登入流程

2. **Login 狀態**：
   - 處理使用者身份驗證
   - 獲取基本遊戲資料
   - 準備進入大廳

3. **Lobby 狀態**：
   - 顯示可用房間列表
   - 提供建立房間功能
   - 管理好友和社交功能

4. **Room 狀態**：
   - 房間配置和準備
   - 顯示其他玩家資訊
   - 提供開始遊戲功能

5. **Gameplay 狀態**：
   - 實際遊戲內容
   - 遊戲進行中的邏輯
   - 處理遊戲結束回到房間

### 狀態響應系統

每個遊戲流程系統都可以響應狀態變化。以下是登入系統範例：

```csharp
public sealed partial class LoginSystem : IGameFlow
{
    public ISystem.ISettings currentSettings;

    public void Initialize()
    {
        currentSettings = new FlowSettings();

        // 訂閱狀態事件
        EventBus.Instance.Subscribe<Action<string, Action>>(GameEvents.State.Enter, OnStateEnter);
        EventBus.Instance.Subscribe<Action<string, Action>>(GameEvents.State.Exit, OnStateExit);
    }

    public void Dispose()
    {
        // 取消訂閱所有事件
        EventBus.Instance.Unsubscribe<Action<string, Action>>(GameEvents.State.Enter, OnStateEnter);
        EventBus.Instance.Unsubscribe<Action<string, Action>>(GameEvents.State.Exit, OnStateExit);
    }

    public void OnStateEnter(string state, Action callback)
    {
        // 只處理進入 Login 狀態的事件
        if (state != GameController.States.Login)
        {
            return;
        }

        // 訂閱 UI 事件
        EventBus.Instance.Subscribe<Action<string, ISystem.ISettings>>(GameEvents.UI.Action, OnUIAction);

        // 處理 Login 狀態進入邏輯
        LoadSettings(() =>
        {
            StartUI(() =>
            {
                callback?.Invoke(); // 完成後呼叫回呼
            });
        });
    }

    public void OnStateExit(string state, Action callback)
    {
        // 只處理退出 Login 狀態的事件
        if (state != GameController.States.Login)
        {
            return;
        }

        // 取消訂閱 UI 事件
        EventBus.Instance.Unsubscribe<Action<string, ISystem.ISettings>>(GameEvents.UI.Action, OnUIAction);

        // 處理 Login 狀態退出邏輯
        EventBus.Instance.Publish(GameEvents.UI.Hide, LoginMenu.MenuName, (Action)(() =>
        {
            callback?.Invoke();
        }));
    }

    // 處理 UI 操作事件
    private void OnUIAction(string actionType, ISystem.ISettings settings)
    {
        switch (actionType)
        {
            case LoginMenu.Actions.ClickLogin:
                // 處理登入按鈕點擊
                ProcessLogin(settings);
                break;

            case LoginMenu.Actions.ClickRegister:
                // 處理註冊按鈕點擊
                ShowRegisterForm();
                break;
        }
    }
}
```

### 系統-UI-玩家通訊流程

通訊流程遵循以下模式：

1. **使用者操作 → UI → 系統**
   - UI 擷取使用者輸入
   - UI 發布 `GameEvents.UI.Action` 事件
   - 系統訂閱並處理該事件

2. **系統 → UI → 顯示**
   - 系統處理完業務邏輯
   - 系統發布 `GameEvents.UI.Update` 事件
   - UI 訂閱並更新顯示內容

3. **系統 → 玩家**
   - 系統發布 `GameEvents.Player.Update` 事件
   - 玩家訂閱並執行相應動作

4. **玩家 → 系統**
   - 玩家發布 `GameEvents.Player.Action` 等事件
   - 系統訂閱並處理玩家行為

### UI 事件使用指南

#### GameEvents.UI.Action 使用場景

UI.Action 事件用於從 UI 向系統發送**使用者操作**訊息：

```csharp
// 定義操作類型常數
public static class LoginMenu
{
    public static class Actions
    {
        public const string ClickLogin = "ClickLogin";
        public const string ClickRegister = "ClickRegister";
    }
}

// 在 UI 中發布 Action 事件
private void OnClickLogin()
{
    // 按鈕點擊事件 - 使用者操作
    EventBus.Instance.Publish(GameEvents.UI.Action, LoginMenu.Actions.ClickLogin, new LoginSystem.Settings());
}

// 在系統中訂閱 Action 事件
EventBus.Instance.Subscribe<Action<string, ISystem.ISettings>>(GameEvents.UI.Action, OnUIAction);

// 系統處理 UI 事件
private void OnUIAction(string actionType, ISystem.ISettings data)
{
    switch (actionType)
    {
        case LoginMenu.Actions.ClickLogin:
            // 處理登入邏輯
            break;
        case LoginMenu.Actions.ClickRegister:
            // 處理註冊邏輯
            break;
    }
}
```

#### GameEvents.UI.Update 使用場景

UI.Update 事件用於從系統向 UI 發送**狀態更新**訊息：

```csharp
// 定義更新類型常數
public static class LobbyMenu
{
    public static class Updates
    {
        public const string RoomList = "RoomList";
        public const string PlayerInfo = "PlayerInfo";
        public const string ErrorMessage = "ErrorMessage";
    }
}

// 在系統中發布 Update 事件
private void UpdateRoomList(List<RoomData> rooms)
{
    // 更新 UI 顯示
    EventBus.Instance.Publish(GameEvents.UI.Update, LobbyMenu.Updates.RoomList, rooms);
}

// 在 UI 中訂閱 Update 事件
EventBus.Instance.Subscribe<Action<string, object>>(GameEvents.UI.Update, OnUIUpdate);

// UI 處理更新事件
private void OnUIUpdate(string updateType, object data)
{
    switch (updateType)
    {
        case LobbyMenu.Updates.RoomList:
            // 更新房間列表
            RefreshRoomList((List<RoomData>)data);
            break;

        case LobbyMenu.Updates.PlayerInfo:
            // 更新玩家訊息顯示
            UpdatePlayerDisplay((PlayerData)data);
            break;

        case LobbyMenu.Updates.ErrorMessage:
            // 顯示錯誤訊息
            ShowError((string)data);
            break;
    }
}
```

### 系統與 UI 職責分離

#### 職責劃分原則

| UI 層職責 | 系統層職責 |
|----------|-----------|
| 視覺呈現 | 業務邏輯 |
| 使用者輸入 | 資料處理 |
| 動畫與效果 | 狀態管理 |
| 佈局與排版 | 網路通訊 |

#### 常見反模式及修正

```csharp
// 反模式：UI 包含業務邏輯
public class BadLoginMenu : MonoBehaviour
{
    private void OnLoginButtonClick()
    {
        // 錯誤：UI 直接處理登入邏輯
        string username = usernameInput.text;
        string password = passwordInput.text;

        // UI 不應直接連接網路
        NetworkManager.Connect(username, password);

        // UI 不應直接管理遊戲狀態
        GameController.Instance.ChangeState("Lobby");
    }
}

// 正確模式：UI 只負責發布事件
public class GoodLoginMenu : MonoBehaviour, IGameMenu
{
    private void OnLoginButtonClick()
    {
        // 正確：UI 只捕獲輸入並發布事件
        string username = usernameInput.text;
        string password = passwordInput.text;

        var loginData = new LoginSystem.LoginSettings
        {
            Username = username,
            Password = password
        };

        // 發布 UI 事件，由系統處理
        EventBus.Instance.Publish(GameEvents.UI.Action, LoginMenu.Actions.LoginRequest, loginData);
    }
}
```

## 系統擴展指南

> **擴展設計原則**：系統擴展遵循「開放封閉原則」—對擴展開放，對修改封閉。新功能應通過添加新系統或擴展現有系統來實現，而非修改核心框架。每個新系統都應有單一的職責，並與其他系統通過事件通訊，保持鬆散耦合。

### 建立新的遊戲系統

以下是建立新系統的基本步驟：

```csharp
// 1. 建立新的遊戲系統類
public sealed partial class RoomSystem : IGameFlow
{
    public ISystem.ISettings currentSettings;

    // 2. 實現初始化方法
    public void Initialize()
    {
        currentSettings = new RoomSettings();

        // 訂閱狀態事件
        EventBus.Instance.Subscribe<Action<string, Action>>(GameEvents.State.Enter, OnStateEnter);
        EventBus.Instance.Subscribe<Action<string, Action>>(GameEvents.State.Exit, OnStateExit);
    }

    // 3. 實現清理方法
    public void Dispose()
    {
        // 取消訂閱事件
        EventBus.Instance.Unsubscribe<Action<string, Action>>(GameEvents.State.Enter, OnStateEnter);
        EventBus.Instance.Unsubscribe<Action<string, Action>>(GameEvents.State.Exit, OnStateExit);
    }

    // 4. 實現狀態響應方法
    public void OnStateEnter(string state, Action callback)
    {
        // 只響應進入 Room 狀態
        if (state != GameController.States.Room)
        {
            return;
        }

        // 處理進入房間邏輯
        LoadRoomData(() => {
            SetupRoom(() => {
                callback?.Invoke();
            });
        });
    }

    public void OnStateExit(string state, Action callback)
    {
        // 只響應退出 Room 狀態
        if (state != GameController.States.Room)
        {
            return;
        }

        // 處理退出房間邏輯
        CleanupRoom(() => {
            callback?.Invoke();
        });
    }
}

// 5. 定義系統設置
public sealed partial class RoomSystem
{
    [Serializable]
    public class RoomSettings : ISystem.ISettings
    {
        public string roomId;
        public string roomName;
        public int maxPlayers;
        // 更多房間設置...
    }
}
```

> **系統品質評估**：一個良好設計的系統應符合以下標準：
> - **關注點分離**：系統只關注自己的職責，不干涉其他系統邏輯
> - **事件使用規範**：只訂閱必要的事件，並在不需要時取消訂閱
> - **狀態處理簡潔**：明確處理狀態的進入和退出，不影響其他狀態
> - **回呼鏈正確性**：所有回呼都被正確呼叫，確保狀態流程順暢
> - **錯誤處理完善**：能夠優雅處理各種異常情況，不中斷遊戲流程

### 建立新的 UI 選單

```csharp
public sealed partial class RoomMenu : MonoBehaviour, IGameMenu
{
    public const string MenuName = nameof(RoomMenu);

    [SerializeField] private Button startGameButton;
    [SerializeField] private Button leaveRoomButton;
    [SerializeField] private Transform playerListContainer;

    // UI 元素引用...

    public string GetName()
    {
        return MenuName;
    }

    public bool IsVisible()
    {
        return gameObject.activeSelf;
    }

    public void Initialize()
    {
        // 註冊按鈕事件
        startGameButton.onClick.AddListener(OnClickStartGame);
        leaveRoomButton.onClick.AddListener(OnClickLeaveRoom);

        // 訂閱 UI 更新事件
        EventBus.Instance.Subscribe<Action<string, object>>(GameEvents.UI.Update, OnUIUpdate);
    }

    public void Dispose()
    {
        // 取消註冊按鈕事件
        startGameButton.onClick.RemoveListener(OnClickStartGame);
        leaveRoomButton.onClick.RemoveListener(OnClickLeaveRoom);

        // 取消訂閱事件
        EventBus.Instance.Unsubscribe<Action<string, object>>(GameEvents.UI.Update, OnUIUpdate);
    }

    public void Show(ISystem.ISettings settings, Action callback)
    {
        var roomSettings = settings as RoomSystem.RoomSettings;
        if (roomSettings != null)
        {
            // 初始化 UI 顯示
            InitRoomDisplay(roomSettings);
        }

        gameObject.SetActive(true);
        callback?.Invoke();
    }

    public void Hide(Action callback)
    {
        gameObject.SetActive(false);
        callback?.Invoke();
    }

    private void OnClickStartGame()
    {
        // 發布開始遊戲事件
        EventBus.Instance.Publish(GameEvents.UI.Action, Actions.StartGame, new RoomSystem.RoomSettings());
    }

    private void OnClickLeaveRoom()
    {
        // 發布離開房間事件
        EventBus.Instance.Publish(GameEvents.UI.Action, Actions.LeaveRoom, null);
    }

    private void OnUIUpdate(string updateType, object data)
    {
        switch (updateType)
        {
            case Updates.PlayerJoined:
                AddPlayerToList((PlayerData)data);
                break;

            case Updates.PlayerLeft:
                RemovePlayerFromList((string)data);
                break;
        }
    }

    // UI 相關方法...

    // 定義操作和更新類型
    public static class Actions
    {
        public const string StartGame = "StartGame";
        public const string LeaveRoom = "LeaveRoom";
    }

    public static class Updates
    {
        public const string PlayerJoined = "PlayerJoined";
        public const string PlayerLeft = "PlayerLeft";
    }
}
```

## 設計模式

本框架採用以下設計模式，確保程式碼的品質和可維護性：

- **發布-訂閱模式**：核心通訊機制，實現組件間的解耦
- **命令模式**：用於封裝操作請求，確保可撤銷與重放
- **狀態模式**：配合事件處理狀態轉換，使狀態邏輯清晰
- **中介者模式**：事件匯流排作為組件間的中介，簡化組件間的溝通
- **單例模式**：管理全域訪問點，確保系統唯一性
- **組合模式**：用於遊戲物件層次結構，管理父子關係
