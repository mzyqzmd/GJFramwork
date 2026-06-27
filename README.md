# GJFramwork
用于参加GameJam48-72小时极限开发的中小型框架

---

# GJFramework - GameJam 轻量框架# GJFramework - A Lightweight Framework for Game JamGJFramework - A Lightweight Framework for Game Jam

## 框架定位

### 不是什么
- 不是商业级框架，不追求架构完美和万无一失的边界处理
- 不是"一套代码通吃所有项目"的万能钥匙
- 不适合需要复杂 AI 行为树、联网同步、资源热更的大型项目

### 是什么
- 一个"开箱即用"的 GameJam 启动模板
- 8 个模块覆盖了 GameJam 90% 的通用需求
- 整个框架不到 30 个类，半小时内能通读全部源码
- 设计哲学：**够用就好，快速出活**

### 客观短板（用之前心里有数）
- FSM 状态机只有单层，复杂 AI 得自己扩展
- UI 面板每次打开都是全新实例，高频开关的面板（比如背包）会有 GC 抖动
- 音频系统没有淡入淡出，BGM 切换比较生硬
- 输入系统只预置了移动和导航，跳跃/攻击/交互需要自己加
- 场景名用字符串硬编码，改名时容易漏改

---

## 必要插件

| 插件 | 说明 |
|------|------|
| **Odin Inspector And SerializerOdin 检查器与序列化器** | 官方购买或反正就是自己想想办法。若导入时报错，删除 `Assets/Plugins/Sirenix/Odin Inspector/Modules/Unity.Mathematics/MathematicsDrawers.cs` 即可 || **Odin Inspector And SerializerOdin 检查器与序列化器** | Purchase officially or obtain on your own. If errors occur upon import, simply delete `Assets/Plugins/Sirenix/Odin Inspector/Modules/Unity.Mathematics/MathematicsDrawers.cs` |
| **TextMeshPro** | Unity 内置，直接安装 || **TextMeshPro** | Built into Unity, install directly |
| **DOTween** | 免费版即可 |
| **InputSystem** | Unity 官方新输入系统 || **InputSystem** | Unity's official new input system |
| **ExcelDataReader** | [GitHub 下载](https://github.com/ExcelDataReader/ExcelDataReader/releases)，将 `netstandard2.0` 的 `ExcelDataReader.dll` 放入 `Assets/Plugins` || **ExcelDataReader** | [Download from GitHub](https://github.com/ExcelDataReader/ExcelDataReader/releases), place the `ExcelDataReader.dll` from `netstandard2.0` into `Assets/Plugins` |

---

## 模块总览

> **注意**：模块 1（Audio）、4（Input）、7（UI）需要先手动创建对应的 SO 文件，并在 `Managers` 预制体中配置引用。

| 模块 | 说明 |
|------|------|
| 1. Audio | 音频系统 |
| 2. Event | 事件系统 |
| 3. FSM | 状态机 |
| 4. Input | 输入系统 |
| 5. Save | 存档系统 |
| 6. Scene | 场景管理 |
| 7. UI | UI 管理系统 |
| 8. Utilities | 工具类（对象池、单例、Excel 导入） |

---

## 1. Audio 音频系统

### 功能说明
播 BGM、播音效、停止音频。无需自己编写 `AudioSource` 管理代码。

### 配置方式
1. 右键 → `GJFramework/AudioDatabase` 新建 SO（未预置是因为提前创建会报错）
2. 将新建的 SO 配置到 `Prefabs/Managers` 预制体中的 `AudioManager` 组件上

### 数据结构
两层嵌套字典：
- 第一层 Key：音频组名（如 `BGM`、`SFX`）
- 第二层 Key：音频名（如 `BGM_Title`、`SFX_Jump`）
- Value：具体音频数据

在 Inspector 中展开即可直接填写。

### 每条音频可配置项

| 配置项 | 说明 |
|--------|------|
| 音频片段列表 | 多个片段时每次随机抽取一个播放 |
| 优先级 | 0~256，数字越小越不容易被挤掉 |
| 音量 / 随机音量 | 勾选随机音量后，在 0.1 ~ 设定值之间随机 |
| 音调 / 随机音调 | 同上 |
| 空间混合 | 0 = 2D，1 = 3D |
| 是否循环 | 勾选后循环播放 |
| Output AudioMixerGroup | 输出到哪个混音组 |

### 全局设置
- **音频发射器预制体**：拖入 `AudioEmitter.prefab`
- **同一音频最大同时播放数量**：默认 10

### 使用示例

```csharp
// 播放 BGM（自动循环）
AudioManager.Instance.PlayAudio("BGM", "BGM_Title");

// 播放跟随角色的 3D 音效
AudioManager.Instance.PlayAudio("SFX", "SFX_Footstep", playerTransform);

// 停止某个音频
AudioManager.Instance.StopAudio("BGM", "BGM_Title");

// 停止所有音频
AudioManager.Instance.StopAllAudio();
```

### 底层机制
使用对象池管理 `AudioEmitter`，播放完毕后自动回收，无需手动管理生命周期。

---

## 2. Event 事件系统

### 功能说明
不同脚本之间传递消息，无需互相持有引用。例如"玩家死亡"事件，玩家脚本发布，UI 脚本和音效脚本各自订阅。

### 创建事件

```csharp
// 新建结构体（不会触发 GC），实现 IGameEvent 空接口
public struct PlayerDiedEvent : IGameEvent
{
    public string KillerName;
}
```

### 发布事件

```csharp
// 立即发布
EventManager.Instance.Publish(new PlayerDiedEvent { KillerName = "Boss" });

// 延迟发布（秒），1.5f 表示 1.5 秒后触发
EventManager.Instance.Publish(new PlayerDiedEvent { KillerName = "Boss" }, 1.5f);
```

### 在 MonoBehaviour 中订阅（推荐）

```csharp
void OnEnable()
{
    this.RegisterEvent<PlayerDiedEvent>(OnPlayerDied);
}

void OnPlayerDied(PlayerDiedEvent e)
{
    Debug.Log($"玩家被 {e.KillerName} 杀死了");
}
// 对象销毁时自动取消订阅，不会内存泄漏
```

### 订阅一次性事件

```csharp
// 触发一次后自动取消（内部包装了一层匿名函数）
this.RegisterOnceEvent<PlayerDiedEvent>(OnPlayerDied);
```

### 在普通类中订阅（需手动取消）

```csharp
EventManager.Instance.Register<PlayerDiedEvent>(OnPlayerDied);
EventManager.Instance.Unregister<PlayerDiedEvent>(OnPlayerDied);
```

### 清理事件

```csharp
// 清空某个事件的所有订阅
EventManager.Instance.RemoveAllSubscribers<PlayerDiedEvent>();

// 清空全部事件的全部订阅
EventManager.Instance.RemoveAllSubscribers();
```

### 注意事项
延迟发布使用协程实现，延迟期间若对象被销毁，回调中的代码可能会报空引用，需自行做好判空处理。

---

## 3. FSM 状态机

### 功能说明
管理角色状态切换（如"待机 → 跑步 → 跳跃 → 待机"）。仅支持单层状态机，GameJam 场景完全够用。

### 使用步骤

**Step 1：编写状态基类，实现 `IState` 接口**

> 注意：方法需加 `virtual`，子类才能重写。

```csharp
public class PlayerState : IState
{
    protected Player _player;
    protected Animator _anim;
    protected string _animationName;
    protected StateMachine<PlayerState> _stateMachine;

    public virtual void Enter() { }
    public virtual void Update() { }
    public virtual void Exit() { }
}
```

**Step 2：编写具体状态，继承状态基类**

```csharp
public class PlayerIdleState : PlayerState
{
    public override void Enter()
    {
        base.Enter();
        // 进入待机状态的逻辑
    }
}
```

**Step 3：在玩家脚本中使用**

```csharp
public class Player : MonoBehaviour
{
    public PlayerIdleState IdleState { get; private set; }
    public PlayerRunState RunState { get; private set; }
    private StateMachine<PlayerState> _stateMachine;

    void Awake()
    {
        _stateMachine = new StateMachine<PlayerState>();
        IdleState = new PlayerIdleState(this, anim, "Idle", _stateMachine);
        RunState = new PlayerRunState(this, anim, "Run", _stateMachine);
    }

    void Start()
    {
        _stateMachine.InitializeState(IdleState);
    }

    void Update()
    {
        _stateMachine.CurrentState.Update();
    }
}
```

### 注意事项
- 状态是普通类（非 MonoBehaviour），初始化所需数据通过构造函数传入
- 状态基类的 `Enter`/`Update`/`Exit` 必须加 `virtual`，否则子类无法 `override`

---

## 4. Input 输入系统

### 功能说明
将 InputSystem 的输入事件包装成 `UnityEvent`，只需订阅即可获取输入值。

### 底层结构
- `GameInputControl.cs`：由 `.inputactions` 文件自动生成，**不要手动修改**
- 定义了 `Player` 和 `UI` 两个 ActionMap
  - `Player` 下：`Movement`（WASD）
  - `UI` 下：`Navigation`（WASD）
- `SO_InputReader`：实现了这两个 ActionMap 的接口，将输入值转为 `UnityEvent` 发出

### 配置方式
1. 右键 → `GJFramework/InputReader` 新建 SO
2. 将新建的 SO 配置到 `Prefabs/Managers` 预制体中的 `SceneControlManager` 组件上

### 基础使用

```csharp
[SerializeField] private SO_InputReader _inputReader;

void OnEnable()
{
    _inputReader.OnPlayerMovement.AddListener(OnMove);
}

void OnMove(Vector2 input)
{
    // input 即为 WASD 的输入值
}
```

### 扩展输入
1. 从 `Assets/GJFramework/Scripts/Input/GameInputControl.inputactions` 添加新输入
2. 让其自动生成对应的 `GameInputControl.cs` 脚本
3. 打开 `SO_InputReader` 脚本，参照已有模式添加新的 `UnityEvent` 和对应的接口方法（脚本内有注释说明）

### 切换输入模式

```csharp
_inputReader.EnablePlayerInput();   // 启用玩家输入
_inputReader.EnableUIInput();       // 启用 UI 输入
_inputReader.DisableAllInput();     // 禁用全部输入
```

---

## 5. Save 存档系统

### 功能说明
将游戏数据（血量、金币、关卡进度等）自动存储到本地文件，下次启动游戏时可读取恢复。

### 整体流程
1. 在 `GameData` 类中声明需要保存的所有字段
2. 需要存档的脚本继承 `SaveableMonoBehaviour`（若不继承，则需手动实现 `ISaveable` 接口并在 `Awake` 中调用 `SaveManager.RegisterSaveable`）
3. 框架自动处理注册、保存、加载、加密

### 使用步骤

**Step 1：在 `GameData.cs` 中添加需要保存的字段**

```csharp
// 示例：public int coin; public float health;
```

**Step 2：让需要存档的脚本继承 `SaveableMonoBehaviour`**

```csharp
public class PlayerData : SaveableMonoBehaviour
{
    public override void SaveData(ref GameData data)
    {
        data.coin = _coin;    // 将自身数据写入
    }

    public override void LoadData(GameData data)
    {
        _coin = data.coin;    // 从存档中读取恢复
    }
}
```

### 自动行为
- `Awake` 时自动注册到 `SaveManager`（若 `SaveManager` 尚未初始化，会等待一帧后再注册）
- `OnDestroy` 时自动取消注册，并先将自身数据同步到 `GameData`
- 场景切换时：旧对象销毁前自动保存数据，新对象创建后自动恢复数据

### 手动操作

```csharp
SaveManager.Instance.SaveGameData();     // 手动保存到本地文件
SaveManager.Instance.LoadGameData();     // 手动从本地文件加载
SaveManager.Instance.DeleteSavedData();  // 删除存档文件
```

### 检测存档是否存在

```csharp
if (SaveManager.Instance.HasSavedData())
{
    // 有存档，显示"继续游戏"按钮
}
```

### 只读访问

```csharp
GameData data = SaveManager.Instance.CurrentGameData;
// 返回的是副本（通过 Odin 序列化深拷贝），无法修改原始数据，安全
```

### 加密
在 Inspector 上勾选"启用加密"并填写密码，存档将使用异或加密。直接打开 JSON 文件看到的是乱码。

### 自动保存时机
- 游戏退出时（`OnApplicationQuit`）
- 切到后台时（`OnApplicationPause`），防止数据丢失

---

## 6. Scene 场景管理

### 功能说明
加载场景时自动处理淡入淡出、禁用输入、清理旧 UI、切换输入模式。无需每次切换场景都手写协程。

### 基础使用

```csharp
// 加载场景（带淡入淡出过渡，默认 1.5 秒）
SceneControlManager.Instance.LoadScene("GameLevel");

// 自定义过渡时间
SceneControlManager.Instance.LoadScene("MainMenu", 2f);

// 退出游戏（带黑屏淡出，Editor 下会自动停止播放）
SceneControlManager.Instance.QuitGame();
```

### 场景输入模式配置
在 `SceneControlManager` 的 Inspector 上有"场景输入模式映射"列表：
- 列表默认包含 `new("MainMenu", ESceneInputMode.UI)`
- 可配置哪些场景加载后自动切换为 UI 输入（如主菜单）
- 不在列表中的场景默认使用玩家输入

### 过渡效果流程
加载场景时自动执行以下步骤：
1. 黑屏淡入（DOTween 控制 `CanvasGroup` alpha `0 → 1`）
2. 禁用所有输入
3. 等待淡入动画播完
4. 清理旧场景的所有 UI 面板
5. 异步加载新场景
6. 黑屏淡出（DOTween 控制 `CanvasGroup` alpha `1 → 0`）
7. 根据配置启用对应的输入模式

---

## 7. UI 管理系统

### 功能说明
使用栈管理所有 UI 面板的打开/关闭/层级排序。只需调用打开方法，无需手动管理 Canvas 的创建。

### 配置方式
1. 右键 → `GJFramework/UIViewDatabase` 新建 SO
2. 将新建的 SO 配置到 `Prefabs/Managers` 预制体中的 `UIManager` 组件上

### Inspector 配置项

| 配置项 | 说明 |
|--------|------|
| 基础 Canvas 预制体 | 拖入 `UI_BasicCanvas` 预制体 |
| UI 面板预制体 | Key 填面板类名（如 `MyMainMenu`），Value 拖入对应预制体 |
| UI 元素预制体 | 同理配置 |

### 面板类型（`EPanelType`）

| 类型 | 说明 |
|------|------|
| `Normal` | 普通面板，打开时下层面板继续显示 |
| `Block` | 遮挡面板，打开时自动隐藏下层（节省性能） |

### 基础使用

```csharp
// 打开一个面板
UIManager.Instance.OpenPanel<MyMainMenu>();

// 关闭最上面的面板
UIManager.Instance.CloseTopPanel();

// 清空所有面板（场景切换时框架会自动调用）
UIManager.Instance.ClearAllPanel();
```

### 编写面板脚本

```csharp
public class MyMainMenu : UI_BasePanel
{
    // OnPanelShow 和 OnPanelHide 是 UnityEvent，
    // 可在 Inspector 中直接拖拽绑定回调，
    // 也可重写 Show/Hide 方法添加自定义逻辑
}
```

### 关闭动画
- 在 Inspector 上设置 `HideDuration`（秒），`UIManager` 会等待动画播完后再销毁面板
- 若有关闭动画，子类需重写 `Hide` 方法自行控制 `SetActive(false)` 的时机

### 底层机制
- 面板用栈管理，每个面板自动分配一个 Canvas 和排序层级，层级越深越靠前
- 打开和关闭 `Block` 类型面板时，会自动处理下层面板的显隐
- 每次 `OpenPanel` 都会 `Instantiate` 新实例，关闭时销毁，保证每次打开都是干净状态
- **缺点**：高频开关的面板会产生 GC 压力

---

## 8. Utilities 工具类

### 8.1 对象池（`MonoObjectPool<T>`）

#### 功能说明
用于子弹、特效等频繁创建销毁的对象，大幅减少 GC。仅支持存储 `MonoBehaviour` 对象。底层封装了 Unity 的 `ObjectPool<T>`。

#### 基础使用

```csharp
MonoObjectPool<Bullet> _bulletPool;

// 最简用法
_bulletPool = new MonoObjectPool<Bullet>(bulletPrefab, transform);

// 指定容量
_bulletPool = new MonoObjectPool<Bullet>(
    bulletPrefab,
    transform,
    defaultCapacity: 10,
    maxSize: 50
);

// 传入回调
_bulletPool = new MonoObjectPool<Bullet>(
    bulletPrefab,
    transform,
    onGet: (b) => b.Reset(),              // Get 时重置状态
    onRelease: (b) => { /* 回收时的额外处理 */ },
    onDestroy: (b) => { /* 销毁时的清理 */ }
);

Bullet b = _bulletPool.Get();       // 取出一个
_bulletPool.Release(b);             // 放回池中
_bulletPool.Clear();                // 全部销毁
```

---

### 8.2 单例（`Singleton<T>` / `PersistentSingleton<T>`）

#### 功能说明

| 类型 | 说明 |
|------|------|
| `Singleton<T>` | 普通单例，不跨场景，切换场景后销毁 |
| `PersistentSingleton<T>` | 跨场景单例，游戏不关闭则一直存在（`Awake` 中自动调用 `DontDestroyOnLoad` 并脱离父节点） |

#### 使用方式

```csharp
public class MyManager : PersistentSingleton<MyManager>
{
    // 直接继承即可
}

// 外部调用
MyManager.Instance.xxx();

// 判断是否已初始化
if (MyManager.IsInitialized) { ... }
```

---

### 8.3 Excel 配置表导入（`ExcelConfigImporter`）

#### 功能说明
将 Excel 中配置的数据自动填充到 ScriptableObject，无需手写 `AssetDatabase` 导入代码。修改 Excel 后点击菜单即可刷新。

#### 代码结构（一个类一个文件）

| 文件 | 说明 |
|------|------|
| `ExcelSheet.cs` | 数据容器：存储表头和数据行，全为 `string` |
| `ExcelReader.cs` | 读取 `.xlsx` 文件，输出 `ExcelSheet` |
| `ExcelHeaderAttribute.cs` | 特性：`[ExcelHeader("表头名")]` 解决字段名与表头不一致的问题 |
| `ExcelFieldFiller.cs` | 反射工具：建立映射、填充字段、类型转换 |
| `EExcelImportMode.cs` | 枚举：`Skip` / `RowsToList` / `EachRowToAsset` |
| `ExcelImportModes.cs` | 三种模式的具体实现 |
| `ImportWindow.cs` | 编辑器窗口：逐文件选择模式、路径配置 |
| `ExcelImporter.cs` | 入口：`MenuItem` + 模式分发 |

#### 三种导入模式

| 模式 | 说明 |
|------|------|
| `Skip` | 跳过该 xlsx，不导出 |
| `RowsToList` | 所有行注入同一个 SO 的 `List<T>` 字段（如等级表） |
| `EachRowToAsset` | 每行生成一个独立的 `.asset`，第一列作为文件名（如角色表） |

#### 路径配置

打开导入配置窗口，顶部可配置三个目录：

| 目录 | 默认值 |
|------|--------|
| Excel 读取目录 | `Assets/GJFramework/Excels` |
| 模板目录 | `Assets/GJFramework/Scriptable/Templates` |
| SO 输出目录 | `Assets/GJFramework/Scriptable` |

- 路径持久化到 `EditorPrefs`，修改一次永久生效
- 可点击"选择"按钮通过文件夹对话框选取，也可手动输入相对路径
- 修改后需点击"保存路径"，也可点击"重置默认"恢复默认路径

#### SO 模板命名要求
- 模板文件必须放在**模板目录**下，文件名 = xlsx 文件名（不含扩展名）
- Windows 上大小写不敏感，但跨平台（macOS/Linux）区分大小写
  - 例：`levelConfig.xlsx` 的模板应为 `Templates/levelConfig.asset`
- 模板必须通过右键菜单 `Create → 对应菜单项` 创建，是合法的 ScriptableObject 资产
- `RowsToList` 模式：模板是"目标类型的容器"，导入时从模板目录复制到输出目录（覆盖），再填入 Excel 数据
- `EachRowToAsset` 模式：模板同样是"目标类型的容器"，但只用来确定每行生成的 `.asset` 的类型，不会复制模板内容

#### SO 类字段要求

**通用规则：**
- 类必须是 `ScriptableObject` 的子类
- 可被导入的字段：`public` 字段，或标记了 `[SerializeField]` 的 `private`/`protected` 字段
- **注意**：属性（Property）不会被识别，必须是字段（Field）

**`RowsToList` 模式的 SO 类结构：**

```csharp
[CreateAssetMenu(fileName = "LevelConfig", menuName = "GJFramework/LevelConfig")]
public class LevelConfig : ScriptableObject
{
    public List<LevelRow> Levels;   // 必须是 List<T>，不支持 T[]
}
```

- 必须包含一个 `List<T>` 字段（只会填充找到的第一个 `List<T>`）
- 若 SO 有多个 `List<T>` 字段，只有第一个会被填入数据，其余忽略
- 若没有 `List<T>` 字段，则回退为"单行填充"——用 Excel 第一行数据直接填充 SO 的平铺字段

**`EachRowToAsset` 模式的 SO 类结构：**

```csharp
[CreateAssetMenu(fileName = "CharacterConfig", menuName = "GJFramework/CharacterConfig")]
public class CharacterConfig : ScriptableObject
{
    public string _characterName;
    public int _levelId;
    public float _timeLimit;
}
```

- 所有字段都是平铺的（没有 `List`），每行数据对应一个字段集合
- 实际运行时，行数据对应的元素类型是 SO 类本身，无需额外定义 Row 类

#### `List` 元素类型（即 `List<T>` 中的 `T`）：
- 可以是任意 `class`（无需继承 `MonoBehaviour` 或 `ScriptableObject`）
- 必须标记 `[System.Serializable]`
- 构造方式：`Activator.CreateInstance`，要求有无参构造函数
- 字段规则同上（`public` 或 `[SerializeField]`）
- **注意**：若元素类型一个 `public`/`[SerializeField]` 字段都没有，导入时会报错"没有 public 或 [SerializeField] 字段"

```csharp
[System.Serializable]
public class LevelRow
{
    [ExcelHeader("关卡名称")]  // 可选，字段名与表头不一致时才需要（大小写敏感！！！）
    public string _levelName;
    public int _levelId;
    public float _timeLimit;
}
```

#### xlsx 命名要求
- 文件名必须与模板 `.asset` 文件名一致（不含扩展名）
- Windows 上大小写不敏感，但跨平台（macOS/Linux）区分大小写
  - 例：`levelConfig.xlsx` ↔ `levelConfig.asset`
- 文件放在"Excel 读取目录"下（默认 `Assets/GJFramework/Excels/`）
- 扩展名必须是 `.xlsx`（不是 `.xls`）
- 只读取第一个 Sheet，后续 Sheet 会被忽略
- 建议：不要用中文命名 xlsx 文件，避免路径处理问题

#### Sheet 布局要求

| 要求 | 说明 |
|------|------|
| 第 1 行 | 表头行，每一列的值为字段名（或 `[ExcelHeader]` 指定的别名） |
| 第 2 行起 | 数据行，每个单元格的值按目标字段类型自动转换 |
| 表头名匹配规则（优先级） | 1. 字段上有 `[ExcelHeader("xxx")]` 则用 `xxx` 匹配；2. 没有则用字段名匹配；3. 表头名自动 Trim |
| 空表头列 | 会被跳过 |
| 空单元格 | 会被跳过（不覆盖字段原值，保留模板中的默认值） |
| 表头行 | 必须存在，否则报错"Excel 没有表头行" |
| 数据行 | 至少有一行，否则警告"Excel 没有数据行"并跳过 |
| `EachRowToAsset` 模式 | 第一列的值作为 `.asset` 文件名；第一列为空时该行会被跳过并警告 |
| 列顺序 | 不重要，表头名匹配与列的位置无关 |

**示例布局（`levelConfig.xlsx`）：**

| 关卡名称 | _levelId | _timeLimit |
|----------|----------|------------|
| 新手村   | 1        | 60         |
| 森林     | 2        | 90         |
| 火山     | 3        | 120        |

#### 支持的类型转换

| 类型 | 格式说明 |
|------|----------|
| `string` | 直接赋值 |
| `int` | `int.Parse` |
| `float` | `float.Parse` |
| `double` | `double.Parse` |
| `bool` | `bool.Parse`（Excel 中写 `"true"`/`"false"`，不要写 `0`/`1`） |
| `enum` | `Enum.Parse`（值与枚举名一致，**大小写敏感**） |
| `Vector2` | 逗号分隔，如 `"1.0,2.5"` |
| `Vector3` | 逗号分隔，如 `"1.0,2.5,3.0"` |
| `Color` | HTML 格式，如 `"#FF0000"` 或 `"red"` |

#### 使用步骤

1. 创建 ScriptableObject 配置类（按"SO 类字段要求"编写）
2. 在模板目录下手动创建空白模板 `.asset`：
   - 右键 → `Create` → `GJFramework` → 对应菜单项
   - 模板内容无所谓（可以是空值），只取其类型和文件名
   - 放在 `Templates/` 下，文件名与 xlsx 同名
3. 在 Excel 目录下放置 xlsx 文件，按"Sheet 布局要求"填写表头和数据
4. 菜单 `Tools → GJFramework → Excel → 打开导入配置窗口`
   - 为每个文件选择模式（默认为 `EachRowToAsset`）
   - 点击"开始导入"（或逐文件点击"导入"）
   - 也可使用"全部 RowsToList""全部 EachRowToAsset""全部跳过"批量设置
5. 数据会自动填入输出目录下对应的 `.asset` 文件

#### 手动读取 Excel 原始数据（不导入 SO）

```csharp
var sheet = ExcelReader.ReadSheet("Assets/GJFramework/Excels/xxx.xlsx");
// sheet.Headers 是表头行（string[]）
// sheet.Rows 是数据行列表（List<string[]>）
foreach (var row in sheet.Rows)
{
    string name = row[0];   // 第一列
    string value = row[1];  // 第二列
}
// 所有数据都是 string，类型转换需自行处理
// 注意：只读取第一个 Sheet，后续 Sheet 会被忽略
