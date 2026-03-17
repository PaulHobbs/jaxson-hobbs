<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  // ==================== GAME CONFIG ====================
  const BODY_TYPES = [
    { id: 'sport', emoji: '🏎️', label: 'Sport', speed: 5, accel: 4, handling: 3 },
    { id: 'truck', emoji: '🛻', label: 'Truck', speed: 3, accel: 3, handling: 4 },
    { id: 'race', emoji: '🏁', label: 'Racer', speed: 5, accel: 5, handling: 2 },
    { id: 'suv', emoji: '🚙', label: 'SUV', speed: 3, accel: 4, handling: 5 },
  ];
  const WHEEL_TYPES = [
    { id: 'normal', emoji: '⚫', label: 'Normal', grip: 3 },
    { id: 'big', emoji: '🔵', label: 'Big', grip: 4 },
    { id: 'spiky', emoji: '💎', label: 'Diamond', grip: 5 },
    { id: 'fire', emoji: '🔥', label: 'Fire', grip: 2 },
  ];
  const ENGINE_TYPES = [
    { id: 'normal', emoji: '⚙️', label: 'Normal', power: 3 },
    { id: 'turbo', emoji: '💨', label: 'Turbo', power: 4 },
    { id: 'rocket', emoji: '🚀', label: 'Rocket', power: 5 },
    { id: 'electric', emoji: '⚡', label: 'Electric', power: 4 },
  ];
  const SPOILER_TYPES = [
    { id: 'none', emoji: '❌', label: 'None', downforce: 0 },
    { id: 'small', emoji: '🔼', label: 'Small', downforce: 2 },
    { id: 'big', emoji: '🔺', label: 'Big', downforce: 4 },
    { id: 'wing', emoji: '🦅', label: 'Wing', downforce: 3 },
  ];
  const PAINT_COLORS = [
    { id: 'red', emoji: '🔴', label: 'Red', color: '#ff4444' },
    { id: 'blue', emoji: '🔵', label: 'Blue', color: '#4488ff' },
    { id: 'green', emoji: '🟢', label: 'Green', color: '#44cc44' },
    { id: 'yellow', emoji: '🟡', label: 'Yellow', color: '#ffdd44' },
    { id: 'purple', emoji: '🟣', label: 'Purple', color: '#bb44ff' },
    { id: 'orange', emoji: '🟠', label: 'Orange', color: '#ff8844' },
    { id: 'white', emoji: '⚪', label: 'White', color: '#eeeeee' },
    { id: 'black', emoji: '⚫', label: 'Black', color: '#333333' },
  ];

  const BUILD_STEPS = [
    { name: 'Pick Your Body!', key: 'body', options: BODY_TYPES },
    { name: 'Pick Your Wheels!', key: 'wheels', options: WHEEL_TYPES },
    { name: 'Pick Your Engine!', key: 'engine', options: ENGINE_TYPES },
    { name: 'Pick Your Spoiler!', key: 'spoiler', options: SPOILER_TYPES },
    { name: 'Paint Your Car!', key: 'paint', options: PAINT_COLORS },
  ];

  // ==================== TRACK CONFIG ====================
  const TRACK_CENTER_X = 600;
  const TRACK_CENTER_Y = 450;
  const TRACK_RX = 450;
  const TRACK_RY = 320;
  const TRACK_WIDTH = 120;
  const CHECKPOINTS = [0, Math.PI / 2, Math.PI, Math.PI * 1.5];
  const LAPS_TO_WIN = 3;

  function getTrackPoint(angle: number) {
    return {
      x: TRACK_CENTER_X + Math.cos(angle) * TRACK_RX,
      y: TRACK_CENTER_Y + Math.sin(angle) * TRACK_RY,
    };
  }

  // ==================== GAME STATE ====================
  type CarParts = {
    body?: (typeof BODY_TYPES)[number];
    wheels?: (typeof WHEEL_TYPES)[number];
    engine?: (typeof ENGINE_TYPES)[number];
    spoiler?: (typeof SPOILER_TYPES)[number];
    paint?: (typeof PAINT_COLORS)[number];
  };

  type Player = {
    connected: boolean;
    pad: Gamepad | null;
    car: CarParts;
    buildStep: number;
    hoverIdx: number;
    ready: boolean;
    x: number;
    y: number;
    angle: number;
    speed: number;
    lap: number;
    checkpoint: number;
    finished: boolean;
    finishTime: number;
  };

  function makePlayer(): Player {
    return {
      connected: false, pad: null, car: {}, buildStep: 0, hoverIdx: 0, ready: false,
      x: 0, y: 0, angle: 0, speed: 0, lap: 0, checkpoint: 0, finished: false, finishTime: 0,
    };
  }

  type GameMode = '2p' | '1p-ai' | '1p-timetrial';

  let gamePhase = $state<'title' | 'build' | 'countdown' | 'race' | 'win'>('title');
  let gameMode = $state<GameMode>('2p');
  let menuIdx = $state(0);
  let players = $state<Player[]>([makePlayer(), makePlayer()]);
  let prevButtons: Record<number, boolean>[] = [{}, {}];
  let countdownTimer = 0;
  let countdownText = $state('');
  let countdownColor = $state('#fff');
  let raceStartTime = 0;
  let raceElapsed = $state(0);
  let winner = $state(0);
  let raceCanvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let animFrame: number;
  let lastTime = 0;

  // AI state
  let aiTargetAngle = 0;
  let aiWobble = 0;
  let aiWobbleTimer = 0;

  // Camera state for pseudo-3D (per player)
  let camX = [0, 0];
  let camY = [0, 0];
  let camAngle = [0, 0];

  // Rendering constants
  const RENDER_W = 640;
  const RENDER_H = 360;
  const FOV = 120;
  const CAMERA_HEIGHT = 80;
  const CAMERA_BEHIND = 60;
  const DRAW_DISTANCE = 1200;

  // Offscreen canvas for Mode 7 rendering
  let offCanvas: HTMLCanvasElement;
  let offCtx: CanvasRenderingContext2D;
  let cachedImgData: ImageData | null = null;

  // Mountain silhouette heights (precomputed)
  const MOUNTAIN_POINTS: number[] = [];
  function initMountains() {
    for (let i = 0; i <= 640; i++) {
      const x = i / 640;
      MOUNTAIN_POINTS[i] = Math.sin(x * 5.3) * 30 + Math.sin(x * 12.7) * 15 +
        Math.sin(x * 2.1 + 1.3) * 25 + Math.cos(x * 8.4) * 10 + 60;
    }
  }
  initMountains();

  // Controller assignment: maps player index (0,1) to actual gamepad index
  let padAssignment: (number | null)[] = [null, null];
  let debugInfo = $state('');

  // Menu options
  const MENU_OPTIONS: { label: string; mode: GameMode; needs2: boolean }[] = [
    { label: '1 Player - VS Computer', mode: '1p-ai', needs2: false },
    { label: '1 Player - Time Trial', mode: '1p-timetrial', needs2: false },
    { label: '2 Players', mode: '2p', needs2: true },
  ];

  function is1P(): boolean {
    return gameMode === '1p-ai' || gameMode === '1p-timetrial';
  }

  // ==================== CONTROLLER INPUT ====================
  function pollGamepads() {
    const gps = navigator.getGamepads();

    // Build debug info on title screen
    if (gamePhase === 'title') {
      let dbg = '';
      for (let i = 0; i < gps.length; i++) {
        const gp = gps[i];
        if (gp && gp.connected) {
          const pressedBtns: number[] = [];
          for (let b = 0; b < gp.buttons.length; b++) {
            if (gp.buttons[b].pressed) pressedBtns.push(b);
          }
          const axes = gp.axes.map((a, idx) => Math.abs(a) > 0.1 ? `ax${idx}:${a.toFixed(2)}` : '').filter(Boolean).join(' ');
          dbg += `[${i}] ${gp.id.substring(0, 40)}`;
          if (pressedBtns.length) dbg += ` BTN:${pressedBtns.join(',')}`;
          if (axes) dbg += ` ${axes}`;
          dbg += '\n';
        }
      }
      debugInfo = dbg;
    }

    // Auto-assign controllers to players: first connected pad = P1, second = P2
    // Re-scan every frame to handle reconnects
    const connectedPads: number[] = [];
    for (let i = 0; i < gps.length; i++) {
      if (gps[i] && gps[i]!.connected) {
        connectedPads.push(i);
      }
    }

    // Assign first two connected gamepads to player 0 and player 1
    for (let pi = 0; pi < 2; pi++) {
      if (pi < connectedPads.length) {
        const gpIdx = connectedPads[pi];
        padAssignment[pi] = gpIdx;
        players[pi].connected = true;
        players[pi].pad = gps[gpIdx];
      } else {
        padAssignment[pi] = null;
        players[pi].connected = false;
        players[pi].pad = null;
      }
    }
  }

  function buttonPressed(pi: number, btnIdx: number): boolean {
    const pad = players[pi].pad;
    if (!pad) return false;
    const curr = pad.buttons[btnIdx]?.pressed ?? false;
    const prev = prevButtons[pi][btnIdx] ?? false;
    return curr && !prev;
  }

  function buttonHeld(pi: number, btnIdx: number): boolean {
    const pad = players[pi].pad;
    if (!pad) return false;
    return pad.buttons[btnIdx]?.pressed ?? false;
  }

  function stickX(pi: number): number {
    const pad = players[pi].pad;
    if (!pad) return 0;
    const val = pad.axes[0] ?? 0;
    return Math.abs(val) > 0.2 ? val : 0;
  }

  function savePrevButtons() {
    for (let i = 0; i < 2; i++) {
      const pad = players[i].pad;
      if (pad) {
        prevButtons[i] = {};
        for (let b = 0; b < pad.buttons.length; b++) {
          prevButtons[i][b] = pad.buttons[b].pressed;
        }
      }
    }
  }

  // Shortcuts
  function dpadLeft(pi: number) { return buttonPressed(pi, 14); }
  function dpadRight(pi: number) { return buttonPressed(pi, 15); }
  function dpadUp(pi: number) { return buttonPressed(pi, 12); }
  function dpadDown(pi: number) { return buttonPressed(pi, 13); }
  function btnA(pi: number) { return buttonPressed(pi, 0); }
  function btnB(pi: number) { return buttonPressed(pi, 1); }
  // Options button on PS4/PS5, also accept touchpad press (17) as alternate start
  function btnStart(pi: number) { return buttonPressed(pi, 9) || buttonPressed(pi, 17); }

  // ==================== TITLE ====================
  function updateTitle() {
    if (!players[0].connected) return;

    // Navigate menu with D-pad
    if (dpadUp(0)) {
      menuIdx = (menuIdx - 1 + MENU_OPTIONS.length) % MENU_OPTIONS.length;
    }
    if (dpadDown(0)) {
      menuIdx = (menuIdx + 1) % MENU_OPTIONS.length;
    }

    // Select with Cross
    if (btnA(0)) {
      const selected = MENU_OPTIONS[menuIdx];
      if (selected.needs2 && !players[1].connected) return; // can't pick 2P without 2 controllers
      gameMode = selected.mode;
      startBuildPhase();
    }
  }

  // ==================== BUILD ====================
  function generateRandomCar(): CarParts {
    return {
      body: BODY_TYPES[Math.floor(Math.random() * BODY_TYPES.length)],
      wheels: WHEEL_TYPES[Math.floor(Math.random() * WHEEL_TYPES.length)],
      engine: ENGINE_TYPES[Math.floor(Math.random() * ENGINE_TYPES.length)],
      spoiler: SPOILER_TYPES[Math.floor(Math.random() * SPOILER_TYPES.length)],
      paint: PAINT_COLORS[Math.floor(Math.random() * PAINT_COLORS.length)],
    };
  }

  function startBuildPhase() {
    gamePhase = 'build';
    for (let i = 0; i < 2; i++) {
      players[i].buildStep = 0;
      players[i].hoverIdx = 0;
      players[i].ready = false;
      players[i].car = {};
    }

    // In AI mode, auto-generate P2's car
    if (gameMode === '1p-ai') {
      players[1].car = generateRandomCar();
      players[1].ready = true;
    }
    // In time trial, no P2 at all
    if (gameMode === '1p-timetrial') {
      players[1].ready = true;
    }
  }

  function updateBuild() {
    // Only P1 builds in 1P modes
    const buildPlayers = is1P() ? [0] : [0, 1];

    for (const pi of buildPlayers) {
      const p = players[pi];
      if (p.ready) continue;

      const step = BUILD_STEPS[p.buildStep];
      const numOpts = step.options.length;

      if (dpadLeft(pi) || dpadUp(pi)) {
        p.hoverIdx = (p.hoverIdx - 1 + numOpts) % numOpts;
      }
      if (dpadRight(pi) || dpadDown(pi)) {
        p.hoverIdx = (p.hoverIdx + 1) % numOpts;
      }

      if (btnA(pi)) {
        const chosen = step.options[p.hoverIdx];
        (p.car as any)[step.key] = chosen;
        p.buildStep++;
        p.hoverIdx = 0;
        if (p.buildStep >= BUILD_STEPS.length) {
          p.ready = true;
        }
      }

      if (btnB(pi) && p.buildStep > 0) {
        p.buildStep--;
        p.hoverIdx = 0;
        delete (p.car as any)[BUILD_STEPS[p.buildStep].key];
      }
    }

    if (players[0].ready && players[1].ready) {
      startCountdown();
    }
  }

  function getCarPreviewEmoji(car: CarParts): string {
    const body = car.body?.emoji ?? '❓';
    const wheels = car.wheels?.emoji ?? '';
    const engine = car.engine?.emoji ?? '';
    const spoiler = car.spoiler?.id !== 'none' ? (car.spoiler?.emoji ?? '') : '';
    return `${body} ${wheels} ${engine} ${spoiler}`;
  }

  // ==================== COUNTDOWN ====================
  function startCountdown() {
    gamePhase = 'countdown';
    countdownTimer = 0;

    const start = getTrackPoint(Math.PI);
    const numRacers = gameMode === '1p-timetrial' ? 1 : 2;
    for (let i = 0; i < numRacers; i++) {
      players[i].x = start.x;
      players[i].y = start.y + (i === 0 ? -25 : 25);
      players[i].angle = 0;
      players[i].speed = 0;
      players[i].lap = 0;
      players[i].checkpoint = 0;
      players[i].finished = false;
    }

    // Initialize AI state
    aiTargetAngle = Math.PI;
    aiWobble = 0;
    aiWobbleTimer = 0;

    // Initialize cameras to avoid lerp from (0,0)
    for (let i = 0; i < numRacers; i++) {
      camX[i] = players[i].x - Math.cos(players[i].angle) * CAMERA_BEHIND;
      camY[i] = players[i].y - Math.sin(players[i].angle) * CAMERA_BEHIND;
      camAngle[i] = players[i].angle;
    }

    requestAnimationFrame(() => {
      if (raceCanvas) {
        raceCanvas.width = window.innerWidth;
        raceCanvas.height = window.innerHeight;
        ctx = raceCanvas.getContext('2d')!;
      }
    });
  }

  function updateCountdown(dt: number) {
    countdownTimer += dt;
    if (countdownTimer < 1) {
      countdownText = '3'; countdownColor = '#ff4444';
    } else if (countdownTimer < 2) {
      countdownText = '2'; countdownColor = '#ffd93d';
    } else if (countdownTimer < 3) {
      countdownText = '1'; countdownColor = '#6bcb77';
    } else if (countdownTimer < 3.5) {
      countdownText = 'GO!'; countdownColor = '#4d96ff';
    } else {
      gamePhase = 'race';
      raceStartTime = performance.now();
    }
    if (ctx) drawRace();
  }

  // ==================== AI ====================
  function updateAI(dt: number) {
    const p = players[1];
    if (p.finished) return;
    const stats = getCarStats(p);

    // Figure out where the AI car is on the ellipse
    const dx = p.x - TRACK_CENTER_X;
    const dy = p.y - TRACK_CENTER_Y;
    const currentAngle = Math.atan2(dy / TRACK_RY, dx / TRACK_RX);

    // Target a point slightly ahead on the ellipse
    const lookAhead = 0.3;
    const targetAngle = currentAngle + lookAhead;
    const target = {
      x: TRACK_CENTER_X + Math.cos(targetAngle) * TRACK_RX,
      y: TRACK_CENTER_Y + Math.sin(targetAngle) * TRACK_RY,
    };

    // Add wobble for imperfection
    aiWobbleTimer += dt;
    if (aiWobbleTimer > 0.5) {
      aiWobbleTimer = 0;
      aiWobble = (Math.random() - 0.5) * 0.4;
    }

    // Steer toward target
    const toTargetAngle = Math.atan2(target.y - p.y, target.x - p.x);
    let steerDiff = toTargetAngle - p.angle + aiWobble;
    // Normalize to [-PI, PI]
    while (steerDiff > Math.PI) steerDiff -= Math.PI * 2;
    while (steerDiff < -Math.PI) steerDiff += Math.PI * 2;

    const steerAmount = Math.max(-1, Math.min(1, steerDiff * 3));

    // Steering
    if (Math.abs(p.speed) > 0.3) {
      p.angle += steerAmount * stats.handling * (p.speed > 0 ? 1 : -0.5);
    }

    // Always accelerate, brake slightly when turning hard
    const gas = Math.abs(steerAmount) > 0.7 ? 0.6 : 1.0;
    p.speed += stats.accel * gas;

    p.speed *= stats.friction;
    if (p.speed > stats.topSpeed) p.speed = stats.topSpeed;

    p.x += Math.cos(p.angle) * p.speed;
    p.y += Math.sin(p.angle) * p.speed;
  }

  // ==================== RACING ====================
  function getCarStats(p: Player) {
    const car = p.car;
    const topSpeed = 3 + (car.body?.speed ?? 0) * 0.4 + (car.engine?.power ?? 0) * 0.3;
    const accel = 0.03 + (car.body?.accel ?? 0) * 0.006 + (car.engine?.power ?? 0) * 0.005;
    const handling = 0.03 + (car.body?.handling ?? 0) * 0.005 +
      (car.wheels?.grip ?? 0) * 0.004 + (car.spoiler?.downforce ?? 0) * 0.003;
    const friction = 0.985;
    return { topSpeed, accel, handling, friction };
  }

  function updateRace(dt: number) {
    // Update elapsed time for time trial display
    raceElapsed = (performance.now() - raceStartTime) / 1000;

    const numRacers = gameMode === '1p-timetrial' ? 1 : 2;

    // AI controls for P2 in 1p-ai mode
    if (gameMode === '1p-ai') {
      updateAI(dt);
    }

    for (let pi = 0; pi < numRacers; pi++) {
      const p = players[pi];
      if (p.finished) continue;

      // Determine if car is on or off track (before steering so handling reduction applies)
      const tdx = p.x - TRACK_CENTER_X;
      const tdy = p.y - TRACK_CENTER_Y;
      const normDist = Math.sqrt((tdx / TRACK_RX) ** 2 + (tdy / TRACK_RY) ** 2);
      const innerEdge = (TRACK_RX - TRACK_WIDTH / 2) / TRACK_RX;
      const outerEdge = (TRACK_RX + TRACK_WIDTH / 2) / TRACK_RX;
      const onTrack = normDist >= innerEdge && normDist <= outerEdge;

      // Hard wall at 1.5x track width to prevent going into oblivion
      const hardInner = (TRACK_RX - TRACK_WIDTH * 1.5) / TRACK_RX;
      const hardOuter = (TRACK_RX + TRACK_WIDTH * 1.5) / TRACK_RX;
      if (normDist > 0.001 && (normDist < hardInner || normDist > hardOuter)) {
        const clampDist = normDist < hardInner ? hardInner : hardOuter;
        const scale = clampDist / normDist;
        p.x = TRACK_CENTER_X + tdx * scale;
        p.y = TRACK_CENTER_Y + tdy * scale;
        p.speed *= 0.5;
      }

      // Skip human input for AI player
      if (gameMode === '1p-ai' && pi === 1) {
        // AI movement already handled in updateAI, just do off-track friction
        if (!onTrack) {
          p.speed *= 0.96;
          const aiStats = getCarStats(p);
          if (Math.abs(p.speed) > aiStats.topSpeed * 0.4) {
            p.speed = Math.sign(p.speed) * aiStats.topSpeed * 0.4;
          }
        }
      } else {
        const stats = getCarStats(p);
        const handlingMult = onTrack ? 1.0 : 0.5;

        const gas = buttonHeld(pi, 7) ? 1 : (buttonHeld(pi, 0) ? 1 : 0);
        const brake = buttonHeld(pi, 6) ? 1 : (buttonHeld(pi, 1) ? 0.5 : 0);
        const steer = stickX(pi);

        if (Math.abs(p.speed) > 0.3) {
          p.angle += steer * stats.handling * handlingMult * (p.speed > 0 ? 1 : -0.5);
        }

        if (gas > 0) p.speed += stats.accel * gas;
        if (brake > 0) p.speed -= stats.accel * brake * 1.5;

        // Off-track: heavier friction and lower speed cap
        const friction = onTrack ? stats.friction : 0.96;
        const speedCap = onTrack ? stats.topSpeed : stats.topSpeed * 0.4;
        p.speed *= friction;
        if (p.speed > speedCap) p.speed = speedCap;
        if (p.speed < -stats.topSpeed * 0.3) p.speed = -stats.topSpeed * 0.3;

        p.x += Math.cos(p.angle) * p.speed;
        p.y += Math.sin(p.angle) * p.speed;
      }

      // Checkpoints and laps (use current position after any wall clamps/movement)
      const cpDx = p.x - TRACK_CENTER_X;
      const cpDy = p.y - TRACK_CENTER_Y;
      const carAngle = Math.atan2(cpDy, cpDx);
      const normalizedAngle = carAngle < 0 ? carAngle + Math.PI * 2 : carAngle;
      const nextCP = CHECKPOINTS[p.checkpoint];
      const cpAngle = nextCP < 0 ? nextCP + Math.PI * 2 : nextCP;
      const angleDiff = Math.abs(normalizedAngle - cpAngle);

      if (angleDiff < 0.4 || angleDiff > Math.PI * 2 - 0.4) {
        p.checkpoint++;
        if (p.checkpoint >= CHECKPOINTS.length) {
          p.checkpoint = 0;
          p.lap++;
          if (p.lap >= LAPS_TO_WIN) {
            p.finished = true;
            p.finishTime = performance.now() - raceStartTime;
          }
        }
      }
    }

    // Check winner
    if (gameMode === '1p-timetrial') {
      if (players[0].finished) {
        winner = 0;
        gamePhase = 'win';
        return;
      }
    } else {
      if (players[0].finished || players[1].finished) {
        winner = players[0].finished ? 0 : 1;
        gamePhase = 'win';
        return;
      }
    }

    // Car-to-car collision (only when 2 racers)
    if (numRacers === 2) {
      const cdx = players[0].x - players[1].x;
      const cdy = players[0].y - players[1].y;
      const dist = Math.sqrt(cdx * cdx + cdy * cdy);
      if (dist < 30 && dist > 0) {
        const nx = cdx / dist;
        const ny = cdy / dist;
        const push = (30 - dist) / 2;
        players[0].x += nx * push;
        players[0].y += ny * push;
        players[1].x -= nx * push;
        players[1].y -= ny * push;
        const avgSpeed = (players[0].speed + players[1].speed) / 2;
        players[0].speed = avgSpeed * 0.9;
        players[1].speed = avgSpeed * 0.9;
      }
    }

    drawRace();
  }

  // ==================== PSEUDO-3D RENDERING ====================

  function isOnTrack(wx: number, wy: number): 'road' | 'curb' | 'grass' {
    const dx = wx - TRACK_CENTER_X;
    const dy = wy - TRACK_CENTER_Y;
    const normDist = Math.sqrt((dx / TRACK_RX) ** 2 + (dy / TRACK_RY) ** 2);
    const inner = (TRACK_RX - TRACK_WIDTH / 2) / TRACK_RX;
    const outer = (TRACK_RX + TRACK_WIDTH / 2) / TRACK_RX;
    const curbWidth = 0.02;
    if (normDist >= inner + curbWidth && normDist <= outer - curbWidth) return 'road';
    if (normDist >= inner && normDist <= outer) return 'curb';
    return 'grass';
  }

  function getGroundColor(wx: number, wy: number, dist: number): [number, number, number] {
    const terrain = isOnTrack(wx, wy);
    const fogT = Math.min(dist / DRAW_DISTANCE, 1);

    let r: number, g: number, b: number;

    if (terrain === 'road') {
      // Alternating road stripes for speed perception
      const dx = wx - TRACK_CENTER_X;
      const dy = wy - TRACK_CENTER_Y;
      const angle = Math.atan2(dy, dx);
      const stripe = Math.floor(angle * 20 / Math.PI);
      if (stripe % 2 === 0) {
        r = 90; g = 90; b = 90;
      } else {
        r = 75; g = 75; b = 75;
      }

      // Start/finish line area
      const finishAngle = Math.PI;
      const angleDiff = Math.abs(angle - finishAngle);
      if (angleDiff < 0.04) {
        const normDist = Math.sqrt((dx / TRACK_RX) ** 2 + (dy / TRACK_RY) ** 2);
        const checker = (Math.floor(normDist * 40) + Math.floor(angle * 200)) % 2;
        if (checker === 0) { r = 255; g = 255; b = 255; }
        else { r = 20; g = 20; b = 20; }
      }
    } else if (terrain === 'curb') {
      // Red/white curb pattern
      const dx = wx - TRACK_CENTER_X;
      const dy = wy - TRACK_CENTER_Y;
      const angle = Math.atan2(dy, dx);
      const curb = Math.floor(angle * 15 / Math.PI);
      if (curb % 2 === 0) {
        r = 220; g = 40; b = 40;
      } else {
        r = 240; g = 240; b = 240;
      }
    } else {
      // Grass with checkerboard
      const cx = Math.floor(wx / 40);
      const cy = Math.floor(wy / 40);
      if ((cx + cy) % 2 === 0) {
        r = 30; g = 100; b = 30;
      } else {
        r = 25; g = 85; b = 25;
      }
    }

    // Fog: fade toward sky color (dark blue-purple)
    const fogR = 40, fogG = 30, fogB = 60;
    r = r + (fogR - r) * fogT;
    g = g + (fogG - g) * fogT;
    b = b + (fogB - b) * fogT;

    return [r, g, b];
  }

  function updateCamera(pi: number) {
    const p = players[pi];
    const targetX = p.x - Math.cos(p.angle) * CAMERA_BEHIND;
    const targetY = p.y - Math.sin(p.angle) * CAMERA_BEHIND;
    const lerpSpeed = 0.12;
    camX[pi] += (targetX - camX[pi]) * lerpSpeed;
    camY[pi] += (targetY - camY[pi]) * lerpSpeed;

    // Smooth angle lerp
    let angleDiff = p.angle - camAngle[pi];
    while (angleDiff > Math.PI) angleDiff -= Math.PI * 2;
    while (angleDiff < -Math.PI) angleDiff += Math.PI * 2;
    camAngle[pi] += angleDiff * lerpSpeed;
  }

  function renderView(targetCtx: CanvasRenderingContext2D, pi: number, vx: number, vy: number, vw: number, vh: number) {
    // Initialize offscreen canvas if needed
    if (!offCanvas) {
      offCanvas = document.createElement('canvas');
      offCanvas.width = RENDER_W;
      offCanvas.height = RENDER_H;
      offCtx = offCanvas.getContext('2d')!;
    }

    const rw = RENDER_W;
    const rh = RENDER_H;
    const horizonY = Math.floor(rh * 0.4);

    // Get ImageData for scanline rendering (reuse allocation)
    if (!cachedImgData || cachedImgData.width !== rw || cachedImgData.height !== rh) {
      cachedImgData = offCtx.createImageData(rw, rh);
    }
    const imgData = cachedImgData;
    const pixels = imgData.data;

    const fovRad = (FOV * Math.PI) / 180;
    const halfFov = fovRad / 2;
    const halfTan = Math.tan(halfFov);

    // Sky gradient (dark blue to warm orange at horizon)
    for (let y = 0; y < horizonY; y++) {
      const t = y / horizonY;
      const sr = 10 + t * 50;
      const sg = 10 + t * 40;
      const sb = 50 + t * 30;
      for (let x = 0; x < rw; x++) {
        const idx = (y * rw + x) * 4;
        pixels[idx] = sr;
        pixels[idx + 1] = sg;
        pixels[idx + 2] = sb;
        pixels[idx + 3] = 255;
      }
    }

    // Mountain silhouettes on sky
    for (let x = 0; x < rw; x++) {
      // Parallax offset based on camera angle
      const parallax = ((camAngle[pi] * 200 / Math.PI) + x) % rw;
      const mIdx = Math.floor(((parallax % rw) + rw) % rw);
      const mHeight = MOUNTAIN_POINTS[mIdx] ?? 40;
      const mBaseY = horizonY;
      const mTopY = mBaseY - mHeight;

      for (let y = Math.max(0, Math.floor(mTopY)); y < mBaseY; y++) {
        const idx = (y * rw + x) * 4;
        // Dark mountain color, slightly lighter near top
        const mt = (y - mTopY) / mHeight;
        pixels[idx] = 20 + mt * 10;
        pixels[idx + 1] = 15 + mt * 10;
        pixels[idx + 2] = 35 + mt * 10;
        pixels[idx + 3] = 255;
      }
    }

    // Ground plane (Mode 7 scanlines)
    for (let y = horizonY; y < rh; y++) {
      const screenY = y - horizonY;
      if (screenY === 0) continue;

      // Perspective: distance from camera for this scanline
      const rowDist = CAMERA_HEIGHT * (rh * 0.5) / screenY;
      if (rowDist > DRAW_DISTANCE) continue;

      for (let x = 0; x < rw; x++) {
        // X offset from center of screen, mapped to angle offset
        const screenXNorm = (x / rw - 0.5) * 2;
        const angleOffset = screenXNorm * halfFov;

        // World position for this pixel
        const rayAngle = camAngle[pi] + angleOffset;
        const wx = camX[pi] + Math.cos(rayAngle) * rowDist;
        const wy = camY[pi] + Math.sin(rayAngle) * rowDist;

        const [cr, cg, cb] = getGroundColor(wx, wy, rowDist);

        const idx = (y * rw + x) * 4;
        pixels[idx] = cr;
        pixels[idx + 1] = cg;
        pixels[idx + 2] = cb;
        pixels[idx + 3] = 255;
      }
    }

    offCtx.putImageData(imgData, 0, 0);

    // Draw opponent car as a projected sprite
    const numRacers = gameMode === '1p-timetrial' ? 1 : 2;
    if (numRacers > 1) {
      const oi = pi === 0 ? 1 : 0;
      drawProjectedCar(offCtx, pi, oi, rw, rh, horizonY);
    }

    // Draw player car sprite at bottom center
    drawPlayerCarSprite(offCtx, pi, rw, rh);

    // Speed lines effect
    const stats = getCarStats(players[pi]);
    const speedPct = Math.abs(players[pi].speed) / stats.topSpeed;
    if (speedPct > 0.6) {
      const intensity = (speedPct - 0.6) / 0.4;
      offCtx.strokeStyle = `rgba(255,255,255,${intensity * 0.3})`;
      offCtx.lineWidth = 1;
      for (let i = 0; i < 8; i++) {
        const sx = Math.random() * rw;
        const sy = rh * 0.5 + Math.random() * rh * 0.5;
        offCtx.beginPath();
        offCtx.moveTo(sx, sy);
        offCtx.lineTo(sx + (sx - rw / 2) * 0.3, sy + 20 + intensity * 30);
        offCtx.stroke();
      }
    }

    // HUD overlay on offscreen canvas
    drawHUD(offCtx, pi, rw, rh);

    // Minimap
    drawMinimap(offCtx, pi, rw, rh);

    // Blit to main canvas, upscaled
    targetCtx.imageSmoothingEnabled = false;
    targetCtx.drawImage(offCanvas, 0, 0, rw, rh, vx, vy, vw, vh);
  }

  function drawProjectedCar(c: CanvasRenderingContext2D, viewerIdx: number, carIdx: number, rw: number, rh: number, horizonY: number) {
    const car = players[carIdx];
    const halfFov = (FOV * Math.PI / 180) / 2;

    // Vector from camera to opponent
    const dx = car.x - camX[viewerIdx];
    const dy = car.y - camY[viewerIdx];
    const dist = Math.sqrt(dx * dx + dy * dy);
    if (dist < 1 || dist > DRAW_DISTANCE) return;

    // Angle from camera forward to opponent
    const worldAngle = Math.atan2(dy, dx);
    let relAngle = worldAngle - camAngle[viewerIdx];
    while (relAngle > Math.PI) relAngle -= Math.PI * 2;
    while (relAngle < -Math.PI) relAngle += Math.PI * 2;

    // Check if in field of view
    if (Math.abs(relAngle) > halfFov * 1.2) return;

    // Screen X position
    // Use tangent-based projection for correct perspective
    const screenX = (Math.tan(relAngle) / Math.tan(halfFov) * 0.5 + 0.5) * rw;

    // Screen Y position (based on distance/perspective)
    const projectedY = CAMERA_HEIGHT * (rh * 0.5) / dist;
    const screenY = horizonY + projectedY;

    // Size based on distance
    const baseSize = 30;
    const size = baseSize * (rh * 0.5) / (dist * 0.8);
    if (size < 2) return;

    const paint = car.car.paint?.color ?? (carIdx === 0 ? '#ff4444' : '#4488ff');

    c.save();
    c.translate(screenX, screenY);

    // Car body (rear view)
    const w = size * 1.4;
    const h = size;
    c.fillStyle = paint;
    c.beginPath();
    c.moveTo(-w / 2, h * 0.3);
    c.lineTo(-w * 0.4, -h * 0.3);
    c.lineTo(-w * 0.2, -h * 0.5);
    c.lineTo(w * 0.2, -h * 0.5);
    c.lineTo(w * 0.4, -h * 0.3);
    c.lineTo(w / 2, h * 0.3);
    c.closePath();
    c.fill();
    c.strokeStyle = '#000';
    c.lineWidth = 1;
    c.stroke();

    // Wheels
    c.fillStyle = '#222';
    c.fillRect(-w / 2 - 3, -h * 0.2, 5, h * 0.5);
    c.fillRect(w / 2 - 2, -h * 0.2, 5, h * 0.5);

    // Rear window
    c.fillStyle = 'rgba(100,150,200,0.5)';
    c.fillRect(-w * 0.25, -h * 0.45, w * 0.5, h * 0.25);

    // Label
    if (size > 10) {
      const label = gameMode === '2p' ? `P${carIdx + 1}` : carIdx === 0 ? 'P1' : 'AI';
      c.fillStyle = '#fff';
      c.font = `bold ${Math.max(8, Math.floor(size * 0.4))}px sans-serif`;
      c.textAlign = 'center';
      c.fillText(label, 0, -h * 0.55 - 4);
    }

    // Exhaust
    if (car.speed > 1) {
      c.fillStyle = `rgba(255, ${100 + Math.random() * 100}, 0, ${0.4 + Math.random() * 0.3})`;
      c.beginPath();
      c.arc(-w * 0.15, h * 0.35 + Math.random() * 3, 2 + Math.random() * 3, 0, Math.PI * 2);
      c.fill();
      c.beginPath();
      c.arc(w * 0.15, h * 0.35 + Math.random() * 3, 2 + Math.random() * 3, 0, Math.PI * 2);
      c.fill();
    }

    c.restore();
  }

  function drawPlayerCarSprite(c: CanvasRenderingContext2D, pi: number, rw: number, rh: number) {
    const p = players[pi];
    const paint = p.car.paint?.color ?? (pi === 0 ? '#ff4444' : '#4488ff');
    const steer = pi < 2 ? stickX(pi) : 0;

    const carW = 70;
    const carH = 50;
    const cx = rw / 2;
    const cy = rh - carH / 2 - 15;

    c.save();
    c.translate(cx, cy);
    // Slight tilt with steering
    c.rotate(steer * 0.08);

    // Shadow
    c.fillStyle = 'rgba(0,0,0,0.3)';
    c.beginPath();
    c.ellipse(0, carH * 0.3, carW * 0.5, 8, 0, 0, Math.PI * 2);
    c.fill();

    // Car body (rear view)
    c.fillStyle = paint;
    c.beginPath();
    c.moveTo(-carW / 2, carH * 0.3);
    c.lineTo(-carW * 0.4, -carH * 0.2);
    c.lineTo(-carW * 0.3, -carH * 0.4);
    c.lineTo(-carW * 0.15, -carH * 0.5);
    c.lineTo(carW * 0.15, -carH * 0.5);
    c.lineTo(carW * 0.3, -carH * 0.4);
    c.lineTo(carW * 0.4, -carH * 0.2);
    c.lineTo(carW / 2, carH * 0.3);
    c.closePath();
    c.fill();
    c.strokeStyle = '#000';
    c.lineWidth = 2;
    c.stroke();

    // Roof/cabin
    c.fillStyle = 'rgba(100,150,200,0.5)';
    c.beginPath();
    c.moveTo(-carW * 0.2, -carH * 0.45);
    c.lineTo(-carW * 0.15, -carH * 0.55);
    c.lineTo(carW * 0.15, -carH * 0.55);
    c.lineTo(carW * 0.2, -carH * 0.45);
    c.closePath();
    c.fill();

    // Wheels (visible from behind)
    const wheelColor = p.car.wheels?.id === 'fire' ? '#ff6600' : '#222';
    c.fillStyle = wheelColor;
    c.fillRect(-carW / 2 - 6, -carH * 0.15, 8, carH * 0.45);
    c.fillRect(carW / 2 - 2, -carH * 0.15, 8, carH * 0.45);

    // Taillights
    c.fillStyle = '#ff2222';
    c.fillRect(-carW * 0.38, carH * 0.15, 8, 5);
    c.fillRect(carW * 0.38 - 8, carH * 0.15, 8, 5);

    // Spoiler
    if (p.car.spoiler && p.car.spoiler.id !== 'none') {
      c.fillStyle = '#555';
      c.fillRect(-carW * 0.35, -carH * 0.5 - 6, carW * 0.7, 4);
      // Spoiler supports
      c.fillRect(-carW * 0.25, -carH * 0.5 - 6, 3, 8);
      c.fillRect(carW * 0.25 - 3, -carH * 0.5 - 6, 3, 8);
    }

    // Exhaust flames
    if (p.speed > 1) {
      const intensity = Math.min(p.speed / 4, 1);
      c.fillStyle = `rgba(255, ${120 + Math.random() * 80}, 0, ${0.5 + intensity * 0.4})`;
      c.beginPath();
      c.arc(-carW * 0.12, carH * 0.35 + Math.random() * 5, 3 + intensity * 5, 0, Math.PI * 2);
      c.fill();
      c.beginPath();
      c.arc(carW * 0.12, carH * 0.35 + Math.random() * 5, 3 + intensity * 5, 0, Math.PI * 2);
      c.fill();
    }

    c.restore();
  }

  function drawHUD(c: CanvasRenderingContext2D, pi: number, rw: number, rh: number) {
    const p = players[pi];
    const stats = getCarStats(p);
    const speedPct = Math.abs(p.speed) / stats.topSpeed;

    // Lap counter (top left)
    c.fillStyle = 'rgba(0,0,0,0.6)';
    c.fillRect(10, 10, 160, 45);
    c.fillStyle = '#ffd93d';
    c.font = 'bold 16px sans-serif';
    c.textAlign = 'left';
    const hudLabel = gameMode === '2p' ? `Player ${pi + 1}` :
                     pi === 0 ? 'You' : 'Computer';
    c.fillText(hudLabel, 18, 28);
    c.fillStyle = '#fff';
    c.font = '14px sans-serif';
    c.fillText(`Lap ${Math.min(p.lap + 1, LAPS_TO_WIN)} / ${LAPS_TO_WIN}`, 18, 47);

    // Speed bar (bottom center, above car)
    const barW = 120;
    const barH = 10;
    const barX = rw / 2 - barW / 2;
    const barY = rh - 80;
    c.fillStyle = 'rgba(0,0,0,0.5)';
    c.fillRect(barX - 2, barY - 2, barW + 4, barH + 4);
    c.fillStyle = '#333';
    c.fillRect(barX, barY, barW, barH);
    c.fillStyle = speedPct > 0.7 ? '#ff4444' : '#6bcb77';
    c.fillRect(barX, barY, barW * speedPct, barH);

    // Speed text
    const kmh = Math.floor(speedPct * 300);
    c.fillStyle = '#fff';
    c.font = 'bold 12px sans-serif';
    c.textAlign = 'center';
    c.fillText(`${kmh} km/h`, rw / 2, barY - 4);

    // Time trial timer
    if (gameMode === '1p-timetrial' && gamePhase === 'race' && pi === 0) {
      const elapsed = (performance.now() - raceStartTime) / 1000;
      const mins = Math.floor(elapsed / 60);
      const secs = Math.floor(elapsed % 60);
      const ms = Math.floor((elapsed % 1) * 100);
      const timeStr = `${mins}:${secs.toString().padStart(2, '0')}.${ms.toString().padStart(2, '0')}`;
      c.fillStyle = 'rgba(0,0,0,0.6)';
      c.fillRect(rw / 2 - 70, 10, 140, 35);
      c.fillStyle = '#ffd93d';
      c.font = 'bold 20px sans-serif';
      c.textAlign = 'center';
      c.fillText(timeStr, rw / 2, 35);
    }
  }

  function drawMinimap(c: CanvasRenderingContext2D, pi: number, rw: number, rh: number) {
    const mapSize = 80;
    const mapX = rw - mapSize - 10;
    const mapY = 10;
    const mapCx = mapX + mapSize / 2;
    const mapCy = mapY + mapSize / 2;

    // Background
    c.fillStyle = 'rgba(0,0,0,0.6)';
    c.fillRect(mapX, mapY, mapSize, mapSize);

    // Scale track to fit minimap
    const scaleX = (mapSize * 0.4) / TRACK_RX;
    const scaleY = (mapSize * 0.4) / TRACK_RY;
    const scale = Math.min(scaleX, scaleY);

    // Draw track ellipse
    c.beginPath();
    c.ellipse(mapCx, mapCy, TRACK_RX * scale, TRACK_RY * scale, 0, 0, Math.PI * 2);
    c.strokeStyle = '#666';
    c.lineWidth = Math.max(2, TRACK_WIDTH * scale);
    c.stroke();

    // Draw car dots
    const numRacers = gameMode === '1p-timetrial' ? 1 : 2;
    const dotColors = ['#ff4444', '#4488ff'];
    for (let i = 0; i < numRacers; i++) {
      const p = players[i];
      const dx = (p.x - TRACK_CENTER_X) * scale;
      const dy = (p.y - TRACK_CENTER_Y) * scale;
      c.fillStyle = dotColors[i];
      c.beginPath();
      c.arc(mapCx + dx, mapCy + dy, i === pi ? 4 : 3, 0, Math.PI * 2);
      c.fill();
      // Bright border for current player
      if (i === pi) {
        c.strokeStyle = '#fff';
        c.lineWidth = 1;
        c.stroke();
      }
    }
  }

  function drawRace() {
    if (!raceCanvas) return;
    if (!ctx || ctx.canvas !== raceCanvas) {
      raceCanvas.width = window.innerWidth;
      raceCanvas.height = window.innerHeight;
      ctx = raceCanvas.getContext('2d')!;
    }
    const W = raceCanvas.width;
    const H = raceCanvas.height;

    // Update cameras
    const numRacers = gameMode === '1p-timetrial' ? 1 : 2;
    for (let i = 0; i < numRacers; i++) {
      updateCamera(i);
    }

    if (gameMode === '2p') {
      // Split screen: top half = P1, bottom half = P2
      const halfH = Math.floor(H / 2);
      renderView(ctx, 0, 0, 0, W, halfH);
      renderView(ctx, 1, 0, halfH, W, halfH);

      // Divider line
      ctx.fillStyle = '#222';
      ctx.fillRect(0, halfH - 2, W, 4);
    } else {
      // Single player: full screen
      renderView(ctx, 0, 0, 0, W, H);
    }
  }

  // ==================== WIN ====================
  function updateWin() {
    if (btnStart(0) || (gameMode === '2p' && btnStart(1))) {
      gamePhase = 'title';
      menuIdx = 0;
    }
  }

  function formatTime(ms: number): string {
    const totalSecs = ms / 1000;
    const mins = Math.floor(totalSecs / 60);
    const secs = Math.floor(totalSecs % 60);
    const centis = Math.floor((totalSecs % 1) * 100);
    return `${mins}:${secs.toString().padStart(2, '0')}.${centis.toString().padStart(2, '0')}`;
  }

  // ==================== MAIN LOOP ====================
  function gameLoop(now: number) {
    const dt = Math.min((now - lastTime) / 1000, 0.1); // Cap at 100ms to prevent huge jumps
    lastTime = now;

    pollGamepads();

    switch (gamePhase) {
      case 'title': updateTitle(); break;
      case 'build': updateBuild(); break;
      case 'countdown': updateCountdown(dt); break;
      case 'race': updateRace(dt); break;
      case 'win': updateWin(); break;
    }

    savePrevButtons();
    animFrame = requestAnimationFrame(gameLoop);
  }

  function handleResize() {
    if (raceCanvas && (gamePhase === 'race' || gamePhase === 'countdown')) {
      raceCanvas.width = window.innerWidth;
      raceCanvas.height = window.innerHeight;
    }
  }

  onMount(() => {
    lastTime = performance.now();
    animFrame = requestAnimationFrame(gameLoop);
    window.addEventListener('resize', handleResize);
  });

  onDestroy(() => {
    if (animFrame) cancelAnimationFrame(animFrame);
    window.removeEventListener('resize', handleResize);
  });
</script>

<svelte:head>
  <title>Car Builder Racing!</title>
</svelte:head>

<div class="game-container">
  <!-- TITLE SCREEN -->
  {#if gamePhase === 'title'}
    <div class="screen title-screen">
      <h1 class="title">Car Builder Racing!</h1>
      <div class="subtitle">Build it. Paint it. Race it!</div>

      {#if players[0].connected}
        <div class="mode-menu">
          {#each MENU_OPTIONS as opt, i}
            <div
              class="menu-item"
              class:selected={menuIdx === i}
              class:disabled={opt.needs2 && !players[1].connected}
            >
              {opt.label}
              {#if opt.needs2 && !players[1].connected}
                <span class="needs-controller">(needs 2 controllers)</span>
              {/if}
            </div>
          {/each}
        </div>
        <div class="menu-hint">D-Pad Up/Down to choose, X (Cross) to select</div>
      {:else}
        <div class="connect-info">
          Connect a controller to play!
        </div>
      {/if}

      <div class="controller-status" class:connected={players[0].connected} class:waiting={!players[0].connected}>
        Player 1: {players[0].connected ? 'Connected!' : 'Press any button on controller...'}
      </div>
      <div class="controller-status" class:connected={players[1].connected} class:waiting={!players[1].connected}>
        Player 2: {players[1].connected ? 'Connected!' : 'Not connected'}
      </div>
      {#if debugInfo}
        <pre class="debug-info">{debugInfo}</pre>
      {/if}
    </div>
  {/if}

  <!-- BUILD SCREEN -->
  {#if gamePhase === 'build'}
    <div class="screen build-screen" class:single-player={is1P()}>
      {#if is1P()}
        <!-- 1 Player build: only show P1 panel, centered -->
        <div class="build-half center">
          <h2 style="color: #ff6b6b">Build Your Car!</h2>

          {#if players[0].ready}
            <div class="car-preview">{getCarPreviewEmoji(players[0].car)}</div>
            <div class="ready-badge">READY!</div>
          {:else}
            <div class="step-label">{BUILD_STEPS[players[0].buildStep].name}</div>
            <div class="car-preview">{getCarPreviewEmoji(players[0].car)}</div>
            <div class="options-grid">
              {#each BUILD_STEPS[players[0].buildStep].options as opt, i}
                <div
                  class="option-card"
                  class:hovered={i === players[0].hoverIdx}
                >
                  <span class="opt-emoji">{opt.emoji}</span>
                  <span class="opt-label">{opt.label}</span>
                </div>
              {/each}
            </div>
            <div class="build-instructions">D-Pad to move / X (Cross) to select / O (Circle) to go back</div>
          {/if}

          {#if gameMode === '1p-ai'}
            <div class="ai-car-info">
              Computer's car: {getCarPreviewEmoji(players[1].car)}
            </div>
          {/if}
        </div>
      {:else}
        <!-- 2 Player build -->
        {#each [0, 1] as pi}
          <div class="build-half" class:left={pi === 0} class:right={pi === 1}>
            <h2 style="color: {pi === 0 ? '#ff6b6b' : '#4d96ff'}">Player {pi + 1}</h2>

            {#if players[pi].ready}
              <div class="car-preview">{getCarPreviewEmoji(players[pi].car)}</div>
              <div class="ready-badge">READY!</div>
            {:else}
              <div class="step-label">{BUILD_STEPS[players[pi].buildStep].name}</div>
              <div class="car-preview">{getCarPreviewEmoji(players[pi].car)}</div>
              <div class="options-grid">
                {#each BUILD_STEPS[players[pi].buildStep].options as opt, i}
                  <div
                    class="option-card"
                    class:hovered={i === players[pi].hoverIdx}
                  >
                    <span class="opt-emoji">{opt.emoji}</span>
                    <span class="opt-label">{opt.label}</span>
                  </div>
                {/each}
              </div>
              <div class="build-instructions">D-Pad to move / X (Cross) to select / O (Circle) to go back</div>
            {/if}
          </div>
        {/each}
      {/if}
    </div>
  {/if}

  <!-- RACE CANVAS (shared across countdown, race, and win phases) -->
  {#if gamePhase === 'countdown' || gamePhase === 'race' || gamePhase === 'win'}
    <canvas bind:this={raceCanvas} class="race-canvas"></canvas>
  {/if}

  <!-- COUNTDOWN -->
  {#if gamePhase === 'countdown'}
    <div class="countdown-overlay" style="color: {countdownColor}">
      {countdownText}
    </div>
  {/if}

  <!-- WIN SCREEN -->
  {#if gamePhase === 'win'}
    <div class="win-overlay">
      {#if gameMode === '1p-timetrial'}
        <h1 class="win-text" style="color: #ffd93d">
          Finished!
        </h1>
        <div class="winner-car">{players[0].car.body?.emoji ?? '🏎️'}</div>
        <div class="finish-time">{formatTime(players[0].finishTime)}</div>
      {:else if gameMode === '1p-ai'}
        <h1 class="win-text" style="color: {winner === 0 ? '#6bcb77' : '#ff6b6b'}">
          {winner === 0 ? 'You Win!' : 'Computer Wins!'}
        </h1>
        <div class="winner-car">{players[winner].car.body?.emoji ?? '🏎️'}</div>
      {:else}
        <h1 class="win-text" style="color: {winner === 0 ? '#ff6b6b' : '#4d96ff'}">
          Player {winner + 1} Wins!
        </h1>
        <div class="winner-car">{players[winner].car.body?.emoji ?? '🏎️'}</div>
      {/if}
      <div class="play-again">Press START to play again!</div>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    background: #1a1a2e;
    font-family: 'Segoe UI', Arial, sans-serif;
    color: white;
    user-select: none;
  }

  .game-container {
    position: fixed;
    inset: 0;
  }

  .screen {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

  /* TITLE */
  .title-screen {
    background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
  }

  .title {
    font-size: 64px;
    background: linear-gradient(90deg, #ff6b6b, #ffd93d, #6bcb77, #4d96ff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    animation: titlePulse 2s ease-in-out infinite;
  }

  @keyframes titlePulse {
    0%, 100% { transform: scale(1); }
    50% { transform: scale(1.05); }
  }

  .subtitle { font-size: 28px; color: #aaa; margin-top: 20px; }

  .connect-info {
    font-size: 22px; color: #ffd93d; margin-top: 40px;
    animation: blink 1.5s ease-in-out infinite;
  }

  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.3; }
  }

  .controller-status {
    font-size: 20px; margin-top: 15px; padding: 10px 20px; border-radius: 10px;
  }
  .connected { background: #2d6a2d; color: #6bcb77; }
  .waiting { background: #6a2d2d; color: #ff6b6b; }

  .debug-info {
    font-size: 12px; color: #666; margin-top: 20px;
    font-family: monospace; text-align: left;
    background: rgba(0,0,0,0.3); padding: 10px; border-radius: 8px;
    max-width: 600px; white-space: pre-wrap;
  }

  /* MODE MENU */
  .mode-menu {
    margin-top: 40px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .menu-item {
    font-size: 26px;
    padding: 15px 40px;
    border: 3px solid #555;
    border-radius: 12px;
    background: #222;
    text-align: center;
    transition: all 0.2s;
  }

  .menu-item.selected {
    border-color: #ffd93d;
    background: #3a3520;
    transform: scale(1.05);
  }

  .menu-item.disabled {
    opacity: 0.4;
  }

  .needs-controller {
    font-size: 14px;
    color: #ff6b6b;
    display: block;
  }

  .menu-hint {
    font-size: 16px;
    color: #888;
    margin-top: 20px;
  }

  /* BUILD */
  .build-screen {
    flex-direction: row;
    background: linear-gradient(135deg, #0f0c29, #302b63);
  }

  .build-screen.single-player {
    justify-content: center;
  }

  .build-half {
    width: 50%; height: 100%;
    display: flex; flex-direction: column;
    align-items: center; padding: 20px;
    justify-content: center;
  }
  .build-half.left { border-right: 3px solid #444; }
  .build-half.center { width: 60%; }

  .step-label { font-size: 24px; color: #ffd93d; margin-bottom: 10px; }

  .car-preview {
    font-size: 50px; margin: 10px 0; min-height: 70px;
    display: flex; align-items: center; justify-content: center;
  }

  .options-grid {
    display: flex; flex-wrap: wrap; justify-content: center; gap: 10px;
    max-width: 400px;
  }

  .option-card {
    width: 100px; height: 100px;
    border: 3px solid #555; border-radius: 12px;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    background: #222; transition: all 0.2s;
  }
  .option-card.hovered {
    border-color: #ffd93d; background: #3a3520; transform: scale(1.1);
  }
  .opt-emoji { font-size: 40px; }
  .opt-label { font-size: 12px; margin-top: 4px; }

  .build-instructions { font-size: 16px; color: #888; margin-top: 15px; }

  .ready-badge {
    font-size: 36px; color: #6bcb77; margin-top: 20px;
    animation: blink 1s ease-in-out infinite;
  }

  .ai-car-info {
    font-size: 20px;
    color: #4d96ff;
    margin-top: 30px;
    padding: 10px 20px;
    background: rgba(77, 150, 255, 0.1);
    border: 2px solid #4d96ff;
    border-radius: 10px;
  }

  /* RACE */
  .race-canvas {
    position: absolute; inset: 0;
    width: 100%; height: 100%;
    image-rendering: pixelated;
    image-rendering: crisp-edges;
  }

  /* COUNTDOWN */
  .countdown-overlay {
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center;
    background: rgba(0,0,0,0.7);
    font-size: 120px; font-weight: bold;
    z-index: 10;
  }

  /* WIN */
  .win-overlay {
    position: absolute; inset: 0;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    background: rgba(0,0,0,0.85);
    z-index: 10;
  }
  .win-text { font-size: 64px; }
  .winner-car { font-size: 100px; margin: 20px; }
  .play-again {
    font-size: 28px; color: #6bcb77; margin-top: 30px;
    animation: blink 1.5s ease-in-out infinite;
  }

  .finish-time {
    font-size: 56px;
    color: #ffd93d;
    font-weight: bold;
    margin: 10px 0;
  }
</style>
