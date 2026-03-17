<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  // ==================== CONSTANTS ====================
  const GRID = 16;
  const CELL = 48;
  const MAP_PX = GRID * CELL;

  const T_GRASS = 0, T_WATER = 1, T_MOUNTAIN = 2, T_CITY = 3;

  // prettier-ignore
  const MAP: number[][] = [
    [0,0,0,2,2,0,0,0,0,0,0,0,0,0,0,0],
    [0,3,0,0,2,0,0,0,0,0,0,3,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,1,1,1,1,0,0,0,0,0,0,3,0],
    [0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,2,2,0,0,0,0],
    [0,0,3,0,0,0,0,0,0,0,0,2,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,3,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,3,0,0],
    [0,0,0,0,0,3,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
  ];

  type Good = { emoji: string; name: string };
  const GOODS: Good[] = [
    { emoji: '🪵', name: 'Wood' },
    { emoji: '🍎', name: 'Food' },
    { emoji: '🧸', name: 'Toys' },
    { emoji: '💎', name: 'Gems' },
    { emoji: '📚', name: 'Books' },
    { emoji: '⚙️', name: 'Iron' },
    { emoji: '🐚', name: 'Coral' },
  ];

  type CityDef = {
    name: string; row: number; col: number;
    produces: number; wants: number[]; color: string;
  };

  const CITIES: CityDef[] = [
    { name: 'Woodville',  row: 1,  col: 1,  produces: 0, wants: [2, 4, 6], color: '#8b4513' },
    { name: 'Toytown',    row: 1,  col: 11, produces: 2, wants: [0, 3],    color: '#ff69b4' },
    { name: 'Farmdale',   row: 7,  col: 2,  produces: 1, wants: [3, 5],    color: '#228b22' },
    { name: 'Gemsburg',   row: 10, col: 8,  produces: 3, wants: [1, 0],    color: '#9370db' },
    { name: 'Bookbury',   row: 13, col: 5,  produces: 4, wants: [1, 2],    color: '#cd853f' },
    { name: 'Ironforge',  row: 4,  col: 14, produces: 5, wants: [0, 4, 3], color: '#708090' },
    { name: 'Coraltown',  row: 12, col: 13, produces: 6, wants: [1, 5],    color: '#20b2aa' },
  ];

  type CostumeDef = {
    id: string; name: string; price: number; color: string; detail: string;
  };

  const COSTUMES: CostumeDef[] = [
    { id: 'classic', name: 'Classic',  price: 0,   color: '#cc2222', detail: 'Standard red steam engine' },
    { id: 'crabby',  name: 'Crabby',   price: 50,  color: '#4488bb', detail: 'Bluish crab claws + eyes' },
    { id: 'rocket',  name: 'Rocket',   price: 100, color: '#bbbbcc', detail: 'Silver with flame trail' },
    { id: 'dragon',  name: 'Dragon',   price: 150, color: '#22aa44', detail: 'Green dragon train' },
    { id: 'golden',  name: 'Golden',   price: 200, color: '#ffaa00', detail: 'Sparkle particles' },
    { id: 'ghost',   name: 'Ghost',    price: 250, color: '#ddeeff', detail: 'Semi-transparent wisps' },
    { id: 'rainbow', name: 'Rainbow',  price: 300, color: '#ff0000', detail: 'Color-shifting body' },
    { id: 'royal',   name: 'Royal',    price: 400, color: '#7722aa', detail: 'Purple with gold crown' },
  ];

  const RANKS = [
    { title: 'Conductor', threshold: 0 },
    { title: 'Engineer', threshold: 200 },
    { title: 'Station Master', threshold: 500 },
    { title: 'Railroad Baron', threshold: 1000 },
    { title: 'Train Tycoon!', threshold: 2000 },
  ];

  const TRACK_COST = 10;
  const EXPRESS_COST = 30;
  const TRAIN_COST = 75;

  type AchievementDef = { id: string; name: string; desc: string; check: () => boolean };

  // ==================== GAME STATE ====================
  type Tool = 'track' | 'express' | 'demolish' | 'select';

  type TrainState = {
    id: number;
    costumeId: string;
    gridRow: number;
    gridCol: number;
    targetRow: number;
    targetCol: number;
    progress: number;
    path: [number, number][];
    pathIdx: number;
    assignedOrder: number | null;
    carrying: number | null;
    wheelAngle: number;
    smokePuffs: { x: number; y: number; age: number; size: number }[];
  };

  type Order = {
    id: number;
    fromCity: number;
    toCity: number;
    good: number;
    reward: number;
    timeLeft: number;
  };

  type GameEvent = {
    type: 'goldrush' | 'storm' | 'bonus';
    timeLeft: number;
    desc: string;
  };

  // State variables
  let phase: 'title' | 'tutorial' | 'play' = 'title';
  let tutorialStep = 0;
  const TUTORIAL_STEPS = [
    'Welcome to Train Tycoon! Click tracks to connect cities.',
    'Use the Track tool to lay rails between tiles.',
    'Buy a train to start delivering goods!',
    'Complete orders to earn coins. Have fun!'
  ];

  let coins = 100;
  let totalEarned = 0;
  let totalDeliveries = 0;
  let playTime = 0;

  let currentTool: Tool = 'track';
  let tracks: boolean[][] = Array.from({ length: GRID }, () => Array(GRID).fill(false));
  let expressTiles: boolean[][] = Array.from({ length: GRID }, () => Array(GRID).fill(false));
  let trains: TrainState[] = [];
  let orders: Order[] = [];
  let nextTrainId = 1;
  let nextOrderId = 1;
  let cityDeliveries: number[] = CITIES.map(() => 0);
  let unlockedCostumes: Set<string> = new Set(['classic']);

  let activeEvent: GameEvent | null = null;
  let eventTimer = 60 + Math.random() * 30;

  // Achievements
  let unlockedAchievements: Set<string> = new Set();
  let achievementToast: string | null = null;
  let toastTimer = 0;

  const ACHIEVEMENTS: AchievementDef[] = [
    { id: 'first_delivery', name: 'First Delivery', desc: 'Complete your first delivery', check: () => totalDeliveries >= 1 },
    { id: 'ten_deliveries', name: 'Regular Service', desc: 'Complete 10 deliveries', check: () => totalDeliveries >= 10 },
    { id: 'speed_demon', name: 'Speed Demon', desc: 'Use express track', check: () => expressTiles.some(r => r.some(c => c)) },
    { id: 'costume4', name: 'Costume Collector', desc: 'Unlock 4 costumes', check: () => unlockedCostumes.size >= 4 },
    { id: 'all_connected', name: 'Connected!', desc: 'Place track adjacent to all cities', check: () => CITIES.every(c => hasTrackNear(c.row, c.col)) },
    { id: 'millionaire', name: 'Millionaire', desc: 'Earn 1000 coins total', check: () => totalEarned >= 1000 },
    { id: 'full_fleet', name: 'Full Fleet', desc: 'Own 5+ trains', check: () => trains.length >= 5 },
    { id: 'city_growth1', name: 'Town Builder', desc: 'Grow a city to level 2', check: () => cityDeliveries.some(d => d >= 15) },
    { id: 'city_growth2', name: 'Metropolis', desc: 'Grow a city to level 3', check: () => cityDeliveries.some(d => d >= 30) },
    { id: 'all_costumes', name: 'Fashionista', desc: 'Unlock all costumes', check: () => unlockedCostumes.size >= COSTUMES.length },
    { id: 'rain_rider', name: 'Rain Rider', desc: 'Deliver during a storm', check: () => false }, // special
    { id: 'baron', name: 'Railroad Baron', desc: 'Reach Baron rank', check: () => totalEarned >= 1000 },
  ];

  // Camera
  let camX = MAP_PX / 2;
  let camY = MAP_PX / 2;
  let zoom = 1;
  let dragging = false;
  let dragStartX = 0;
  let dragStartY = 0;
  let camStartX = 0;
  let camStartY = 0;
  let dragMoved = false;

  // Day/night cycle (5 min = 300s)
  let dayTime = 0;
  const DAY_CYCLE = 300;

  // Weather
  let raining = false;
  let rainTimer = 0;
  let rainDrops: { x: number; y: number; speed: number }[] = [];

  // Shop/Stats modals
  let showShop = false;
  let showStats = false;

  // Canvas
  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let canvasW = 800;
  let canvasH = 600;
  let animFrame: number;
  let lastTime = 0;

  // Touch
  let touchIds: Map<number, { x: number; y: number }> = new Map();
  let pinchDist = 0;

  // ==================== HELPERS ====================
  function hasTrackNear(row: number, col: number): boolean {
    for (let dr = -1; dr <= 1; dr++) {
      for (let dc = -1; dc <= 1; dc++) {
        if (dr === 0 && dc === 0) continue;
        const r = row + dr, c = col + dc;
        if (r >= 0 && r < GRID && c >= 0 && c < GRID && tracks[r][c]) return true;
      }
    }
    return false;
  }

  function screenToGrid(sx: number, sy: number): [number, number] {
    const wx = camX + (sx - canvasW / 2) / zoom;
    const wy = camY + (sy - canvasH / 2) / zoom;
    return [Math.floor(wy / CELL), Math.floor(wx / CELL)];
  }

  function gridToScreen(row: number, col: number): [number, number] {
    const wx = col * CELL + CELL / 2;
    const wy = row * CELL + CELL / 2;
    return [
      (wx - camX) * zoom + canvasW / 2,
      (wy - camY) * zoom + canvasH / 2,
    ];
  }

  function getRank(): string {
    let rank = RANKS[0].title;
    for (const r of RANKS) {
      if (totalEarned >= r.threshold) rank = r.title;
    }
    return rank;
  }

  function getCityLevel(idx: number): number {
    const d = cityDeliveries[idx];
    if (d >= 30) return 3;
    if (d >= 15) return 2;
    if (d >= 5) return 1;
    return 0;
  }

  function getDaylight(): number {
    // 0=midnight, 0.5=noon
    const t = (dayTime % DAY_CYCLE) / DAY_CYCLE;
    return 0.5 + 0.5 * Math.cos(2 * Math.PI * (t - 0.5));
  }

  function findPath(sr: number, sc: number, er: number, ec: number): [number, number][] | null {
    if (sr === er && sc === ec) return [[sr, sc]];
    const visited = Array.from({ length: GRID }, () => Array(GRID).fill(false));
    const prev: ([number, number] | null)[][] = Array.from({ length: GRID }, () => Array(GRID).fill(null));
    const queue: [number, number][] = [[sr, sc]];
    visited[sr][sc] = true;

    while (queue.length > 0) {
      const [r, c] = queue.shift()!;
      if (r === er && c === ec) {
        const path: [number, number][] = [];
        let cur: [number, number] | null = [er, ec];
        while (cur) {
          path.unshift(cur);
          cur = prev[cur[0]][cur[1]];
        }
        return path;
      }
      for (const [dr, dc] of [[0, 1], [0, -1], [1, 0], [-1, 0]]) {
        const nr = r + dr, nc = c + dc;
        if (nr < 0 || nr >= GRID || nc < 0 || nc >= GRID) continue;
        if (visited[nr][nc]) continue;
        // Allow: track tiles, any city tile (traversable), or destination city
        const isTrack = tracks[nr][nc];
        const isCity = MAP[nr][nc] === T_CITY;
        if (!isTrack && !isCity) continue;
        visited[nr][nc] = true;
        prev[nr][nc] = [r, c];
        queue.push([nr, nc]);
      }
    }
    return null;
  }

  function spawnOrders() {
    while (orders.length < 3) {
      const from = Math.floor(Math.random() * CITIES.length);
      let to = from;
      while (to === from) to = Math.floor(Math.random() * CITIES.length);
      const city = CITIES[from];
      const good = city.produces;
      const isBonus = activeEvent?.type === 'bonus';
      const base = 15 + Math.floor(Math.random() * 20);
      const reward = isBonus ? base * 2 : base;
      orders.push({
        id: nextOrderId++,
        fromCity: from,
        toCity: to,
        good,
        reward,
        timeLeft: 120,
      });
    }
  }

  function checkAchievements() {
    for (const a of ACHIEVEMENTS) {
      if (!unlockedAchievements.has(a.id) && a.check()) {
        unlockedAchievements.add(a.id);
        achievementToast = a.name;
        toastTimer = 3;
        // Bonus coins for achievement
        coins += 25;
        totalEarned += 25;
      }
    }
  }

  // ==================== INPUT HANDLERS ====================
  function handleGridAction(row: number, col: number) {
    const terrain = MAP[row][col];
    if (currentTool === 'track') {
      if (terrain === T_WATER || terrain === T_MOUNTAIN) return;
      if (tracks[row][col]) return;
      if (coins < TRACK_COST) return;
      tracks[row][col] = true;
      coins -= TRACK_COST;
    } else if (currentTool === 'express') {
      if (!tracks[row][col]) return;
      if (expressTiles[row][col]) return;
      if (coins < EXPRESS_COST) return;
      expressTiles[row][col] = true;
      coins -= EXPRESS_COST;
    } else if (currentTool === 'demolish') {
      if (!tracks[row][col]) return;
      // Don't demolish if a train is on it or about to move onto it
      const trainOnTile = trains.some(t =>
        (t.gridRow === row && t.gridCol === col) ||
        (t.path.length > 0 && t.pathIdx < t.path.length - 1 &&
         t.path[t.pathIdx + 1][0] === row && t.path[t.pathIdx + 1][1] === col)
      );
      if (trainOnTile) return;
      tracks[row][col] = false;
      expressTiles[row][col] = false;
      coins += Math.floor(TRACK_COST / 2);
    }
  }

  function onCanvasClick(e: MouseEvent) {
    if (phase === 'title') {
      phase = 'tutorial';
      tutorialStep = 0;
      return;
    }
    if (phase === 'tutorial') {
      tutorialStep++;
      if (tutorialStep >= TUTORIAL_STEPS.length) {
        phase = 'play';
        spawnOrders();
      }
      return;
    }
    if (dragMoved) return;

    const rect = canvas.getBoundingClientRect();
    const sx = e.clientX - rect.left;
    const sy = e.clientY - rect.top;
    const [row, col] = screenToGrid(sx, sy);

    if (row < 0 || row >= GRID || col < 0 || col >= GRID) return;
    handleGridAction(row, col);
  }

  function onMouseDown(e: MouseEvent) {
    dragging = true;
    dragMoved = false;
    dragStartX = e.clientX;
    dragStartY = e.clientY;
    camStartX = camX;
    camStartY = camY;
  }

  function onMouseMove(e: MouseEvent) {
    if (!dragging) return;
    const dx = e.clientX - dragStartX;
    const dy = e.clientY - dragStartY;
    if (Math.abs(dx) > 3 || Math.abs(dy) > 3) dragMoved = true;
    camX = camStartX - dx / zoom;
    camY = camStartY - dy / zoom;
    clampCamera();
  }

  function onMouseUp() {
    dragging = false;
  }

  function onWheel(e: WheelEvent) {
    e.preventDefault();
    const factor = e.deltaY > 0 ? 0.9 : 1.1;
    zoom = Math.max(0.3, Math.min(3, zoom * factor));
    clampCamera();
  }

  function onTouchStart(e: TouchEvent) {
    for (const t of Array.from(e.changedTouches)) {
      touchIds.set(t.identifier, { x: t.clientX, y: t.clientY });
    }
    if (touchIds.size === 1) {
      const t = Array.from(e.touches)[0];
      dragging = true;
      dragMoved = false;
      dragStartX = t.clientX;
      dragStartY = t.clientY;
      camStartX = camX;
      camStartY = camY;
    } else if (touchIds.size === 2) {
      const pts = Array.from(touchIds.values());
      pinchDist = Math.hypot(pts[0].x - pts[1].x, pts[0].y - pts[1].y);
    }
  }

  function onTouchMove(e: TouchEvent) {
    e.preventDefault();
    for (const t of Array.from(e.changedTouches)) {
      touchIds.set(t.identifier, { x: t.clientX, y: t.clientY });
    }
    if (touchIds.size === 1 && dragging) {
      const t = Array.from(e.touches)[0];
      const dx = t.clientX - dragStartX;
      const dy = t.clientY - dragStartY;
      if (Math.abs(dx) > 3 || Math.abs(dy) > 3) dragMoved = true;
      camX = camStartX - dx / zoom;
      camY = camStartY - dy / zoom;
      clampCamera();
    } else if (touchIds.size === 2) {
      const pts = Array.from(touchIds.values());
      const newDist = Math.hypot(pts[0].x - pts[1].x, pts[0].y - pts[1].y);
      if (pinchDist > 0) {
        zoom = Math.max(0.3, Math.min(3, zoom * (newDist / pinchDist)));
      }
      pinchDist = newDist;
    }
  }

  function onTouchEnd(e: TouchEvent) {
    for (const t of Array.from(e.changedTouches)) {
      touchIds.delete(t.identifier);
    }
    if (touchIds.size === 0) {
      if (!dragMoved && phase === 'play') {
        // Simulate click at last touch position
        const t = e.changedTouches[0];
        const rect = canvas.getBoundingClientRect();
        const sx = t.clientX - rect.left;
        const sy = t.clientY - rect.top;
        const [row, col] = screenToGrid(sx, sy);
        if (row >= 0 && row < GRID && col >= 0 && col < GRID) {
          handleGridAction(row, col);
        }
      } else if (phase !== 'play') {
        // Advance title/tutorial
        if (phase === 'title') { phase = 'tutorial'; tutorialStep = 0; }
        else if (phase === 'tutorial') {
          tutorialStep++;
          if (tutorialStep >= TUTORIAL_STEPS.length) { phase = 'play'; spawnOrders(); }
        }
      }
      dragging = false;
      dragMoved = false;
    }
  }

  function clampCamera() {
    const halfW = canvasW / (2 * zoom);
    const halfH = canvasH / (2 * zoom);
    camX = Math.max(halfW - CELL, Math.min(MAP_PX + CELL - halfW, camX));
    camY = Math.max(halfH - CELL, Math.min(MAP_PX + CELL - halfH, camY));
  }

  // ==================== GAME ACTIONS ====================
  function canSpawnTrain(): boolean {
    return CITIES.some(c => hasTrackNear(c.row, c.col));
  }

  function buyTrain(costumeId: string) {
    if (coins < TRAIN_COST) return;
    // Find a city with adjacent track to spawn on
    let spawnRow = -1;
    let spawnCol = -1;
    outer: for (const city of CITIES) {
      for (let dr = -1; dr <= 1; dr++) {
        for (let dc = -1; dc <= 1; dc++) {
          if (dr === 0 && dc === 0) continue;
          const r = city.row + dr, c = city.col + dc;
          if (r >= 0 && r < GRID && c >= 0 && c < GRID && tracks[r][c]) {
            spawnRow = r;
            spawnCol = c;
            break outer;
          }
        }
      }
    }
    if (spawnRow === -1) return; // No valid spawn point
    coins -= TRAIN_COST;
    trains.push({
      id: nextTrainId++,
      costumeId,
      gridRow: spawnRow,
      gridCol: spawnCol,
      targetRow: spawnRow,
      targetCol: spawnCol,
      progress: 0,
      path: [],
      pathIdx: 0,
      assignedOrder: null,
      carrying: null,
      wheelAngle: 0,
      smokePuffs: [],
    });
  }

  function buyCostume(id: string) {
    const c = COSTUMES.find(c => c.id === id);
    if (!c || unlockedCostumes.has(id)) return;
    if (coins < c.price) return;
    coins -= c.price;
    unlockedCostumes.add(id);
  }

  // ==================== UPDATE LOOP ====================
  function update(dt: number) {
    if (phase !== 'play') return;

    playTime += dt;
    dayTime += dt;

    // Event timer
    eventTimer -= dt;
    if (eventTimer <= 0 && !activeEvent) {
      const roll = Math.random();
      if (roll < 0.33) {
        activeEvent = { type: 'goldrush', timeLeft: 30, desc: 'Gold Rush! 2x rewards!' };
      } else if (roll < 0.66) {
        activeEvent = { type: 'storm', timeLeft: 20, desc: 'Storm! Trains slowed!' };
        raining = true;
      } else {
        activeEvent = { type: 'bonus', timeLeft: 30, desc: 'Bonus Orders available!' };
      }
      eventTimer = 60 + Math.random() * 30;
    }

    if (activeEvent) {
      activeEvent.timeLeft -= dt;
      if (activeEvent.timeLeft <= 0) {
        if (activeEvent.type === 'storm') raining = false;
        activeEvent = null;
      }
    }

    // Weather
    if (raining) {
      rainTimer += dt;
      if (rainTimer > 0.03) {
        rainTimer = 0;
        rainDrops.push({
          x: Math.random() * canvasW,
          y: -10,
          speed: 300 + Math.random() * 200,
        });
      }
      rainDrops = rainDrops.filter(d => {
        d.y += d.speed * dt;
        return d.y < canvasH + 10;
      });
    } else {
      rainDrops = [];
    }

    // Toast timer
    if (toastTimer > 0) {
      toastTimer -= dt;
      if (toastTimer <= 0) achievementToast = null;
    }

    // Order timers
    orders = orders.filter(o => {
      o.timeLeft -= dt;
      return o.timeLeft > 0;
    });
    spawnOrders();

    // Update trains
    for (const train of trains) {
      // Auto-assign to orders
      if (train.assignedOrder === null && train.path.length === 0) {
        // Find unassigned order
        for (const order of orders) {
          if (trains.some(t => t.assignedOrder === order.id)) continue;
          // Try to path to pickup city
          const fromCity = CITIES[order.fromCity];
          const path = findPath(train.gridRow, train.gridCol, fromCity.row, fromCity.col);
          if (path && path.length > 1) {
            train.assignedOrder = order.id;
            train.path = path;
            train.pathIdx = 0;
            train.progress = 0;
            break;
          }
        }
      }

      // Move along path
      if (train.path.length > 0 && train.pathIdx < train.path.length - 1) {
        let speed = 2.0; // tiles per second
        if (activeEvent?.type === 'storm') speed *= 0.5;
        if (expressTiles[train.gridRow]?.[train.gridCol]) speed *= 2;

        train.progress += speed * dt;
        train.wheelAngle += speed * dt * 8;

        if (train.progress >= 1) {
          train.progress = 0;
          train.pathIdx++;
          if (train.pathIdx < train.path.length) {
            train.gridRow = train.path[train.pathIdx][0];
            train.gridCol = train.path[train.pathIdx][1];
          }
        }

        // Smoke puffs
        if (Math.random() < dt * 5) {
          train.smokePuffs.push({ x: 0, y: -10, age: 0, size: 3 + Math.random() * 4 });
        }
        train.smokePuffs = train.smokePuffs.filter(p => {
          p.age += dt;
          p.y -= 20 * dt;
          p.x += (Math.random() - 0.5) * 10 * dt;
          return p.age < 1.5;
        });
      }

      // Arrived at destination?
      if (train.path.length > 0 && train.pathIdx >= train.path.length - 1) {
        const order = orders.find(o => o.id === train.assignedOrder);
        if (order) {
          const fromCity = CITIES[order.fromCity];
          const toCity = CITIES[order.toCity];

          if (train.carrying === null && train.gridRow === fromCity.row && train.gridCol === fromCity.col) {
            // Pick up goods
            train.carrying = order.good;
            // Path to destination
            const path = findPath(train.gridRow, train.gridCol, toCity.row, toCity.col);
            if (path && path.length > 1) {
              train.path = path;
              train.pathIdx = 0;
              train.progress = 0;
            } else {
              // Can't reach destination, drop order
              train.carrying = null;
              train.assignedOrder = null;
              train.path = [];
            }
          } else if (train.carrying !== null && train.gridRow === toCity.row && train.gridCol === toCity.col) {
            // Deliver!
            let reward = order.reward;
            if (activeEvent?.type === 'goldrush') reward *= 2;
            coins += reward;
            totalEarned += reward;
            totalDeliveries++;
            cityDeliveries[order.toCity]++;

            // Storm delivery achievement
            if (activeEvent?.type === 'storm' && !unlockedAchievements.has('rain_rider')) {
              unlockedAchievements.add('rain_rider');
              achievementToast = 'Rain Rider';
              toastTimer = 3;
              coins += 25;
              totalEarned += 25;
            }

            orders = orders.filter(o => o.id !== order.id);
            train.carrying = null;
            train.assignedOrder = null;
            train.path = [];
          }
        } else {
          // Order expired
          train.assignedOrder = null;
          train.carrying = null;
          train.path = [];
        }
      }
    }

    checkAchievements();
  }

  // ==================== DRAWING ====================
  function draw() {
    if (!ctx) return;
    ctx.clearRect(0, 0, canvasW, canvasH);

    if (phase === 'title') {
      drawTitle();
      return;
    }
    if (phase === 'tutorial') {
      drawTutorial();
      return;
    }

    // Day/night tint
    const daylight = getDaylight();
    const skyR = Math.floor(135 * daylight + 10 * (1 - daylight));
    const skyG = Math.floor(206 * daylight + 10 * (1 - daylight));
    const skyB = Math.floor(235 * daylight + 40 * (1 - daylight));
    ctx.fillStyle = `rgb(${skyR},${skyG},${skyB})`;
    ctx.fillRect(0, 0, canvasW, canvasH);

    ctx.save();
    // Camera transform
    ctx.translate(canvasW / 2, canvasH / 2);
    ctx.scale(zoom, zoom);
    ctx.translate(-camX, -camY);

    drawGrid(daylight);
    drawTracks();
    drawCities();
    drawTrains();

    ctx.restore();

    // Rain overlay
    if (raining) drawRain();

    // Night overlay
    if (daylight < 0.5) {
      ctx.fillStyle = `rgba(0,0,30,${(1 - daylight * 2) * 0.3})`;
      ctx.fillRect(0, 0, canvasW, canvasH);
    }
  }

  function drawTitle() {
    ctx.fillStyle = '#1a3a2a';
    ctx.fillRect(0, 0, canvasW, canvasH);

    // Animated track line
    const t = (Date.now() / 1000) % 10;
    ctx.strokeStyle = '#654321';
    ctx.lineWidth = 8;
    ctx.beginPath();
    ctx.moveTo(0, canvasH * 0.6);
    ctx.lineTo(canvasW, canvasH * 0.6);
    ctx.stroke();

    // Rails
    ctx.strokeStyle = '#999';
    ctx.lineWidth = 2;
    for (let x = -50 + (t * 30) % 30; x < canvasW + 50; x += 30) {
      ctx.beginPath();
      ctx.moveTo(x, canvasH * 0.6 - 5);
      ctx.lineTo(x, canvasH * 0.6 + 5);
      ctx.stroke();
    }

    // Animated train on title
    const trainX = ((t * 80) % (canvasW + 100)) - 50;
    drawTrainShape(trainX, canvasH * 0.6 - 15, 'classic', '#cc2222', t * 5);

    ctx.fillStyle = '#ffdd44';
    ctx.font = 'bold 48px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('🚂 Train Tycoon! 🚂', canvasW / 2, canvasH * 0.3);

    ctx.fillStyle = '#aaddaa';
    ctx.font = '20px monospace';
    ctx.fillText('Click or tap to start!', canvasW / 2, canvasH * 0.75);
  }

  function drawTutorial() {
    ctx.fillStyle = '#1a3a2a';
    ctx.fillRect(0, 0, canvasW, canvasH);

    ctx.fillStyle = '#ffdd44';
    ctx.font = 'bold 32px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(`Step ${tutorialStep + 1} of ${TUTORIAL_STEPS.length}`, canvasW / 2, canvasH * 0.3);

    ctx.fillStyle = '#ffffff';
    ctx.font = '18px monospace';
    const text = TUTORIAL_STEPS[tutorialStep];
    ctx.fillText(text, canvasW / 2, canvasH * 0.5);

    ctx.fillStyle = '#aaddaa';
    ctx.font = '16px monospace';
    ctx.fillText('Click to continue...', canvasW / 2, canvasH * 0.7);
  }

  function drawGrid(daylight: number) {
    const t = Date.now() / 1000;
    for (let r = 0; r < GRID; r++) {
      for (let c = 0; c < GRID; c++) {
        const x = c * CELL;
        const y = r * CELL;
        const terrain = MAP[r][c];

        if (terrain === T_WATER) {
          // Animated water
          const wave = Math.sin(t * 2 + r * 0.5 + c * 0.7) * 15;
          const bVal = Math.floor(180 + wave);
          ctx.fillStyle = `rgb(60,100,${bVal})`;
        } else if (terrain === T_MOUNTAIN) {
          ctx.fillStyle = `rgb(${Math.floor(120 * daylight + 40)},${Math.floor(110 * daylight + 35)},${Math.floor(100 * daylight + 30)})`;
        } else if (terrain === T_CITY) {
          ctx.fillStyle = `rgb(${Math.floor(180 * daylight + 40)},${Math.floor(170 * daylight + 35)},${Math.floor(140 * daylight + 30)})`;
        } else {
          ctx.fillStyle = `rgb(${Math.floor(100 * daylight + 30)},${Math.floor(160 * daylight + 40)},${Math.floor(80 * daylight + 25)})`;
        }
        ctx.fillRect(x, y, CELL, CELL);

        // Grid lines
        ctx.strokeStyle = `rgba(0,0,0,0.1)`;
        ctx.lineWidth = 0.5;
        ctx.strokeRect(x, y, CELL, CELL);

        // Mountain decoration
        if (terrain === T_MOUNTAIN) {
          ctx.fillStyle = '#888';
          ctx.beginPath();
          ctx.moveTo(x + CELL * 0.2, y + CELL * 0.9);
          ctx.lineTo(x + CELL * 0.5, y + CELL * 0.2);
          ctx.lineTo(x + CELL * 0.8, y + CELL * 0.9);
          ctx.closePath();
          ctx.fill();
          // Snow cap
          ctx.fillStyle = '#fff';
          ctx.beginPath();
          ctx.moveTo(x + CELL * 0.4, y + CELL * 0.4);
          ctx.lineTo(x + CELL * 0.5, y + CELL * 0.2);
          ctx.lineTo(x + CELL * 0.6, y + CELL * 0.4);
          ctx.closePath();
          ctx.fill();
        }
      }
    }
  }

  function drawTracks() {
    for (let r = 0; r < GRID; r++) {
      for (let c = 0; c < GRID; c++) {
        if (!tracks[r][c]) continue;
        const x = c * CELL;
        const y = r * CELL;
        const cx = x + CELL / 2;
        const cy = y + CELL / 2;

        const isExpress = expressTiles[r][c];

        // Track bed
        ctx.fillStyle = isExpress ? '#aa8844' : '#654321';
        ctx.fillRect(x + 4, y + 4, CELL - 8, CELL - 8);

        // Rails
        ctx.strokeStyle = isExpress ? '#ddcc88' : '#999';
        ctx.lineWidth = 2;

        // Connect to neighbors
        const hasUp = r > 0 && (tracks[r - 1][c] || MAP[r - 1][c] === T_CITY);
        const hasDown = r < GRID - 1 && (tracks[r + 1][c] || MAP[r + 1][c] === T_CITY);
        const hasLeft = c > 0 && (tracks[r][c - 1] || MAP[r][c - 1] === T_CITY);
        const hasRight = c < GRID - 1 && (tracks[r][c + 1] || MAP[r][c + 1] === T_CITY);

        if (hasUp || hasDown) {
          ctx.beginPath();
          ctx.moveTo(cx - 5, hasUp ? y : cy);
          ctx.lineTo(cx - 5, hasDown ? y + CELL : cy);
          ctx.stroke();
          ctx.beginPath();
          ctx.moveTo(cx + 5, hasUp ? y : cy);
          ctx.lineTo(cx + 5, hasDown ? y + CELL : cy);
          ctx.stroke();
          // Ties
          const startY = hasUp ? y + 4 : cy - 4;
          const endY = hasDown ? y + CELL - 4 : cy + 4;
          for (let ty = startY; ty <= endY; ty += 8) {
            ctx.beginPath();
            ctx.moveTo(cx - 8, ty);
            ctx.lineTo(cx + 8, ty);
            ctx.stroke();
          }
        }
        if (hasLeft || hasRight) {
          ctx.beginPath();
          ctx.moveTo(hasLeft ? x : cx, cy - 5);
          ctx.lineTo(hasRight ? x + CELL : cx, cy - 5);
          ctx.stroke();
          ctx.beginPath();
          ctx.moveTo(hasLeft ? x : cx, cy + 5);
          ctx.lineTo(hasRight ? x + CELL : cx, cy + 5);
          ctx.stroke();
          const startX = hasLeft ? x + 4 : cx - 4;
          const endX = hasRight ? x + CELL - 4 : cx + 4;
          for (let tx = startX; tx <= endX; tx += 8) {
            ctx.beginPath();
            ctx.moveTo(tx, cy - 8);
            ctx.lineTo(tx, cy + 8);
            ctx.stroke();
          }
        }
        if (!hasUp && !hasDown && !hasLeft && !hasRight) {
          // Isolated track: draw cross
          ctx.beginPath();
          ctx.moveTo(cx - 8, cy);
          ctx.lineTo(cx + 8, cy);
          ctx.stroke();
          ctx.beginPath();
          ctx.moveTo(cx, cy - 8);
          ctx.lineTo(cx, cy + 8);
          ctx.stroke();
        }

        // Express sparkle
        if (isExpress) {
          const t = Date.now() / 300;
          const sparkle = Math.sin(t + r * 3 + c * 7) * 0.5 + 0.5;
          ctx.fillStyle = `rgba(255,255,200,${sparkle * 0.4})`;
          ctx.fillRect(x + 8, y + 8, CELL - 16, CELL - 16);
        }
      }
    }
  }

  function drawCities() {
    for (let i = 0; i < CITIES.length; i++) {
      const city = CITIES[i];
      const x = city.col * CELL;
      const y = city.row * CELL;
      const level = getCityLevel(i);

      // Base buildings
      ctx.fillStyle = city.color;
      // Main building
      ctx.fillRect(x + 8, y + 12, 16, 24);
      ctx.fillRect(x + 24, y + 16, 14, 20);

      // Rooftops
      ctx.fillStyle = '#553322';
      ctx.fillRect(x + 6, y + 10, 20, 4);
      ctx.fillRect(x + 22, y + 14, 18, 4);

      // Level 2+: more buildings
      if (level >= 1) {
        ctx.fillStyle = city.color;
        ctx.fillRect(x + 2, y + 20, 10, 16);
        ctx.fillStyle = '#553322';
        ctx.fillRect(x + 0, y + 18, 14, 4);
      }
      if (level >= 2) {
        ctx.fillStyle = city.color;
        ctx.fillRect(x + 34, y + 14, 10, 22);
        ctx.fillStyle = '#553322';
        ctx.fillRect(x + 32, y + 12, 14, 4);
      }
      if (level >= 3) {
        // Tall building
        ctx.fillStyle = city.color;
        ctx.fillRect(x + 14, y + 2, 12, 34);
        ctx.fillStyle = '#ffdd44';
        ctx.fillRect(x + 12, y + 0, 16, 4);
      }

      // Windows
      ctx.fillStyle = '#ffee88';
      ctx.fillRect(x + 12, y + 16, 3, 3);
      ctx.fillRect(x + 18, y + 16, 3, 3);
      ctx.fillRect(x + 12, y + 22, 3, 3);
      ctx.fillRect(x + 28, y + 20, 3, 3);

      // City name
      ctx.fillStyle = '#fff';
      ctx.font = '9px monospace';
      ctx.textAlign = 'center';
      ctx.fillText(city.name, x + CELL / 2, y + CELL + 10);

      // Produces icon
      ctx.font = '12px serif';
      ctx.fillText(GOODS[city.produces].emoji, x + CELL / 2, y - 2);
    }
  }

  function drawTrains() {
    for (const train of trains) {
      let drawRow = train.gridRow;
      let drawCol = train.gridCol;

      // Interpolate position
      if (train.pathIdx < train.path.length - 1) {
        const nextR = train.path[train.pathIdx + 1][0];
        const nextC = train.path[train.pathIdx + 1][1];
        drawRow = train.gridRow + (nextR - train.gridRow) * train.progress;
        drawCol = train.gridCol + (nextC - train.gridCol) * train.progress;
      }

      const x = drawCol * CELL + CELL / 2;
      const y = drawRow * CELL + CELL / 2;
      const costume = COSTUMES.find(c => c.id === train.costumeId);
      const color = costume?.color || '#cc2222';

      drawTrainShape(x, y, train.costumeId, color, train.wheelAngle);

      // Smoke puffs
      for (const puff of train.smokePuffs) {
        const alpha = Math.max(0, 1 - puff.age / 1.5);
        ctx.fillStyle = `rgba(200,200,200,${alpha * 0.6})`;
        ctx.beginPath();
        ctx.arc(x + puff.x, y + puff.y, puff.size * (1 + puff.age), 0, Math.PI * 2);
        ctx.fill();
      }

      // Carrying indicator
      if (train.carrying !== null) {
        ctx.font = '10px serif';
        ctx.textAlign = 'center';
        ctx.fillText(GOODS[train.carrying].emoji, x, y - 16);
      }
    }
  }

  function drawTrainShape(x: number, y: number, costumeId: string, color: string, wheelAngle: number) {
    ctx.save();
    ctx.translate(x, y);

    // Body
    if (costumeId === 'ghost') {
      ctx.globalAlpha = 0.5 + Math.sin(Date.now() / 300) * 0.2;
    }

    if (costumeId === 'rainbow') {
      const hue = (Date.now() / 10) % 360;
      ctx.fillStyle = `hsl(${hue}, 80%, 55%)`;
    } else {
      ctx.fillStyle = color;
    }

    // Main body rectangle
    ctx.fillRect(-14, -8, 28, 16);

    // Cabin
    ctx.fillStyle = darken(color, 0.3);
    ctx.fillRect(-16, -10, 8, 20);

    // Chimney
    ctx.fillStyle = '#444';
    ctx.fillRect(6, -14, 6, 6);

    // Wheels
    ctx.fillStyle = '#333';
    const wOff = Math.sin(wheelAngle) * 1.5;
    ctx.beginPath();
    ctx.arc(-8, 10 + wOff, 4, 0, Math.PI * 2);
    ctx.fill();
    ctx.beginPath();
    ctx.arc(8, 10 - wOff, 4, 0, Math.PI * 2);
    ctx.fill();

    // Wheel spokes
    ctx.strokeStyle = '#666';
    ctx.lineWidth = 1;
    for (const wx of [-8, 8]) {
      const wo = wx === -8 ? wOff : -wOff;
      ctx.beginPath();
      ctx.moveTo(wx - 3, 10 + wo);
      ctx.lineTo(wx + 3, 10 + wo);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(wx, 7 + wo);
      ctx.lineTo(wx, 13 + wo);
      ctx.stroke();
    }

    // Costume-specific decorations
    if (costumeId === 'crabby') {
      // Crab eyes
      ctx.fillStyle = '#fff';
      ctx.beginPath();
      ctx.arc(-4, -12, 3, 0, Math.PI * 2);
      ctx.fill();
      ctx.beginPath();
      ctx.arc(4, -12, 3, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = '#000';
      ctx.beginPath();
      ctx.arc(-4, -12, 1.5, 0, Math.PI * 2);
      ctx.fill();
      ctx.beginPath();
      ctx.arc(4, -12, 1.5, 0, Math.PI * 2);
      ctx.fill();
      // Claws
      ctx.strokeStyle = color;
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.arc(-18, 0, 6, -0.5, 1.5);
      ctx.stroke();
      ctx.beginPath();
      ctx.arc(18, 0, 6, 1.5, 3.5);
      ctx.stroke();
    } else if (costumeId === 'rocket') {
      // Flame trail
      const flicker = Math.random() * 4;
      ctx.fillStyle = '#ff4400';
      ctx.beginPath();
      ctx.moveTo(-16, -4);
      ctx.lineTo(-24 - flicker, 0);
      ctx.lineTo(-16, 4);
      ctx.closePath();
      ctx.fill();
      ctx.fillStyle = '#ffaa00';
      ctx.beginPath();
      ctx.moveTo(-16, -2);
      ctx.lineTo(-20 - flicker * 0.5, 0);
      ctx.lineTo(-16, 2);
      ctx.closePath();
      ctx.fill();
    } else if (costumeId === 'dragon') {
      // Spiky back
      ctx.fillStyle = '#116622';
      for (let i = -10; i <= 10; i += 5) {
        ctx.beginPath();
        ctx.moveTo(i - 3, -8);
        ctx.lineTo(i, -16);
        ctx.lineTo(i + 3, -8);
        ctx.closePath();
        ctx.fill();
      }
      // Dragon snout
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.moveTo(14, -4);
      ctx.lineTo(22, 0);
      ctx.lineTo(14, 4);
      ctx.closePath();
      ctx.fill();
    } else if (costumeId === 'golden') {
      // Sparkle particles
      ctx.fillStyle = '#fff';
      const t = Date.now() / 200;
      for (let i = 0; i < 5; i++) {
        const sx = Math.sin(t + i * 1.3) * 16;
        const sy = Math.cos(t + i * 2.1) * 12;
        ctx.beginPath();
        ctx.arc(sx, sy, 1.5, 0, Math.PI * 2);
        ctx.fill();
      }
    } else if (costumeId === 'ghost') {
      // Wispy trail
      ctx.fillStyle = 'rgba(200,220,255,0.3)';
      const t = Date.now() / 500;
      for (let i = 0; i < 3; i++) {
        const wx = -18 - i * 6 + Math.sin(t + i) * 3;
        const wy = Math.cos(t + i * 0.7) * 5;
        ctx.beginPath();
        ctx.arc(wx, wy, 4 + i, 0, Math.PI * 2);
        ctx.fill();
      }
    } else if (costumeId === 'royal') {
      // Crown
      ctx.fillStyle = '#ffaa00';
      ctx.fillRect(-6, -16, 12, 4);
      ctx.fillRect(-8, -20, 4, 6);
      ctx.fillRect(-1, -20, 4, 6);
      ctx.fillRect(5, -20, 4, 6);
      // Jewels on crown
      ctx.fillStyle = '#ff0044';
      ctx.beginPath();
      ctx.arc(-6, -18, 1.5, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = '#0044ff';
      ctx.beginPath();
      ctx.arc(1, -18, 1.5, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = '#00ff44';
      ctx.beginPath();
      ctx.arc(7, -18, 1.5, 0, Math.PI * 2);
      ctx.fill();
      // Gold trim
      ctx.strokeStyle = '#ffaa00';
      ctx.lineWidth = 1.5;
      ctx.strokeRect(-14, -8, 28, 16);
    }

    ctx.globalAlpha = 1;
    ctx.restore();
  }

  function darken(hex: string, amount: number): string {
    const r = parseInt(hex.slice(1, 3), 16);
    const g = parseInt(hex.slice(3, 5), 16);
    const b = parseInt(hex.slice(5, 7), 16);
    return `rgb(${Math.floor(r * (1 - amount))},${Math.floor(g * (1 - amount))},${Math.floor(b * (1 - amount))})`;
  }

  function drawRain() {
    ctx.strokeStyle = 'rgba(150,180,255,0.4)';
    ctx.lineWidth = 1;
    for (const drop of rainDrops) {
      ctx.beginPath();
      ctx.moveTo(drop.x, drop.y);
      ctx.lineTo(drop.x - 1, drop.y + 8);
      ctx.stroke();
    }
  }

  // ==================== GAME LOOP ====================
  function gameLoop(timestamp: number) {
    const dt = Math.min((timestamp - lastTime) / 1000, 0.1);
    lastTime = timestamp;

    update(dt);
    draw();

    animFrame = requestAnimationFrame(gameLoop);
  }

  function resizeCanvas() {
    if (!canvas) return;
    const container = canvas.parentElement;
    if (!container) return;
    canvasW = container.clientWidth;
    canvasH = container.clientHeight;
    canvas.width = canvasW;
    canvas.height = canvasH;
  }

  onMount(() => {
    ctx = canvas.getContext('2d')!;
    resizeCanvas();

    window.addEventListener('resize', resizeCanvas);
    lastTime = performance.now();
    animFrame = requestAnimationFrame(gameLoop);
  });

  onDestroy(() => {
    if (animFrame) cancelAnimationFrame(animFrame);
    if (typeof window !== 'undefined') {
      window.removeEventListener('resize', resizeCanvas);
    }
  });

  // Reactive: format time
  function fmtTime(s: number): string {
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60);
    return `${m}:${sec.toString().padStart(2, '0')}`;
  }

  function getDayNightIcon(): string {
    const dl = getDaylight();
    if (dl > 0.7) return '☀️';
    if (dl > 0.3) return '🌅';
    return '🌙';
  }
</script>

<svelte:head>
  <title>Train Tycoon!</title>
</svelte:head>

<div class="game-container">
  <!-- Canvas -->
  <div class="canvas-wrap">
    <canvas
      bind:this={canvas}
      on:click={onCanvasClick}
      on:mousedown={onMouseDown}
      on:mousemove={onMouseMove}
      on:mouseup={onMouseUp}
      on:mouseleave={onMouseUp}
      on:wheel={onWheel}
      on:touchstart={onTouchStart}
      on:touchmove={onTouchMove}
      on:touchend={onTouchEnd}
    ></canvas>
  </div>

  {#if phase === 'play'}
    <!-- HUD: Top-left -->
    <div class="hud top-left">
      <div class="coins">🪙 {coins}</div>
      <div class="rank">{getRank()}</div>
      <div class="day-indicator">{getDayNightIcon()}</div>
      {#if activeEvent}
        <div class="event-banner">{activeEvent.desc} ({Math.ceil(activeEvent.timeLeft)}s)</div>
      {/if}
    </div>

    <!-- Tools: Left -->
    <div class="hud tools-left">
      <button class:active={currentTool === 'track'} on:click={() => currentTool = 'track'} title={`Place Track (${TRACK_COST} coins)`}>
        🛤️ Track
      </button>
      <button class:active={currentTool === 'express'} on:click={() => currentTool = 'express'} title={`Express Upgrade (${EXPRESS_COST} coins)`}>
        ⚡ Express
      </button>
      <button class:active={currentTool === 'demolish'} on:click={() => currentTool = 'demolish'} title="Remove Track">
        🔨 Demolish
      </button>
      <button class:active={currentTool === 'select'} on:click={() => currentTool = 'select'} title="Select">
        👆 Select
      </button>
    </div>

    <!-- Orders: Right -->
    <div class="hud orders-right">
      <h3>📦 Orders</h3>
      {#each orders as order}
        <div class="order-card">
          <div>{GOODS[order.good].emoji} {GOODS[order.good].name}</div>
          <div class="order-route">{CITIES[order.fromCity].name} → {CITIES[order.toCity].name}</div>
          <div class="order-reward">🪙 {order.reward} | ⏱ {Math.ceil(order.timeLeft)}s</div>
          {#if trains.some(t => t.assignedOrder === order.id)}
            <div class="order-assigned">🚂 Assigned</div>
          {/if}
        </div>
      {/each}
    </div>

    <!-- Train roster: Bottom -->
    <div class="hud bottom-bar">
      <div class="train-roster">
        {#each trains as train}
          <div class="train-chip" style="background:{COSTUMES.find(c => c.id === train.costumeId)?.color || '#cc2222'}">
            🚂 {train.id}
            {#if train.carrying !== null}
              {GOODS[train.carrying].emoji}
            {/if}
          </div>
        {/each}
        {#if trains.length === 0}
          <span class="hint">No trains yet! Buy one from the shop.</span>
        {/if}
      </div>
    </div>

    <!-- Shop button: Bottom-right -->
    <button class="hud shop-btn" on:click={() => showShop = true}>🏪 Shop</button>

    <!-- Stats button: Top-right -->
    <button class="hud stats-btn" on:click={() => showStats = true}>📊 Stats</button>

    <!-- Achievement toast -->
    {#if achievementToast}
      <div class="achievement-toast">
        🏆 Achievement: {achievementToast}! (+25 coins)
      </div>
    {/if}

    <!-- Shop Modal -->
    {#if showShop}
      <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
      <div class="modal-overlay" on:click={() => showShop = false}>
        <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
        <div class="modal" on:click|stopPropagation>
          <h2>🏪 Shop</h2>
          <div class="shop-section">
            <h3>🚂 Buy Train ({TRAIN_COST} coins)</h3>
            <div class="costume-grid">
              {#each COSTUMES as costume}
                <div class="costume-card">
                  <div class="costume-preview" style="background:{costume.color}; opacity:{costume.id === 'ghost' ? 0.5 : 1}">🚂</div>
                  <div class="costume-name">{costume.name}</div>
                  <div class="costume-detail">{costume.detail}</div>
                  {#if unlockedCostumes.has(costume.id)}
                    <button on:click={() => { buyTrain(costume.id); showShop = false; }} disabled={coins < TRAIN_COST || !canSpawnTrain()}>
                      {canSpawnTrain() ? `Buy Train (${TRAIN_COST})` : 'Need track near city'}
                    </button>
                  {:else}
                    <button on:click={() => buyCostume(costume.id)} disabled={coins < costume.price}>
                      Unlock ({costume.price})
                    </button>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
          <button class="close-btn" on:click={() => showShop = false}>Close</button>
        </div>
      </div>
    {/if}

    <!-- Stats Modal -->
    {#if showStats}
      <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
      <div class="modal-overlay" on:click={() => showStats = false}>
        <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
        <div class="modal" on:click|stopPropagation>
          <h2>📊 Stats & Achievements</h2>
          <div class="stats-grid">
            <div>Total Deliveries: {totalDeliveries}</div>
            <div>Total Earned: {totalEarned} coins</div>
            <div>Trains: {trains.length}</div>
            <div>Play Time: {fmtTime(playTime)}</div>
            <div>Rank: {getRank()}</div>
            <div>Costumes: {unlockedCostumes.size}/{COSTUMES.length}</div>
          </div>
          <h3>🏆 Achievements</h3>
          <div class="achievements-list">
            {#each ACHIEVEMENTS as ach}
              <div class="ach-item" class:unlocked={unlockedAchievements.has(ach.id)}>
                <span>{unlockedAchievements.has(ach.id) ? '✅' : '🔒'}</span>
                <span class="ach-name">{ach.name}</span>
                <span class="ach-desc">{ach.desc}</span>
              </div>
            {/each}
          </div>
          <h3>🏙️ City Growth</h3>
          <div class="city-growth-list">
            {#each CITIES as city, i}
              <div class="city-growth-item">
                <span style="color:{city.color}">{city.name}</span>
                <span>Lv.{getCityLevel(i)} ({cityDeliveries[i]} deliveries)</span>
              </div>
            {/each}
          </div>
          <button class="close-btn" on:click={() => showStats = false}>Close</button>
        </div>
      </div>
    {/if}
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    overflow: hidden;
    background: #111;
  }

  .game-container {
    position: fixed;
    inset: 0;
    overflow: hidden;
  }

  .canvas-wrap {
    width: 100%;
    height: 100%;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
    touch-action: none;
  }

  .hud {
    position: absolute;
    pointer-events: auto;
  }

  .top-left {
    top: 10px;
    left: 10px;
    color: #fff;
    text-shadow: 1px 1px 3px #000;
    font-family: monospace;
  }

  .coins {
    font-size: 22px;
    font-weight: bold;
  }

  .rank {
    font-size: 14px;
    color: #ffdd44;
  }

  .day-indicator {
    font-size: 16px;
  }

  .event-banner {
    margin-top: 4px;
    background: rgba(255, 200, 0, 0.85);
    color: #222;
    padding: 4px 10px;
    border-radius: 6px;
    font-weight: bold;
    font-size: 13px;
  }

  .tools-left {
    top: 120px;
    left: 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .tools-left button {
    background: rgba(0, 0, 0, 0.7);
    color: #fff;
    border: 2px solid #555;
    border-radius: 8px;
    padding: 8px 12px;
    font-family: monospace;
    font-size: 13px;
    cursor: pointer;
    transition: all 0.15s;
  }

  .tools-left button:hover {
    background: rgba(50, 50, 50, 0.9);
  }

  .tools-left button.active {
    border-color: #ffdd44;
    background: rgba(80, 70, 0, 0.8);
    color: #ffdd44;
  }

  .orders-right {
    top: 10px;
    right: 10px;
    width: 200px;
    background: rgba(0, 0, 0, 0.75);
    border-radius: 10px;
    padding: 10px;
    color: #fff;
    font-family: monospace;
    font-size: 12px;
    max-height: 300px;
    overflow-y: auto;
  }

  .orders-right h3 {
    margin: 0 0 8px 0;
    font-size: 15px;
  }

  .order-card {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 6px;
    padding: 6px;
    margin-bottom: 6px;
  }

  .order-route {
    color: #aaf;
    font-size: 11px;
  }

  .order-reward {
    color: #fda;
    font-size: 11px;
  }

  .order-assigned {
    color: #afa;
    font-size: 11px;
  }

  .bottom-bar {
    bottom: 10px;
    left: 10px;
    right: 70px;
    background: rgba(0, 0, 0, 0.7);
    border-radius: 10px;
    padding: 8px 12px;
    color: #fff;
    font-family: monospace;
  }

  .train-roster {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    align-items: center;
  }

  .train-chip {
    padding: 4px 10px;
    border-radius: 8px;
    font-size: 13px;
    color: #fff;
    text-shadow: 1px 1px 2px #000;
  }

  .hint {
    color: #aaa;
    font-size: 13px;
  }

  .shop-btn {
    bottom: 10px;
    right: 10px;
    background: rgba(40, 100, 40, 0.9);
    color: #fff;
    border: 2px solid #4a4;
    border-radius: 10px;
    padding: 10px 14px;
    font-family: monospace;
    font-size: 15px;
    cursor: pointer;
  }

  .shop-btn:hover {
    background: rgba(60, 140, 60, 0.95);
  }

  .stats-btn {
    top: 10px;
    right: 220px;
    background: rgba(40, 40, 100, 0.9);
    color: #fff;
    border: 2px solid #66a;
    border-radius: 10px;
    padding: 6px 12px;
    font-family: monospace;
    font-size: 13px;
    cursor: pointer;
  }

  .stats-btn:hover {
    background: rgba(60, 60, 140, 0.95);
  }

  .achievement-toast {
    position: absolute;
    top: 60px;
    left: 50%;
    transform: translateX(-50%);
    background: linear-gradient(135deg, #ffaa00, #ff6600);
    color: #fff;
    padding: 12px 24px;
    border-radius: 12px;
    font-family: monospace;
    font-size: 16px;
    font-weight: bold;
    text-shadow: 1px 1px 2px #000;
    animation: toastIn 0.3s ease;
    z-index: 100;
  }

  @keyframes toastIn {
    from { transform: translateX(-50%) translateY(-20px); opacity: 0; }
    to { transform: translateX(-50%) translateY(0); opacity: 1; }
  }

  .modal-overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.7);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 200;
  }

  .modal {
    background: #1a2a1a;
    border: 2px solid #4a4;
    border-radius: 16px;
    padding: 20px;
    color: #fff;
    font-family: monospace;
    max-width: 600px;
    max-height: 80vh;
    overflow-y: auto;
    width: 90%;
  }

  .modal h2 {
    margin: 0 0 12px;
    color: #ffdd44;
  }

  .modal h3 {
    margin: 12px 0 8px;
    color: #aaf;
  }

  .costume-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(130px, 1fr));
    gap: 10px;
  }

  .costume-card {
    background: rgba(255, 255, 255, 0.08);
    border-radius: 10px;
    padding: 10px;
    text-align: center;
  }

  .costume-preview {
    width: 40px;
    height: 40px;
    border-radius: 8px;
    margin: 0 auto 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 20px;
  }

  .costume-name {
    font-weight: bold;
    font-size: 13px;
  }

  .costume-detail {
    font-size: 10px;
    color: #aaa;
    margin: 4px 0;
  }

  .costume-card button {
    background: #335533;
    color: #fff;
    border: 1px solid #4a4;
    border-radius: 6px;
    padding: 4px 8px;
    font-family: monospace;
    font-size: 11px;
    cursor: pointer;
    width: 100%;
  }

  .costume-card button:hover:not(:disabled) {
    background: #447744;
  }

  .costume-card button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  .close-btn {
    margin-top: 12px;
    background: #553333;
    color: #fff;
    border: 1px solid #a44;
    border-radius: 8px;
    padding: 8px 20px;
    font-family: monospace;
    cursor: pointer;
    width: 100%;
  }

  .close-btn:hover {
    background: #774444;
  }

  .stats-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 6px;
    font-size: 13px;
  }

  .achievements-list {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .ach-item {
    display: flex;
    gap: 8px;
    align-items: center;
    font-size: 12px;
    opacity: 0.5;
  }

  .ach-item.unlocked {
    opacity: 1;
  }

  .ach-name {
    font-weight: bold;
    color: #ffdd44;
  }

  .ach-desc {
    color: #aaa;
  }

  .city-growth-list {
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 12px;
  }

  .city-growth-item {
    display: flex;
    justify-content: space-between;
  }

  /* Mobile adjustments */
  @media (max-width: 600px) {
    .orders-right {
      width: 150px;
      font-size: 10px;
      max-height: 200px;
    }

    .tools-left button {
      padding: 6px 8px;
      font-size: 11px;
    }

    .stats-btn {
      right: 170px;
    }
  }
</style>
