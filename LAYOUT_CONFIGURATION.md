# BetterHud Layout Configuration Reference

This document describes all available configuration fields for layout components (images, texts, and heads).

## Table of Contents
- [Common Fields (All Components)](#common-fields-all-components)
- [Image Component Fields](#image-component-fields)
- [Text Component Fields](#text-component-fields)
- [Head Component Fields](#head-component-fields)
- [Conditions and Color Overrides](#conditions-and-color-overrides)

---

## Common Fields (All Components)

These fields are available for all component types: **images**, **texts**, and **heads**.

### Position and Location

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `x` | Integer | `0` | Horizontal offset in pixels |
| `y` | Integer | `0` | Vertical offset in pixels |
| `opacity` | Double | `1.0` | Component opacity (0.0 to 1.0) |

### Appearance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `outline` | Integer/String | `0` | Shadow/outline color. Can be `"false"` (0), `"true"` (0xFF000000), or hex color |
| `layer` | Integer | `0` | Rendering layer for z-ordering |
| `color` | String | `white` | Base color for the component (hex color like `#FF0000` or named color) |

### Behavior

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `follow` | String | - | Placeholder for player to follow (e.g., `%player_name%`) |
| `cancel-if-follower-not-exists` | Boolean | `true` | Hide component if followed player doesn't exist |
| `tick` | Long | `1` | Update frequency in ticks (20 ticks = 1 second) |

### Visual Effects

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `properties` | List | `[]` | Visual effects to apply. Options: `wave`, `rainbow`, `tiny_rainbow` |

**Example:**
```yaml
properties:
  - wave
  - rainbow
```

### Scaling

| Field | Type | Description |
|-------|------|-------------|
| `render-scale` | Object | Configure render scaling |
| `render-scale.x` | Double | Horizontal scale multiplier (default: `1.0`) |
| `render-scale.y` | Double | Vertical scale multiplier (default: `1.0`) |
| `render-scale.static-scale` | Boolean | Use static scaling (default: `false`) |

**Example:**
```yaml
render-scale:
  x: 1.5
  y: 1.5
  static-scale: false
```

### Placeholder Configuration

| Field | Type | Description |
|-------|------|-------------|
| `placeholder-option` | Object | Options for placeholder parsing |
| `placeholder-string-format` | Object | String formatting options for placeholders |

---

## Image Component Fields

Additional fields specific to image components.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Name of the image element to use (defined in image files) |

### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `scale` | Double | `1.0` | Scale multiplier for the image |
| `space` | Integer | `1` | Spacing between stacked images |
| `stack` | String | - | Placeholder for stack count (e.g., `%player_health%`) |
| `max-stack` | String | - | Placeholder for maximum stack count |
| `reversed` | Boolean | `false` | Reverse the stack rendering order |
| `clear-listener` | Boolean | `false` | Clear listener when conditions are not met |

**Example:**
```yaml
example_layout:
  images:
    1:
      name: health_bar
      x: 128
      y: 0
      color: white
      scale: 1.5
      stack: "%player_health%"
      max-stack: 20
```

---

## Text Component Fields

Additional fields specific to text components.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Name of the text element to use (defined in text files) |
| `pattern` | String | Text pattern with placeholders (e.g., `Health: %player_health%`) |

### Appearance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `color` | String | `white` | Text color (hex like `#FF0000` or named color) |
| `scale` | Double | `1.0` | Scale multiplier for text |
| `space` | Integer | `0` | Extra spacing between characters |
| `align` | String | `left` | Horizontal alignment: `left`, `center`, `right` |
| `line-align` | String | `left` | Alignment for multi-line text: `left`, `center`, `right` |

### Multi-line Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `line` | Integer | `1` | Number of lines (must be >= 1) |
| `split-width` | Integer | `200` | Width to split lines at (only applies if `line` > 1) |
| `line-width` | Integer | `10` | Spacing between lines in pixels |
| `force-split` | Boolean | `false` | Force split at exact width without word breaks |

### Number Formatting

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `disable-number-format` | Boolean | `true` | Disable automatic number formatting |
| `number-format` | String | (global) | Java DecimalFormat pattern (e.g., `#,###.##`) |
| `number-equation` | String | `t` | Equation to apply to numbers before formatting |

### Legacy Format

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `use-legacy-format` | Boolean | (global) | Enable legacy color code parsing (`&` or `§`) |
| `legacy-serializer` | String | (global) | Legacy format parser: `ampersand`, `section`, or `both` |

### Background

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `background` | String | - | Name of background element to use |
| `background-scale` | Double | (same as `scale`) | Scale for background element |

### Emoji Configuration

| Field | Type | Description |
|-------|------|-------------|
| `emoji-pixel` | Object | Emoji positioning offset |
| `emoji-pixel.x` | Integer | Horizontal emoji offset (default: `0`) |
| `emoji-pixel.y` | Integer | Vertical emoji offset (default: `0`) |
| `emoji-scale` | Double | Emoji scale multiplier (default: `1.0`, must be > 0) |

**Example:**
```yaml
example_layout:
  texts:
    1:
      name: health_text
      pattern: "Health: %player_health%"
      x: 0
      y: 48
      color: "#00FF00"
      scale: 0.5
      align: center
      line-align: center
      line: 3
      split-width: 200
      line-width: 10
      number-format: "#,###.##"
```

---

## Head Component Fields

Additional fields specific to head components (player heads).

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Name of the head element to use (defined in head files) |

### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | String | `standard` | Render type: `standard` or `fancy` |
| `align` | String | `left` | Alignment: `left`, `center`, `right` (ignored if type is `fancy`) |

**Note:** When `type` is `fancy`, alignment is always `center`.

**Example:**
```yaml
example_layout:
  heads:
    1:
      name: player_head
      x: 0
      y: 0
      type: standard
      align: center
```

---

## Conditions and Color Overrides

These features can be applied at the **layout level** (affects all components in the layout) or **component level** (affects individual components).

### Conditions

Control when components are visible based on placeholder values.

```yaml
conditions:
  1:
    first: "%player_health%"
    second: 10
    operation: ">"
  2:
    first: "%player_world%"
    second: "world"
    operation: "=="
```

**Available operations:**
- `==` - Equal
- `!=` - Not equal
- `<` - Less than
- `<=` - Less than or equal
- `>` - Greater than
- `>=` - Greater than or equal

### Color Overrides

Dynamically change component colors based on conditions. **The color-override always replaces the component's color completely.**

```yaml
color-overrides:
  1:
    color: "#FF0000"  # Red when health is low
    conditions:
      1:
        first: "%player_health%"
        second: 5
        operation: "<"
  2:
    color: "#FFFF00"  # Yellow when health is medium
    conditions:
      1:
        first: "%player_health%"
        second: 10
        operation: "<"
  3:
    color: "#00FF00"  # Green otherwise (no conditions = always true)
  default-color: "#FFFFFF"  # Optional fallback color
```

**Important:** Color-overrides are evaluated in order (1, 2, 3, ...). The **first matching condition** is used.

---

## Complete Example

```yaml
example_layout:
  # Layout-level conditions (apply to all components)
  conditions:
    1:
      first: "%player_gamemode%"
      second: "SURVIVAL"
      operation: "=="

  # Layout-level color overrides
  color-overrides:
    1:
      color: "#FF0000"
      conditions:
        1:
          first: "%player_health%"
          second: 5
          operation: "<"
    2:
      color: "#00FF00"

  # Image component
  images:
    1:
      name: health_bar
      x: 128
      y: 0
      color: white
      scale: 1.5
      stack: "%player_health%"
      max-stack: 20
      properties:
        - wave

  # Text component
  texts:
    1:
      name: health_text
      pattern: |
        Health: %player_health%
        Max: 20
      x: 0
      y: 48
      color: white
      scale: 0.5
      align: center
      line-align: center
      line: 2
      number-format: "#,###"
      # Component-level color override (combined with layout-level)
      color-overrides:
        1:
          color: "#FFAA00"
          conditions:
            1:
              first: "%player_health%"
              second: 15
              operation: "<"

  # Head component
  heads:
    1:
      name: player_head
      x: 200
      y: 0
      type: fancy
      scale: 1.0
