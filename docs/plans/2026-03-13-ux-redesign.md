# DnD Battle Tracker UX Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Redesign the battle tracker so every common combat action is one tap, optimized for phone/tablet use.

**Architecture:** Replace DOM-based state with a JS data model (array of character objects) that auto-persists to localStorage. Re-render cards from data on every change. Mobile-first single-column layout with floating add button and bottom sheet form. Inline HP buttons on cards. Turn tracker with round counter.

**Tech Stack:** Vanilla HTML/CSS/JS, single `index.html` file, no dependencies.

---

### Task 1: Extract JS Data Model and Render Loop

**Files:**
- Modify: `index.html` (JS section, lines 685-852)

**Context:** Currently all state lives in the DOM and is parsed back out via fragile selectors like `querySelector(".chealth").nextSibling.textContent.trim()`. We need a clean data layer.

**Step 1: Add data model and render function at top of `<script>` block**

Replace the entire `<script>` block (lines 685-852) with a new architecture. Keep the Google Analytics script (lines 643-650) untouched.

Add this at the start of the new script block:

```javascript
// --- Data Model ---
let characters = [];
let currentTurnIndex = -1; // -1 means combat not started
let currentRound = 0;
let nextId = 1;

function saveState() {
    localStorage.setItem('dnd-tracker', JSON.stringify({
        characters, currentTurnIndex, currentRound, nextId
    }));
}

function loadState() {
    try {
        const data = JSON.parse(localStorage.getItem('dnd-tracker'));
        if (data) {
            characters = data.characters || [];
            currentTurnIndex = data.currentTurnIndex ?? -1;
            currentRound = data.currentRound ?? 0;
            nextId = data.nextId ?? 1;
        }
    } catch (e) {
        characters = [];
    }
}

function sortCharacters() {
    characters.sort((a, b) => {
        if (b.initiative !== a.initiative) return b.initiative - a.initiative;
        return a.name.localeCompare(b.name);
    });
}

function addCharacter(name, hp, initiative, notes) {
    characters.push({
        id: nextId++,
        name,
        hp: parseInt(hp),
        maxHp: parseInt(hp),
        initiative: parseInt(initiative),
        notes: notes || ''
    });
    sortCharacters();
    saveState();
    render();
}

function deleteCharacter(id) {
    const idx = characters.findIndex(c => c.id === id);
    if (idx === -1) return;
    // Adjust turn index if needed
    if (currentTurnIndex >= 0) {
        if (idx < currentTurnIndex) currentTurnIndex--;
        else if (idx === currentTurnIndex) {
            if (currentTurnIndex >= characters.length - 1) {
                currentTurnIndex = 0;
                currentRound++;
            }
        }
    }
    characters.splice(idx, 1);
    if (characters.length === 0) {
        currentTurnIndex = -1;
        currentRound = 0;
    }
    saveState();
    render();
}

function adjustHp(id, delta) {
    const char = characters.find(c => c.id === id);
    if (!char) return;
    char.hp = Math.max(0, Math.min(char.maxHp, char.hp + delta));
    saveState();
    render();
}

function updateCharacter(id, name, hp, maxHp, initiative, notes) {
    const char = characters.find(c => c.id === id);
    if (!char) return;
    char.name = name;
    char.hp = parseInt(hp);
    char.maxHp = parseInt(maxHp);
    char.initiative = parseInt(initiative);
    char.notes = notes || '';
    sortCharacters();
    saveState();
    render();
}

function nextTurn() {
    if (characters.length === 0) return;
    if (currentTurnIndex === -1) {
        currentTurnIndex = 0;
        currentRound = 1;
    } else {
        currentTurnIndex++;
        if (currentTurnIndex >= characters.length) {
            currentTurnIndex = 0;
            currentRound++;
        }
    }
    saveState();
    render();
}

function resetCombat() {
    currentTurnIndex = -1;
    currentRound = 0;
    saveState();
    render();
}

function clearAll() {
    characters = [];
    currentTurnIndex = -1;
    currentRound = 0;
    nextId = 1;
    saveState();
    render();
}
```

**Step 2: Add the render function**

```javascript
function render() {
    const list = document.getElementById('characterList');
    list.innerHTML = '';

    // Update round display
    const roundDisplay = document.getElementById('roundDisplay');
    if (roundDisplay) {
        roundDisplay.textContent = currentRound > 0 ? `Round ${currentRound}` : '';
    }

    characters.forEach((char, index) => {
        const li = document.createElement('li');
        li.classList.add('character');
        if (index === currentTurnIndex) li.classList.add('active-turn');

        const hpPercent = char.maxHp > 0 ? (char.hp / char.maxHp) * 100 : 0;
        let hpColor = 'var(--color-hp-good)';
        if (hpPercent <= 25) hpColor = 'var(--color-hp-danger)';
        else if (hpPercent <= 50) hpColor = 'var(--color-hp-warning)';

        const notesHtml = char.notes
            ? `<p class="char-notes"><strong>Notes:</strong> <span>${char.notes.replace(/</g, '&lt;').replace(/>/g, '&gt;')}</span></p>`
            : '';

        li.innerHTML = `
            <button class="delete" aria-label="Delete ${char.name.replace(/</g, '&lt;')}" data-id="${char.id}"></button>
            <div class="card-body" data-id="${char.id}">
                <div class="card-header">
                    <h2 class="cname">${char.name.replace(/</g, '&lt;').replace(/>/g, '&gt;')}</h2>
                    <span class="initiative-badge">${char.initiative}</span>
                </div>
                <div class="hp-bar-container">
                    <div class="hp-bar" style="width: ${hpPercent}%; background: ${hpColor};"></div>
                </div>
                <div class="hp-controls">
                    <button class="hp-btn hp-minus" data-id="${char.id}" data-delta="-5">-5</button>
                    <button class="hp-btn hp-minus" data-id="${char.id}" data-delta="-1">-1</button>
                    <span class="hp-display">${char.hp} / ${char.maxHp}</span>
                    <button class="hp-btn hp-plus" data-id="${char.id}" data-delta="1">+1</button>
                    <button class="hp-btn hp-plus" data-id="${char.id}" data-delta="5">+5</button>
                </div>
                ${notesHtml}
            </div>
        `;

        list.appendChild(li);
    });

    // Event delegation is set up once in init(), not here
}
```

**Step 3: Add init function and event delegation**

```javascript
function init() {
    loadState();

    const list = document.getElementById('characterList');

    // Event delegation for character list
    list.addEventListener('click', function(e) {
        const hpBtn = e.target.closest('.hp-btn');
        if (hpBtn) {
            e.stopPropagation();
            adjustHp(parseInt(hpBtn.dataset.id), parseInt(hpBtn.dataset.delta));
            return;
        }

        const deleteBtn = e.target.closest('.delete');
        if (deleteBtn) {
            e.stopPropagation();
            const id = parseInt(deleteBtn.dataset.id);
            // Fade out animation
            const li = deleteBtn.closest('.character');
            li.classList.add('fade-out');
            li.addEventListener('animationend', () => deleteCharacter(id));
            return;
        }

        const cardBody = e.target.closest('.card-body');
        if (cardBody) {
            openEditModal(parseInt(cardBody.dataset.id));
        }
    });

    // Add character form (desktop sidebar)
    document.getElementById('characterForm').addEventListener('submit', function(e) {
        e.preventDefault();
        const name = document.getElementById('name').value.trim();
        const hp = document.getElementById('health').value;
        const initiative = document.getElementById('initiative').value;
        const notes = document.getElementById('notes').value;
        if (!name || !hp || !initiative) return;
        addCharacter(name, hp, initiative, notes);
        this.reset();
    });

    // Mobile add form (bottom sheet)
    document.getElementById('mobileCharacterForm')?.addEventListener('submit', function(e) {
        e.preventDefault();
        const name = document.getElementById('mName').value.trim();
        const hp = document.getElementById('mHealth').value;
        const initiative = document.getElementById('mInitiative').value;
        const notes = document.getElementById('mNotes').value;
        if (!name || !hp || !initiative) return;
        addCharacter(name, hp, initiative, notes);
        this.reset();
        closeMobileForm();
    });

    // Turn controls
    document.getElementById('nextTurnBtn')?.addEventListener('click', nextTurn);
    document.getElementById('resetCombatBtn')?.addEventListener('click', resetCombat);
    document.getElementById('clearAllBtn')?.addEventListener('click', function() {
        if (confirm('Clear all characters?')) clearAll();
    });

    // Mobile FAB
    document.getElementById('fab')?.addEventListener('click', openMobileForm);
    document.getElementById('mobileFormOverlay')?.addEventListener('click', closeMobileForm);
    document.getElementById('mobileFormClose')?.addEventListener('click', closeMobileForm);

    // Edit modal
    const modal = document.getElementById('myModal');
    const closeBtn = modal.querySelector('.close');
    closeBtn.addEventListener('click', closeEditModal);

    let clickStartedOutsideModal = false;
    window.addEventListener('mousedown', (e) => { clickStartedOutsideModal = e.target === modal; });
    window.addEventListener('click', (e) => {
        if (e.target === modal && clickStartedOutsideModal) closeEditModal();
    });

    render();
}

let editingId = null;

function openEditModal(id) {
    const char = characters.find(c => c.id === id);
    if (!char) return;
    editingId = id;
    const modal = document.getElementById('myModal');
    document.getElementById('editName').value = char.name;
    document.getElementById('editHealth').value = char.hp;
    document.getElementById('editMaxHealth').value = char.maxHp;
    document.getElementById('editInitiative').value = char.initiative;
    document.getElementById('editNotes').value = char.notes;
    modal.style.display = 'block';
    requestAnimationFrame(() => modal.classList.add('show'));
}

function closeEditModal() {
    const modal = document.getElementById('myModal');
    modal.classList.remove('show');
    setTimeout(() => { modal.style.display = 'none'; }, 100);
}

document.getElementById('editForm')?.addEventListener('submit', function(e) {
    e.preventDefault();
    if (editingId == null) return;
    updateCharacter(
        editingId,
        document.getElementById('editName').value.trim(),
        document.getElementById('editHealth').value,
        document.getElementById('editMaxHealth').value,
        document.getElementById('editInitiative').value,
        document.getElementById('editNotes').value
    );
    closeEditModal();
});

function openMobileForm() {
    document.getElementById('mobileFormSheet').classList.add('open');
    document.getElementById('mobileFormOverlay').classList.add('open');
    document.getElementById('mName').focus();
}

function closeMobileForm() {
    document.getElementById('mobileFormSheet').classList.remove('open');
    document.getElementById('mobileFormOverlay').classList.remove('open');
}

init();
```

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: replace DOM state with JS data model, add render loop and persistence"
```

---

### Task 2: Rewrite HTML Structure

**Files:**
- Modify: `index.html` (HTML body, lines 651-683)

**Context:** The HTML needs to support: mobile FAB + bottom sheet, turn tracker bar, new card layout with HP buttons, and the edit modal with maxHp field.

**Step 1: Replace the HTML body content (between `<body>` and `<script>`) with:**

```html
<body>
    <!-- Desktop sidebar -->
    <div class="sidebar" id="sidebar">
        <h1>Battle Tracker</h1>
        <form id="characterForm">
            <input type="text" id="name" placeholder="Name" required>
            <div class="form-row">
                <input type="number" id="health" placeholder="HP" required>
                <input type="number" id="initiative" placeholder="Initiative" required>
            </div>
            <textarea id="notes" placeholder="Notes (optional)" rows="2"></textarea>
            <input type="submit" value="Add Character">
        </form>
        <button id="clearAllBtn" class="clear-btn">Clear All</button>
    </div>

    <!-- Main content -->
    <div class="main-content">
        <div class="turn-bar">
            <button id="nextTurnBtn" class="turn-btn">Next Turn</button>
            <span id="roundDisplay" class="round-display"></span>
            <button id="resetCombatBtn" class="turn-btn turn-btn-secondary">Reset</button>
        </div>
        <ul id="characterList"></ul>
    </div>

    <!-- Mobile FAB -->
    <button id="fab" class="fab" aria-label="Add character">+</button>

    <!-- Mobile bottom sheet -->
    <div id="mobileFormOverlay" class="mobile-overlay"></div>
    <div id="mobileFormSheet" class="bottom-sheet">
        <div class="sheet-handle"></div>
        <button id="mobileFormClose" class="sheet-close"></button>
        <h2>Add Character</h2>
        <form id="mobileCharacterForm">
            <input type="text" id="mName" placeholder="Name" required>
            <div class="form-row">
                <input type="number" id="mHealth" placeholder="HP" required>
                <input type="number" id="mInitiative" placeholder="Initiative" required>
            </div>
            <textarea id="mNotes" placeholder="Notes (optional)" rows="2"></textarea>
            <input type="submit" value="Add Character">
        </form>
    </div>

    <!-- Edit modal -->
    <div id="myModal" class="modal">
        <div class="modal-content">
            <span class="close"></span>
            <h2>Edit Character</h2>
            <form id="editForm">
                <p class="modalp">Name:</p>
                <input type="text" id="editName" placeholder="Name" required>
                <p class="modalp">HP / Max HP:</p>
                <div class="form-row">
                    <input type="number" id="editHealth" placeholder="HP" required>
                    <input type="number" id="editMaxHealth" placeholder="Max HP" required>
                </div>
                <p class="modalp">Initiative:</p>
                <input type="number" id="editInitiative" placeholder="Initiative" required>
                <p class="modalp">Notes:</p>
                <textarea id="editNotes" placeholder="Notes (optional)" rows="3"></textarea>
                <input type="submit" value="Save Changes">
            </form>
        </div>
    </div>
```

**Step 2: Commit**

```bash
git add index.html
git commit -m "feat: rewrite HTML for mobile-first layout with turn bar and bottom sheet"
```

---

### Task 3: Rewrite CSS for Mobile-First Layout

**Files:**
- Modify: `index.html` (CSS section, lines 7-640)

**Context:** Replace the entire `<style>` block. Keep the same color variables. New layout: mobile-first single column, sidebar hidden on mobile, FAB + bottom sheet for adding characters. New card design with HP bar and inline buttons. Turn tracker bar at top.

**Step 1: Replace the `<style>` block with new mobile-first CSS**

Key sections to include (this is the full replacement):

```css
:root {
    /* Keep existing color variables exactly */
    --color-background-dark: #2c2019;
    --color-background-darker: #241812;
    --color-background-light: #3a2820;
    --color-background-lighter: #4a332a;
    --color-text-light: #e8d5b5;
    --color-text-dark: #2c2019;
    --color-text-filled: #cd8c50;
    --color-accent: #8b4513;
    --color-accent-light: #a0522d;
    --color-accent-lighter: #cd8c50;
    --color-transparent-dark: rgba(44, 32, 25, 0.9);
    --color-transparent-darker: rgba(44, 32, 25, 0.95);
    --color-transparent-accent: rgba(139, 69, 19, 0.3);
    --color-transparent-accent-hover: rgba(139, 69, 19, 0.5);
    --color-transparent-shadow: rgba(139, 69, 19, 0.2);
    --color-form-background: rgba(232, 213, 181, 0.05);
    --color-input-focus: #e8d5b5;
    --color-placeholder-focus: #2c2019;

    /* New HP colors */
    --color-hp-good: #4a7c3f;
    --color-hp-warning: #b8860b;
    --color-hp-danger: #8b2500;

    /* New active turn */
    --color-turn-glow: rgba(205, 140, 80, 0.6);
}

* { margin: 0; padding: 0; box-sizing: border-box; }
html, body { height: 100%; overflow: hidden; }
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, var(--color-background-dark) 0%, var(--color-background-light) 100%);
    color: var(--color-text-light);
    display: flex;
    flex-direction: column;
}

/* --- Sidebar (desktop only) --- */
.sidebar {
    display: none; /* hidden on mobile */
    flex-direction: column;
    width: 340px;
    min-width: 300px;
    height: 100vh;
    overflow-y: auto;
    background: linear-gradient(180deg, var(--color-background-dark) 0%, var(--color-background-darker) 100%);
    border-right: 1px solid var(--color-transparent-accent);
    padding: 20px;
    box-shadow: 2px 0 10px rgba(0,0,0,0.2);
}
.sidebar h1 {
    text-align: center;
    font-size: 1.8em;
    color: var(--color-text-light);
    margin-bottom: 20px;
    padding-bottom: 12px;
    border-bottom: 2px solid var(--color-transparent-accent);
}
.sidebar form {
    background: var(--color-form-background);
    padding: 20px;
    border-radius: 12px;
    border: 1px solid var(--color-transparent-accent);
}
.clear-btn {
    margin-top: 12px;
    width: 100%;
    padding: 8px;
    background: transparent;
    border: 1px solid var(--color-transparent-accent);
    color: var(--color-text-light);
    border-radius: 8px;
    cursor: pointer;
    font-size: 13px;
    opacity: 0.7;
    transition: opacity 0.2s;
}
.clear-btn:hover { opacity: 1; }

/* --- Main content --- */
.main-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    height: 100vh;
    overflow: hidden;
}

/* --- Turn bar --- */
.turn-bar {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px 16px;
    background: var(--color-background-darker);
    border-bottom: 1px solid var(--color-transparent-accent);
    flex-shrink: 0;
}
.turn-btn {
    padding: 8px 20px;
    background: var(--color-accent);
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.2s;
    white-space: nowrap;
}
.turn-btn:hover { background: var(--color-accent-light); }
.turn-btn:active { transform: scale(0.97); }
.turn-btn-secondary {
    background: transparent;
    border: 1px solid var(--color-transparent-accent);
    color: var(--color-text-light);
    padding: 8px 14px;
    font-weight: 400;
}
.turn-btn-secondary:hover { background: var(--color-transparent-accent); }
.round-display {
    font-size: 14px;
    color: var(--color-accent-lighter);
    font-weight: 600;
    margin-left: auto;
}

/* --- Character list --- */
#characterList {
    list-style: none;
    padding: 12px;
    overflow-y: auto;
    flex: 1;
}

/* --- Character card --- */
.character {
    background: linear-gradient(180deg, var(--color-background-light) 0%, var(--color-background-dark) 100%);
    border-radius: 12px;
    margin-bottom: 10px;
    position: relative;
    overflow: hidden;
    box-shadow: 0 2px 6px rgba(0,0,0,0.15);
    transition: box-shadow 0.2s, transform 0.2s;
}
.character::after {
    content: '';
    position: absolute;
    left: 0; top: 6px; bottom: 6px;
    width: 3px;
    border-radius: 2px;
    background: linear-gradient(to bottom, var(--color-accent-lighter), var(--color-accent));
}
.character:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

/* Active turn highlight */
.character.active-turn {
    box-shadow: 0 0 0 2px var(--color-turn-glow), 0 4px 16px rgba(205, 140, 80, 0.3);
}
.character.active-turn::before {
    content: '\25B6'; /* right-pointing triangle */
    position: absolute;
    left: 10px;
    top: 50%;
    transform: translateY(-50%);
    color: var(--color-accent-lighter);
    font-size: 14px;
    z-index: 1;
}

/* --- Card layout --- */
.card-body {
    padding: 12px 14px 10px 20px;
    cursor: pointer;
}
.card-header {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    margin-bottom: 4px;
}
.cname {
    font-size: 1.3em;
    font-weight: 600;
    color: var(--color-text-light);
    margin: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    flex: 1;
    padding-right: 10px;
}
.initiative-badge {
    background: var(--color-transparent-accent);
    color: var(--color-accent-lighter);
    padding: 2px 10px;
    border-radius: 12px;
    font-size: 0.85em;
    font-weight: 600;
    white-space: nowrap;
}

/* --- HP bar --- */
.hp-bar-container {
    height: 4px;
    background: rgba(255,255,255,0.08);
    border-radius: 2px;
    margin: 6px 0;
    overflow: hidden;
}
.hp-bar {
    height: 100%;
    border-radius: 2px;
    transition: width 0.3s ease, background 0.3s ease;
}

/* --- HP controls --- */
.hp-controls {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    margin: 8px 0 4px;
}
.hp-btn {
    width: 40px;
    height: 34px;
    border: 1px solid var(--color-transparent-accent);
    border-radius: 8px;
    background: var(--color-background-darker);
    color: var(--color-text-light);
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.15s, transform 0.1s;
    display: flex;
    align-items: center;
    justify-content: center;
}
.hp-btn:hover { background: var(--color-transparent-accent-hover); }
.hp-btn:active { transform: scale(0.93); }
.hp-minus { color: var(--color-hp-danger); border-color: rgba(139, 37, 0, 0.3); }
.hp-minus:hover { background: rgba(139, 37, 0, 0.15); }
.hp-plus { color: var(--color-hp-good); border-color: rgba(74, 124, 63, 0.3); }
.hp-plus:hover { background: rgba(74, 124, 63, 0.15); }
.hp-display {
    min-width: 70px;
    text-align: center;
    font-weight: 600;
    font-size: 1em;
    color: var(--color-text-light);
}

/* --- Notes --- */
.char-notes {
    font-size: 0.85em;
    color: var(--color-text-light);
    opacity: 0.8;
    margin-top: 6px;
    line-height: 1.4;
    white-space: pre-wrap;
}
.char-notes strong { color: var(--color-accent-lighter); }

/* --- Delete button --- */
.delete {
    position: absolute;
    right: 8px; top: 8px;
    width: 26px; height: 26px;
    border-radius: 50%;
    background: var(--color-background-dark);
    border: 1px solid var(--color-transparent-accent);
    color: var(--color-text-light);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    font-size: 18px;
    line-height: 0;
    opacity: 0;
    transition: opacity 0.2s, background 0.2s;
    z-index: 2;
    padding: 0;
    font-family: Arial, sans-serif;
}
.delete::after { content: '\00d7'; position: relative; top: 1px; }
.character:hover .delete { opacity: 0.7; }
.delete:hover { opacity: 1 !important; background: var(--color-accent); }

/* --- Animations --- */
.fade-in { animation: fadeIn 0.3s ease forwards; opacity: 0; }
.fade-out { animation: fadeOut 0.25s ease forwards; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(-8px); } to { opacity: 1; transform: none; } }
@keyframes fadeOut { from { opacity: 1; } to { opacity: 0; transform: translateX(30px); } }

/* --- FAB (mobile only) --- */
.fab {
    display: flex; /* shown on mobile, hidden on desktop */
    position: fixed;
    bottom: 24px; right: 20px;
    width: 56px; height: 56px;
    border-radius: 50%;
    background: var(--color-accent);
    color: white;
    font-size: 32px;
    border: none;
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 14px rgba(0,0,0,0.35);
    cursor: pointer;
    z-index: 10;
    transition: transform 0.2s, background 0.2s;
}
.fab:hover { background: var(--color-accent-light); }
.fab:active { transform: scale(0.92); }

/* --- Mobile bottom sheet --- */
.mobile-overlay {
    display: none;
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.5);
    z-index: 20;
}
.mobile-overlay.open { display: block; }

.bottom-sheet {
    position: fixed;
    bottom: 0; left: 0; right: 0;
    background: linear-gradient(180deg, var(--color-background-light) 0%, var(--color-background-dark) 100%);
    border-radius: 20px 20px 0 0;
    padding: 16px 20px 30px;
    transform: translateY(100%);
    transition: transform 0.3s ease;
    z-index: 25;
}
.bottom-sheet.open { transform: translateY(0); }
.sheet-handle {
    width: 40px; height: 4px;
    background: var(--color-transparent-accent-hover);
    border-radius: 2px;
    margin: 0 auto 12px;
}
.sheet-close {
    position: absolute;
    right: 16px; top: 16px;
    width: 28px; height: 28px;
    border-radius: 50%;
    background: var(--color-background-dark);
    border: 1px solid var(--color-transparent-accent);
    color: var(--color-text-light);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    font-size: 18px;
    padding: 0;
    font-family: Arial, sans-serif;
}
.sheet-close::after { content: '\00d7'; }
.bottom-sheet h2 {
    color: var(--color-text-light);
    font-size: 1.3em;
    margin-bottom: 14px;
}

/* --- Form inputs (shared) --- */
.form-row {
    display: flex;
    gap: 10px;
}
.form-row input { flex: 1; }

input[type="text"],
input[type="number"],
textarea {
    width: 100%;
    padding: 10px 12px;
    margin-bottom: 10px;
    border: 2px solid var(--color-transparent-accent);
    border-radius: 8px;
    font-size: 15px;
    background: var(--color-transparent-dark);
    color: var(--color-text-filled);
    transition: background 0.2s, border-color 0.2s;
    resize: vertical;
}
input[type="text"]:hover,
input[type="number"]:hover,
textarea:hover {
    background: var(--color-transparent-darker);
    border-color: var(--color-transparent-accent-hover);
}
input[type="text"]:focus,
input[type="number"]:focus,
textarea:focus {
    background: var(--color-input-focus);
    color: var(--color-text-dark);
    border-color: var(--color-accent);
    box-shadow: 0 0 0 3px var(--color-transparent-shadow);
    outline: none;
}
::placeholder { color: var(--color-text-light); opacity: 0.5; }
input:focus::placeholder,
textarea:focus::placeholder { color: var(--color-placeholder-focus); }

input[type="submit"] {
    width: 100%;
    padding: 10px;
    background: var(--color-accent);
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 15px;
    font-weight: 600;
    transition: background 0.2s;
}
input[type="submit"]:hover { background: var(--color-accent-light); }
input[type="submit"]:active { transform: scale(0.98); }

/* --- Modal --- */
.modal {
    display: none;
    position: fixed;
    inset: 0;
    background: var(--color-transparent-dark);
    opacity: 0;
    transition: opacity 0.1s;
    z-index: 30;
}
.modal.show { display: block; opacity: 1; }
.modal-content {
    position: relative;
    background: linear-gradient(180deg, var(--color-background-light) 0%, var(--color-background-dark) 100%);
    margin: 8% auto;
    padding: 24px;
    border: 1px solid var(--color-transparent-accent);
    border-radius: 16px;
    width: 92%;
    max-width: 440px;
    max-height: 85vh;
    overflow-y: auto;
    color: var(--color-text-light);
}
.modal-content h2 {
    font-size: 1.4em;
    margin-bottom: 16px;
}
.modalp { margin-bottom: 4px; margin-top: 8px; font-size: 0.9em; opacity: 0.8; }
.close {
    position: absolute;
    right: 14px; top: 14px;
    width: 28px; height: 28px;
    border-radius: 50%;
    background: var(--color-background-dark);
    border: 1px solid var(--color-transparent-accent);
    color: var(--color-text-light);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    font-size: 18px;
    padding: 0;
    font-family: Arial, sans-serif;
    opacity: 0.8;
}
.close::after { content: '\00d7'; position: relative; top: 1px; }
.close:hover { background: var(--color-accent); opacity: 1; }

/* --- Desktop layout (768px+) --- */
@media screen and (min-width: 769px) {
    body { flex-direction: row; }
    .sidebar { display: flex; }
    .fab { display: none; }
    .main-content { height: 100vh; }
    .character.active-turn::before { left: 12px; }
    .cname { font-size: 1.5em; }
    .hp-btn { width: 44px; height: 36px; }
    .card-body { padding: 14px 18px 12px 24px; }
}

/* --- Small phones --- */
@media screen and (max-width: 380px) {
    .hp-btn { width: 34px; height: 30px; font-size: 13px; }
    .hp-display { font-size: 0.85em; min-width: 58px; }
    .cname { font-size: 1.1em; }
    .turn-bar { padding: 8px 10px; }
    .turn-btn { padding: 6px 14px; font-size: 13px; }
}
```

**Step 2: Commit**

```bash
git add index.html
git commit -m "feat: mobile-first CSS with HP bars, turn tracker, FAB, and bottom sheet"
```

---

### Task 4: Integration Testing and Polish

**Files:**
- Modify: `index.html` (minor tweaks as needed)

**Step 1: Open in browser and test the following scenarios**

1. Add 3 characters with different initiative values → verify they sort correctly
2. Tap HP -1 and -5 buttons → verify HP updates and bar changes color
3. HP should not go below 0 or above maxHp
4. Tap "Next Turn" → verify first character highlights, round shows "Round 1"
5. Advance through all characters → verify round increments to "Round 2"
6. Tap "Reset" → verify highlight removed, round display clears
7. Refresh the page → verify all characters persist
8. Delete a character → verify fade-out and removal
9. Click a card body → verify edit modal opens with correct values including maxHp
10. Test on mobile viewport → verify sidebar hidden, FAB visible, bottom sheet works

**Step 2: Fix any issues found**

**Step 3: Commit**

```bash
git add index.html
git commit -m "fix: polish and integration testing fixes"
```

---

### Task 5: Touch Polish for Mobile

**Files:**
- Modify: `index.html` (CSS section)

**Step 1: Add touch-specific improvements**

- Ensure all tap targets are at least 44x44px (Apple HIG minimum)
- Add `-webkit-tap-highlight-color: transparent` to interactive elements
- Add `touch-action: manipulation` to prevent double-tap zoom on buttons
- Ensure delete button is always visible on touch devices (no hover state)

Add to the CSS:

```css
/* Touch devices */
@media (hover: none) {
    .delete { opacity: 0.5; }
    .hp-btn, .turn-btn, .fab, .delete, input[type="submit"] {
        -webkit-tap-highlight-color: transparent;
        touch-action: manipulation;
    }
}
```

**Step 2: Commit**

```bash
git add index.html
git commit -m "feat: touch-friendly polish for mobile devices"
```
