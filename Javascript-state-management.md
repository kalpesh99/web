# JavaScript State Management with jQuery

A guide to managing application state and keeping the DOM in sync using two approaches: Classic (explicit updates) and Proxy-based (automatic reactivity).

## The Problem

When building interactive web pages, you need to:
1. Store data (state) somewhere
2. Update the DOM when state changes
3. Update state when users interact with the page

Without a system, this becomes messy and error-prone.

---

## HTML Structure

Both approaches use the same HTML with custom data attributes:

```html
<h1 data-tag="titleColor" data-color>jDux is super cool!</h1>

<div>
    <button class="buttonRed">red</button>
    <button class="buttonBlue">blue</button>
    <button class="buttonGreen">green</button>
</div>

<div>
    Count = <span data-tag="num" data-text>#</span>
    <button type="button" id="inc">+</button>
    <button type="button" id="dec">-</button>
</div>

<div>
    <p>First Name: <span data-tag="firstName" data-text></span></p>
    <input id="firstNameInput" type="text" />
    <p>Last Name: <span data-tag="lastName" data-text></span></p>
    <input id="lastNameInput" type="text" />
</div>
```

### Data Attributes Explained

| Attribute | Purpose |
|-----------|---------|
| `data-tag` | Links element to a state property name |
| `data-text` | Marks element for text content updates |
| `data-color` | Marks element for color attribute updates |

### CSS for Color Binding

```css
[data-color='black'] { color: black; }
[data-color='red'] { color: red; }
[data-color='blue'] { color: blue; }
[data-color='green'] { color: green; }
```

When `data-color="red"` is set on an element, CSS attribute selectors automatically apply the color.

---

## Approach 1: Classic (Explicit Update Function)

### Concept

Create a State object with an `updateState()` method. Every time you change state, you must call this method.

### Code

```javascript
var State = {
    num: 0,
    firstName: "",
    lastName: "",
    titleColor: "black",
    updateState: function(key, value) {
        this[key] = value;

        // Update all text displays
        $("[data-text]").each(function(index, elem) {
            var tag = $(elem).attr("data-tag");
            $(elem).text(State[tag]);
        });

        // Update all color displays
        $("[data-color]").each(function(index, elem) {
            var tag = $(elem).attr("data-tag");
            $(elem).attr("data-color", State[tag]);
        });
    }
};

// Initialize on page load
$(document).ready(function() {
    State.updateState();
});

// Event handlers
$("#inc").on("click", function() {
    State.updateState("num", State.num + 1);
});

$("#dec").on("click", function() {
    State.updateState("num", State.num - 1);
});

$("#firstNameInput").on("input", function() {
    State.updateState("firstName", $(this).val());
});

$("#lastNameInput").on("input", function() {
    State.updateState("lastName", $(this).val());
});

$('[class^=button]').on("click", function(e) {
    State.updateState("titleColor", e.target.innerText);
});
```

### How It Works

1. **State Object**: Holds all application data plus the `updateState` method
2. **updateState(key, value)**:
   - Sets `this[key] = value`
   - Loops through ALL elements with `data-text`, updates their text
   - Loops through ALL elements with `data-color`, updates their attribute
3. **Event Handlers**: Must explicitly call `State.updateState()` with key and new value

### Pros
- Simple to understand
- Works in all browsers (including old ones)
- Easy to debug - you can see exactly when updates happen

### Cons
- Must remember to call `updateState()` every time
- Verbose syntax: `State.updateState("num", State.num + 1)`
- Re-renders ALL elements on every change (inefficient for large apps)

---

## Approach 2: Proxy-Based Reactivity

### Concept

Use JavaScript's `Proxy` to intercept property assignments. The DOM updates automatically when you change any state value.

### What is Proxy?

`Proxy` is a built-in JavaScript feature (ES6/2015) that wraps an object and intercepts operations like getting or setting properties.

```javascript
var proxy = new Proxy(targetObject, {
    get(obj, key) { /* intercepts reads */ },
    set(obj, key, value) { /* intercepts writes */ }
});
```

### Code

```javascript
var State = new Proxy({
    num: 0,
    firstName: "",
    lastName: "",
    titleColor: "black"
}, {
    set(obj, key, value) {
        obj[key] = value;
        $('[data-tag="' + key + '"][data-text]').text(value);
        $('[data-tag="' + key + '"][data-color]').attr("data-color", value);
        return true;
    }
});

// Initialize on page load
$(document).ready(function() {
    for (var key in State) {
        $('[data-tag="' + key + '"][data-text]').text(State[key]);
        $('[data-tag="' + key + '"][data-color]').attr("data-color", State[key]);
    }
});

// Event handlers - much simpler!
$("#inc").on("click", function() {
    State.num++;
});

$("#dec").on("click", function() {
    State.num--;
});

$("#firstNameInput").on("input", function() {
    State.firstName = $(this).val();
});

$("#lastNameInput").on("input", function() {
    State.lastName = $(this).val();
});

$('[class^=button]').on("click", function(e) {
    State.titleColor = e.target.innerText;
});
```

### How It Works

When you write `State.num++`, here's what happens behind the scenes:

```
State.num++
    ↓
Expands to: State.num = State.num + 1
    ↓
┌───────────────────────────────┐
│  Proxy intercepts             │
├───────────────────────────────┤
│  1. GET trap → returns 0      │
│  2. JavaScript does 0 + 1 = 1 │
│  3. SET trap → receives 1     │
│     - stores in obj.num       │
│     - updates DOM             │
└───────────────────────────────┘
    ↓
<span data-tag="num"> shows "1"
```

### Pros
- Natural JavaScript syntax: `State.num++` instead of `State.updateState("num", State.num + 1)`
- Impossible to forget to trigger updates
- Only updates elements matching the changed key (more efficient)
- This is how Vue.js implements reactivity

### Cons
- Not supported in Internet Explorer
- Slightly harder to understand initially
- Debugging can be trickier (updates happen "magically")

### Browser Support

| Browser | Minimum Version |
|---------|-----------------|
| Chrome | 49+ (2016) |
| Firefox | 18+ (2013) |
| Safari | 10+ (2016) |
| Edge | 12+ (2015) |
| IE | Not supported |

---

## Comparison

| Aspect | Classic | Proxy |
|--------|---------|-------|
| Syntax | `State.updateState("num", State.num + 1)` | `State.num++` |
| Forget to update? | Possible | Impossible |
| Browser support | All browsers | Modern browsers |
| Performance | Updates all elements | Updates only matching elements |
| Learning curve | Easy | Medium |
| Used by | Simple apps | Vue.js, MobX |

---

## Which Should You Use?

**Use Classic if:**
- You need to support Internet Explorer
- You want explicit control over when updates happen
- You're building a simple page

**Use Proxy if:**
- You only support modern browsers
- You want cleaner, more natural code
- You're building a more complex application

---

## Credits

- Classic approach inspired by [Meredith Matthews' Redux with jQuery example](https://codepen.io/meredithmatthews/pen/dyoOMKE)
- Proxy approach inspired by Vue.js reactivity system
