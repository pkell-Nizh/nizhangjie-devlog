# M1 — Foundation (EventBus + Core)

Milestone này tạo nền tảng để các milestone sau xây lên. **Không có gameplay logic nào ở đây.**

## Cần làm gì trong Unity sau khi drop folder này vào `Assets/Scripts/`

1. Tạo scene `Bootstrap.unity` (hoặc dùng scene khởi đầu hiện có)
2. Tạo empty GameObject tên `_Bootstrap`
3. Add component `GameBootstrap` (sẽ tự add `GameClock`)
4. Mở **XianXian → Event Debugger** từ menu để xem event log realtime khi play
5. (Optional) Project Settings → Script Execution Order: kéo `GameBootstrap` lên đầu, value `-1000`

## Cấu trúc đã tạo

```
Core/
├── Bootstrap/GameBootstrap.cs      Entry point
├── Enums/Faction.cs                Meta-game faction
├── Enums/Team.cs                   Combat-only team
├── Events/
│   ├── GameEvent.cs                Base class cho mọi event
│   ├── IEventBus.cs                Interface + IEventBusInterceptor
│   ├── EventBus.cs                 Implementation (priority, cancel)
│   ├── ScopedBuses.cs              GlobalEventBus, CombatEventBus, LocalMapEventBus
│   ├── EventListenerBehaviour.cs   Base MonoBehaviour auto-unsubscribe
│   ├── EventLogger.cs              Ring buffer logger
│   ├── SampleEvents.cs             Skeleton event types cho M2+
│   └── Editor/EventDebuggerWindow.cs   Editor window xem log
├── Time/GameClock.cs               Tick system (Fast/Slow/Day)
├── Save/                           (M6)
└── ServiceLocator.cs               Service locator
```

Các folder khác (`Data/`, `Stats/`, `Combat/`, ...) chỉ chứa placeholder `.txt`
để giữ cấu trúc, sẽ implement ở các milestone tương ứng.

## Cách dùng EventBus

### Publish event

```csharp
using XianXian.Core;
using XianXian.Core.Events;
using XianXian.Core.Events.SampleEvents;

var bus = ServiceLocator.Get<GlobalEventBus>();
bus.Publish(new DiscipleDiedEvent {
    DiscipleId = "d_001",
    KillerId = null,
    KillerFaction = Faction.None,
    CauseTag = "tribulation"
});
```

### Subscribe (cách 1 — class kế thừa MonoBehaviour)

Khuyến nghị dùng `EventListenerBehaviour` để khỏi quên unsubscribe:

```csharp
using XianXian.Core.Events;
using XianXian.Core.Events.SampleEvents;

public class SectUIPanel : EventListenerBehaviour
{
    private void OnEnable()
    {
        ListenGlobal<DiscipleDiedEvent>(OnDiscipleDied);
        ListenGlobal<RealmBreakthroughEvent>(OnBreakthrough, priority: -100); // UI chạy sau
    }

    private void OnDiscipleDied(DiscipleDiedEvent e)
    {
        if (e.IsCancelled) return; // pháp bảo đã cứu, không show notification
        // refresh UI...
    }

    private void OnBreakthrough(RealmBreakthroughEvent e) { /* ... */ }

    // KHÔNG cần OnDestroy / OnDisable để unsubscribe
}
```

### Subscribe (cách 2 — class thường, tự quản lý)

```csharp
public class KarmaTracker
{
    private readonly GlobalEventBus _bus;
    public KarmaTracker() {
        _bus = ServiceLocator.Get<GlobalEventBus>();
        _bus.Subscribe<DiscipleDiedEvent>(OnDeath, priority: 50);
    }
    public void Dispose() => _bus.Unsubscribe<DiscipleDiedEvent>(OnDeath);
    private void OnDeath(DiscipleDiedEvent e) { /* ... */ }
}
```

### Pháp bảo intercept (use case cancel)

```csharp
public class ResurrectionTalisman : EventListenerBehaviour
{
    [SerializeField] private string _ownerId;
    private bool _used;

    private void OnEnable()
    {
        // Priority +1000 → chạy TRƯỚC mọi handler khác
        ListenGlobal<DiscipleDiedEvent>(OnAnyDeath, priority: 1000);
    }

    private void OnAnyDeath(DiscipleDiedEvent e)
    {
        if (_used) return;
        if (e.DiscipleId != _ownerId) return;

        e.Cancel("Resurrection Talisman triggered");
        _used = true;
        // gọi service hồi sinh, restore HP, etc.
    }
}
```

Các handler priority thấp (UI, reputation, karma) khi nhận event sẽ kiểm tra
`e.IsCancelled` và tự quyết định có chạy không.

### Bridge giữa các bus

Ví dụ: combat unit chết trong trận → cần raise meta event để Sect biết:

```csharp
public class CombatToGlobalBridge : EventListenerBehaviour
{
    private void OnEnable()
    {
        ListenCombat<CombatUnitDiedEvent>(OnCombatDeath);
    }

    private void OnCombatDeath(CombatUnitDiedEvent e)
    {
        // Convert ID combat unit → disciple ID (qua registry, sẽ làm ở M3/M4)
        var discipleId = ResolveDiscipleId(e.UnitId);
        if (discipleId == null) return; // unit không phải đệ tử thật (vd: NPC summon)

        ServiceLocator.Get<GlobalEventBus>().Publish(new DiscipleDiedEvent {
            DiscipleId = discipleId,
            KillerId = e.KillerUnitId,
            CauseTag = "combat"
        });
    }

    private string ResolveDiscipleId(string unitId) => null; // M3 sẽ implement
}
```

### Tạo / hủy bus theo scope

```csharp
// CombatEncounter.cs (sẽ làm ở M3)
public void StartEncounter() {
    var bus = new CombatEventBus();
    ServiceLocator.Register(bus);
}
public void EndEncounter() {
    var bus = ServiceLocator.TryGet<CombatEventBus>();
    bus?.Clear();
    ServiceLocator.Unregister<CombatEventBus>();
}
```

## Migration code cũ (sẽ làm ở M3)

Code cũ:
```csharp
unit.OnDeath += OnUnitDeath;
unit.RegisterDeath();
```

Code mới (M3):
```csharp
// trong UnitController.Die()
ServiceLocator.Get<CombatEventBus>().Publish(new CombatUnitDiedEvent {
    UnitId = id, Team = team, KillerUnitId = killerId
});

// trong BattleManager / CombatEncounter
ListenCombat<CombatUnitDiedEvent>(OnUnitDeath);
```

## Quy ước priority

| Priority | Dùng cho |
|---------:|----------|
| +1000    | Pháp bảo intercept, cancel sớm |
| +500     | Buff/debuff protect (vd: invincibility frame) |
| +100     | Core game logic (sect reputation, karma) |
| 0        | Default |
| -100     | UI update, audio cue |
| -1000    | Logger, analytics (chạy sau cùng) |

## Lưu ý

- `EventBus` **không thread-safe**. Chỉ publish từ main thread.
- Event payload nên dùng `init`-only field, không mutable.
- KHÔNG để `GameObject` / `MonoBehaviour` reference trong event field — dùng ID string.
- Khi handler subscribe/unsubscribe handler khác giữa lúc publish, **an toàn**
  (EventBus copy snapshot trước khi iterate).
- Event được fire ngay (synchronous). Không có deferred / queue ở M1.

## Test thử

1. Play scene có `GameBootstrap`
2. Mở **XianXian → Event Debugger**
3. Trong code bất kỳ, thử:
```csharp
ServiceLocator.Get<GlobalEventBus>().Publish(new DayPassedEvent { TotalDays = 42 });
```
4. Sẽ thấy entry mới trong Event Debugger window
