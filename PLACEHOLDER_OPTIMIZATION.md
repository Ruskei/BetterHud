# Placeholder Performance Optimization Guide

This guide explains strategies for optimizing expensive placeholder computations in BetterHud.

## Table of Contents
- [Understanding the Update System](#understanding-the-update-system)
- [Built-in Optimization: Lazy Listeners](#built-in-optimization-lazy-listeners)
- [Manual Caching Strategy](#manual-caching-strategy)
- [Event-Driven Updates with Popups](#event-driven-updates-with-popups)
- [Best Practices](#best-practices)

---

## Understanding the Update System

### Two-Phase Update Model

BetterHud uses a **two-phase update model**:

1. **Build Phase** (on `UpdateEvent`):
   - Your placeholder's `invoke(args, UpdateEvent)` method is called
   - Returns a `Function<HudPlayer, T>` that will be used for this update cycle
   - Triggered by game events (player move, health change, etc.) or popups

2. **Evaluation Phase** (every tick):
   - The returned function is called for each player: `Function<HudPlayer, T>.apply(player)`
   - This happens **every tick** (20 times per second by default)
   - Use the `tick` field in layout config to reduce frequency

### When Does Your Code Run?

```java
numCont.addPlaceholder(
    "example_1",
    HudPlaceholder.of { _, event ->        // Build Phase: Called on UpdateEvent
        Function { player ->                // Evaluation Phase: Called EVERY TICK
            return@Function expensiveComputation()  // ⚠️ THIS RUNS EVERY TICK!
        }
    }
)
```

**Problem**: `expensiveComputation()` runs 20 times per second per player!

---

## Built-in Optimization: Lazy Listeners

BetterHud provides a **lazy listener** system that automatically caches values with configurable smoothing.

### When to Use

- Expensive computations that can be cached
- Values that don't need frame-perfect accuracy
- Smooth transitions are acceptable (e.g., health bars)

### How to Configure

In your **image element** configuration (not layout!):

```yaml
health_bar:
  type: listener
  listener:
    class: placeholder
    value: "%my_expensive_placeholder%"
    lazy: true                    # Enable caching
    initial-value: "0"           # Starting value
    delay: 5                     # Wait 5 ticks before updating
    multiplier: 0.5              # Smoothing factor (0.0-1.0)
    expiring-second: 5           # Cache expiry time in seconds
```

### Configuration Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `lazy` | Boolean | `false` | Enable lazy evaluation with caching |
| `initial-value` | String (placeholder) | `1.0` | Initial value when first accessed |
| `delay` | Integer | `0` | Number of ticks to wait before updating cache |
| `multiplier` | Double | `0.5` | Smoothing factor: `new = old * (1 - m) + actual * m` |
| `expiring-second` | Long | `5` | Cache entry expiration time (minimum 1 second) |

### How It Works

```
Frame 1:  actual=100, cached=50  → cached becomes 75  (50 * 0.5 + 100 * 0.5)
Frame 2:  actual=100, cached=75  → cached becomes 87.5 (75 * 0.5 + 100 * 0.5)
Frame 3:  actual=100, cached=87.5 → cached becomes 93.75
...
Converges smoothly to actual value
```

**Benefits**:
- Reduces computation frequency
- Smooth visual transitions
- Automatic per-player caching
- Configurable smoothing

**Limitations**:
- Only works with **listener-based images** (not text or direct placeholders)
- Values are Numbers only (0.0 to 1.0 for images)
- Slight delay before reaching actual value

---

## Manual Caching Strategy

For placeholders used in **text** or when you need more control, implement caching yourself.

### Strategy 1: Cache in Build Phase

Cache the computation result in the `UpdateEvent` phase:

```java
numCont.addPlaceholder(
    "cached_example",
    HudPlaceholder.of { _, event ->
        // Expensive computation runs ONCE per update event
        int cachedValue = expensiveComputation();

        // Return function that uses cached value
        Function { player ->
            return@Function cachedValue  // Fast lookup, no recomputation
        }
    }
)
```

**When to use**: Value is the same for all players, expensive to compute.

---

### Strategy 2: Per-Player Cache with UpdateEvent Key

Use the `UpdateEvent.getKey()` to cache per update cycle:

```java
private static class CachedPlaceholder implements HudPlaceholder<Number> {
    private final Map<Object, Map<UUID, Integer>> cache = new ConcurrentHashMap<>();

    @Override
    public Function<HudPlayer, Number> invoke(List<String> args, UpdateEvent reason) {
        Object eventKey = reason.getKey();
        Map<UUID, Integer> perPlayerCache = cache.computeIfAbsent(
            eventKey,
            k -> new ConcurrentHashMap<>()
        );

        return player -> {
            return perPlayerCache.computeIfAbsent(
                player.uuid(),
                uuid -> expensivePerPlayerComputation(player)
            );
        };
    }

    @Override
    public int getRequiredArgsLength() {
        return 0;
    }
}

// Register it
numCont.addPlaceholder("smart_cached", new CachedPlaceholder());
```

**When to use**: Value is player-specific, expensive to compute.

**Important**: Clean up old cache entries periodically to avoid memory leaks!

---

### Strategy 3: Time-Based Caching

Cache values for a fixed duration:

```java
private static class TimeCachedPlaceholder implements HudPlaceholder<Number> {
    private final Map<UUID, CachedValue> cache = new ConcurrentHashMap<>();
    private final long cacheDurationMs = 1000; // 1 second

    private static class CachedValue {
        final int value;
        final long timestamp;

        CachedValue(int value) {
            this.value = value;
            this.timestamp = System.currentTimeMillis();
        }

        boolean isExpired(long duration) {
            return System.currentTimeMillis() - timestamp > duration;
        }
    }

    @Override
    public Function<HudPlayer, Number> invoke(List<String> args, UpdateEvent reason) {
        return player -> {
            UUID uuid = player.uuid();
            CachedValue cached = cache.get(uuid);

            if (cached == null || cached.isExpired(cacheDurationMs)) {
                int newValue = expensiveComputation(player);
                cache.put(uuid, new CachedValue(newValue));
                return newValue;
            }

            return cached.value;
        };
    }

    @Override
    public int getRequiredArgsLength() {
        return 0;
    }
}
```

**When to use**: Value doesn't change often, can tolerate staleness.

---

## Event-Driven Updates with Popups

For **truly event-driven updates** (only update when something changes), use **Popups** triggered by Bukkit events.

### How It Works

1. Create a popup layout (updated only when shown)
2. Trigger popup on specific events
3. Placeholder runs ONLY when popup is triggered

### Example: Health Change Popup

**1. Create popup configuration** (`popups/health_warning.yml`):

```yaml
health_warning:
  move: 0               # Don't move on screen
  gui-scale: false
  duration: 20          # Show for 1 second (20 ticks)
  layouts:
    1:
      images:
        1:
          name: warning_icon
          x: 128
          y: -20
      texts:
        1:
          name: warning_text
          pattern: "Health: %my_expensive_health_calc%"
          x: 128
          y: 0
```

**2. Trigger from your plugin when health changes**:

```java
import kr.toxicity.hud.api.BetterHudAPI;
import kr.toxicity.hud.api.bukkit.update.BukkitEventUpdateEvent;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.entity.EntityDamageEvent;

public class HealthListener implements Listener {

    @EventHandler
    public void onDamage(EntityDamageEvent event) {
        if (!(event.getEntity() instanceof Player)) return;

        Player player = (Player) event.getEntity();
        HudPlayer hudPlayer = BetterHudAPI.inst().getHudPlayer(player.getUniqueId());

        if (hudPlayer != null) {
            // Create update event from Bukkit event
            BukkitEventUpdateEvent updateEvent = new BukkitEventUpdateEvent(
                event,
                UUID.randomUUID()  // Unique key for this update
            );

            // Show popup (placeholder will be computed NOW, not every tick)
            Popup popup = BetterHudAPI.inst().getPopupManager().getPopup("health_warning");
            if (popup != null) {
                popup.show(updateEvent, hudPlayer);
            }
        }
    }
}
```

### Benefits

- **Computation runs ONLY when event fires**
- Perfect for rare or specific events
- Can pass event data to placeholders via UpdateEvent
- Clean event-driven architecture

### Accessing Event Data in Placeholders

```java
numCont.addPlaceholder(
    "event_damage",
    HudPlaceholder.of { _, event ->
        // Check if this is a Bukkit event
        if (event instanceof BukkitEventUpdateEvent bukkitEvent) {
            Event rawEvent = bukkitEvent.event();

            if (rawEvent instanceof EntityDamageEvent damageEvent) {
                double damage = damageEvent.getDamage();

                return player -> (int) damage;  // Return damage value
            }
        }

        return player -> 0;  // Default value
    }
)
```

---

## Best Practices

### 1. Choose the Right Strategy

| Situation | Recommended Approach |
|-----------|---------------------|
| Image listener with smooth animation | **Lazy Listener** (config-based) |
| Expensive per-player calculation | **Manual per-player cache** |
| Global expensive calculation | **Cache in build phase** |
| Event-specific data | **Popup with event trigger** |
| Moderate computation, text-based | **Time-based cache** |

### 2. Reduce Update Frequency

Use the `tick` field in layouts to update less frequently:

```yaml
example_layout:
  texts:
    1:
      name: health_text
      pattern: "%expensive_placeholder%"
      tick: 20  # Update once per second instead of 20 times per second
```

### 3. Profile Before Optimizing

Not every placeholder needs optimization. Profile your server to identify actual bottlenecks:

```java
// Add timing to your placeholder
public Function<HudPlayer, Number> invoke(List<String> args, UpdateEvent reason) {
    return player -> {
        long start = System.nanoTime();
        int result = myComputation(player);
        long elapsed = System.nanoTime() - start;

        if (elapsed > 1_000_000) { // > 1ms
            System.out.println("Slow placeholder: " + elapsed/1_000_000.0 + "ms");
        }

        return result;
    };
}
```

### 4. Clean Up Caches

Always clean up expired cache entries to prevent memory leaks:

```java
// Periodic cleanup (run every 5 minutes)
Bukkit.getScheduler().runTaskTimerAsynchronously(plugin, () -> {
    cache.entrySet().removeIf(entry -> {
        return entry.getValue().isExpired();
    });
}, 6000L, 6000L);
```

### 5. Consider Async Computation

For very expensive operations, compute asynchronously:

```java
private final ExecutorService executor = Executors.newCachedThreadPool();
private final Map<UUID, CompletableFuture<Integer>> pending = new ConcurrentHashMap<>();

public Function<HudPlayer, Number> invoke(List<String> args, UpdateEvent reason) {
    return player -> {
        UUID uuid = player.uuid();

        CompletableFuture<Integer> future = pending.computeIfAbsent(uuid, u ->
            CompletableFuture.supplyAsync(() -> {
                return veryExpensiveComputation(player);
            }, executor).whenComplete((result, ex) -> {
                pending.remove(u);
            })
        );

        return future.getNow(-1);  // Return -1 if still computing
    };
}
```

**Warning**: Be careful with async operations - ensure thread safety!

---

## Summary

**Quick Decision Tree**:

1. **Is it an image listener?**
   - Yes → Use **lazy listener** config
   - No → Continue

2. **Does it need event-specific data?**
   - Yes → Use **popup with event trigger**
   - No → Continue

3. **Is it expensive and player-specific?**
   - Yes → **Manual per-player cache**
   - No → Continue

4. **Is it expensive and global?**
   - Yes → **Cache in build phase**
   - No → Continue

5. **Is it moderately expensive?**
   - Yes → **Time-based cache** or reduce `tick` frequency
   - No → No optimization needed!

---

For more information:
- See `LAYOUT_CONFIGURATION.md` for layout configuration reference
- Check BetterHud API documentation for full API details
