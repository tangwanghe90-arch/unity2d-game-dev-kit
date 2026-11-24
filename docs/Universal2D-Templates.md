# Unity 2D 万能模板代码与操作手册

适用 Unity 2021.3 / 2022.3 / Unity 6 的 2D 项目。下面是一套可以直接复制进 `Assets/Scripts` 的独立脚本，覆盖平台跳跃、俯视移动、Flappy、弹射、射击、打地鼠、打砖块和通用规则层。

> 最快路线：先复制 `Rb2D.cs` 和 `GameFlow2D.cs`，再按题型复制一个玩家脚本和两三个交互脚本。整套复制也不会和仓库里的示例组件重名。

---

## 0. 先选一种做法

| 你手里有什么 | 怎么做 |
|---|---|
| 已有完整 Unity 工程 | 只把脚本复制到 `Assets/Scripts/Universal2D`，不要替换原有 `Assets` |
| 只有素材、没有场景 | 新建 2D 工程，复制脚本，按第 11 节搭一个灰盒场景 |
| 什么都没有 | 直接解压仓库里的 `Unity2D_Full_Project.zip`，用 Unity Hub 打开 |

新建工程时选 **2D**。如果 Console 报 `Input` 相关错误，打开 `Project Settings > Player > Active Input Handling`，改成 **Both**，Unity 会重启。

---

## 0.5 Unity 面板地图

先认清这几个面板，后面所有“挂脚本、改数值、拖引用”都在这里完成。

| 面板 | 位置 | 用途 |
|---|---|---|
| Hierarchy | 左侧 | 新建、选中、父子物体；改名；复制 |
| Scene | 中间 | 拖动、缩放、旋转物体；点左上角 `2D` 切 2D 视图 |
| Inspector | 右侧 | 挂脚本、改字段、设 Tag/Layer、调 Collider |
| Project | 底部 | 放脚本、Prefab、Sprite、场景 |
| Console | 底部标签 | 看红色报错；点 Clear 清空 |
| Game | 中间顶部 | 按 Play 后的运行画面 |

一次点击路径：

1. 新建空物体：Hierarchy 空白处右键 > `Create Empty`。
2. 新建方块：Hierarchy 右键 > `2D Object > Sprites > Square`。
3. 挂脚本：选中物体 > Inspector 最下方 `Add Component` > 输入脚本名 > 回车。
4. 改字段：Inspector 里脚本标题下面的输入框；有 `Rules`、`UI` 这类分组时，点左边三角展开。
5. 拖引用：从 Hierarchy 或 Project 把物体/资源拖到 Inspector 的槽位（显示 `None (Transform)` 的框）。
6. 新建 Tag：Inspector 顶部 `Tag` 下拉 > `Add Tag...` > `Tags` 列表的 `+` > 输入 `Player` > `Save`。
7. 新建 Layer：Inspector 顶部 `Layer` 下拉 > `Add Layer...` > `User Layer 6` 输入 `Ground` > 回车；回到物体把 `Layer` 选 `Ground`。
8. 存 Prefab：把 Hierarchy 里的物体拖到 Project 的 `Assets/Prefabs` 文件夹。
9. 复制物体：选中后 `Ctrl+D`。
10. 保存场景：`Ctrl+S`；标题栏场景名没有 `*` 才算保存。
11. 加入构建：`File > Build Settings > Add Open Scenes`，勾选当前场景。

Inspector 里最容易找不到的三样东西：

- `Tag` 和 `Layer` 在 Inspector **最顶部**，不在脚本组件里。
- `Add Component` 在 Inspector **最下面**，选中物体才出现。
- 脚本字段被折叠时只显示标题，点标题左边的三角展开。

## 0.6 新建 Player 标签和 Ground 层

很多模板靠 `Player` 标签判断“谁吃金币、谁受伤”，靠 `Ground` 层判断“能不能跳”。先建好这两个，后面不用反复改。

新建 `Player` 标签：

1. Hierarchy 选中玩家物体。
2. Inspector 顶部 `Tag` 下拉 > `Add Tag...`。
3. 在 Tags 列表底部点 `+`，输入 `Player`，点 `Save`。
4. 回到玩家物体，`Tag` 下拉选 `Player`。

新建 `Ground` 层：

1. 选中地面物体。
2. Inspector 顶部 `Layer` 下拉 > `Add Layer...`。
3. 在 `User Layer 6` 输入 `Ground`，回车。
4. 回到地面物体，`Layer` 下拉选 `Ground`。
5. 选中玩家，`PlatformerPlayer2D > Ground Mask` 点开，只勾 `Ground`；如果暂时找不到，先选 `Everything`，跑通后再改。

注意：Tag 是“身份”，Layer 是“物理分类”。玩家要 Tag `Player`，地面要 Layer `Ground`，两个不是同一个东西。

## 1. 最小可玩结构

一个能交差的 2D 小游戏，先只搭这 5 个对象：

```text
GameFlow          空物体，挂 GameFlow2D
Player            玩家，挂玩家脚本 + Rigidbody2D + Collider2D
Main Camera       挂 FollowCamera2D
Pickup            金币/星星，挂 Pickup2D + Collider2D（Is Trigger）
Hazard / Goal     尖刺/敌人/终点，挂 Hazard2D 或 Goal2D
```

层级示例：

```text
SampleScene
├── GameFlow
├── Player
├── Main Camera
├── Ground
├── Pickup (1..5)
├── Hazard
└── Goal
```

先把“能开始、能移动、能得分、能失败、能胜利、能重开”跑通，再换美术。

---

## 2. 通用规则层：GameFlow2D.cs

负责开始、计时、分数、生命、胜负和重开。所有玩法脚本都通过它记分和结束游戏。

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

namespace Universal2D
{
    public enum GameState { Ready, Playing, Won, Lost }

    [DefaultExecutionOrder(-100)]
    public class GameFlow2D : MonoBehaviour
    {
        public static GameFlow2D I { get; private set; }

        [Header("Rules")]
        public float timeLimit = 90f;
        public int targetScore = 5;
        public int lives = 3;

        [Header("Respawn")]
        public Transform respawnPoint;

        [Header("UI")]
        public bool showBuiltInUI = true;
        public KeyCode restartKey = KeyCode.R;

        public GameState State { get; private set; } = GameState.Ready;
        public int Score { get; private set; }
        public int Lives => lives;
        public float TimeLeft { get; private set; }

        void Awake()
        {
            if (I != null && I != this) { Destroy(gameObject); return; }
            I = this;
            TimeLeft = timeLimit;
            Time.timeScale = 0f;
        }

        void OnDestroy()
        {
            if (I == this) I = null;
        }

        void Update()
        {
            if (State == GameState.Ready)
            {
                if (Input.GetKeyDown(KeyCode.Space) ||
                    Input.GetKeyDown(KeyCode.Return) ||
                    Input.GetMouseButtonDown(0))
                    Begin();
                return;
            }

            if (State == GameState.Playing)
            {
                TimeLeft -= Time.deltaTime;
                if (timeLimit > 0f && TimeLeft <= 0f) Lose();
            }

            if (Input.GetKeyDown(restartKey)) Restart();
        }

        public void Begin()
        {
            if (State != GameState.Ready) return;
            State = GameState.Playing;
            TimeLeft = timeLimit;
            Time.timeScale = 1f;
        }

        public void AddScore(int amount = 1)
        {
            if (State != GameState.Playing) return;
            Score += amount;
            if (targetScore > 0 && Score >= targetScore) Win();
        }

        public void TakeDamage(int amount = 1)
        {
            if (State != GameState.Playing) return;
            lives -= amount;
            if (lives <= 0) { Lose(); return; }

            var player = GameObject.FindGameObjectWithTag("Player");
            if (player != null && respawnPoint != null)
            {
                player.transform.position = respawnPoint.position;
                var rb = player.GetComponent<Rigidbody2D>();
                if (rb != null) Rb2D.Set(rb, Vector2.zero);
            }
        }

        public void Win()
        {
            if (State != GameState.Playing) return;
            State = GameState.Won;
            Time.timeScale = 0f;
        }

        public void Lose()
        {
            if (State != GameState.Playing) return;
            State = GameState.Lost;
            Time.timeScale = 0f;
        }

        public void Restart()
        {
            Time.timeScale = 1f;
            SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex);
        }

        void OnGUI()
        {
            if (!showBuiltInUI) return;

            var hud = new GUIStyle(GUI.skin.label)
            {
                fontSize = 22,
                fontStyle = FontStyle.Bold
            };
            hud.normal.textColor = Color.white;

            GUI.Label(new Rect(16, 12, 620, 34),
                $"SCORE {Score}/{targetScore}    LIVES {lives}    TIME {Mathf.CeilToInt(TimeLeft)}",
                hud);

            var center = new GUIStyle(hud)
            {
                alignment = TextAnchor.MiddleCenter,
                fontSize = 36
            };

            if (State == GameState.Ready)
                GUI.Label(new Rect(0, Screen.height / 2 - 50, Screen.width, 100),
                    "PRESS SPACE / CLICK TO START", center);
            else if (State == GameState.Won)
                GUI.Label(new Rect(0, Screen.height / 2 - 80, Screen.width, 160),
                    "YOU WIN\nPRESS R TO RESTART", center);
            else if (State == GameState.Lost)
                GUI.Label(new Rect(0, Screen.height / 2 - 80, Screen.width, 160),
                    "GAME OVER\nPRESS R TO RESTART", center);
        }
    }
}
```

挂法：

1. 新建空物体，命名 `GameFlow`。
2. 挂 `GameFlow2D`。
3. `Target Score` 设 5，`Lives` 设 3，`Time Limit` 设 90。
4. 把玩家出生点拖到 `Respawn Point`，没有就留空。

---

## 3. Unity 6 速度兼容：Rb2D.cs

Unity 6 把 `Rigidbody2D.velocity` 改名为 `linearVelocity`。这个工具类让同一套代码在 2022 和 Unity 6 都能编译。

```csharp
using UnityEngine;

namespace Universal2D
{
    public static class Rb2D
    {
        public static Vector2 Get(Rigidbody2D rb)
        {
#if UNITY_6000_0_OR_NEWER
            return rb.linearVelocity;
#else
            return rb.velocity;
#endif
        }

        public static void Set(Rigidbody2D rb, Vector2 value)
        {
#if UNITY_6000_0_OR_NEWER
            rb.linearVelocity = value;
#else
            rb.velocity = value;
#endif
        }
    }
}
```

---

## 4. 三种玩家控制

按题型只选一个，不要同时挂两个移动脚本。

### 4.1 平台跳跃：PlatformerPlayer2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D), typeof(Collider2D))]
    public class PlatformerPlayer2D : MonoBehaviour
    {
        public float moveSpeed = 6f;
        public float jumpForce = 11f;
        public Transform groundCheck;
        public float groundRadius = 0.16f;
        public LayerMask groundMask = ~0;

        Rigidbody2D rb;
        bool grounded;
        float move;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.gravityScale = 2.5f;
            rb.freezeRotation = true;

            if (groundCheck == null)
            {
                var go = new GameObject("GroundCheck");
                go.transform.SetParent(transform);
                go.transform.localPosition = new Vector3(0f, -0.5f, 0f);
                groundCheck = go.transform;
            }
        }

        void Update()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing)
            {
                move = 0f;
                return;
            }

            move = 0f;
            if (Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.LeftArrow)) move -= 1f;
            if (Input.GetKey(KeyCode.D) || Input.GetKey(KeyCode.RightArrow)) move += 1f;

            if (Input.GetKeyDown(KeyCode.Space) && grounded)
                Rb2D.Set(rb, new Vector2(Rb2D.Get(rb).x, jumpForce));
        }

        void FixedUpdate()
        {
            var hits = Physics2D.OverlapCircleAll(groundCheck.position, groundRadius, groundMask);
            grounded = false;
            foreach (var hit in hits)
            {
                if (hit.attachedRigidbody != rb) { grounded = true; break; }
            }

            Rb2D.Set(rb, new Vector2(move * moveSpeed, Rb2D.Get(rb).y));
        }
    }
}
```

挂法：

1. 玩家物体挂 `Rigidbody2D`，`Gravity Scale` 设 2.5，`Freeze Rotation Z` 勾上。
2. 挂 `BoxCollider2D` 或 `CapsuleCollider2D`。
3. 挂 `PlatformerPlayer2D`。
4. `Ground Check` 留空会自动在脚底生成；也可以手动放一个空物体在脚底。
5. 地面物体设 `Ground` 层；玩家的 `Ground Mask` 选 `Ground`（或先选 Everything）。
6. 玩家物体 Tag 设 `Player`。

### 4.2 俯视移动：TopDownPlayer2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D))]
    public class TopDownPlayer2D : MonoBehaviour
    {
        public float moveSpeed = 5f;
        public bool faceMoveDirection = true;

        Rigidbody2D rb;
        Vector2 input;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.gravityScale = 0f;
            rb.freezeRotation = true;
        }

        void Update()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing)
            {
                input = Vector2.zero;
                return;
            }

            input = new Vector2(Input.GetAxisRaw("Horizontal"), Input.GetAxisRaw("Vertical"));
            if (input.sqrMagnitude > 1f) input.Normalize();

            if (faceMoveDirection && input.sqrMagnitude > 0.01f)
            {
                float angle = Mathf.Atan2(input.y, input.x) * Mathf.Rad2Deg;
                transform.rotation = Quaternion.Euler(0f, 0f, angle - 90f);
            }
        }

        void FixedUpdate()
        {
            Rb2D.Set(rb, input * moveSpeed);
        }
    }
}
```

挂法：

1. 玩家挂 `Rigidbody2D`，`Gravity Scale` 设 0，`Freeze Rotation Z` 勾上。
2. 挂 `CircleCollider2D` 或 `BoxCollider2D`。
3. 挂 `TopDownPlayer2D`。
4. 如果素材默认朝上，保留 `Face Move Direction`；如果素材朝右，取消勾选并自己在 Sprite 上转向。

### 4.3 Flappy：FlappyPlayer2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D))]
    public class FlappyPlayer2D : MonoBehaviour
    {
        public float autoRunSpeed = 3f;
        public float flapVelocity = 6f;

        Rigidbody2D rb;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.gravityScale = 2.2f;
            rb.freezeRotation = true;
        }

        void Update()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing) return;

            if (Input.GetKeyDown(KeyCode.Space) || Input.GetMouseButtonDown(0))
                Rb2D.Set(rb, new Vector2(autoRunSpeed, flapVelocity));
        }

        void FixedUpdate()
        {
            Rb2D.Set(rb, new Vector2(autoRunSpeed, Rb2D.Get(rb).y));
        }
    }
}
```

挂法：

1. 玩家挂 `Rigidbody2D`，`Gravity Scale` 设 2.2。
2. 挂 `CircleCollider2D`。
3. 挂 `FlappyPlayer2D`。
4. 管道用 `Spawner2D` 生成，或手动摆几根，挂 `DestroyBelow2D` 和 `Hazard2D`。

---

## 5. 相机：FollowCamera2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class FollowCamera2D : MonoBehaviour
    {
        public Transform target;
        public Vector3 offset = new Vector3(0f, 1f, -10f);
        public float smooth = 6f;
        public bool clampToBounds;
        public Vector2 min = new Vector2(-20f, -10f);
        public Vector2 max = new Vector2(20f, 10f);

        void LateUpdate()
        {
            if (target == null)
            {
                var player = GameObject.FindGameObjectWithTag("Player");
                if (player != null) target = player.transform;
            }
            if (target == null) return;

            var want = target.position + offset;
            if (clampToBounds)
                want = new Vector3(
                    Mathf.Clamp(want.x, min.x, max.x),
                    Mathf.Clamp(want.y, min.y, max.y),
                    want.z);

            float t = 1f - Mathf.Exp(-smooth * Time.unscaledDeltaTime);
            transform.position = Vector3.Lerp(transform.position, want, t);
        }
    }
}
```

挂法：`Main Camera` 挂 `FollowCamera2D`，`Target` 拖玩家；不拖会自动找 `Player` 标签。

---

## 6. 拾取、伤害、终点

### 6.1 拾取：Pickup2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Collider2D))]
    public class Pickup2D : MonoBehaviour
    {
        public static int Alive { get; private set; }

        public int score = 1;
        public bool winWhenAllCollected = true;

        void Awake()
        {
            GetComponent<Collider2D>().isTrigger = true;
        }

        void OnEnable() { Alive++; }
        void OnDisable() { Alive--; }

        void OnTriggerEnter2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;

            if (GameFlow2D.I != null)
            {
                GameFlow2D.I.AddScore(score);
                if (winWhenAllCollected && Alive <= 1) GameFlow2D.I.Win();
            }

            Destroy(gameObject);
        }
    }
}
```

挂法：金币/星星挂 `Pickup2D` + `CircleCollider2D`，勾 `Is Trigger`；玩家 Tag 设 `Player`。

### 6.2 伤害：Hazard2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class Hazard2D : MonoBehaviour
    {
        public int damage = 1;
        public bool destroySelf;

        float nextHit;

        void OnTriggerEnter2D(Collider2D other) { Hit(other); }
        void OnCollisionEnter2D(Collision2D collision) { Hit(collision.collider); }

        void Hit(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            if (Time.unscaledTime < nextHit) return;
            nextHit = Time.unscaledTime + 0.6f;

            if (GameFlow2D.I != null) GameFlow2D.I.TakeDamage(damage);
            if (destroySelf) Destroy(gameObject);
        }
    }
}
```

挂法：尖刺/敌人/掉落物挂 `Hazard2D`；静态伤害勾 `Is Trigger`，实体碰撞不勾。

### 6.3 终点：Goal2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Collider2D))]
    public class Goal2D : MonoBehaviour
    {
        void Awake()
        {
            GetComponent<Collider2D>().isTrigger = true;
        }

        void OnTriggerEnter2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            if (GameFlow2D.I != null) GameFlow2D.I.Win();
        }
    }
}
```

挂法：终点门/宝箱挂 `BoxCollider2D`，勾 `Is Trigger`，再挂 `Goal2D`。

---

## 7. 敌人与刷怪

### 7.1 巡逻敌人：PatrolEnemy2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D))]
    public class PatrolEnemy2D : MonoBehaviour
    {
        public float speed = 2f;
        public float leftX = -3f;
        public float rightX = 3f;
        public int hp = 1;
        public int score = 1;
        public int damage = 1;

        Rigidbody2D rb;
        int dir = 1;
        float nextHit;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.gravityScale = 0f;
            rb.freezeRotation = true;
        }

        void FixedUpdate()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing)
            {
                Rb2D.Set(rb, Vector2.zero);
                return;
            }

            if (transform.position.x <= leftX) dir = 1;
            else if (transform.position.x >= rightX) dir = -1;

            Rb2D.Set(rb, new Vector2(dir * speed, Rb2D.Get(rb).y));
        }

        public void Hit(int amount)
        {
            hp -= amount;
            if (hp > 0) return;
            if (GameFlow2D.I != null) GameFlow2D.I.AddScore(score);
            Destroy(gameObject);
        }

        void OnTriggerEnter2D(Collider2D other) { Damage(other); }
        void OnCollisionEnter2D(Collision2D collision) { Damage(collision.collider); }

        void Damage(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            if (Time.unscaledTime < nextHit) return;
            nextHit = Time.unscaledTime + 0.6f;
            if (GameFlow2D.I != null) GameFlow2D.I.TakeDamage(damage);
        }
    }
}
```

挂法：

1. 敌人挂 `Rigidbody2D`（Gravity Scale 0）+ `BoxCollider2D`。
2. 挂 `PatrolEnemy2D`。
3. `Left X` / `Right X` 设巡逻范围的世界坐标。
4. 敌人 Tag 不要设 `Player`。

### 7.2 定时刷怪：Spawner2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class Spawner2D : MonoBehaviour
    {
        public GameObject prefab;
        public float interval = 1.5f;
        public float minX = -8f;
        public float maxX = 8f;
        public float spawnY = 6f;
        public bool parentToSpawner;

        float timer;

        void Update()
        {
            if (prefab == null) return;
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing) return;

            timer -= Time.deltaTime;
            if (timer > 0f) return;
            timer = Mathf.Max(0.1f, interval);

            var pos = new Vector3(Random.Range(minX, maxX), spawnY, 0f);
            Instantiate(prefab, pos, Quaternion.identity, parentToSpawner ? transform : null);
        }
    }
}
```

挂法：空物体挂 `Spawner2D`，`Prefab` 拖敌人/管道；`Min X` 要小于 `Max X`。

### 7.3 掉出屏幕销毁：DestroyBelow2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class DestroyBelow2D : MonoBehaviour
    {
        public float y = -8f;

        void Update()
        {
            if (transform.position.y < y) Destroy(gameObject);
        }
    }
}
```

挂法：管道、掉落物、飞出屏幕的子弹都可以挂，`Y` 设摄像机下方。

---

### 7.4 向左移动：MoveLeft2D.cs

Flappy 的管道、跑酷的障碍都可以用这个脚本持续向左移动。

```csharp
using UnityEngine;

namespace Universal2D
{
    public class MoveLeft2D : MonoBehaviour
    {
        public float speed = 3f;

        void Update()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing) return;
            transform.position += Vector3.left * speed * Time.deltaTime;
        }
    }
}
```

挂法：管道 Prefab 根物体挂 `MoveLeft2D`，`Speed` 设 3；再挂 `DestroyBelow2D`，`Y` 设 -8。

## 8. 射击

### 8.1 自动射击：AutoShooter2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class AutoShooter2D : MonoBehaviour
    {
        public GameObject bulletPrefab;
        public float interval = 0.25f;
        public float bulletSpeed = 12f;
        public bool autoFire = true;

        static Sprite whiteSprite;
        float timer;

        void Update()
        {
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing) return;

            timer -= Time.deltaTime;
            bool wantFire = autoFire || Input.GetKey(KeyCode.Space);
            if (!wantFire || timer > 0f) return;
            timer = Mathf.Max(0.05f, interval);
            Fire();
        }

        void Fire()
        {
            var pos = transform.position + transform.up * 0.6f;

            if (bulletPrefab != null)
            {
                var bullet = Instantiate(bulletPrefab, pos, transform.rotation);
                var rb = bullet.GetComponent<Rigidbody2D>();
                if (rb != null) Rb2D.Set(rb, transform.up * bulletSpeed);
                return;
            }

            var go = new GameObject("Bullet");
            go.transform.position = pos;
            go.transform.rotation = transform.rotation;
            go.transform.localScale = Vector3.one * 0.25f;

            if (whiteSprite == null)
                whiteSprite = Sprite.Create(Texture2D.whiteTexture, new Rect(0f, 0f, 1f, 1f),
                    new Vector2(0.5f, 0.5f), 1f);

            var sr = go.AddComponent<SpriteRenderer>();
            sr.sprite = whiteSprite;
            sr.color = Color.yellow;
            sr.sortingOrder = 10;

            var col = go.AddComponent<CircleCollider2D>();
            col.isTrigger = true;
            col.radius = 0.5f;

            var rb2 = go.AddComponent<Rigidbody2D>();
            rb2.gravityScale = 0f;
            Rb2D.Set(rb2, transform.up * bulletSpeed);

            go.AddComponent<Bullet2D>();
        }
    }
}
```

挂法：

1. 玩家挂 `AutoShooter2D`。
2. `Bullet Prefab` 可以留空，脚本会生成黄色方块子弹。
3. 玩家 Sprite 的“正前方”要朝上；俯视玩家用 `TopDownPlayer2D` 的 `Face Move Direction` 自动转向。

### 8.2 子弹：Bullet2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class Bullet2D : MonoBehaviour
    {
        public float life = 2f;
        public int damage = 1;

        void Start()
        {
            Destroy(gameObject, life);
        }

        void OnTriggerEnter2D(Collider2D other)
        {
            if (other.CompareTag("Player")) return;

            var enemy = other.GetComponentInParent<PatrolEnemy2D>();
            if (enemy != null) enemy.Hit(damage);

            var brick = other.GetComponentInParent<Brick2D>();
            if (brick != null) brick.Hit();

            Destroy(gameObject);
        }
    }
}
```

如果用自己的子弹 Prefab，Prefab 需要 `Rigidbody2D`（Gravity Scale 0）+ `Collider2D`（Is Trigger）+ `Bullet2D`。

---

## 9. 弹射：Slingshot2D.cs

适合愤怒的小鸟类：鼠标按住鸟往后拖，松开发射。

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D))]
    public class Slingshot2D : MonoBehaviour
    {
        public float maxDrag = 2.5f;
        public float launchPower = 8f;
        public LineRenderer line;

        Rigidbody2D rb;
        Collider2D col;
        Camera cam;
        bool dragging;
        Vector3 startPos;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            col = GetComponent<Collider2D>();
            cam = Camera.main;
            rb.gravityScale = 0f;
        }

        void Update()
        {
            if (cam == null) cam = Camera.main;
            if (cam == null) return;
            if (GameFlow2D.I != null && GameFlow2D.I.State != GameState.Playing) return;

            if (Input.GetMouseButtonDown(0))
            {
                var world = cam.ScreenToWorldPoint(Input.mousePosition);
                if (Vector2.Distance(world, transform.position) < 1.3f)
                {
                    dragging = true;
                    startPos = transform.position;
                    Rb2D.Set(rb, Vector2.zero);
                    rb.bodyType = RigidbodyType2D.Kinematic;
                    if (col != null) col.enabled = false;
                }
            }

            if (dragging && Input.GetMouseButton(0))
            {
                var world = cam.ScreenToWorldPoint(Input.mousePosition);
                world.z = 0f;
                var delta = (Vector2)(world - startPos);
                delta = Vector2.ClampMagnitude(delta, maxDrag);
                transform.position = startPos + (Vector3)delta;
                DrawLine();
            }

            if (dragging && Input.GetMouseButtonUp(0))
            {
                dragging = false;
                rb.bodyType = RigidbodyType2D.Dynamic;
                rb.gravityScale = 2.5f;
                if (col != null) col.enabled = true;

                var launch = (startPos - transform.position) * launchPower;
                Rb2D.Set(rb, launch);
                if (line != null) line.positionCount = 0;
            }
        }

        void DrawLine()
        {
            if (line == null) return;
            line.positionCount = 2;
            line.SetPosition(0, startPos);
            line.SetPosition(1, transform.position);
        }
    }
}
```

挂法：

1. 鸟挂 `Rigidbody2D`（Gravity Scale 2.5）+ `CircleCollider2D`。
2. 挂 `Slingshot2D`。
3. 可选：给鸟加 `LineRenderer`，拖到 `Line` 字段，宽度 0.05。
4. 猪挂 `PatrolEnemy2D` 或 `Hazard2D`，木块挂 `Brick2D`。
5. 相机要带 `MainCamera` 标签。

---

## 10. 打地鼠：ClickTarget2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Collider2D))]
    public class ClickTarget2D : MonoBehaviour
    {
        public int score = 10;
        public float hideTime = 1f;
        public bool randomRespawn = true;
        public Vector2 randomRange = new Vector2(3f, 2f);

        Vector3 home;
        SpriteRenderer sr;
        Collider2D col;
        float timer;
        bool hidden;

        void Awake()
        {
            home = transform.position;
            sr = GetComponent<SpriteRenderer>();
            col = GetComponent<Collider2D>();
        }

        void Update()
        {
            if (!hidden) return;
            timer -= Time.deltaTime;
            if (timer > 0f) return;

            hidden = false;
            if (sr != null) sr.enabled = true;
            if (col != null) col.enabled = true;

            if (randomRespawn)
                transform.position = home + new Vector3(
                    Random.Range(-randomRange.x, randomRange.x),
                    Random.Range(-randomRange.y, randomRange.y),
                    0f);
        }

        void OnMouseDown()
        {
            if (hidden) return;

            if (GameFlow2D.I != null) GameFlow2D.I.AddScore(score);

            hidden = true;
            timer = hideTime;
            if (sr != null) sr.enabled = false;
            if (col != null) col.enabled = false;
        }
    }
}
```

挂法：

1. 地鼠挂 `SpriteRenderer` + `BoxCollider2D`。
2. 挂 `ClickTarget2D`。
3. `Score` 设 10，`Hide Time` 设 1。
4. 复制 6–9 个，摆在不同位置。

---

## 11. 打砖块

### 11.1 球：Ball2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    [RequireComponent(typeof(Rigidbody2D))]
    public class Ball2D : MonoBehaviour
    {
        public float speed = 7f;
        public Vector2 startDirection = new Vector2(0.4f, 1f);

        Rigidbody2D rb;

        void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.gravityScale = 0f;
        }

        void Start()
        {
            var dir = startDirection.sqrMagnitude < 0.01f ? Vector2.up : startDirection.normalized;
            Rb2D.Set(rb, dir * speed);
        }

        void FixedUpdate()
        {
            var v = Rb2D.Get(rb);
            if (v.sqrMagnitude < 0.01f) v = Vector2.up;
            Rb2D.Set(rb, v.normalized * speed);
        }

        void OnCollisionEnter2D(Collision2D collision)
        {
            var brick = collision.collider.GetComponentInParent<Brick2D>();
            if (brick != null) brick.Hit();
        }
    }
}
```

### 11.2 砖块：Brick2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class Brick2D : MonoBehaviour
    {
        public int hp = 1;
        public int score = 10;

        public void Hit()
        {
            hp--;
            if (hp > 0) return;
            if (GameFlow2D.I != null) GameFlow2D.I.AddScore(score);
            Destroy(gameObject);
        }
    }
}
```

### 11.3 球掉落复位：BallLost2D.cs

```csharp
using UnityEngine;

namespace Universal2D
{
    public class BallLost2D : MonoBehaviour
    {
        public float y = -6f;
        public Transform spawnPoint;

        void Update()
        {
            if (transform.position.y >= y) return;

            if (GameFlow2D.I != null) GameFlow2D.I.TakeDamage(1);
            transform.position = spawnPoint != null ? spawnPoint.position : Vector3.zero;

            var rb = GetComponent<Rigidbody2D>();
            if (rb != null) Rb2D.Set(rb, new Vector2(0.4f, 1f).normalized * 7f);
        }
    }
}
```

挂法：

1. 球挂 `Rigidbody2D`（Gravity Scale 0）+ `CircleCollider2D` + `Ball2D` + `BallLost2D`。
2. 砖块挂 `BoxCollider2D` + `Brick2D`，摆成 4×5。
3. 挡板挂 `BoxCollider2D` + `Rigidbody2D`（Kinematic），用键盘控制 `transform.position.x`。
4. 底部放一个空物体 `SpawnPoint`，拖到 `BallLost2D`。

挡板最小代码：

```csharp
using UnityEngine;

namespace Universal2D
{
    public class Paddle2D : MonoBehaviour
    {
        public float speed = 8f;
        public float minX = -7f;
        public float maxX = 7f;

        void Update()
        {
            float h = Input.GetAxisRaw("Horizontal");
            var p = transform.position;
            p.x = Mathf.Clamp(p.x + h * speed * Time.deltaTime, minX, maxX);
            transform.position = p;
        }
    }
}
```

---

## 12. 按题型搭场景

### 12.1 平台收集（最稳的保底）

1. 新建 2D 场景。
2. `GameFlow` 空物体挂 `GameFlow2D`，`Target Score` 设 5。
3. 玩家挂 `Rigidbody2D` + `BoxCollider2D` + `PlatformerPlayer2D`，Tag 设 `Player`。
4. 地面用 `Cube` 压扁，或 `Sprite` + `BoxCollider2D`，Layer 设 `Ground`。
5. 放 5 个金币，挂 `Pickup2D` + `CircleCollider2D`（Is Trigger）。
6. 放 1 个尖刺，挂 `Hazard2D` + `BoxCollider2D`（Is Trigger）。
7. `Main Camera` 挂 `FollowCamera2D`，`Target` 拖玩家。
8. 按 Play，点 START，吃满 5 个金币出现 `YOU WIN`。

### 12.2 俯视射击

1. 玩家挂 `Rigidbody2D`（Gravity 0）+ `CircleCollider2D` + `TopDownPlayer2D` + `AutoShooter2D`。
2. 敌人 Prefab：`Rigidbody2D`（Gravity 0）+ `CircleCollider2D` + `PatrolEnemy2D`。
3. 空物体挂 `Spawner2D`，`Prefab` 拖敌人，`Interval` 设 1.2。
4. 相机挂 `FollowCamera2D`。
5. 玩家撞敌人扣命；子弹打死敌人加分；分数到 `Target Score` 获胜。

### 12.3 Flappy

1. 玩家挂 `Rigidbody2D`（Gravity 2.2）+ `CircleCollider2D` + `FlappyPlayer2D`。
2. 管道 Prefab：上下两个 `BoxCollider2D` + `Hazard2D` + `DestroyBelow2D`。
3. 空物体挂 `Spawner2D`，`Prefab` 拖管道，`Interval` 设 1.8。
4. 相机固定，不挂跟随；或挂 `FollowCamera2D` 只跟 Y。
5. 穿过一根管道加 1 分：给管道加一个 Trigger 子物体挂 `Pickup2D`，`Score` 设 1。

### 12.4 弹射

1. 鸟挂 `Rigidbody2D`（Gravity 2.5）+ `CircleCollider2D` + `Slingshot2D`。
2. 猪挂 `Rigidbody2D` + `CircleCollider2D` + `PatrolEnemy2D`（`Hp` 设 1）。
3. 木块挂 `BoxCollider2D` + `Brick2D`。
4. 相机挂 `FollowCamera2D`，`Target` 拖鸟。
5. 拖鸟向后，松手；猪死光或分数到目标获胜。

### 12.5 打地鼠

1. 背景放一张全屏 Sprite。
2. 地鼠挂 `SpriteRenderer` + `BoxCollider2D` + `ClickTarget2D`。
3. 复制 6–9 个，`Score` 设 10。
4. `GameFlow2D.Target Score` 设 50，`Time Limit` 设 30。
5. 鼠标点击地鼠，分数到 50 获胜。

### 12.6 打砖块

1. 球挂 `Rigidbody2D`（Gravity 0）+ `CircleCollider2D` + `Ball2D` + `BallLost2D`。
2. 砖块挂 `BoxCollider2D` + `Brick2D`，摆 4 行 5 列。
3. 挡板挂 `BoxCollider2D` + `Rigidbody2D`（Kinematic）+ `Paddle2D`。
4. 底部放 `SpawnPoint`，拖到 `BallLost2D`。
5. `Target Score` 设 200（20 块 × 10 分）。

---

## 13. 30 分钟保底流程

| 时间 | 做什么 | 通过标准 |
|---|---|---|
| 00–03 | 新建/打开 2D 工程，复制 `Rb2D.cs` 和 `GameFlow2D.cs` | Console 无红字 |
| 03–08 | 选一个玩家脚本，搭地面和玩家 | 按 Play 能移动/跳跃 |
| 08–15 | 加拾取、伤害、终点 | 能得分、能扣命、能胜利 |
| 15–22 | 换素材、调碰撞体 | 画面像题目要求 |
| 22–27 | 加敌人/刷怪/子弹 | 有失败压力 |
| 27–30 | 保存场景，加入 Build Settings | 场景名无星号 |

时间不够时的优先级：

1. 能开始。
2. 能操作。
3. 能得分。
4. 能失败。
5. 能胜利。
6. 能重开。
7. 最后才是美术和音效。

---

## 14. 常见报错

| 现象 | 原因 | 处理 |
|---|---|---|
| Console 红色 CS 错误 | 脚本没编译，Play 不可信 | 先修第一条红色错误，点 Clear 再等编译 |
| 玩家不动 | 没有 `Rigidbody2D` / `Collider2D`，或两个移动脚本抢速度 | 只保留一个移动脚本 |
| 跳不起来 | `Ground Check` 不在脚底，或 `Ground Mask` 不对 | 把 GroundCheck 移到脚底，Mask 先设 Everything |
| 吃不到金币 | Collider 没勾 `Is Trigger` | 金币 Collider 勾 Is Trigger |
| 点击没反应 | 没有 Collider，或场景没有 EventSystem | 打地鼠用 `OnMouseDown`，只要有 Collider 即可；UI 按钮才需要 EventSystem |
| 键盘没反应 | Active Input Handling 配置不对 | Player Settings 改成 Both，重启 Unity |
| Unity 6 报 velocity 过时 | API 改名 | 用本文的 `Rb2D.Get/Set` |
| 重开后分数没清零 | 用了 `DontDestroyOnLoad` 或静态变量 | 不要给 GameFlow 加 DontDestroyOnLoad |

---

## 15. 交卷前 60 秒

- [ ] Play 后出现 START，点击或按 Enter 能开始
- [ ] WASD/方向键能移动、跳跃或发射
- [ ] 拾取一个道具，SCORE 增加
- [ ] 碰敌人/尖刺，LIVES 减少，归零出现 GAME OVER
- [ ] 吃满道具或到终点，出现 YOU WIN
- [ ] 点 RESTART 或按 R，场景恢复
- [ ] 场景标题栏没有星号
- [ ] 当前场景在 `File > Build Settings > Scenes In Build` 并勾选

---

## 16. 文件清单

把下面这些脚本放进 `Assets/Scripts/Universal2D/`：

```text
Rb2D.cs
GameFlow2D.cs
PlatformerPlayer2D.cs
TopDownPlayer2D.cs
FlappyPlayer2D.cs
FollowCamera2D.cs
Pickup2D.cs
Hazard2D.cs
Goal2D.cs
PatrolEnemy2D.cs
Spawner2D.cs
DestroyBelow2D.cs
MoveLeft2D.cs
AutoShooter2D.cs
Bullet2D.cs
Slingshot2D.cs
ClickTarget2D.cs
Ball2D.cs
Brick2D.cs
BallLost2D.cs
Paddle2D.cs
```

只想快速跑通的话，最小组合是：

```text
平台收集：Rb2D + GameFlow2D + PlatformerPlayer2D + FollowCamera2D + Pickup2D + Hazard2D + Goal2D
俯视射击：Rb2D + GameFlow2D + TopDownPlayer2D + FollowCamera2D + AutoShooter2D + Bullet2D + PatrolEnemy2D
Flappy：Rb2D + GameFlow2D + FlappyPlayer2D + Spawner2D + MoveLeft2D + Hazard2D + DestroyBelow2D
```

---

## 17. 和仓库里的完整工程怎么选

- 想最快交差：直接用 `02_Full_Project/Unity2D_Full_Project.zip`，打开 `Assets/Scenes/Graybox.unity`。
- 已有工程：只导入 `01_Starter_Kit/Unity2D_Starter.unitypackage`，不要替换原有 `Assets`。
- 想自己写：复制本文脚本，按第 12 节搭场景。

模板的目标是让你把时间花在题目要求的核心玩法上，而不是重复写菜单、计分、胜负和重开。

---

## 18. 面板操作逐项清单

这一节按“选中哪个物体、点哪个按钮、每个框填什么”来写。第一次搭场景时照着做，不用猜 Inspector。

### 18.1 GameFlow：规则总开关

1. Hierarchy 空白处右键 > `Create Empty`，改名 `GameFlow`。
2. 选中 `GameFlow`，Inspector 右上角 `Transform` 点齿轮 > `Reset`，位置回到 `(0,0,0)`。
3. Inspector 最下方点 `Add Component`，输入 `GameFlow2D`，回车。
4. 在 `Game Flow 2D` 组件里填：

| 字段 | 设成 | 说明 |
|---|---|---|
| Time Limit | 90 | 倒计时秒数；填 0 表示不限时 |
| Target Score | 5 | 吃满 5 个道具获胜；填 0 表示只靠终点 |
| Lives | 3 | 生命数；归零 GAME OVER |
| Respawn Point | 拖 Player | 受伤后回到这里；没有就留空 |
| Show Built In UI | 勾选 | 用内置的分数/生命/胜负文字 |
| Restart Key | R | 按 R 重开 |

5. `GameFlow` 的 Tag 保持 `Untagged`，Layer 保持 `Default`。
6. 如果 `Respawn Point` 拖不进去，先确认 Player 已经创建；拖 Player 物体本身，不要拖它的子物体。

### 18.2 平台玩家：Player + PlatformerPlayer2D

1. Hierarchy 右键 > `2D Object > Sprites > Square`，改名 `Player`。
2. Inspector `Transform`：Position `(0,-2,0)`，Scale `(0.8,0.8,1)`。
3. Inspector 顶部 `Tag` 下拉，选 `Player`；没有就按第 0.6 节先新建。
4. `Layer` 保持 `Default`。
5. `Add Component` > `Rigidbody 2D`，填：

| 字段 | 设成 |
|---|---|
| Body Type | Dynamic |
| Gravity Scale | 2.5 |
| Collision Detection | Continuous |
| Interpolate | Interpolate |
| Constraints > Freeze Rotation Z | 勾选 |

6. `Add Component` > `Box Collider 2D`：`Size` 设 `(1,1)`，`Offset` 设 `(0,0)`，`Is Trigger` 不勾。
7. `Add Component` > `PlatformerPlayer2D`，填：

| 字段 | 设成 | 说明 |
|---|---|---|
| Move Speed | 6 | 左右速度 |
| Jump Force | 11 | 跳跃力度 |
| Ground Check | 留空 | 脚本自动在脚底生成 |
| Ground Radius | 0.16 | 脚底检测圈 |
| Ground Mask | Everything | 跑通后改成 Ground |

8. `Sprite Renderer`：`Color` 设蓝色，`Order in Layer` 设 0。
9. 做地面：Hierarchy 右键 > `2D Object > Sprites > Square`，改名 `Ground`，Position `(0,-4,0)`，Scale `(12,1,1)`；`Layer` 选 `Ground`；`Add Component` > `Box Collider 2D`，`Size` 保持 `(1,1)`。
10. 做相机：选中 `Main Camera`，Inspector 里 `Projection` 选 `Orthographic`，`Size` 设 5，Position `(0,0,-10)`；`Add Component` > `FollowCamera2D`，`Target` 拖 Player。
11. 按 Play：点 START，用 A/D 或方向键移动，空格跳。跳不起来就看 `Ground Check` 是否在脚底、`Ground Mask` 是否包含 Ground。

### 18.3 俯视玩家：TopDownPlayer2D

1. 复制 18.2 的 Player，或新建 `2D Object > Sprites > Square`。
2. `Tag` 设 `Player`。
3. `Rigidbody 2D`：`Body Type` Dynamic，`Gravity Scale` **0**，`Collision Detection` Continuous，`Freeze Rotation Z` 勾选。
4. `Circle Collider 2D`：`Radius` 0.4，`Is Trigger` 不勾。
5. `Add Component` > `TopDownPlayer2D`：`Move Speed` 5，`Face Move Direction` 勾选。
6. 相机 `Projection` Orthographic，`Size` 6。
7. 按 Play：WASD 八方向移动；玩家会朝移动方向旋转。

如果素材默认朝右而不是朝上：取消 `Face Move Direction`，把 Sprite 的初始旋转设成朝右，或把脚本里的 `angle - 90f` 改成 `angle`。

### 18.4 Flappy 玩家和管道

玩家：

1. 新建 `2D Object > Sprites > Square`，改名 `Player`，`Tag` 设 `Player`。
2. `Rigidbody 2D`：`Gravity Scale` 2.2，`Collision Detection` Continuous，`Freeze Rotation Z` 勾选。
3. `Circle Collider 2D`：`Radius` 0.3。
4. `Add Component` > `FlappyPlayer2D`：`Auto Run Speed` 3，`Flap Velocity` 6。
5. 相机 Orthographic，`Size` 5，Position `(0,0,-10)`，不挂跟随。

管道 Prefab：

1. Hierarchy 右键 > `Create Empty`，改名 `PipePair`，Position `(0,0,0)`。
2. 在 `PipePair` 下新建两个 `2D Object > Sprites > Square`，改名 `TopPipe`、`BottomPipe`。
3. `TopPipe`：Position `(0,3,0)`，Scale `(1.5,4,1)`；`BottomPipe`：Position `(0,-3,0)`，Scale `(1.5,4,1)`。
4. 两个管道都加 `Box Collider 2D`，`Is Trigger` 勾选；都加 `Hazard2D`，`Damage` 1。
5. 选中 `PipePair`，加 `MoveLeft2D`（`Speed` 3）和 `DestroyBelow2D`（`Y` -8）。
6. 在两根管道中间新建空物体 `ScoreZone`，Position `(0,0,0)`，加 `Box Collider 2D`（`Size` `(1.2,2)`，`Is Trigger` 勾选）和 `Pickup2D`（`Score` 1，`Win When All Collected` 取消勾选）。
7. 把 `PipePair` 从 Hierarchy 拖到 Project 的 `Assets/Prefabs`，生成 Prefab；场景里的可以删掉。

刷管道：

1. Hierarchy 新建空物体 `PipeSpawner`，Position `(0,0,0)`。
2. `Add Component` > `Spawner2D`。
3. `Prefab` 拖 `PipePair`；`Interval` 1.8；`Min X` -8；`Max X` 8；`Spawn Y` 0；`Parent To Spawner` 不勾。
4. 按 Play：点 START 后管道从右边出现并向左移动；穿过间隙加 1 分。

### 18.5 金币/星星：Pickup2D

1. 新建 `2D Object > Sprites > Square`，改名 `Pickup`。
2. `Tag` 保持 `Untagged`。
3. `Add Component` > `Circle Collider 2D`：`Radius` 0.4，`Is Trigger` **勾选**。
4. `Add Component` > `Pickup2D`：`Score` 1，`Win When All Collected` 勾选。
5. `Sprite Renderer`：`Color` 黄色，`Order in Layer` 1。
6. 选中 `Pickup`，按 `Ctrl+D` 复制 4 个，一共 5 个；在 Scene 里拖到不同位置。
7. 想让金币变成 Prefab：把 `Pickup` 拖到 Project 的 `Assets/Prefabs`。

### 18.6 尖刺/伤害：Hazard2D

1. 新建 `2D Object > Sprites > Square`，改名 `Spike`。
2. `Transform`：Scale `(1,0.3,1)`，Position 放在地面上。
3. `Add Component` > `Box Collider 2D`：`Is Trigger` **勾选**（静态伤害）。
4. `Add Component` > `Hazard2D`：`Damage` 1，`Destroy Self` 不勾。
5. `Sprite Renderer`：`Color` 红色。
6. 如果要实体敌人碰撞（不勾 Is Trigger），敌人需要 `Rigidbody 2D`；静态尖刺不勾 Trigger 也可以，因为玩家自己有 Rigidbody。

### 18.7 终点：Goal2D

1. 新建 `2D Object > Sprites > Square`，改名 `Goal`。
2. `Transform`：Scale `(1,2,1)`，Position 放在关卡右侧。
3. `Add Component` > `Box Collider 2D`：`Is Trigger` **勾选**。
4. `Add Component` > `Goal2D`。
5. `Sprite Renderer`：`Color` 绿色。

### 18.8 巡逻敌人：PatrolEnemy2D

1. 新建 `2D Object > Sprites > Square`，改名 `Enemy`。
2. `Add Component` > `Rigidbody 2D`：`Body Type` Dynamic，`Gravity Scale` 0，`Collision Detection` Continuous，`Freeze Rotation Z` 勾选。
3. `Add Component` > `Box Collider 2D`：`Size` `(0.8,0.8)`，`Is Trigger` 不勾。
4. `Add Component` > `PatrolEnemy2D`，填：

| 字段 | 设成 | 说明 |
|---|---|---|
| Speed | 2 | 移动速度 |
| Left X | -3 | 左边界世界坐标 |
| Right X | 3 | 右边界世界坐标 |
| Hp | 1 | 被打几下死 |
| Score | 1 | 死亡加分 |
| Damage | 1 | 撞玩家扣几命 |

5. `Sprite Renderer`：`Color` 红色。
6. 如果敌人在 x=5，`Left X` 设 3、`Right X` 设 7；不要让 Left 大于 Right。
7. 把 `Enemy` 拖到 Project 的 `Assets/Prefabs`，给 `Spawner2D` 用。

### 18.9 刷怪：Spawner2D

1. Hierarchy 新建空物体 `Spawner`，Position `(0,6,0)`。
2. `Add Component` > `Spawner2D`。
3. `Prefab` 从 Project 拖 `Enemy`；`Interval` 1.2；`Min X` -8；`Max X` 8；`Spawn Y` 6；`Parent To Spawner` 不勾。
4. 如果生成物要跟着 Spawner 移动，才勾 `Parent To Spawner`；一般不要勾。
5. 按 Play，START 后每 1.2 秒生成一个敌人。

### 18.10 射击：AutoShooter2D + Bullet2D

1. 选中俯视玩家，`Add Component` > `AutoShooter2D`。
2. `Bullet Prefab` 可以留空，脚本会生成黄色方块子弹。
3. `Interval` 0.25；`Bullet Speed` 12；`Auto Fire` 勾选。
4. 玩家 Sprite 的“正前方”要朝上；`TopDownPlayer2D` 的 `Face Move Direction` 勾选后，玩家会朝移动方向，子弹沿玩家上方发射。

如果用自己的子弹 Prefab：

1. 新建 `2D Object > Sprites > Square`，改名 `Bullet`，Scale `(0.2,0.2,1)`。
2. `Circle Collider 2D`：`Radius` 0.5，`Is Trigger` 勾选。
3. `Rigidbody 2D`：`Gravity Scale` 0，`Body Type` Dynamic。
4. `Add Component` > `Bullet2D`：`Life` 2，`Damage` 1。
5. 拖到 Project 的 `Assets/Prefabs`，再拖到 `AutoShooter2D > Bullet Prefab`。

### 18.11 弹射：Slingshot2D

1. 新建 `2D Object > Sprites > Circle`，改名 `Bird`。
2. `Tag` 保持 `Untagged`；`Rigidbody 2D`：`Gravity Scale` 2.5，`Collision Detection` Continuous。
3. `Circle Collider 2D`：`Radius` 0.3。
4. `Add Component` > `Slingshot2D`：`Max Drag` 2.5，`Launch Power` 8。
5. 可选：`Add Component` > `Effects > Line Renderer`，`Width` 0.05，把组件拖到 `Slingshot2D > Line`。
6. 猪：新建 `2D Object > Sprites > Circle`，`Rigidbody 2D`（Gravity 0）+ `Circle Collider 2D` + `PatrolEnemy2D`（`Hp` 1，`Speed` 0）。
7. 木块：`2D Object > Sprites > Square`，`Box Collider 2D` + `Brick2D`。
8. 相机要 Tag `MainCamera`；`Camera.main` 找不到相机时，弹射脚本直接不工作。
9. 按 Play，START 后用鼠标按住鸟往后拖，松开发射。

### 18.12 打地鼠：ClickTarget2D

1. 新建 `2D Object > Sprites > Square`，改名 `Mole`，Scale `(0.8,0.8,1)`。
2. `Add Component` > `Box Collider 2D`：`Size` `(1,1)`，`Is Trigger` 不勾。
3. `Add Component` > `ClickTarget2D`：`Score` 10，`Hide Time` 1，`Random Respawn` 勾选，`Random Range` `(3,2)`。
4. `Sprite Renderer`：`Color` 棕色，`Order in Layer` 2。
5. `Ctrl+D` 复制 8 个，在 Scene 里摆成 3×3；每个的 Position 例如 `(-3,2,0)`、`(0,2,0)`、`(3,2,0)`，第二行 y=0，第三行 y=-2。
6. 按 Play，点 START，鼠标点地鼠加分；不用 Canvas，也不用 EventSystem。

### 18.13 打砖块：球、砖、挡板

球：

1. 新建 `2D Object > Sprites > Circle`，改名 `Ball`，Scale `(0.5,0.5,1)`。
2. `Rigidbody 2D`：`Gravity Scale` 0，`Collision Detection` Continuous。
3. `Circle Collider 2D`：`Radius` 0.5。
4. `Add Component` > `Ball2D`：`Speed` 7，`Start Direction` `(0.4,1)`。
5. `Add Component` > `BallLost2D`：`Y` -6，`Spawn Point` 拖场景里的 `SpawnPoint`。

砖块：

1. 新建 `2D Object > Sprites > Square`，改名 `Brick`，Scale `(1.4,0.5,1)`。
2. `Box Collider 2D`：`Size` `(1,1)`。
3. `Add Component` > `Brick2D`：`Hp` 1，`Score` 10。
4. 放到 Position `(-6,3,0)`，`Ctrl+D` 复制；第一行 x = -6、-3、0、3、6，y=3；第二行 y=2；第三行 y=1；第四行 y=0。

挡板：

1. 新建 `2D Object > Sprites > Square`，改名 `Paddle`，Scale `(2,0.3,1)`，Position `(0,-4,0)`。
2. `Rigidbody 2D`：`Body Type` **Kinematic**，`Gravity Scale` 0。
3. `Box Collider 2D`：`Size` `(1,1)`。
4. `Add Component` > `Paddle2D`：`Speed` 8，`Min X` -7，`Max X` 7。

出生点：

1. Hierarchy 新建空物体 `SpawnPoint`，Position `(0,-3.5,0)`。
2. 选中 `Ball`，把 `SpawnPoint` 拖到 `BallLost2D > Spawn Point`。
3. 相机 `Size` 设 6。

### 18.14 Canvas 按钮（可选）

模板默认用 `GameFlow2D` 的 `OnGUI` 画分数和胜负，不需要 Canvas。如果题目要求按钮，按下面做：

1. Hierarchy 右键 > `UI > Canvas`；Canvas 的 `Canvas Scaler` 选 `Scale With Screen Size`，Reference Resolution 设 `1920 × 1080`。
2. Hierarchy 右键 > `UI > Button - TextMeshPro`；如果弹出 TMP 导入窗口，点 `Import TMP Essentials`。
3. 选中 Button，Inspector `Rect Transform`：`Anchor` 选底部居中，`Pos Y` 60，`Width` 220，`Height` 60。
4. Button 的 `OnClick` 点 `+`，把 `GameFlow` 从 Hierarchy 拖到对象槽，下拉选 `GameFlow2D > Begin`。
5. 再做一个 Button，`OnClick` 选 `GameFlow2D > Restart`。
6. Canvas 下会自动生成 `EventSystem`；没有就右键 > `UI > Event System`。
7. 如果 Canvas 和 OnGUI 的分数重复，把 `GameFlow2D > Show Built In UI` 取消勾选，再自己做分数 Text。

### 18.15 保存和 Build Settings

1. `Ctrl+S` 保存场景；标题栏场景名没有 `*` 才算保存。
2. `File > Build Settings`。
3. 点 `Add Open Scenes`，当前场景出现在 `Scenes In Build`。
4. 确认场景左边勾选。
5. `Platform` 选 `Windows, Mac, Linux`；`Target Platform` 选 Windows。
6. 点 `Build And Run`，选一个空文件夹；能生成 exe 并运行，说明工程完整。

### 18.16 面板排查

| 现象 | 面板原因 | 处理 |
|---|---|---|
| Add Component 搜不到脚本 | 脚本有红错，Unity 没编译 | 看 Console 第一条红错，修完等右下角转圈结束 |
| Inspector 没有脚本字段 | 选错物体，或组件折叠 | 确认选的是挂脚本的物体；点组件标题左边三角 |
| Tag 下拉没有 Player | 还没新建 Tag | Inspector 顶部 Tag > Add Tag... 新建 |
| Layer 下拉没有 Ground | 还没新建 Layer | Inspector 顶部 Layer > Add Layer... 新建 |
| 物体槽拖不进去 | 类型不匹配 | 看清槽位要 Transform、GameObject 还是 Rigidbody2D |
| Play 后物体不动 | Rigidbody 的 Gravity 0，或 GameFlow 还没开始 | 先点 START；检查 Gravity Scale 和 Body Type |
| 相机里看不到物体 | 相机 Z 不是 -10，或 Projection 不是 Orthographic | 调相机 Position 和 Projection |
| 改了 Inspector，Play 后恢复 | 在 Play 模式里改的 | 先 Stop，改完再 Play |
| Console 一直刷红 | 缺引用或空物体 | 先修第一条；不要同时改五个地方 |
