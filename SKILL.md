---
name: qq-bot-platform-cdp
description: "Automate QQ bot management platform (q.qq.com/qqbot) via Chrome DevTools Protocol. Use when configuring bot commands, services, menus, or any QQ bot settings through the web dashboard."
version: 1.5.0
author: Hermes Agent
platforms: [linux]
metadata:
  hermes:
    tags: [qq, bot, cdp, browser-automation, vue]
---

# QQ Bot Platform CDP Automation

Automate the QQ bot management dashboard (q.qq.com/qqbot) using Chrome DevTools Protocol over WebSocket. The platform is a Vue SPA — standard DOM click events don't trigger Vue handlers; you must use CDP `Input.dispatchMouseEvent`.

## Prerequisites

Chrome running with remote debugging on port 9222:
```bash
google-chrome --remote-debugging-port=9222 --no-first-run
```

Verify: `curl -s http://127.0.0.1:9222/json/version`

## CDP Connection

```python
from websocket import create_connection
ws = create_connection("ws://127.0.0.1:9222/devtools/page/<PAGE_ID>", timeout=15)
```

Get page IDs: `curl -s http://127.0.0.1:9222/json/list`

## Core Patterns

### Reliable Clicking (Vue SPA)

JavaScript `.click()` is unreliable on Vue apps. Use CDP mouse events:

```python
def send(msg):
    ws.send(json.dumps(msg))
    return json.loads(ws.recv())

def click_at(x, y):
    send({"method":"Input.dispatchMouseEvent","params":{"type":"mousePressed","x":x,"y":y,"button":"left","clickCount":1}})
    send({"method":"Input.dispatchMouseEvent","params":{"type":"mouseReleased","x":x,"y":y,"button":"left","clickCount":1}})

# Get element position then click
r = js("""(function(){
    var el = document.querySelector('.my-button');
    var rect = el.getBoundingClientRect();
    return JSON.stringify({x:rect.left+rect.width/2, y:rect.top+rect.height/2});
})()""")
pos = json.loads(r)
click_at(pos["x"], pos["y"])
```

### Filling Vue Input Fields

Standard `el.value = 'x'` doesn't trigger Vue reactivity. Use the native setter + dispatch:

```javascript
var d = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value');
d.set.call(inputEl, 'my_value');
inputEl.dispatchEvent(new Event('input', {bubbles: true}));
```

### Element Out of Viewport

Use `scrollIntoView` before `getBoundingClientRect`:
```javascript
el.scrollIntoView({block: 'center'});
var rect = el.getBoundingClientRect();
```

Or scroll the container: `document.querySelector('.config-content-config-body').scrollTop = 0;`

## QQ Bot Platform — 指令配置 Workflow

### Key Selectors

| Element | Selector |
|---------|----------|
| Edit mode button | `button` with text "重新配置" |
| Add command (+) | `svg.t-icon-add-circle` (only visible in edit mode) |
| Command items | `.command-item-wrap` |
| Command header (click to select) | `.command-item-wrap .form-box-header` |
| Command name input | `input.t-input__inner` (index 1 — index 0 is the "/" prefix) |
| Command description input | `input.t-input__inner` (index 2) |
| Save button | `button` with text "保存配置" |
| Usage scene items | `.usescene-item` (lives INSIDE each `.command-item-wrap`, NOT global) |
| Usage scene active class | `usescene-item-active` |
| Disabled scene class | `usescene-item-forbit` |
| Identity verification QR | `.common-qrcode-main canvas` |

### Full Workflow: Add Commands

1. **Navigate** to `https://q.qq.com/qqbot/#/developer/publish-config/function-config?appid=<id>`.
   Command configuration IS per-appid — each bot has its own separate 指令配置.
   **ALWAYS open a fresh CDP target (tab)** when switching between bots. Navigating in the same tab can show stale/cached data from the previous bot (see Pitfall #13).

2. **Enter edit mode**: Click "重新配置" button. Verify with "取消"/"保存配置" buttons appearing and `.t-icon-add-circle` being visible.

3. **Add command rows**: Click the `+` icon N times:
   ```python
   for _ in range(count):
       r = js("el.getBoundingClientRect...")
       click_at(r["x"], r["y"])
       time.sleep(0.3)
   ```
   Re-query position each time — the list grows and the `+` moves down.

4. **Fill each command**: Click command header to select it, then fill inputs:
   ```python
   def click_item(idx):
       r = js(f"items[{idx}].querySelector('.form-box-header').getBoundingClientRect...")
       click_at(r["x"], r["y"])
   
   def fill(cmd, desc):
       js(f"d.set.call(ins[1], '{cmd}'); ins[1].dispatchEvent(...)")
       js(f"d.set.call(ins[2], '{desc}'); ins[2].dispatchEvent(...)")
   
   for idx, (cmd, desc) in enumerate(commands):
       click_item(idx)
       fill(cmd, desc)
   ```

5. **Check usage scenarios PER COMMAND** (CRITICAL — each command has its OWN usage scenes, NOT global):
   After filling each command's name and description, select "消息列表" for THAT SPECIFIC command. The `.usescene-item` elements live inside each `.command-item-wrap`, not at the page level.

   ```python
   for idx, (cmd, desc) in enumerate(commands):
       click_item(idx)
       fill(cmd, desc)
       # THEN select usage scenes for THIS command
       js(f"""
       (function(){{
           var item = document.querySelectorAll('.command-item-wrap')[{idx}];
           var scenes = item.querySelectorAll('.usescene-item');
           for (var i = 0; i < scenes.length; i++) {{
               var isMsgList = scenes[i].textContent.trim() === '消息列表';
               var isForbid = scenes[i].classList.contains('usescene-item-forbit');
               var isActive = scenes[i].classList.contains('usescene-item-active');
               if (!isForbid && isActive && !isMsgList) {{ scenes[i].click(); }}
               if (isMsgList && !isActive) {{ scenes[i].click(); }}
           }}
       }})()
       """)
   ```

   ⚠️ Do NOT iterate `document.querySelectorAll('.usescene-item')` at the global level — that only affects whichever command happens to be expanded. Every command must be expanded and have its own scenes set individually.

6. **Save (two-step)**: Click "保存配置" via CDP mouse event.

   **Step A — Confirmation dialog**: The platform first shows a confirmation dialog (`.t-dialog__ctx` with display ≠ none) with text "保存功能配置 是否需要保存已填写的服务？" and buttons "取消" / "确认". You MUST click "确认" on this dialog before the QR step. The dialog has a mask overlay (`.t-dialog__mask`) that blocks further clicks until dismissed.
   
   ```python
   # After clicking save, wait for confirmation dialog
   time.sleep(1.5)
   # Find visible dialog's 确认 button and click it
   r = js("""...querySelectorAll('.t-dialog__ctx')...find visible...querySelector('button')...text='确认'...getBoundingClientRect()...""")
   click_at(r["x"], r["y"])
   time.sleep(1)
   ```

   **Step B — Identity verification QR**: After confirming, the QR code dialog appears.

7. **Identity verification**: Extract QR code from canvas, user scans on VNC page. Dialog auto-closes on success, page returns to read-only mode ("返回"/"重新配置" buttons).

### Verifying Results

**Read-only view** (after save):
- Buttons: "返回", "重新配置"
- Commands in `.function-card__expand-name span`

**Edit mode** (after clicking "重新配置"):
- Buttons: "取消", "保存配置"
- Commands in `.command-item-wrap .form-box-expand-name`

### Limits

- Max 24 commands per bot (platform tooltip)
- Command name: only Chinese/English characters + digits (no hyphens, underscores, or symbols)
- Description: same character restrictions
- Three usage scenarios (QQ频道, 频道私信, QQ群) are disabled by default with `forbit` class — need sandbox config first
- Identity verification required for every save

### Common Pitfalls

1. **JS `.click()` doesn't work** — Vue event handlers require CDP `Input.dispatchMouseEvent`.
2. **`+` icon position changes** after each click — re-query `getBoundingClientRect` every time.
3. **Element out of viewport** — `scrollIntoView({block:'center'})` before getting position.
4. **Vue input values** — must use native setter + `input` event, not `.value =`.
5. **WebSocket timeout** — CDP connection can drop after rapid operations; reconnect if needed.
6. **Save is a TWO-STEP process (confirmation dialog → QR)** — clicking "保存配置" shows a confirmation dialog FIRST. Only after clicking "确认" does the QR code appear. Skipping the confirmation step causes silent save failure (page stuck in edit mode).
7. **Confirmation dialog may be hidden (`display:none`)** — after clicking save, the dialog `.t-dialog__ctx` may exist but be invisible. Check `window.getComputedStyle(d).display`; if `none`, set to `''` to reveal it, then CDP-click "确认". Do NOT clear `.t-dialog__mask` — let the dialog manage its own overlay.
8. **Usage scenes are PER-COMMAND, NOT global** 🔥 — Each `.command-item-wrap` has its own `.usescene-item` elements inside it. Global scene selectors only affect the currently-expanded command. You MUST: click each command's header → select "消息列表" in ITS scenes → repeat for all 24. This is the #1 cause of failed saves.
9. **Commands are PER-BOT (独立配置)** — each appid has independent 指令配置. Use `Target.createTarget` for a fresh tab when switching bots. Same-tab navigation caches stale Vue state.
10. **Header shows "N/A"/"?" after input** — the `.form-box-expand-name` header sometimes lags after Vue native-setter input. The underlying `input.t-input__inner` values are correct; it resolves after save. Trust the input values, not the header display.
11. **Cannot reorder via CDP** — command positions are fixed to slots. Replace commands in-place; manual drag-and-drop is needed for reordering.
12. **Wrong descriptions block silent save** — typos or truncated descriptions cause the platform to reject save without error. Verify every description matches user's exact wording.
13. **Scene availability varies by bot** — check `.usescene-item-forbit` on each scene. On multi-scene bots, select ONLY "消息列表" to avoid conflicts.
14. **Always refresh stale edit state** — if save fails repeatedly, navigate `about:blank` → target URL for a clean slate. Lingering dialog/mask state from prior attempts corrupts subsequent saves.
