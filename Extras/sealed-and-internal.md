# 善用 `sealed` 與 `internal` 強化架構邊界

## 核心概念

`sealed` 與 `internal` 是 C# 提供的兩個存取修飾詞，它們是我們在**編譯時期**強制執行架構規則、避免誤用的強力工具。

- `sealed`: 防止一個類別被繼承。
- `internal`: 將類型或成員的存取權限限制在其定義的**組件 (Assembly / .asmdef)** 內部。

## 為何至關重要

- **明確設計意圖 (`sealed`)**: 使用 `sealed` 等同於向其他開發者宣告：「這個類別的設計是完整的，不應被繼承以修改其行為。」這不僅能防止因意外繼承而導致的脆弱基類問題，也與我們的「**組合優於繼承**」原則相呼應，鼓勵開發者創建新的組件而非擴展既有類別。

- **強制架構邊界 (`internal`)**: `internal` 是實現模組化架構的**基石**。當我們為每個模組（`Application`, `Infrastructure` 等）定義了 `.asmdef` 後，`internal` 就能確保模組內部的實作細節不會洩漏到外部。這能有效防止其他模組「非法」引用不該被引用的類別，從而在程式碼提交前就杜絕潛在的架構破壞。

## 在本架構中的實踐

### 預設使用 `sealed`
對於絕大多數非抽象的具體類別，如果沒有明確且必要的繼承需求，**一律預設使用 `sealed`**。這適用於：
- `UseCase` 類別 (例如：`public sealed class LoginUseCase`)
- `Store` 類別 (例如：`public sealed class PlayerStore`)
- `Presenter` 類別 (例如：`public sealed class HudPresenter`)
- `Infrastructure` 中的 `Port` 實作 (例如：`public sealed class ApiUserRepository : IUserRepository`)

### 策略性使用 `internal`

`internal` 的核心用途是**隱藏實作細節，只暴露必要的公共 API**。

**關鍵應用場景： 保護 `Store` 的寫入權限**

`Store` 的狀態只能由 `UseCase` 修改。為了在編譯層級強制此規則，我們將 `Store` 的寫入方法設為 `internal`。

```csharp
public sealed class PlayerStore
{
    internal readonly ReactiveProperty<int> health = new(100);
    public IReadOnlyReactiveProperty<int> Health => health; 
}
```

**運作方式：**
- **`UseCase`** (在同一個 `.asmdef` 內) 可以存取 `internal` 的 `health` 屬性，並直接賦值：`_playerStore.health.Value = newHealth;`。
- **`Presenter`** (在不同的 `.asmdef` 外) 只能看到 `public` 的 `Health` 屬性，其型別為 `IReadOnlyReactiveProperty<int>`，無法寫入。