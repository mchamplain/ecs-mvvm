# 🎨 UI Architecture (ECS-MVVM + UI Toolkit)

## 🏗️ Why Use ECS-MVVM?

The **ECS-MVVM (Entity Component System - Model View ViewModel)** pattern ensures a **clean, scalable, and reactive UI architecture** while integrating seamlessly with **Unity ECS**.

### ✅ Benefits of ECS-MVVM:
- **Separation of Concerns:** ECS game logic does not directly interact with UI logic.
- **Data-Driven UI:** UI elements automatically update when ECS updates the ViewModel.
- **Scalability:** New UI features can be added without modifying ECS systems.
- **Performance:** ECS handles game logic efficiently, while UI updates remain optimized through bindings.
- **Maintainability:** Less coupling between UI and game logic makes long-term development easier.

---

## 📂 UI File Breakdown

### **ViewModel (`<Feature>ViewModel.asset`)**
- Stores UI-related **state and data bindings**.
- Acts as a **bridge between ECS and the UI**.
- **Updated by ECS systems** and **observed by the UI**.

### **View (`<Feature>View.cs`)**
- A MonoBehaviour that **binds UI elements** to the ViewModel.
- Handles UI initialization, but **contains no game logic**.
- Uses **UI Toolkit bindings** to update UI elements dynamically.

### **Layout (`<Feature>.uxml`)**
- **Defines the UI structure** using Unity UI Toolkit.
- Specifies the **hierarchy of UI elements** (labels, buttons, progress bars, etc.).

### **Styles (`<Feature>.uss`)**
- **Defines the visual appearance** of UI elements.
- Similar to **CSS** in web development.

---

## 🔗 How It Works

1️⃣ **Model (ECS Data - Components & Systems)** → ECS stores gameplay data (e.g., `PlayerHealthComponent`).  
2️⃣ **ViewModel (`ScriptableObject`)** → Acts as a bridge between ECS and UI, holding UI state.  
3️⃣ **View (UI Toolkit)** → Displays the UI and updates automatically using UI Toolkit **data binding**.

### 🎯 Example Flow:
1. **ECS updates the `PlayerHealthViewModel`** (e.g., health changes from 100 → 75).
2. **UI is automatically updated** via **UI Toolkit bindings** (e.g., health bar decreases).
3. **No direct interaction between ECS and UI**—only the `ViewModel` is modified.

---

## 🛠️ Implementation Details

### **1. ViewModel - `BaseUIViewModel`**
The ViewModel acts as the bridge between ECS and UI:
```csharp
public abstract class BaseUIViewModel : ScriptableObject {}
```
#### **Example: `PlayerHealthViewModel`**
```csharp
[CreateAssetMenu(fileName = "PlayerHealthViewModel", menuName = "UI/Player Health ViewModel")]
public class PlayerHealthViewModel: BaseUIViewModel
{
    [Header("ECS-Driven Variables")]
    public int health = 100;
    
    [Header("UI-Only Variables")]
    [SerializeField] private Color healthBarColor = Color.green;
    public Color HealthBarColor { get => healthBarColor; set => healthBarColor = value; }

    [SerializeField] private bool showHealthAsText = true;
    public bool ShowHealthAsText { get => showHealthAsText; set => showHealthAsText = value; }
}
```

### **2. ECS System - `UISystemBase<TViewModel>`**
Handles ViewModel updates from ECS data:
```csharp
public abstract partial class UISystemBase<TViewModel> : SystemBase where TViewModel : ScriptableObject
{
    protected TViewModel ViewModel { get; private set; }
    private AsyncOperationHandle<TViewModel> loadHandle;
    protected abstract string AddressKey { get; }

    protected override void OnStartRunning()
    {
        loadHandle = Addressables.LoadAssetAsync<TViewModel>(AddressKey);
        loadHandle.Completed += OnViewModelLoaded;
    }
    
    private void OnViewModelLoaded(AsyncOperationHandle<TViewModel> obj)
    {
        if (obj.Status == AsyncOperationStatus.Succeeded)
            ViewModel = obj.Result;
        else
            Debug.LogError($"Failed to load {typeof(TViewModel).Name} from Addressables key: {AddressKey}");
    }
    
    protected override void OnDestroy()
    {
        if (loadHandle.IsValid())
            Addressables.Release(loadHandle);
    }
}
```

#### **Example: `PlayerHealthUISystem`**
```csharp
public partial class PlayerHealthUISystem: UISystemBase<PlayerHealthViewModel>
{
    protected override string AddressKey => "PlayerHealthViewModel";
    
    protected override void OnUpdate()
    {
        if (ViewModel == null) return;

        foreach (RefRO<HealthComponent> playerHealth in SystemAPI.Query<RefRO<HealthComponent>>())
        {
            ViewModel.health = playerHealth.ValueRO.CurrentHealth;
        }
    }
}
```

### **3. UI View - `PlayerHealthView`**
Binds UI elements to the ViewModel:
```csharp
public class PlayerHealthView : BaseUIView<PlayerHealthViewModel>
{
    private Label healthLabel;
    private ProgressBar healthBar;

    private void Awake()
    {
        VisualElement root = uiDocument.rootVisualElement;
        healthLabel = root.Q<Label>("healthLabel");
        healthBar = root.Q<ProgressBar>("healthBar");

        DataBinding healthBinding = new()
        {
            dataSource = viewModel,
            dataSourcePath = PropertyPath.FromName(nameof(PlayerHealthViewModel.health)),
            bindingMode = BindingMode.ToTarget
        };
        
        healthBar.SetBinding("value", healthBinding);

        if (viewModel.ShowHealthAsText)
        {
            healthBinding.sourceToUiConverters.AddConverter((ref int health) => $"My Health: {health}");
        }
        
        healthLabel.SetBinding("text", healthBinding);
        healthBar.SetBinding("title", healthBinding);
    }
}
```
