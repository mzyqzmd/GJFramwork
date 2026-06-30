# GJFramework - GameJam 轻量框架

## 框架定位

### 不是什么
- 不是商业级框架，不追求架构完美和万无一失的边界处理
- 不是"一套代码通吃所有项目"的万能钥匙
- 不适合需要复杂 AI 行为树、联网同步、资源热更的大型项目

### 是什么
- 一个"开箱即用"的 GameJam 启动模板
- 8 个模块覆盖了 GameJam 90% 的通用需求
- 整个框架不到 30 个类，半小时内能通读全部源码
- 设计哲学：够用就好，快速出活

### 客观短板（用之前心里有数）
- UI 面板每次打开都是全新实例，高频开关的面板（比如背包）会有 GC 抖动
- 音频系统没有淡入淡出，BGM 切换比较生硬
- 输入系统只预置了移动和导航，跳跃/攻击/交互需要自己加
- 场景名用字符串硬编码，改名时容易漏改

## 必要插件

- **Odin Inspector And Serializer**（官方购买或购物平台上自己找）
  - 如果导入 Odin 时报一堆错，大概率是下面这个文件引起的：
    `Assets/Plugins/Sirenix/Odin Inspector/Modules/Unity.Mathematics/MathematicsDrawers.cs`
  - 直接删掉这个文件就行。
- **TextMeshPro**
- **DOTween**（免费版就行）
- **Unity 官方的 InputSystem**
- 需要 **ExcelDataReader** 库
  - GitHub: https://github.com/ExcelDataReader/ExcelDataReader/releases
  - 下载 netstandard2.0 的 `ExcelDataReader.dll` 放到项目的 `Assets/Plugins`（属于一批特殊文件夹之一）下。

## 模块说明

> 1.Audio、4.Input、7.UI 三个模块需要先手动建对应的 SO 文件并配置 Managers 预制体里面的引用。

---

### 1. Audio 音频系统

#### 干什么的
播 BGM、播音效、停止音频。不用自己写 AudioSource 的管理代码。

#### 配置在哪
- 用右键菜单 `GJFramework/AudioDatabase` 新建（没有提前做因为提前做的话总是有报错）
- 新建完之后要再 Prefabs 文件夹下面的 Managers 预制体中 AudioManager 中配置

#### 怎么配
它是一个两层嵌套字典：第一层 Key 是"音频组名"（比如 BGM、SFX），第二层 Key 是"音频名"（比如 BGM_Title、SFX_Jump），Value 是具体的音频数据。在 Inspector 里展开就能看到结构，直接填就行。

**每个音频可以配：**
- 音频片段列表（多个的话每次随机抽一个播放）
- 优先级（0\~256，数字越小越不容易被挤掉）
- 音量 / 随机音量（勾选后会在 0.1 \~ 你设的值之间随机）
- 音调 / 随机音调
- 空间混合（0=2D，1=3D）
- 是否循环
- 输出到哪个 AudioMixerGroup

**两个全局设置：**
- 音频发射器预制体（拖 `AudioEmitter.prefab`）
- 同一音频最大同时播放数量（默认10）

#### 怎么用
```csharp
// 播放一个BGM，自动循环
AudioManager.Instance.PlayAudio("BGM", "BGM_Title");

// 播放一个跟着角色走的3D音效
AudioManager.Instance.PlayAudio("SFX", "SFX_Footstep", playerTransform);

// 停止某个音频
AudioManager.Instance.StopAudio("BGM", "BGM_Title");

// 停止所有
AudioManager.Instance.StopAllAudio();
```

#### 底层机制
用对象池管理 AudioEmitter，播放完自动回收，不用你管生命周期。

---

### 2. Event 事件系统

#### 干什么的
不同脚本之间互相传消息，不用互相持有引用。比如"玩家死了"这个消息，玩家脚本发布，UI 脚本和音效脚本各自订阅。

#### 创建事件
新建一个结构体（不会触发GC），实现 `IGameEvent` 空接口：

```csharp   “‘csharp
public struct PlayerDiedEvent : IGameEvent
{
    public string KillerName;
}
```

#### 发布事件
```csharp
EventManager.Instance.Publish(new PlayerDiedEvent { KillerName = "Boss" });
// 第二个参数可以传延迟（秒），比如 1.5f 表示 1.5 秒后再触发
```

#### 在 MonoBehaviour 里订阅（推荐）
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

#### 订阅一次性事件
触发一次后自动取消，内部是包装了一层匿名函数：

```csharp
this.RegisterOnceEvent<PlayerDiedEvent>(OnPlayerDied);
```

#### 在普通类里订阅
需要手动取消，不然会内存泄漏：

```csharp
EventManager.Instance.Register<PlayerDiedEvent>(OnPlayerDied);
EventManager.Instance.Unregister<PlayerDiedEvent>(OnPlayerDied);
```

#### 清理事件
```csharp
EventManager.Instance.RemoveAllSubscribers<PlayerDiedEvent>(); // 清空某个事件的所有订阅
EventManager.Instance.RemoveAllSubscribers();                   // 清空全部事件的全部订阅
```

#### 注意
延迟发布是用协程实现的，延迟期间如果对象被销毁了，回调里的代码可能会报空引用，自己做好判空。

---

### 3. FSM 状态机

#### 干什么的
管理角色的状态切换，比如"待机→跑步→跳跃→待机"。支持条件自动转换、转换优先级、分层状态机（父子层同时Update）。

#### 文件说明
| 文件 | 说明 |
|------|------|
| `IState.cs` | 状态接口：Enter / Update / FixedUpdate / Exit |
| `BaseState.cs` | 状态基类，封装了状态机和宿主引用，推荐继承此类来写状态 |
| `StateMachine.cs` | 状态机核心：状态切换、条件转换、子状态机 |
| `Transition.cs` | 转换规则结构体：from→to 的触发条件 |
| `ETransitionPriority.cs` | 转换优先级枚举：Low / Normal / High |

#### 使用步骤

**1. 继承 `BaseState<T>` 写状态基类（T 为宿主类型，如 Player）：**

```csharp
public class PlayerState : BaseState<PlayerTest>
{
    protected Animator _anim;
    protected string _animationName;

    public PlayerState(StateMachine<BaseState<PlayerTest>> stateMachine_,
        PlayerTest owner_, Animator anim_, string animationName_)
        : base(stateMachine_, owner_)
    {
        _anim = anim_;
        _animationName = animationName_;
    }

    public override void Enter()
    {
        base.Enter();
        _anim.SetBool(_animationName, true);
    }

    public override void Exit()
    {
        base.Exit();
        _anim.SetBool(_animationName, false);
    }
}
```

**2. 写具体状态，继承状态基类：**

```csharp   “‘csharp
public class PlayerIdleState : PlayerState { /* ... */ }
public class PlayerRunState  : PlayerState { /* ... */ }
```

**3. 在宿主脚本里创建状态机并初始化：**

```csharp
public PlayerIdleState IdleState { get; private set; }
public PlayerRunState RunState { get; private set; }
private StateMachine<BaseState<PlayerTest>> _stateMachine;

void Awake()
{
    _stateMachine = new StateMachine<BaseState<PlayerTest>>();
    IdleState = new PlayerIdleState(_stateMachine, this, anim, "Idle");
    RunState  = new PlayerRunState(_stateMachine, this, anim, "Run");
}

void Start()
{
    _stateMachine.Initialize(IdleState);
}

void Update()
{
    _stateMachine.Update();   // 自动执行 CurrentState.Update + EvaluateTransitions
}

void FixedUpdate()
{
    _stateMachine.FixedUpdate();
}
```

#### 条件自动转换
```csharp
// 指定状态转换：当前状态为 IdleState 且 isRunning==true 时自动切到 RunState
_stateMachine.AddTransition(IdleState, RunState, () => _isRunning);

// 任意状态转换：无论当前什么状态，isDead==true 时自动切到 DeadState
_stateMachine.AddAnyTransition(DeadState, () => _isDead);

// 转换优先级：同帧多个条件满足时，High > Normal > Low，同优先级按注册顺序
_stateMachine.AddTransition(IdleState, RunState,  () => _isRunning, ETransitionPriority.Normal);
_stateMachine.AddTransition(RunState,  JumpState, () => _wantToJump, ETransitionPriority.High);
```

#### 分层状态机（子状态机）
```csharp
// 父状态机处理大状态（Idle / Run / Dead），子状态机处理跑动中的子状态
_stateMachine = new StateMachine<BaseState<PlayerTest>>();
_stateMachine.Initialize(IdleState);

// 在 RunState 的 Enter 里创建子状态机，子层和父层同时Update
var childFsm = _stateMachine.SetChildFsm(RunSubState);
childFsm.AddTransition(RunSubState, SprintSubState, () => _isSprinting);
```

#### 状态切换事件
```csharp
_stateMachine.OnStateChanged += (from, to) =>
{
    Debug.Log($"状态切换：{from} → {to}");
};
```

#### 属性
- `CurrentState`：当前状态
- `PreviousState`：上一个状态

#### 注意
- 状态是普通类（不是 MonoBehaviour），初始化需要的东西通过构造函数传入。
- 推荐继承 `BaseState<T>` 来写状态，它已封装好 `_myStateMachine` 和 `_owner` 引用。
- 如果不继承 `BaseState<T>`，可以直接实现 `IState` 接口。
- `StateMachine.Update()` 内部已包含 `EvaluateTransitions()`，不需要手动调用。

---

### 4. Input 输入系统

#### 干什么的
把 InputSystem 的输入事件包装成 UnityEvent，你只管订阅就能拿到输入值。

#### 底层
有一个 `GameInputControl.cs`（自动生成的文件，不要手动改），里面定义了 Player 和 UI 两个 ActionMap。Player 下有一个 Movement（WASD），UI 下有一个 Navigation（也是 WASD）。`SO_InputReader` 实现了这两个 ActionMap 的接口，把输入值转成 UnityEvent 发出去。

#### 配置在哪
- 用右键菜单 `GJFramework/InputReader` 新建（没有提前做因为提前做的话总是有报错）
- 新建完之后要再 Prefabs 文件夹下面的 Managers 预制体中 SceneControlManager 中配置

#### 怎么用
1. 在你的脚本里声明一个 `SO_InputReader` 字段
2. 把 `InputReader.asset` 拖到 Inspector 上
3. 在代码里订阅事件：

```csharp   “‘csharp
[SerializeField] private SO_InputReader _inputReader;

void OnEnable()
{
    _inputReader.OnPlayerMovement.AddListener(OnMove);
}

void OnMove(Vector2 input)
{
    // input 就是 WASD 的输入值
}
```

#### 扩展输入
- 从 `Assets/GJFramework/Scripts/Input/GameInputControl.inputactions` 这个路径去添加新的输入
- 然后让他自动生成对应的 `GameInputControl.cs` 脚本
- 打开 `SO_InputReader` 脚本，照着已有的模式添加新的 UnityEvent 和对应的接口方法就行。脚本里有注释说明怎么做。

#### 切换输入模式
```csharp
_inputReader.EnablePlayerInput();   // 启用玩家输入
_inputReader.EnableUIInput();       // 启用UI输入
_inputReader.DisableAllInput();     // 禁用全部输入
```

---

### 5. Save 存档系统

#### 干什么的
把游戏数据（血量、金币、关卡进度等）自动存到本地文件，下次打开游戏能读回来。

#### 整体流程
1. 在 `GameData` 类里声明你要保存的所有字段（血量、金币、关卡进度等等）
2. 需要存档的脚本继承 `SaveableMonoBehaviour`（如果不继承的话就要手动继承 `ISaveable` 接口然后在 Awake 中调用 `SaveManager.RegisterSaveable`）
3. 框架会自动处理注册、保存、加载、加密

#### 使用步骤
```csharp
// 第一步：打开 GameData.cs，在里面加你要保存的字段
// 比如 public int coin; public float health;

// 第二步：让需要存档的脚本继承 SaveableMonoBehaviour
public class PlayerData : SaveableMonoBehaviour
{
    public override void SaveData(ref GameData data)
    {
        data.coin = _coin;    // 把自己的数据写进去
    }

    public override void LoadData(GameData data)
    {
        _coin = data.coin;    // 从存档里读回来
    }
}
```

#### 自动行为
- Awake 时自动注册到 SaveManager（如果 SaveManager 还没初始化会等一帧再注册）
- OnDestroy 时自动取消注册，并先把自己的数据同步到 GameData
- 场景切换时，旧对象销毁前自动保存数据，新对象创建后自动恢复数据

#### 手动操作
```csharp
SaveManager.Instance.SaveGameData();     // 手动保存到本地文件
SaveManager.Instance.LoadGameData();     // 手动从本地文件加载
SaveManager.Instance.DeleteSavedData();  // 删除存档文件
```

#### 检测存档是否存在
```csharp
if (SaveManager.Instance.HasSavedData())
{
    // 有存档，显示"继续游戏"按钮
}
```

#### 只读访问
```csharp
GameData data = SaveManager.Instance.CurrentGameData;
// 拿到的是副本（通过 Odin 序列化深拷贝），改不了原始数据，安全
```

#### 加密
Inspector 上勾选"启用加密"并填个密码，存档就会用异或加密。别人直接打开 JSON 文件看到的是乱码。

#### 自动保存时机
- 游戏退出时（`OnApplicationQuit`）
- 切到后台时（`OnApplicationPause`），防止丢数据

---

### 6. Scene 场景管理

#### 干什么的
加载场景时自动处理淡入淡出、禁用输入、清理旧 UI、切换输入模式。不用每次切场景都手写一堆协程。

#### 怎么用
```csharp
// 加载场景（带淡入淡出过渡，默认 1.5 秒）
SceneControlManager.Instance.LoadScene("GameLevel");

// 自定义过渡时间
SceneControlManager.Instance.LoadScene("MainMenu", 2f);

// 退出游戏（带黑屏淡出，Editor下会自动停止播放）
SceneControlManager.Instance.QuitGame();
```

#### 场景输入模式配置
在 SceneControlManager 的 Inspector 上有个"场景输入模式映射"列表，列表默认有一个 `new("MainMenu", ESceneInputMode.UI)`，可以配置哪些场景加载后自动切换为 UI 输入（比如主菜单），不在列表里的默认用玩家输入。

#### 过渡效果
加载场景时会自动：
1. 黑屏淡入（DOTween 控制 CanvasGroup alpha 0→1）
2. 禁用所有输入
3. 等待淡入动画播完
4. 清理旧场景的所有 UI 面板
5. 异步加载新场景
6. 黑屏淡出（DOTween 控制 CanvasGroup alpha 1→0）
7. 根据配置启用对应的输入模式

---

### 7. UI 管理系统

#### 干什么的
用栈来管理所有 UI 面板的打开/关闭/层级排序。你只管说"打开主菜单"，不用管 Canvas 怎么创建。

#### 配置在哪
- 右键菜单 `GJFramework/UIViewDatabase` 新建（没有提前做因为提前做的话总是有报错）
- 新建完之后要再 Prefabs 文件夹下面的 Managers 预制体中 UIManager 中配置

#### 怎么配
- 基础 Canvas 预制体：拖一个 `UI_BasicCanvas` 的预制体进去
- UI 面板预制体：Key 填面板的类名（比如 `MyMainMenu`），Value 拖对应的预制体
- UI 元素预制体：同理

#### 面板类型（EPanelType）
- `Normal`：普通面板，打开时下层面板继续显示
- `Block`：遮挡面板，打开时自动隐藏下层（省性能）

#### 怎么用
```csharp
// 打开一个面板
UIManager.Instance.OpenPanel<MyMainMenu>();

// 关闭最上面的面板
UIManager.Instance.CloseTopPanel();

// 清空所有面板（场景切换时框架会自动调）
UIManager.Instance.ClearAllPanel();
```

#### 写面板脚本
```csharp
public class MyMainMenu : UI_BasePanel
{
    // OnPanelShow 和 OnPanelHide 是 UnityEvent，
    // 可以在 Inspector 里直接拖拽绑定回调
    // 也可以重写 Show/Hide 方法加自定义逻辑
}
```

#### 关闭动画
在 Inspector 上设 `HideDuration`（秒），UIManager 会等动画播完再销毁面板。如果有关闭动画，子类需要重写 `Hide` 方法自己控制 `SetActive(false)` 的时机。

#### 底层机制
- 面板用栈管理，每个面板自动分配一个 Canvas 和排序层级，层级越深越靠前。
- 打开和关闭 Block 类型面板时会自动处理下层面板的显隐。
- 每次 OpenPanel 都会 Instantiate 新实例，关闭时销毁，保证每次打开都是干净状态，但高频开关的面板会有 GC 压力。

---

### 8. Utilities 工具类

#### 对象池（MonoObjectPool）

**干什么的：** 子弹、特效这些频繁创建销毁的东西，用对象池能大幅减少 GC。只能存 MonoBehaviour 对象。底层封装了 Unity 的 `ObjectPool<T>`。

**用法：**
```csharp
MonoObjectPool<Bullet> _bulletPool;
_bulletPool = new MonoObjectPool<Bullet>(bulletPrefab, transform);

// 可选参数：defaultCapacity(初始容量,默认5), maxSize(最大容量,默认10)
_bulletPool = new MonoObjectPool<Bullet>(bulletPrefab, transform,
    defaultCapacity: 10, maxSize: 50);

// 还可以传回调：onGet, onRelease, onDestroy
_bulletPool = new MonoObjectPool<Bullet>(bulletPrefab, transform,
    onGet: (b) => b.Reset(),           // Get时重置状态
    onRelease: (b) => { /* 回收时的额外处理 */ },
    onDestroy: (b) => { /* 销毁时的清理 */ });

Bullet b = _bulletPool.Get();       // 拿出一个
_bulletPool.Release(b);             // 放回去
_bulletPool.Clear();                // 全部销毁
```

#### 单例

| 类型 | 说明 |
|------|------|
| `Singleton<T>` | 普通单例，不跨场景，切换场景就没了 |
| `PersistentSingleton<T>` | 跨场景单例，游戏不关它就在（Awake里自动调 `DontDestroyOnLoad` 并脱离父节点） |

**用法：**
```csharp
public class MyManager : PersistentSingleton<MyManager>
{
    // 直接继承就行
}

// 外部调用
MyManager.Instance.xxx();
if (MyManager.IsInitialized) { ... }
```

#### Excel 配置表导入（ExcelConfigImporter）

**干什么的：** 把策划在 Excel 里配的数据自动填进 ScriptableObject，不用手写一堆 AssetDatabase 导入代码，改完 Excel 点一下菜单就刷新。

##### 代码结构（一个类一个文件）

| 文件 | 说明 |
|------|------|
| `ExcelSheet.cs` | 数据容器：存表头和数据行，全是 string |
| `ExcelReader.cs` | 读取 .xlsx 文件，输出 ExcelSheet |
| `ExcelHeaderAttribute.cs` | 特性：`[ExcelHeader("表头名")]` 解决字段名和表头不一致 |
| `ExcelFieldFiller.cs` | 反射工具：建映射、填字段、类型转换 |
| `EExcelImportMode.cs` | 枚举：Skip / RowsToList / EachRowToAsset |
| `ExcelImportModes.cs` | 三种模式的具体实现 |
| `ImportWindow.cs` | 编辑器窗口：逐文件选模式、路径配置 |
| `ExcelImporter.cs` | 入口：MenuItem + 模式分发 |

##### 三种导入模式

| 模式 | 说明 |
|------|------|
| `Skip` | 跳过该 xlsx，不导出 |
| `RowsToList` | 所有行注入同一个 SO 的 `List<T>` 字段（如等级表） |
| `EachRowToAsset` | 每行生成一个独立的 .asset，第一列作为文件名（如角色表） |

##### 路径配置
- 打开导入配置窗口，顶部可配置三个目录：Excel读取目录、模板目录、SO输出目录
- 路径持久化到 EditorPrefs，改一次永久生效
- 点击"选择"按钮用文件夹对话框选，也可以手动输入相对路径
- 修改后需点"保存路径"，也可点"重置默认"恢复默认路径
- 默认路径：
  - Excel = `Assets/GJFramework/Excels`
  - 模板 = `Assets/GJFramework/Scriptable/Templates`
  - 输出 = `Assets/GJFramework/Scriptable`

##### SO 模板命名要求
- 模板文件必须放在"模板目录"下，文件名 = xlsx 文件名（不含扩展名）
- Windows 上大小写不敏感，但是跨平台（macOS/Linux）区分大小写
  - 例：`levelConfig.xlsx` 的模板应为 `Templates/levelConfig.asset`
- 模板必须是通过右键菜单 `Create → 对应菜单项` 创建的 ScriptableObject 资产
- **RowsToList 模式**：模板是"目标类型的容器"，模板类型决定了导入结果 SO 的类型。导入时从模板目录复制到输出目录（覆盖），再填入 Excel 数据。模板的内容无所谓（可以是空的），只取其类型和文件名。
- **EachRowToAsset 模式**：模板同样是"目标类型的容器"，但只用来确定每行生成的 .asset 的类型，不会复制模板的内容。
- 每个 xlsx 文件对应一个模板，模板名必须和 xlsx 名严格一致。

##### SO 类字段要求

**1. 类必须是 ScriptableObject 子类：**
```csharp
public class LevelConfig : ScriptableObject { ... }
```

**2. 可被导入的字段：**
- public 字段
- 标记了 `[SerializeField]` 的 private / protected 字段
- 注意：属性（Property）不会被识别，必须是字段（Field）

**3. RowsToList 模式的 SO 类结构：**
- 必须包含一个 `List<T>` 字段（只会填充找到的第一个 `List<T>`）
- 如果 SO 有多个 `List<T>` 字段，只有第一个会被填入数据，其余忽略
- 如果没有 `List<T>` 字段，则回退为"单行填充"——用 Excel 第一行数据直接填 SO 的平铺字段

```csharp
[CreateAssetMenu(fileName = "LevelConfig", menuName = "GJFramework/LevelConfig")]
public class LevelConfig : ScriptableObject
{
    public List<LevelRow> Levels;   // ← 必须是 List<T>，不支持 T[]
}
```

**4. EachRowToAsset 模式的 SO 类结构：**
- 所有字段都是平铺的（没有 List），每行数据对应一个字段集合
- 实际运行时，行数据对应的元素类型是 SO 类本身，不需要额外定义 Row 类

```csharp
[CreateAssetMenu(fileName = "CharacterConfig", menuName = "GJFramework/CharacterConfig")]
public class CharacterConfig : ScriptableObject
{
    public string _characterName;
    public int _levelId;
    public float _timeLimit;
}
```

**5. List 元素类型（即 `List<T>` 中的 T）：**
- 可以是任意 class（不需要继承 MonoBehaviour 或 ScriptableObject）
- 必须标记 `[System.Serializable]`
- 构造方式：`Activator.CreateInstance`，要求有无参构造函数
- 字段规则同上（public 或 `[SerializeField]`）

> 注意：如果元素类型一个 public/`[SerializeField]` 字段都没有，导入时会报错"没有 public 或 [SerializeField] 字段"。> Note: If an element type has no public or `[SerializeField]` fields, an error will occur during import stating "No public or [SerializeField] fields." 。

```csharp   “‘csharp
[System.Serializable]
public class LevelRow
{
    [ExcelHeader("关卡名称")]  // 可选，字段名与表头不一致时才需要(大小写敏感!!!)
    public string _levelName;
    public int _levelId;
    public float _timeLimit;   public float _时间
}
```

##### xlsx 命名要求
- 文件名必须与模板 .asset 文件名一致（不含扩展名）
- Windows 上大小写不敏感，但是跨平台（macOS/Linux）区分大小写
  - 例：`levelConfig.xlsx` ↔ `levelConfig.asset`- Example: `levelConfig.xlsx` ↔ `levelConfig.asset`
- 文件放在"Excel 读取目录"下（默认 `Assets/GJFramework/Excels/`）- Place the file under the "Excel Read Directory" (default: `Assets/GJFramework/Excels/`)
- 扩展名必须是 `.xlsx`（不是 `.xls`）
- 只读取第一个 Sheet，后续 Sheet 会被忽略
- 建议：不要用中文命名 xlsx 文件，避免路径处理问题

##### Sheet 布局要求
- 第 1 行 = 表头行，每一列的值为字段名（或 `[ExcelHeader]` 指定的别名）
- 第 2 行起 = 数据行，每个单元格的值会按目标字段类型自动转换
- **表头名匹配规则（按优先级）：**
  1. 字段上有 `[ExcelHeader("xxx")]` 则用 xxx 匹配1. If a field has `[ExcelHeader("xxx")   ExcelHeader("xxx")], use xxx for matching
  2. 没有该特性则用字段名匹配
  3. 表头名忽略首尾空格（自动 Trim）
- 空表头列会被跳过
- 空单元格会被跳过（不覆盖字段原值，保留模板中的默认值）
- 表头行必须存在，否则报错"Excel 没有表头行"
- 至少有一行数据，否则警告"Excel 没有数据行"并跳过
- EachRowToAsset 模式额外要求：   - Additional requirements for the EachRowToAsset pattern:
  - 第一列的值作为 .asset 文件名
  - 第一列不能为空，空名称的行会被跳过并警告
- 列的顺序不重要，表头名匹配与列的位置无关

**示例布局（levelConfig.xlsx）：****Example Layout (levelConfig.xlsx):**

| 关卡名称 | \_levelId | \_timeLimit || Level Name | \_levelId | \_timeLimit |
|----------|-----------|-------------|
| 新手村   | 1         | 60          |
| 森林     | 2         | 90          |
| 火山     | 3         | 120         |

##### 支持的类型转换

| 类型 | 转换方式 |
|------|----------|
| `string` | 直接赋值 |
| `int` | `int.Parse` |   b| ' int ' b| ' int。解析“|
| `float` | `float.Parse` |
| `double` | `double.Parse` |
| `bool` | `bool.Parse`（注意：Excel 中写 "true"/"false"，不要写 0/1） || `bool   保龄球` | `bool.Parse   保龄球。解析` (Note: In Excel, write "true"/"false", not 0/1) |
| `enum` | `Enum.Parse`（值与枚举名一致，大小写敏感） || `enum   枚举` | `Enum.Parse   枚举。解析` (value matches enum name, case-sensitive) |
| `Vector2` | 逗号分隔，如 "1.0,2.5" || `Vector2` | Comma-separated, e.g., "1.0,2.5" |
| `Vector3` | 逗号分隔，如 "1.0,2.5,3.0" || `Vector3` | Comma-separated, e.g., "1.0,2.5,3.0" |
| `Color` | HTML 格式，如 "#FF0000" 或 "red" || `Color   颜色` | HTML format, such as " #FF0000" or " red" |

##### 使用步骤

1. 创建 ScriptableObject 配置类（按上面"SO 类字段要求"写）1. Create a ScriptableObject configuration class (written according to the "SO Class Field Requirements" above)

2. 在模板目录下手动创建空白模板 .asset：
   - 右键 → `Create` → `GJFramework` → 对应菜单项Right-click → `Create   创建` → `GJFramework` → Corresponding menu item
   - 模板内容无所谓（可以是空值），只取其类型和文件名
   - 放在 `Templates/` 下，文件名与 xlsx 同名- Place it under `Templates/   模板/`, with the same filename as the xlsx file

3. 在 Excel 目录下放 xlsx 文件，按上面"Sheet 布局要求"填写表头和数据

4. 菜单 `Tools` → `GJFramework` → `Excel` → `打开导入配置窗口`，为每个文件选模式（默认是 EachRowToAsset），点"开始导入"（或逐文件点"导入"）。也可以用"全部RowsToList""全部EachRowToAsset""全部跳过"批量设置。4. Menu `Tools   工具` → `GJFramework` → `Excel` → Open the import configuration window, select a mode for each file (default is EachRowToAsset), then click "Start Import" (or click "Import" for each file individually). You can also use "AllRowsToList". " EachRowToAsset " Skip all "Batch Settings."

5. 数据会自动填入输出目录下对应的 .asset 文件

##### 手动读取 Excel 原始数据（不导入 SO）
```csharp   “‘csharp   ```csharp   “‘csharp
var sheet = ExcelReader.ReadSheet("Assets/GJFramework/Excels/xxx.xlsx");var sheet = ExcelReader.ReadSheet("Assets/GJFramework/Excels/xxx.xlsx");
// sheet.Headers 是表头行（string[]）// sheet.Headers is the header row (string[])var sheet = ExcelReader.ReadSheet("Assets/GJFramework/Excels/xxx.xlsx");var sheet = ExcelReader.ReadSheet("Assets/GJFramework/Excels/xxx.xlsx");// sheet.Headers 是表头行（string[]）
// sheet.Rows 是数据行列表（List<string[]>）// sheet.Rows is a list of data rows (List)// sheet.Rows is a list of data rows (List)
foreach (var row in sheet.Rows)
{
    string name = row[0];   // 第一列string name = row[0];   // First columnstring name = row[0];   // 第一列
    string value = row[1];  // 第二列string value = row[1];  // Second columnstring value = row[1];  // 第二列
}
// 所有数据都是 string，类型转换自己处理
// 注意：只读取第一个 Sheet，后续 Sheet 会被忽略
```
