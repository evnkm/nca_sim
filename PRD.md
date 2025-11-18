# Product Requirements Document: Economy of Things

## Economic Strategy Evolution Simulator using Neural Cellular Automata

**Version:** 1.0  
**Date:** November 18, 2025  
**Author:** Evan  
**Target Completion:** 4 weeks

---

## Executive Summary

Economy of Things is a web-based multiplayer game where players design economic policies for cellular automata that compete in an evolutionary environment. The system demonstrates how macro-economic patterns emerge from micro-level rules, making abstract economic concepts tangible and interactive.

**Core Innovation:** Using Neural Cellular Automata to simulate economic agent behavior where local trade rules generate emergent economic systems (capitalism, socialism, mixed economies) without top-down coordination.

---

## 1. Project Goals

### Primary Objectives

1. Create engaging multiplayer experience where economic strategies compete evolutionarily
2. Demonstrate emergent economic phenomena (monopolies, market crashes, inequality, cooperation)
3. Build viral, shareable content showcasing NCA applications to social systems
4. Produce portfolio piece for Google DeepMind applications (Mordvintsev, Parker-Holder teams)

### Success Metrics

- 1,000+ unique players in first month
- 50+ custom strategies created by community
- 10+ emergent behaviors documented
- Social shares: 100+ tweets, 5+ YouTube videos, 3+ Reddit posts with 100+ upvotes

---

## 2. Technical Architecture

### 2.1 Technology Stack

**Frontend:**

- **Framework:** Vanilla JavaScript + HTML5 Canvas OR Three.js
- **Rendering:** WebGL for grid visualization (target: 60 FPS with 100,000+ cells)
- **UI Framework:** Minimal CSS + vanilla JS (no React/Vue to reduce complexity)
- **WebSocket Client:** Native WebSocket API for real-time updates

**Backend:**

- **Language:** Python 3.10+
- **Framework:** FastAPI (lightweight, async support)
- **WebSocket Server:** FastAPI WebSocket support
- **Database:** SQLite for development, PostgreSQL for production (store strategies, match results)
- **Hosting:** Vercel/Netlify for frontend, Railway/Render for backend

**Computation:**

- **Primary:** Client-side JavaScript (CA updates run in browser)
- **Server Role:** Matchmaking, leaderboard, strategy storage only
- **Fallback:** Server-side Python if client can't handle computation

### 2.2 Core Data Structures

```python
# Cell State (each grid cell)
class Cell:
    resource: float  # 0.0 to 100.0, current energy/capital
    policy_id: str   # reference to player's strategy
    age: int         # generations survived
    lineage: str     # identifier for tracking evolutionary lines
    is_alive: bool   # active or dead

# Economic Policy (player-designed)
class EconomicPolicy:
    id: str
    name: str
    creator: str

    # Core parameters (0.0 to 1.0 sliders)
    accumulation_rate: float    # 0=spend all, 1=hoard all
    trade_openness: float       # 0=autarky, 1=free trade
    redistribution_rate: float  # 0=keep gains, 1=share equally
    investment_rate: float      # 0=extract, 1=reinvest
    risk_tolerance: float       # 0=conservative, 1=aggressive

    # Derived behaviors
    reproduction_threshold: float  # resource level to spawn
    trade_radius: int              # how many neighbors to trade with

# Grid State
class Grid:
    width: int  # 200-500 cells
    height: int
    cells: List[List[Cell]]
    generation: int
    statistics: GridStatistics

class GridStatistics:
    total_cells: int
    resources_by_policy: Dict[str, float]
    gini_coefficient: float
    survival_rates: Dict[str, float]
```

### 2.3 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser Client                        │
│  ┌─────────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Policy Editor  │  │  CA Engine   │  │  Visualizer   │  │
│  │  (UI Controls)  │──│  (WebGL/JS)  │──│  (Canvas/GL)  │  │
│  └─────────────────┘  └──────────────┘  └───────────────┘  │
│           │                    │                  │          │
│           └────────────────────┴──────────────────┘          │
│                              │                               │
│                         WebSocket                            │
└──────────────────────────────┼──────────────────────────────┘
                               │
┌──────────────────────────────┼──────────────────────────────┐
│                      FastAPI Backend                         │
│  ┌─────────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Matchmaking    │  │  Leaderboard │  │  Strategy DB  │  │
│  │  (Room Mgmt)    │  │  (Rankings)  │  │  (SQLite)     │  │
│  └─────────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Core Game Mechanics

### 3.1 Cellular Automata Rules

**Update Loop (executed every frame, ~16ms):**

```python
def update_cell(cell, neighbors, grid):
    """
    Update single cell state based on policy and neighbors
    """
    if not cell.is_alive:
        return

    policy = get_policy(cell.policy_id)

    # 1. RESOURCE DECAY (entropy)
    cell.resource *= 0.99  # 1% decay per step

    # 2. TRADE PHASE
    trade_partners = get_neighbors_within_radius(
        cell,
        policy.trade_radius
    )

    for partner in trade_partners:
        if should_trade(cell, partner, policy):
            execute_trade(cell, partner, policy)

    # 3. REDISTRIBUTION PHASE
    if policy.redistribution_rate > 0:
        redistribute_resources(cell, neighbors, policy)

    # 4. INVESTMENT PHASE
    if policy.investment_rate > 0:
        cell.resource += generate_returns(cell, policy)

    # 5. REPRODUCTION CHECK
    if cell.resource >= policy.reproduction_threshold:
        spawn_offspring(cell, empty_neighbors, policy)
        cell.resource *= 0.5  # reproduction costs half resources

    # 6. DEATH CHECK
    if cell.resource <= 0:
        cell.is_alive = False

    cell.age += 1
```

**Trade Mechanics:**

```python
def should_trade(cell_a, cell_b, policy_a):
    """
    Determine if trade occurs based on policies
    """
    # Only trade if both parties willing
    policy_b = get_policy(cell_b.policy_id)

    trade_willingness = (policy_a.trade_openness + policy_b.trade_openness) / 2
    resource_difference = abs(cell_a.resource - cell_b.resource)

    # Trade more likely when:
    # - High trade openness
    # - Large resource difference (comparative advantage)
    trade_probability = trade_willingness * (resource_difference / 100)

    return random() < trade_probability

def execute_trade(cell_a, cell_b, policy_a):
    """
    Execute resource exchange
    """
    policy_b = get_policy(cell_b.policy_id)

    # Trade volume based on risk tolerance
    trade_volume = min(
        cell_a.resource * policy_a.risk_tolerance * 0.1,
        cell_b.resource * policy_b.risk_tolerance * 0.1
    )

    # Simple exchange: both parties gain from trade
    # (models gains from trade, not zero-sum)
    efficiency = (policy_a.trade_openness + policy_b.trade_openness) / 2
    gain = trade_volume * efficiency * 0.2

    cell_a.resource += gain
    cell_b.resource += gain
```

**Redistribution Mechanics:**

```python
def redistribute_resources(cell, neighbors, policy):
    """
    Share resources with neighbors based on policy
    """
    alive_neighbors = [n for n in neighbors if n.is_alive]
    if not alive_neighbors:
        return

    # Amount to redistribute
    redistribution_amount = cell.resource * policy.redistribution_rate * 0.1

    # Equal distribution to all neighbors
    per_neighbor = redistribution_amount / len(alive_neighbors)

    cell.resource -= redistribution_amount
    for neighbor in alive_neighbors:
        neighbor.resource += per_neighbor
```

**Investment Returns:**

```python
def generate_returns(cell, policy):
    """
    Generate investment returns based on accumulation
    """
    # Returns scale with accumulated capital (compound growth)
    capital = cell.resource * policy.accumulation_rate
    base_return = capital * 0.02  # 2% return per step

    # Risk tolerance affects return variance
    risk_factor = policy.risk_tolerance
    volatility = random_normal(0, risk_factor * 0.5)

    return base_return * (1 + volatility)
```

### 3.2 Grid Initialization

**Match Start:**

```python
def initialize_grid(width, height, policies):
    """
    Set up initial grid state for match
    """
    grid = Grid(width, height)

    # Randomly distribute policies
    for x in range(width):
        for y in range(height):
            # 30% starting density
            if random() < 0.3:
                policy = random.choice(policies)
                grid.cells[x][y] = Cell(
                    resource=50.0,  # everyone starts equal
                    policy_id=policy.id,
                    age=0,
                    lineage=generate_id(),
                    is_alive=True
                )
            else:
                grid.cells[x][y] = Cell(is_alive=False)

    return grid
```

### 3.3 Win Conditions & Match Length

**Match Duration:** 5,000 generations (~5 minutes at 15 FPS)

**Victory Conditions (tracked simultaneously):**

1. **Dominance:** Policy controlling most cells at end
2. **Total Resources:** Policy with highest aggregate resources
3. **Survival:** Policy with highest average cell age
4. **Gini:** Policy creating most equal distribution (lowest Gini coefficient)

**Scoring:**

```python
def calculate_scores(grid, policies):
    scores = {}
    for policy in policies:
        policy_cells = get_cells_by_policy(grid, policy.id)

        scores[policy.id] = {
            'dominance': len(policy_cells) / grid.total_cells,
            'resources': sum(c.resource for c in policy_cells),
            'avg_age': mean(c.age for c in policy_cells),
            'gini': calculate_gini(policy_cells)
        }
    return scores
```

---

## 4. User Interface Specification

### 4.1 Screen Layout

```
┌────────────────────────────────────────────────────────────────┐
│  ECONOMY OF THINGS                    [Settings] [Leaderboard] │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────┐  ┌────────────────────┐   │
│  │                                │  │   POLICY EDITOR    │   │
│  │                                │  │                    │   │
│  │                                │  │ Name: [________]   │   │
│  │         GRID DISPLAY           │  │                    │   │
│  │      (Canvas/WebGL)            │  │ Accumulation:     │   │
│  │                                │  │ [====    ] 0.6     │   │
│  │      Color-coded by            │  │                    │   │
│  │      policy                    │  │ Trade Openness:   │   │
│  │                                │  │ [======  ] 0.7     │   │
│  │                                │  │                    │   │
│  │                                │  │ Redistribution:   │   │
│  │                                │  │ [==      ] 0.3     │   │
│  │                                │  │                    │   │
│  └────────────────────────────────┘  │ Investment:       │   │
│                                       │ [=====   ] 0.6     │   │
│  Generation: 1,234 / 5,000           │                    │   │
│  [=============================>]     │ Risk Tolerance:   │   │
│                                       │ [========] 0.9     │   │
│  ┌────────────────────────────────┐  │                    │   │
│  │   LIVE STATISTICS              │  │ [Save Policy]     │   │
│  │                                │  │ [Start Match]     │   │
│  │  Top Policy: Capitalist (42%)  │  └────────────────────┘   │
│  │  Gini Coefficient: 0.67        │                           │
│  │  Total Resources: 8,432        │  ┌────────────────────┐   │
│  │  Alive Cells: 2,156            │  │  PRESET POLICIES   │   │
│  │                                │  │                    │   │
│  │  [Policy Distribution Chart]   │  │  • Pure Capitalist│   │
│  │                                │  │  • Dem Socialist  │   │
│  └────────────────────────────────┘  │  • Mercantilist   │   │
│                                       │  • Cooperative    │   │
│  [⏸ Pause] [⏩ 2x Speed] [📊 Stats]   │  • Anarchist      │   │
│                                       └────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 Visual Design

**Color Scheme:**

- Background: `#0a0e27` (dark blue)
- Grid lines: `#1a1f3a` (subtle)
- UI panels: `#151935` with `rgba(255,255,255,0.05)` border
- Text: `#e0e6ff` (light blue-white)
- Accent: `#00d9ff` (cyan) for buttons/highlights

**Cell Visualization:**

```javascript
// Each cell rendered as colored square
// Color encodes policy type
// Brightness encodes resource level

const policyColors = {
  capitalist: "#ff0055", // Red
  socialist: "#00ff88", // Green
  mercantilist: "#ffaa00", // Orange
  cooperative: "#0088ff", // Blue
  anarchist: "#aa00ff", // Purple
};

function renderCell(cell, ctx) {
  const baseColor = policyColors[cell.policy_id];
  const brightness = cell.resource / 100.0; // 0 to 1

  ctx.fillStyle = adjustBrightness(baseColor, brightness);
  ctx.fillRect(cell.x * cellSize, cell.y * cellSize, cellSize, cellSize);
}
```

**Animation:**

- Smooth color transitions (50ms ease)
- Resource transfer visualization (particle effects)
- Birth/death flicker effects
- Trade connections (brief line flash between trading cells)

### 4.3 Interactive Elements

**Policy Sliders:**

```html
<div class="slider-group">
  <label>Accumulation Rate</label>
  <input
    type="range"
    min="0"
    max="100"
    value="50"
    id="accumulation"
    class="slider"
  />
  <span id="accumulation-value">0.50</span>
  <p class="hint">Higher = save more, lower = spend more</p>
</div>
```

**Preset Policies (click to load):**

```javascript
const presets = {
  pure_capitalist: {
    accumulation_rate: 0.8,
    trade_openness: 0.9,
    redistribution_rate: 0.1,
    investment_rate: 0.9,
    risk_tolerance: 0.7,
  },
  dem_socialist: {
    accumulation_rate: 0.4,
    trade_openness: 0.6,
    redistribution_rate: 0.7,
    investment_rate: 0.5,
    risk_tolerance: 0.3,
  },
  // ... more presets
};
```

**Real-time Stats Display:**

```javascript
function updateStatsDisplay(grid) {
  // Update every 100 generations to avoid lag
  if (grid.generation % 100 !== 0) return;

  document.getElementById("gini").textContent =
    grid.statistics.gini_coefficient.toFixed(3);

  document.getElementById("total-resources").textContent =
    grid.statistics.total_resources.toLocaleString();

  updatePolicyDistributionChart(grid.statistics.resources_by_policy);
}
```

---

## 5. Preset Economic Strategies

### 5.1 Pure Capitalist

```python
{
    'name': 'Pure Capitalist',
    'description': 'Maximize individual wealth. High accumulation, aggressive investment.',
    'accumulation_rate': 0.8,
    'trade_openness': 0.9,
    'redistribution_rate': 0.1,
    'investment_rate': 0.9,
    'risk_tolerance': 0.7,
    'color': '#ff0055'
}
```

**Expected Behavior:** Fast early growth, high inequality, vulnerable to crashes

### 5.2 Democratic Socialist

```python
{
    'name': 'Democratic Socialist',
    'description': 'Balance growth with equality. Moderate redistribution.',
    'accumulation_rate': 0.4,
    'trade_openness': 0.6,
    'redistribution_rate': 0.7,
    'investment_rate': 0.5,
    'risk_tolerance': 0.3,
    'color': '#00ff88'
}
```

**Expected Behavior:** Slower but stable growth, survives resource scarcity

### 5.3 Mercantilist

```python
{
    'name': 'Mercantilist',
    'description': 'Hoard resources, trade cautiously, build reserves.',
    'accumulation_rate': 0.9,
    'trade_openness': 0.3,
    'redistribution_rate': 0.2,
    'investment_rate': 0.4,
    'risk_tolerance': 0.2,
    'color': '#ffaa00'
}
```

**Expected Behavior:** Defensive, long survival, slow expansion

### 5.4 Anarcho-Cooperative

```python
{
    'name': 'Anarcho-Cooperative',
    'description': 'Maximum sharing, no accumulation, community focus.',
    'accumulation_rate': 0.1,
    'trade_openness': 1.0,
    'redistribution_rate': 0.9,
    'investment_rate': 0.3,
    'risk_tolerance': 0.5,
    'color': '#0088ff'
}
```

**Expected Behavior:** Extreme equality, vulnerable to exploitation

### 5.5 Parasitic Capitalist

```python
{
    'name': 'Parasitic',
    'description': 'Extract value from neighbors without contributing.',
    'accumulation_rate': 0.9,
    'trade_openness': 0.8,  # High to extract
    'redistribution_rate': 0.0,
    'investment_rate': 0.1,  # Low, just extract
    'risk_tolerance': 0.9,
    'color': '#aa00ff'
}
```

**Expected Behavior:** Benefits from socialist neighbors, collapses in homogeneous environment

---

## 6. Multiplayer Implementation

### 6.1 Match Flow

```
1. LOBBY PHASE (30 seconds)
   └─> Players join room, select/design policies
   └─> Display preview of each policy's parameters
   └─> Ready check

2. INITIALIZATION
   └─> Generate grid (400x400 default)
   └─> Randomly distribute policies (equal starting positions)
   └─> Sync initial state to all clients

3. SIMULATION PHASE (5,000 generations)
   └─> Client-side CA updates (synchronized via generation counter)
   └─> Stats sent to server every 100 generations
   └─> Server broadcasts leaderboard updates

4. RESULTS PHASE
   └─> Display final statistics
   └─> Winner announcements (4 categories)
   └─> Save match to database
   └─> Update player ratings
```

### 6.2 Synchronization Strategy

**Problem:** Each client runs CA locally for performance. How to keep synchronized?

**Solution: Deterministic seeded randomness**

```python
# Server generates match seed
match_seed = random.randint(0, 2**32)

# All clients use same seed
random.seed(match_seed)

# Now all random events (trades, mutations) are identical
```

**Validation:**

```python
# Server runs lightweight validation every 500 generations
def validate_sync(client_states):
    # Check total resources match (approximate due to float precision)
    for state in client_states:
        if abs(state.total_resources - expected_total) > 100:
            log_desync(state.client_id)
```

### 6.3 WebSocket Protocol

```python
# Message Types

# Client -> Server
{
    'type': 'join_match',
    'policy': {...}
}

{
    'type': 'update_stats',
    'generation': 1234,
    'stats': {...}
}

# Server -> Client
{
    'type': 'match_start',
    'seed': 12345678,
    'policies': [...],
    'grid_config': {'width': 400, 'height': 400}
}

{
    'type': 'leaderboard_update',
    'rankings': [...]
}

{
    'type': 'match_end',
    'results': {...}
}
```

### 6.4 Matchmaking

**Simple Version (MVP):**

- Single global queue
- Matches start when 2-8 players ready
- 30 second countdown

**Future:**

- Rating-based matchmaking (ELO)
- Private rooms with invite codes
- Tournament brackets

---

## 7. Statistical Tracking

### 7.1 Real-time Metrics

**Gini Coefficient (inequality measure):**

```python
def calculate_gini(cells):
    """
    Gini coefficient: 0 = perfect equality, 1 = maximum inequality
    """
    if not cells:
        return 0

    resources = sorted([c.resource for c in cells if c.is_alive])
    n = len(resources)

    if n == 0 or sum(resources) == 0:
        return 0

    cumsum = 0
    for i, r in enumerate(resources):
        cumsum += (2 * i - n + 1) * r

    return cumsum / (n * sum(resources))
```

**Survival Rates:**

```python
def calculate_survival_rates(grid):
    """
    Track which policies last longest
    """
    lineages = defaultdict(list)

    for cell in grid.all_cells():
        if cell.is_alive:
            lineages[cell.lineage].append(cell.age)

    return {
        policy_id: mean(ages)
        for policy_id, ages in lineages.items()
    }
```

**Market Concentration (HHI):**

```python
def calculate_hhi(grid):
    """
    Herfindahl-Hirschman Index: measure of market concentration
    0 = perfect competition, 10,000 = monopoly
    """
    policy_counts = Counter(c.policy_id for c in grid.all_cells() if c.is_alive)
    total = sum(policy_counts.values())

    if total == 0:
        return 0

    return sum((count / total) ** 2 * 10000 for count in policy_counts.values())
```

### 7.2 Historical Data Collection

```python
class MatchHistory:
    match_id: str
    timestamp: datetime
    policies: List[EconomicPolicy]
    generations: int

    # Time series data (sampled every 100 generations)
    snapshots: List[Snapshot]

class Snapshot:
    generation: int
    gini_coefficient: float
    total_resources: float
    hhi: float
    policy_distributions: Dict[str, float]
    alive_cells: int
```

**Storage:**

```sql
CREATE TABLE matches (
    match_id TEXT PRIMARY KEY,
    timestamp TIMESTAMP,
    duration_seconds REAL,
    total_generations INTEGER,
    winner_policy_id TEXT
);

CREATE TABLE snapshots (
    match_id TEXT,
    generation INTEGER,
    gini REAL,
    total_resources REAL,
    hhi REAL,
    alive_cells INTEGER,
    policy_distributions JSON,
    FOREIGN KEY (match_id) REFERENCES matches(match_id)
);

CREATE TABLE policies (
    policy_id TEXT PRIMARY KEY,
    name TEXT,
    creator TEXT,
    parameters JSON,
    total_matches INTEGER,
    total_wins INTEGER,
    avg_survival_rate REAL
);
```

---

## 8. Development Phases

### Phase 1: Core CA Engine (Week 1)

**Goal:** Working single-player simulation

**Tasks:**

- [ ] Set up project structure (HTML, JS, Python)
- [ ] Implement `Cell` and `Grid` classes
- [ ] Implement core update loop (trade, redistribution, reproduction)
- [ ] Canvas rendering (basic colored squares)
- [ ] Single policy running on 200x200 grid at 30 FPS
- [ ] Basic stats display (total cells, resources)

**Deliverable:** Can watch single policy evolve in isolation

**Testing:**

```javascript
// Test 1: Pure accumulation policy should grow steadily
// Test 2: Zero accumulation should die out quickly
// Test 3: High redistribution should spread resources evenly
```

### Phase 2: Policy Editor & Presets (Week 2)

**Goal:** User can design and test custom policies

**Tasks:**

- [ ] Build policy editor UI (5 sliders)
- [ ] Implement preset policies (5 strategies)
- [ ] Add "load preset" functionality
- [ ] Add "save custom policy" (localStorage for now)
- [ ] Multi-policy matches (2-4 policies competing)
- [ ] Color-coded visualization by policy
- [ ] Enhanced stats (per-policy resources, Gini coefficient)

**Deliverable:** Can compare capitalist vs socialist strategies visually

**Testing:**

```javascript
// Test 1: Capitalist should dominate early, socialist stabilizes later
// Test 2: Extreme accumulation (0.9+) should create monopolies
// Test 3: High redistribution (0.8+) should keep Gini < 0.3
```

### Phase 3: Multiplayer Backend (Week 3)

**Goal:** Real-time multiplayer matches

**Tasks:**

- [ ] FastAPI server with WebSocket support
- [ ] Matchmaking queue (simple: wait for 2-4 players)
- [ ] Match state synchronization (seeded randomness)
- [ ] Leaderboard calculation and storage (SQLite)
- [ ] Match history recording
- [ ] Deploy backend to Railway/Render
- [ ] Deploy frontend to Vercel/Netlify

**Deliverable:** 2+ players can compete in real-time match

**Testing:**

```javascript
// Test 1: Two clients start with identical grids
// Test 2: Stats sync across clients every 100 generations
// Test 3: Match completes and records results correctly
```

### Phase 4: Polish & Content (Week 4)

**Goal:** Production-ready, shareable product

**Tasks:**

- [ ] Visual polish (animations, particle effects, smooth transitions)
- [ ] Sound effects (births, deaths, trades)
- [ ] Detailed results screen (graphs, charts, replay)
- [ ] "Share match" functionality (generate shareable URL)
- [ ] Optimize performance (WebGL rendering, web workers for CA updates)
- [ ] Mobile responsive design
- [ ] Tutorial/onboarding flow
- [ ] Create demo video (3 minutes)
- [ ] Write blog post with examples
- [ ] Generate 10+ example matches with interesting emergent behaviors

**Deliverable:** Production website + marketing materials

---

## 9. Technical Challenges & Solutions

### Challenge 1: Performance with 100k+ cells

**Problem:** 400x400 grid = 160k cells, 30 updates/sec = 4.8M ops/sec

**Solutions:**

- **Sparse representation:** Only store alive cells in hash map
- **Spatial partitioning:** Update only "active regions" where cells exist
- **WebGL rendering:** GPU-accelerated drawing
- **Web Workers:** Run CA updates in background thread
- **Adaptive quality:** Reduce grid size on low-end devices

```javascript
// Sparse grid optimization
class SparseGrid {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.cells = new Map(); // Only stores alive cells
  }

  getCell(x, y) {
    const key = `${x},${y}`;
    return this.cells.get(key) || { is_alive: false };
  }

  setCell(x, y, cell) {
    if (cell.is_alive) {
      this.cells.set(`${x},${y}`, cell);
    } else {
      this.cells.delete(`${x},${y}`);
    }
  }

  *aliveCells() {
    for (const [key, cell] of this.cells) {
      yield cell;
    }
  }
}
```

### Challenge 2: Synchronization across clients

**Problem:** Floating point errors accumulate, causing desync

**Solutions:**

- **Fixed-point arithmetic:** Use integers for resources (multiply by 100)
- **Periodic server validation:** Server checks total resources every 500 gens
- **Checksum validation:** Hash grid state every 1000 gens
- **Replay system:** Server can replay from seed to verify

```javascript
// Fixed-point arithmetic
const RESOURCE_SCALE = 100;

class Cell {
  constructor() {
    this._resource = 5000; // represents 50.00
  }

  get resource() {
    return this._resource / RESOURCE_SCALE;
  }

  set resource(value) {
    this._resource = Math.round(value * RESOURCE_SCALE);
  }
}
```

### Challenge 3: Interesting emergent behaviors

**Problem:** Parameters might lead to boring outcomes (all cells die, or stagnation)

**Solutions:**

- **Resource injection:** Slowly add resources to prevent death spirals
- **Environmental variation:** Different zones with different conditions
- **Random events:** Occasional "economic shocks" (resources added/removed)
- **Parameter tuning:** Extensive testing to find sweet spots

```javascript
// Resource injection to prevent extinction
function injectResources(grid) {
  if (grid.total_alive < grid.width * grid.height * 0.05) {
    // Less than 5% alive? Inject resources
    const injection_points = selectRandomCells(grid, 10);
    for (const point of injection_points) {
      point.resource += 50;
    }
  }
}
```

---

## 10. Marketing & Content Strategy

### 10.1 Launch Content

**Primary Pieces:**

1. **Demo Video (3 minutes)**
   - Opening: "What if you could simulate capitalism vs socialism?"
   - Show time-lapse of match evolution
   - Highlight 3 surprising emergent behaviors
   - End with call-to-action:
