<script>
  import { onDestroy, onMount, tick } from 'svelte';
  import { MoveDiagonal2, RotateCcw, RotateCw, SkipForward, Timer } from 'lucide-svelte';
  import { createClient } from '@supabase/supabase-js';
  import { UMAP } from 'umap-js';
  import gsap from 'gsap';
  import './app.css';

  const WORDS = [
    'to roll',
    'to crease',
    'to fold',
    'to store',
    'to bend',
    'to shorten',
    'to twist',
    'to dapple',
    'to crumple',
    'to shave',
    'to tear',
    'to chip',
    'to split',
    'to cut',
    'to sever',
    'to drop',
    'to remove',
    'to simplify',
    'to differ',
    'to disarrange',
    'to open',
    'to mix',
    'to splash',
    'to knot',
    'to spill',
    'to droop',
    'to flow',
    'to curve',
    'to lift',
    'to inlay',
    'to impress',
    'to fire',
    'to bury',
    'to smear',
    'to rotate',
    'to swirl',
    'to support',
    'to hook',
    'to suspend',
    'to spread',
    'to hang',
    'to collect',
    'of tension',
    'of gravity',
    'of entropy',
    'of nature',
    'of grouping',
    'of layering',
    'of felting',
    'to grasp',
    'to tighten',
    'to bundle',
    'to heap',
    'to gather',
    'to scatter',
    'to arrange',
    'to repair',
    'to discard',
    'to pair',
    'to distribute',
    'to surfeit',
    'to compliment',
    'to enclose',
    'to surround',
    'to encircle',
    'to hole',
    'to cover',
    'to wrap',
    'to dig',
    'to tie',
    'to bind',
    'to weave',
    'to join',
    'to match',
    'to laminate',
    'to bond',
    'to hinge',
    'to mark',
    'to expand',
    'to dilute',
    'to light',
    'to modulate',
    'to distill',
    'of waves',
    'of electromagnetic',
    'of inertia',
    'of ionization',
    'of polarization',
    'of refraction',
    'of tides',
    'of reflection',
    'of equilibrium',
    'of symmetry',
    'of friction',
    'to stretch',
    'to bounce',
    'to erase',
    'to spray',
    'to systematize',
    'to refer',
    'to force',
    'of mapping',
    'of location',
    'of context',
    'of time',
    'of carbonization',
    'to continue'
  ];

  const WORDS_PER_ROUND = 5;
  const MINUTES_PER_WORD = 1;
  const SECONDS_TOTAL = MINUTES_PER_WORD * 60;
  const MIN_SQUARE_SIZE = 2;
  const DRAG_ACTIVATION_THRESHOLD = 0.2;
  const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL;
  const SUPABASE_ANON_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY;
  const supabase = SUPABASE_URL && SUPABASE_ANON_KEY
    ? createClient(SUPABASE_URL, SUPABASE_ANON_KEY)
    : null;
  const PIXEL_TILE_SIZE = 130;
  const PIXEL_TILE_GAP = 34;
  const PIXEL_MIN_SCALE = 0.24;
  const PIXEL_MAX_SCALE = 3.6;
  const PIXEL_SPACE_CACHE_KEY = 'gestalt-pixel-space-cache-v1';
  const PIXEL_SPACE_FETCH_LIMIT = 900;
  const PIXEL_SPACE_CAMERA_LERP = 0.2;
  const PIXEL_SPACE_KEYBOARD_PAN_STEP = 44;
  const PIXEL_SPACE_INERTIA_FRICTION = 0.92;
  const PIXEL_SPACE_MIN_INERTIA_SPEED = 0.015;
  const CORNER_HANDLES = [
    { x: -1, y: -1, key: 'tl' },
    { x: 1, y: -1, key: 'tr' },
    { x: -1, y: 1, key: 'bl' },
    { x: 1, y: 1, key: 'br' }
  ];

  function shuffle(arr) {
    const a = [...arr];
    for (let i = a.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [a[i], a[j]] = [a[j], a[i]];
    }
    return a;
  }

  let started = false;
  let roundFinished = false;
  let roundWords = [];
  let roundResults = []; // { word, squares } for each word when round ends
  let wordIndex = 0;
  let squares = [];
  let nextId = 1;
  let historyPast = [];
  let historyFuture = [];
  let timeLeft = SECONDS_TOTAL;
  let timerId = null;
  let gameMode = 'timed'; // 'timed' | 'free'

  let canvasWrap = null;
  let borderSquareRef = null;
  let mouseCanvasX = 50;
  let mouseCanvasY = 50;
  let creationState = null; // { pointerId, startX, startY, currentX, currentY }
  let dragState = null; // { pointerId, squareId, selectedIds, startPointerX, startPointerY, startPositions, moved, hasSnapshot, copySourceId, copySourceIds, copyStartX, copyStartY }
  let transformState = null; // { pointerId, squareId, mode, hasSnapshot, ... }
  let selectedSquareIds = [];
  let selectionAnchorId = null;
  let altPressed = false;
  let hoverTransformMode = null; // 'resize' | 'rotate' | null
  let copiedSquares = [];
  let pasteCount = 0;
  let username = '';
  let submitStatus = '';
  let submitBusy = false;
  let showPixelSpace = false;
  let pixelSpaceWrap = null;
  let pixelSpaceDrawings = [];
  let pixelSpaceLoading = false;
  let pixelSpaceError = '';
  let pixelSpacePanState = null; // { pointerId, startClientX, startClientY, startX, startY, lastClientX, lastClientY, lastTime, velocityX, velocityY }
  let pixelSpaceCamera = { x: 0, y: 0, scale: 1 };
  let pixelSpaceCameraTarget = { x: 0, y: 0, scale: 1 };
  let pixelSpaceCameraFrame = null;
  let pixelSpaceInertiaFrame = null;
  let pixelSpaceRealtimeChannel = null;
  let pixelSpaceSelectedDrawingId = null;
  let pixelSpaceLastLoadedAt = '';
  let pixelSpaceLoadedFromCache = false;
  let pixelSpaceStatus = '';

  $: currentWord = roundWords[wordIndex] ?? '';
  $: pixelSpaceTransform = `translate(${pixelSpaceCamera.x}px, ${pixelSpaceCamera.y}px) scale(${pixelSpaceCamera.scale})`;
  $: pixelSpaceSelectedDrawing = pixelSpaceDrawings.find((drawing) => drawing.id === pixelSpaceSelectedDrawingId) ?? null;

  onMount(() => {
    if (typeof window === 'undefined') return;
    const savedUsername = window.localStorage.getItem('gestalt-username');
    if (savedUsername) {
      username = savedUsername;
    }
  });

  onDestroy(() => {
    stopPixelSpaceCameraAnimation();
    stopPixelSpaceInertia();
    unsubscribeFromPixelSpaceRealtime();
  });

  function getRectangleFromDrag(startX, startY, currentX, currentY) {
    const dx = currentX - startX;
    const dy = currentY - startY;
    const side = Math.max(Math.abs(dx), Math.abs(dy));
    const x = startX + (dx < 0 ? -side : 0);
    const y = startY + (dy < 0 ? -side : 0);
    return { x, y, width: side, height: side, rotation: 0 };
  }

  function cloneSquares(items) {
    return items.map((square) => ({ ...square }));
  }

  function normalizeRotation(rotation) {
    const value = rotation % 360;
    return value < 0 ? value + 360 : value;
  }

  function getSquareCenter(square) {
    return {
      x: square.x + square.width / 2,
      y: square.y + square.height / 2
    };
  }

  function rotatePoint(x, y, angleDeg) {
    const radians = (angleDeg * Math.PI) / 180;
    const cos = Math.cos(radians);
    const sin = Math.sin(radians);
    return {
      x: x * cos - y * sin,
      y: x * sin + y * cos
    };
  }

  function worldToSquareLocal(square, worldX, worldY) {
    const center = getSquareCenter(square);
    const dx = worldX - center.x;
    const dy = worldY - center.y;
    return rotatePoint(dx, dy, -square.rotation);
  }

  function squareLocalToWorld(square, localX, localY) {
    const center = getSquareCenter(square);
    const rotated = rotatePoint(localX, localY, square.rotation);
    return {
      x: center.x + rotated.x,
      y: center.y + rotated.y
    };
  }

  function pointerAngleFromSquare(square, worldX, worldY) {
    const center = getSquareCenter(square);
    const dx = worldX - center.x;
    const dy = worldY - center.y;
    return (Math.atan2(dy, dx) * 180) / Math.PI;
  }

  function getSquareWorldCorners(square) {
    const halfWidth = square.width / 2;
    const halfHeight = square.height / 2;
    return CORNER_HANDLES.map((corner) => squareLocalToWorld(
      square,
      corner.x * halfWidth,
      corner.y * halfHeight
    ));
  }

  function getSquareWorldBounds(square) {
    const corners = getSquareWorldCorners(square);
    const xs = corners.map((corner) => corner.x);
    const ys = corners.map((corner) => corner.y);
    const minX = Math.min(...xs);
    const maxX = Math.max(...xs);
    const minY = Math.min(...ys);
    const maxY = Math.max(...ys);
    return {
      x: minX,
      y: minY,
      width: maxX - minX,
      height: maxY - minY
    };
  }

  function getSelectionBounds(items) {
    if (!items.length) return null;
    const bounds = items.map((square) => getSquareWorldBounds(square));
    const minX = Math.min(...bounds.map((bound) => bound.x));
    const minY = Math.min(...bounds.map((bound) => bound.y));
    const maxX = Math.max(...bounds.map((bound) => bound.x + bound.width));
    const maxY = Math.max(...bounds.map((bound) => bound.y + bound.height));
    return {
      x: minX,
      y: minY,
      width: maxX - minX,
      height: maxY - minY
    };
  }

  function captureDrawingState() {
    return {
      squares: cloneSquares(squares),
      nextId
    };
  }

  function applyDrawingState(state) {
    squares = cloneSquares(state.squares);
    nextId = state.nextId;
  }

  function resetHistory() {
    historyPast = [];
    historyFuture = [];
  }

  function pushHistorySnapshot() {
    historyPast = [...historyPast, captureDrawingState()];
    historyFuture = [];
  }

  function undoDrawing() {
    if (historyPast.length === 0) return;
    const previous = historyPast[historyPast.length - 1];
    const current = captureDrawingState();
    historyPast = historyPast.slice(0, -1);
    historyFuture = [...historyFuture, current];
    applyDrawingState(previous);
  }

  function redoDrawing() {
    if (historyFuture.length === 0) return;
    const next = historyFuture[historyFuture.length - 1];
    const current = captureDrawingState();
    historyFuture = historyFuture.slice(0, -1);
    historyPast = [...historyPast, current];
    applyDrawingState(next);
  }

  function clearSelection() {
    selectedSquareIds = [];
    selectionAnchorId = null;
  }

  function setSingleSelection(squareId) {
    selectedSquareIds = [squareId];
    selectionAnchorId = squareId;
  }

  function selectRangeTo(squareId) {
    const targetIndex = squares.findIndex((square) => square.id === squareId);
    if (targetIndex === -1) return;
    if (selectionAnchorId === null) {
      selectionAnchorId = squareId;
    }
    const anchorId = selectionAnchorId ?? squareId;
    const anchorIndex = squares.findIndex((square) => square.id === anchorId);
    if (anchorIndex === -1) {
      setSingleSelection(squareId);
      return;
    }
    const start = Math.min(anchorIndex, targetIndex);
    const end = Math.max(anchorIndex, targetIndex);
    const rangeIds = squares.slice(start, end + 1).map((square) => square.id);
    selectedSquareIds = Array.from(new Set([...selectedSquareIds, ...rangeIds]));
  }

  function deleteSelectedSquares() {
    if (selectedSquareIds.length === 0) return;
    const selectedIdSet = new Set(selectedSquareIds);
    const hasTargetSquare = squares.some((square) => selectedIdSet.has(square.id));
    if (!hasTargetSquare) return;
    pushHistorySnapshot();
    squares = squares.filter((square) => !selectedIdSet.has(square.id));
    clearSelection();
  }

  function copySelectedSquares() {
    if (selectedSquareIds.length === 0) return;
    const selectedIdSet = new Set(selectedSquareIds);
    const selectedSquares = squares.filter((square) => selectedIdSet.has(square.id));
    if (selectedSquares.length === 0) return;
    copiedSquares = cloneSquares(selectedSquares);
    pasteCount = 0;
  }

  function pasteCopiedSquares() {
    if (copiedSquares.length === 0) return;
    const pasteOffset = 2 + pasteCount * 2;
    pushHistorySnapshot();
    const pastedIds = [];
    const pastedSquares = copiedSquares.map((square) => {
      const pastedSquare = {
        ...square,
        id: nextId++,
        x: square.x + pasteOffset,
        y: square.y + pasteOffset
      };
      pastedIds.push(pastedSquare.id);
      return pastedSquare;
    });
    squares = [...squares, ...pastedSquares];
    selectedSquareIds = pastedIds;
    selectionAnchorId = pastedIds[pastedIds.length - 1] ?? null;
    pasteCount += 1;
  }

  $: previewSquare = (() => {
    if (!creationState) return null;
    const rectangle = getRectangleFromDrag(
      creationState.startX,
      creationState.startY,
      creationState.currentX,
      creationState.currentY
    );
    return rectangle.width > 0 && rectangle.height > 0 ? rectangle : null;
  })();

  function mean(values) {
    if (!values.length) return 0;
    return values.reduce((sum, value) => sum + value, 0) / values.length;
  }

  function stdDev(values) {
    if (!values.length) return 0;
    const avg = mean(values);
    const variance = mean(values.map((value) => (value - avg) ** 2));
    return Math.sqrt(variance);
  }

  function buildEmbedding(items) {
    if (!items.length) {
      return Array.from({ length: 26 }, () => 0);
    }

    const areas = items.map((square) => (square.width * square.height) / 10000);
    const centers = items.map((square) => ({
      x: (square.x + square.width / 2) / 100,
      y: (square.y + square.height / 2) / 100
    }));
    const rotations = items.map((square) => (square.rotation * Math.PI) / 180);
    const centerXs = centers.map((center) => center.x);
    const centerYs = centers.map((center) => center.y);

    const grid = Array.from({ length: 16 }, () => 0);
    for (const center of centers) {
      const gx = Math.max(0, Math.min(3, Math.floor(center.x * 4)));
      const gy = Math.max(0, Math.min(3, Math.floor(center.y * 4)));
      grid[gy * 4 + gx] += 1;
    }
    const normalizedGrid = grid.map((value) => value / items.length);

    return [
      Math.min(1, items.length / 24),
      Math.min(1, areas.reduce((sum, area) => sum + area, 0)),
      mean(areas),
      stdDev(areas),
      mean(centerXs),
      mean(centerYs),
      stdDev(centerXs),
      stdDev(centerYs),
      mean(rotations.map((rotation) => Math.sin(rotation))),
      mean(rotations.map((rotation) => Math.cos(rotation))),
      ...normalizedGrid
    ];
  }

  function parseEmbedding(rawEmbedding) {
    if (!Array.isArray(rawEmbedding)) return [];
    return rawEmbedding
      .map((value) => Number(value))
      .filter((value) => Number.isFinite(value));
  }

  function padEmbedding(embedding, dimensions) {
    if (embedding.length >= dimensions) return embedding.slice(0, dimensions);
    return [...embedding, ...Array.from({ length: dimensions - embedding.length }, () => 0)];
  }

  function normalizeCoordinates(pairs, targetRadius) {
    if (pairs.length === 0) return [];
    const xs = pairs.map((pair) => pair[0]);
    const ys = pairs.map((pair) => pair[1]);
    const minX = Math.min(...xs);
    const maxX = Math.max(...xs);
    const minY = Math.min(...ys);
    const maxY = Math.max(...ys);
    const spanX = Math.max(1e-6, maxX - minX);
    const spanY = Math.max(1e-6, maxY - minY);
    const scale = targetRadius * 2;
    return pairs.map((pair) => ([
      ((pair[0] - minX) / spanX - 0.5) * scale,
      ((pair[1] - minY) / spanY - 0.5) * scale
    ]));
  }

  function hashToUnit(value) {
    const text = String(value ?? '');
    let hash = 0;
    for (let i = 0; i < text.length; i += 1) {
      hash = (hash * 31 + text.charCodeAt(i)) % 1000003;
    }
    return (hash % 1000) / 1000;
  }

  function layoutEmbeddedDrawings(items) {
    if (items.length === 0) {
      return { drawings: [] };
    }

    const dimensions = Math.max(...items.map((item) => item.embedding.length), 1);
    const vectors = items.map((item) => padEmbedding(item.embedding, dimensions));
    let projected = vectors.map((vector) => [vector[0] ?? 0, vector[1] ?? 0]);

    if (vectors.length >= 3) {
      try {
        const neighborCount = Math.max(2, Math.min(32, Math.round(Math.sqrt(vectors.length) * 2.2)));
        const umap = new UMAP({
          nComponents: 2,
          nNeighbors: neighborCount,
          minDist: 0.2,
          spread: 1.0
        });
        projected = umap.fit(vectors);
      } catch (error) {
        // Fall back to first dimensions when UMAP fails.
        projected = vectors.map((vector) => [vector[0] ?? 0, vector[1] ?? 0]);
      }
    }

    const targetRadius = Math.max(380, Math.sqrt(items.length) * (PIXEL_TILE_SIZE + PIXEL_TILE_GAP) * 0.9);
    const normalized = normalizeCoordinates(projected, targetRadius);
    const drawings = items.map((item, index) => {
      const [baseX, baseY] = normalized[index] ?? [0, 0];
      const jitterStrength = 7;
      const jitterX = (hashToUnit(`${item.id}-x`) - 0.5) * jitterStrength;
      const jitterY = (hashToUnit(`${item.id}-y`) - 0.5) * jitterStrength;
      return {
        ...item,
        worldX: baseX + jitterX,
        worldY: baseY + jitterY
      };
    });
    return { drawings };
  }

  function normalizeStoredSquares(rawSquares) {
    if (!Array.isArray(rawSquares)) return [];
    return rawSquares
      .map((square, index) => ({
        id: square?.id ?? index,
        x: Number(square?.x) || 0,
        y: Number(square?.y) || 0,
        width: Math.max(0, Number(square?.width) || 0),
        height: Math.max(0, Number(square?.height) || 0),
        rotation: Number(square?.rotation) || 0
      }))
      .filter((square) => square.width > 0 && square.height > 0);
  }

  function centerPixelSpaceCamera() {
    if (!pixelSpaceWrap) return;
    const rect = pixelSpaceWrap.getBoundingClientRect();
    setPixelSpaceCamera({
      x: rect.width / 2,
      y: rect.height / 2,
      scale: 1
    }, { syncTarget: true });
  }

  function openPixelSpace() {
    showPixelSpace = true;
    tick().then(() => {
      centerPixelSpaceCamera();
      subscribeToPixelSpaceRealtime();
      if (pixelSpaceDrawings.length === 0 && !pixelSpaceLoading) {
        loadPixelSpaceDrawings();
      }
    });
  }

  function closePixelSpace() {
    showPixelSpace = false;
    pixelSpacePanState = null;
    stopPixelSpaceInertia();
    unsubscribeFromPixelSpaceRealtime();
    pixelSpaceStatus = '';
  }

  function clampPixelScale(scale) {
    return Math.max(PIXEL_MIN_SCALE, Math.min(PIXEL_MAX_SCALE, scale));
  }

  function setPixelSpaceCamera(nextCamera, options = {}) {
    const safeCamera = {
      x: Number(nextCamera.x) || 0,
      y: Number(nextCamera.y) || 0,
      scale: clampPixelScale(nextCamera.scale ?? pixelSpaceCamera.scale)
    };
    pixelSpaceCamera = safeCamera;
    if (options.syncTarget) {
      pixelSpaceCameraTarget = { ...safeCamera };
    }
  }

  function stopPixelSpaceCameraAnimation() {
    if (!pixelSpaceCameraFrame) return;
    cancelAnimationFrame(pixelSpaceCameraFrame);
    pixelSpaceCameraFrame = null;
  }

  function startPixelSpaceCameraAnimation() {
    if (pixelSpaceCameraFrame) return;
    const animate = () => {
      const dx = pixelSpaceCameraTarget.x - pixelSpaceCamera.x;
      const dy = pixelSpaceCameraTarget.y - pixelSpaceCamera.y;
      const ds = pixelSpaceCameraTarget.scale - pixelSpaceCamera.scale;
      if (Math.abs(dx) < 0.2 && Math.abs(dy) < 0.2 && Math.abs(ds) < 0.0015) {
        setPixelSpaceCamera(pixelSpaceCameraTarget, { syncTarget: true });
        stopPixelSpaceCameraAnimation();
        return;
      }
      setPixelSpaceCamera({
        x: pixelSpaceCamera.x + dx * PIXEL_SPACE_CAMERA_LERP,
        y: pixelSpaceCamera.y + dy * PIXEL_SPACE_CAMERA_LERP,
        scale: pixelSpaceCamera.scale + ds * PIXEL_SPACE_CAMERA_LERP
      });
      pixelSpaceCameraFrame = requestAnimationFrame(animate);
    };
    pixelSpaceCameraFrame = requestAnimationFrame(animate);
  }

  function stopPixelSpaceInertia() {
    if (!pixelSpaceInertiaFrame) return;
    cancelAnimationFrame(pixelSpaceInertiaFrame);
    pixelSpaceInertiaFrame = null;
  }

  function startPixelSpaceInertia(initialVelocityX, initialVelocityY) {
    stopPixelSpaceInertia();
    let velocityX = initialVelocityX;
    let velocityY = initialVelocityY;
    const step = () => {
      velocityX *= PIXEL_SPACE_INERTIA_FRICTION;
      velocityY *= PIXEL_SPACE_INERTIA_FRICTION;
      const speed = Math.hypot(velocityX, velocityY);
      if (speed < PIXEL_SPACE_MIN_INERTIA_SPEED) {
        pixelSpaceInertiaFrame = null;
        return;
      }
      setPixelSpaceCamera({
        x: pixelSpaceCamera.x + velocityX * 16,
        y: pixelSpaceCamera.y + velocityY * 16,
        scale: pixelSpaceCamera.scale
      }, { syncTarget: true });
      pixelSpaceInertiaFrame = requestAnimationFrame(step);
    };
    pixelSpaceInertiaFrame = requestAnimationFrame(step);
  }

  function focusPixelSpaceDrawing(drawingId, preferredScale = null) {
    if (!pixelSpaceWrap) return;
    const drawing = pixelSpaceDrawings.find((item) => item.id === drawingId);
    if (!drawing) return;
    const rect = pixelSpaceWrap.getBoundingClientRect();
    const targetScale = clampPixelScale(preferredScale ?? Math.max(pixelSpaceCamera.scale, 1.1));
    pixelSpaceSelectedDrawingId = drawing.id;
    pixelSpaceCameraTarget = {
      x: rect.width / 2 - drawing.worldX * targetScale,
      y: rect.height / 2 - drawing.worldY * targetScale,
      scale: targetScale
    };
    startPixelSpaceCameraAnimation();
  }

  function fitPixelSpaceToDrawings() {
    if (!pixelSpaceWrap || pixelSpaceDrawings.length === 0) {
      centerPixelSpaceCamera();
      return;
    }
    const rect = pixelSpaceWrap.getBoundingClientRect();
    const xs = pixelSpaceDrawings.map((item) => item.worldX);
    const ys = pixelSpaceDrawings.map((item) => item.worldY);
    const minX = Math.min(...xs) - PIXEL_TILE_SIZE;
    const maxX = Math.max(...xs) + PIXEL_TILE_SIZE;
    const minY = Math.min(...ys) - PIXEL_TILE_SIZE;
    const maxY = Math.max(...ys) + PIXEL_TILE_SIZE;
    const worldWidth = Math.max(1, maxX - minX);
    const worldHeight = Math.max(1, maxY - minY);
    const padding = 72;
    const fitScale = clampPixelScale(Math.min(
      (rect.width - padding) / worldWidth,
      (rect.height - padding) / worldHeight
    ));
    const centerWorldX = (minX + maxX) / 2;
    const centerWorldY = (minY + maxY) / 2;
    pixelSpaceCameraTarget = {
      x: rect.width / 2 - centerWorldX * fitScale,
      y: rect.height / 2 - centerWorldY * fitScale,
      scale: fitScale
    };
    startPixelSpaceCameraAnimation();
  }

  function applyPixelSpaceZoomAtClientPoint(clientX, clientY, zoomFactor) {
    if (!pixelSpaceWrap) return;
    const rect = pixelSpaceWrap.getBoundingClientRect();
    const pointerX = clientX - rect.left;
    const pointerY = clientY - rect.top;
    const targetScale = clampPixelScale(pixelSpaceCamera.scale * zoomFactor);
    if (Math.abs(targetScale - pixelSpaceCamera.scale) < 0.00001) return;
    const worldX = (pointerX - pixelSpaceCamera.x) / pixelSpaceCamera.scale;
    const worldY = (pointerY - pixelSpaceCamera.y) / pixelSpaceCamera.scale;
    setPixelSpaceCamera({
      x: pointerX - worldX * targetScale,
      y: pointerY - worldY * targetScale,
      scale: targetScale
    }, { syncTarget: true });
  }

  function cachePixelSpaceDrawings(drawings) {
    if (typeof window === 'undefined') return;
    try {
      const payload = {
        updatedAt: new Date().toISOString(),
        drawings
      };
      window.localStorage.setItem(PIXEL_SPACE_CACHE_KEY, JSON.stringify(payload));
    } catch (error) {
      // Ignore storage failures and continue with network-backed state.
    }
  }

  function loadPixelSpaceDrawingsFromCache() {
    if (typeof window === 'undefined') return false;
    try {
      const raw = window.localStorage.getItem(PIXEL_SPACE_CACHE_KEY);
      if (!raw) return false;
      const parsed = JSON.parse(raw);
      if (!Array.isArray(parsed?.drawings) || parsed.drawings.length === 0) return false;
      pixelSpaceDrawings = parsed.drawings;
      pixelSpaceLoadedFromCache = true;
      pixelSpaceLastLoadedAt = parsed.updatedAt ?? '';
      return true;
    } catch (error) {
      return false;
    }
  }

  function withDrawingWorldPosition(baseDrawing) {
    if (pixelSpaceDrawings.length === 0) {
      return { ...baseDrawing, worldX: 0, worldY: 0 };
    }
    const neighbor = pixelSpaceDrawings.reduce((best, drawing) => {
      const length = Math.max(baseDrawing.embedding.length, drawing.embedding.length, 1);
      const a = padEmbedding(baseDrawing.embedding, length);
      const b = padEmbedding(drawing.embedding, length);
      let distance = 0;
      for (let i = 0; i < length; i += 1) {
        const diff = a[i] - b[i];
        distance += diff * diff;
      }
      if (!best || distance < best.distance) {
        return { distance, drawing };
      }
      return best;
    }, null)?.drawing;
    if (!neighbor) {
      return { ...baseDrawing, worldX: 0, worldY: 0 };
    }
    const jitterRadius = PIXEL_TILE_SIZE * 0.85;
    const angle = hashToUnit(`${baseDrawing.id}-angle`) * Math.PI * 2;
    return {
      ...baseDrawing,
      worldX: neighbor.worldX + Math.cos(angle) * jitterRadius,
      worldY: neighbor.worldY + Math.sin(angle) * jitterRadius
    };
  }

  function unsubscribeFromPixelSpaceRealtime() {
    if (!pixelSpaceRealtimeChannel || !supabase) return;
    supabase.removeChannel(pixelSpaceRealtimeChannel);
    pixelSpaceRealtimeChannel = null;
  }

  function subscribeToPixelSpaceRealtime() {
    if (!supabase || pixelSpaceRealtimeChannel) return;
    pixelSpaceRealtimeChannel = supabase
      .channel('pixel-space-drawings')
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'drawings'
      }, (payload) => {
        const row = payload.new ?? {};
        const rowId = row.id ?? null;
        if (!rowId) return;
        const idText = String(rowId);
        if (pixelSpaceDrawings.some((drawing) => String(drawing.id) === idText)) return;
        const json = row.drawing_json ?? {};
        const rawSquares = Array.isArray(json?.squares) ? json.squares : (Array.isArray(json) ? json : []);
        const normalizedSquares = normalizeStoredSquares(rawSquares);
        const parsedEmbedding = parseEmbedding(row.embedding);
        const embedding = parsedEmbedding.length > 0 ? parsedEmbedding : buildEmbedding(normalizedSquares);
        const baseDrawing = {
          id: row.id,
          username: (row.username ?? 'anon').slice(0, 40),
          word: row.word ?? '',
          createdAt: row.created_at ?? '',
          worldX: 0,
          worldY: 0,
          squares: normalizedSquares,
          embedding
        };
        const drawingWithWorldPosition = withDrawingWorldPosition(baseDrawing);
        pixelSpaceDrawings = [drawingWithWorldPosition, ...pixelSpaceDrawings];
        cachePixelSpaceDrawings(pixelSpaceDrawings);
      })
      .subscribe();
  }

  async function loadPixelSpaceDrawings() {
    pixelSpaceError = '';
    pixelSpaceStatus = '';
    if (pixelSpaceDrawings.length === 0) {
      const didLoadCache = loadPixelSpaceDrawingsFromCache();
      if (didLoadCache) {
        pixelSpaceStatus = 'Showing cached drawings while syncing...';
      }
    }
    if (!supabase) {
      pixelSpaceError = 'Supabase env vars are missing. Add both URL and anon key.';
      return;
    }
    pixelSpaceLoading = true;
    const { data, error } = await supabase
      .from('drawings')
      .select('id, created_at, username, word, drawing_json, embedding')
      .order('created_at', { ascending: false })
      .limit(PIXEL_SPACE_FETCH_LIMIT);
    pixelSpaceLoading = false;
    if (error) {
      pixelSpaceError = error.message;
      return;
    }
    const entries = (data ?? []).map((row, index) => {
      const json = row.drawing_json ?? {};
      const rawSquares = Array.isArray(json?.squares) ? json.squares : (Array.isArray(json) ? json : []);
      const normalizedSquares = normalizeStoredSquares(rawSquares);
      const parsedEmbedding = parseEmbedding(row.embedding);
      const embedding = parsedEmbedding.length > 0 ? parsedEmbedding : buildEmbedding(normalizedSquares);
      return {
        id: row.id ?? `drawing-${index}`,
        username: (row.username ?? 'anon').slice(0, 40),
        word: row.word ?? '',
        createdAt: row.created_at ?? '',
        worldX: 0,
        worldY: 0,
        squares: normalizedSquares,
        embedding
      };
    });
    const { drawings } = layoutEmbeddedDrawings(entries);
    pixelSpaceDrawings = drawings;
    pixelSpaceLastLoadedAt = new Date().toISOString();
    pixelSpaceLoadedFromCache = false;
    pixelSpaceStatus = '';
    cachePixelSpaceDrawings(drawings);
    if (pixelSpaceSelectedDrawingId && !drawings.some((drawing) => drawing.id === pixelSpaceSelectedDrawingId)) {
      pixelSpaceSelectedDrawingId = null;
    }
  }

  function ensureUsername() {
    const trimmed = username.trim();
    if (trimmed) return trimmed;
    if (typeof window === 'undefined') return '';
    const prompted = window.prompt('Enter a username for submissions (1-40 chars):', '') ?? '';
    const cleaned = prompted.trim().slice(0, 40);
    if (!cleaned) return '';
    username = cleaned;
    window.localStorage.setItem('gestalt-username', cleaned);
    return cleaned;
  }

  async function submitCurrentDrawing() {
    submitStatus = '';
    if (squares.length === 0) {
      submitStatus = 'Draw something first.';
      return;
    }
    if (!supabase) {
      submitStatus = 'Supabase env vars are missing.';
      return;
    }
    const ensuredUsername = ensureUsername();
    if (!ensuredUsername) {
      submitStatus = 'Submission cancelled (missing username).';
      return;
    }
    submitBusy = true;
    const drawingSquares = normalizeStoredSquares(squares);
    const drawingPayload = {
      squares: drawingSquares,
      source: 'gestalt-exercizes',
      mode: gameMode
    };
    const embedding = buildEmbedding(drawingSquares);
    const { error } = await supabase.from('drawings').insert({
      username: ensuredUsername,
      word: currentWord || 'Untitled',
      drawing_json: drawingPayload,
      embedding
    });
    submitBusy = false;
    if (error) {
      submitStatus = `Save failed: ${error.message}`;
      return;
    }
    submitStatus = 'Submitted to Drawing Space.';
    if (showPixelSpace) {
      loadPixelSpaceDrawings();
    }
  }

  function handlePixelSpacePointerDown(e) {
    if (e.button !== 0) return;
    stopPixelSpaceInertia();
    stopPixelSpaceCameraAnimation();
    pixelSpacePanState = {
      pointerId: e.pointerId,
      startClientX: e.clientX,
      startClientY: e.clientY,
      startX: pixelSpaceCamera.x,
      startY: pixelSpaceCamera.y,
      lastClientX: e.clientX,
      lastClientY: e.clientY,
      lastTime: performance.now(),
      velocityX: 0,
      velocityY: 0
    };
    if (typeof e.currentTarget.setPointerCapture === 'function') {
      e.currentTarget.setPointerCapture(e.pointerId);
    }
  }

  function handlePixelSpacePointerMove(e) {
    if (!pixelSpacePanState || e.pointerId !== pixelSpacePanState.pointerId) return;
    const now = performance.now();
    const elapsed = Math.max(1, now - pixelSpacePanState.lastTime);
    setPixelSpaceCamera({
      x: pixelSpacePanState.startX + (e.clientX - pixelSpacePanState.startClientX),
      y: pixelSpacePanState.startY + (e.clientY - pixelSpacePanState.startClientY),
      scale: pixelSpaceCamera.scale
    }, { syncTarget: true });
    pixelSpacePanState = {
      ...pixelSpacePanState,
      velocityX: (e.clientX - pixelSpacePanState.lastClientX) / elapsed,
      velocityY: (e.clientY - pixelSpacePanState.lastClientY) / elapsed,
      lastClientX: e.clientX,
      lastClientY: e.clientY,
      lastTime: now
    };
  }

  function handlePixelSpacePointerUp(e) {
    if (!pixelSpacePanState || e.pointerId !== pixelSpacePanState.pointerId) return;
    const speed = Math.hypot(pixelSpacePanState.velocityX, pixelSpacePanState.velocityY);
    if (speed >= PIXEL_SPACE_MIN_INERTIA_SPEED) {
      startPixelSpaceInertia(pixelSpacePanState.velocityX, pixelSpacePanState.velocityY);
    }
    pixelSpacePanState = null;
  }

  function handlePixelSpaceWheel(e) {
    e.preventDefault();
    const zoomDirection = e.deltaY < 0 ? 1.12 : 0.89;
    applyPixelSpaceZoomAtClientPoint(e.clientX, e.clientY, zoomDirection);
  }

  function handlePixelSpaceTileClick(e, drawing) {
    e.stopPropagation();
    pixelSpaceSelectedDrawingId = drawing.id;
  }

  function focusSelectedPixelDrawing() {
    if (!pixelSpaceSelectedDrawing) return;
    focusPixelSpaceDrawing(pixelSpaceSelectedDrawing.id);
  }

  function getRandomWord() {
    return WORDS[Math.floor(Math.random() * WORDS.length)];
  }

  function start(mode = 'timed') {
    gameMode = mode;
    showPixelSpace = false;
    started = true;
    roundFinished = false;
    roundResults = [];
    squares = [];
    resetHistory();
    creationState = null;
    dragState = null;
    transformState = null;
    clearSelection();
    if (gameMode === 'timed') {
      roundWords = shuffle([...WORDS]).slice(0, WORDS_PER_ROUND);
      wordIndex = 0;
      resetTimer();
    } else {
      roundWords = [getRandomWord()];
      wordIndex = 0;
      if (timerId) clearInterval(timerId);
      timerId = null;
      timeLeft = SECONDS_TOTAL;
    }
    tick().then(() => {
      if (borderSquareRef) {
        gsap.fromTo(
          borderSquareRef,
          { opacity: 1, width: 1, height: 1 },
          {
            opacity: 1,
            width: '100%',
            height: '100%',
            duration: 0.45,
            ease: 'power2.out'
          }
        );
      }
    });
  }

  const BORDER_SHRINK_DURATION = 0.25;
  const BORDER_GROW_DURATION = 0.35;

  function animateBorderGrow() {
    if (!borderSquareRef) return;
    gsap.to(borderSquareRef, {
      width: '100%',
      height: '100%',
      duration: BORDER_GROW_DURATION,
      ease: 'power2.out'
    });
  }

  function resetTimer() {
    if (timerId) clearInterval(timerId);
    timeLeft = SECONDS_TOTAL;
    timerId = setInterval(() => {
      timeLeft -= 1;
      if (timeLeft <= 0) {
        if (timerId) clearInterval(timerId);
        timerId = null;
        nextWord();
        resetTimer();
      }
    }, 1000);
  }

  function addSquareFromDrag(startX, startY, currentX, currentY) {
    const { x, y, width, height, rotation } = getRectangleFromDrag(startX, startY, currentX, currentY);
    if (width <= 0 || height <= 0) return;
    pushHistorySnapshot();
    squares = [...squares, {
      id: nextId++,
      x, y, width, height, rotation
    }];
  }

  function applyNextWord() {
    if (gameMode === 'timed') {
      roundResults = [...roundResults, { word: roundWords[wordIndex], squares: squares.map(s => ({ ...s })) }];
      wordIndex += 1;
      squares = [];
      resetHistory();
      creationState = null;
      dragState = null;
      transformState = null;
      clearSelection();
      if (wordIndex >= roundWords.length) {
        roundFinished = true;
        if (timerId) clearInterval(timerId);
        timerId = null;
      } else if (started) {
        resetTimer();
      }
      return;
    }

    roundWords = [getRandomWord()];
    wordIndex = 0;
    squares = [];
    resetHistory();
    creationState = null;
    dragState = null;
    transformState = null;
    clearSelection();
  }

  function nextWord() {
    if (!borderSquareRef) {
      applyNextWord();
      return;
    }
    gsap.to(borderSquareRef, {
      width: 1,
      height: 1,
      duration: BORDER_SHRINK_DURATION,
      ease: 'power2.in',
      onComplete: () => {
        applyNextWord();
        if (!roundFinished) {
          animateBorderGrow();
        }
      }
    });
  }

  function playAgain() {
    start(gameMode);
  }

  function clientToCanvasPercent(clientX, clientY) {
    if (!canvasWrap) return { x: 0, y: 0 };
    const rect = canvasWrap.getBoundingClientRect();
    return {
      x: ((clientX - rect.left) / rect.width) * 100,
      y: ((clientY - rect.top) / rect.height) * 100
    };
  }

  function beginCreationDrag(e) {
    if (!canvasWrap || e.button !== 0) return;
    clearSelection();
    const { x, y } = clientToCanvasPercent(e.clientX, e.clientY);
    creationState = {
      pointerId: e.pointerId,
      startX: x,
      startY: y,
      currentX: x,
      currentY: y
    };
    if (typeof e.currentTarget.setPointerCapture === 'function') {
      e.currentTarget.setPointerCapture(e.pointerId);
    }
  }

  function handleSquarePointerDown(e, square) {
    e.stopPropagation();
    if (e.button !== 0) return;
    if (e.shiftKey) {
      selectRangeTo(square.id);
      return;
    }
    const activeSquare = squares.find((s) => s.id === square.id);
    if (!activeSquare) return;
    const alreadySelected = selectedSquareIds.includes(activeSquare.id);
    if (!alreadySelected || selectedSquareIds.length <= 1) {
      setSingleSelection(activeSquare.id);
    }
    const { x, y } = clientToCanvasPercent(e.clientX, e.clientY);
    const selectedIds = selectedSquareIds.includes(activeSquare.id) && alreadySelected
      ? [...selectedSquareIds]
      : [activeSquare.id];
    const selectedIdSet = new Set(selectedIds);
    const selectedSquaresAtStart = squares.filter((s) => selectedIdSet.has(s.id));
    const startPositions = {};
    for (const selectedSquare of selectedSquaresAtStart) {
      startPositions[selectedSquare.id] = { x: selectedSquare.x, y: selectedSquare.y };
    }
    const isCopyDrag = e.altKey && selectedIds.length > 0;
    if (!isCopyDrag) {
      const unselectedSquares = squares.filter((s) => !selectedIdSet.has(s.id));
      const reorderedSquares = [...unselectedSquares, ...selectedSquaresAtStart.map((s) => ({ ...s }))];
      squares = reorderedSquares;
    }
    dragState = {
      pointerId: e.pointerId,
      squareId: activeSquare.id,
      selectedIds,
      startPointerX: x,
      startPointerY: y,
      startPositions,
      moved: false,
      hasSnapshot: false,
      copySourceId: isCopyDrag ? activeSquare.id : null,
      copySourceIds: isCopyDrag ? [...selectedIds] : [],
      copyStartX: activeSquare.x,
      copyStartY: activeSquare.y
    };
    if (typeof e.currentTarget.setPointerCapture === 'function') {
      e.currentTarget.setPointerCapture(e.pointerId);
    }
  }

  function handleResizePointerDown(e, handleX, handleY) {
    e.stopPropagation();
    e.preventDefault();
    if (e.button !== 0) return;
    if (selectedSquares.length === 0) return;
    if (selectedSquares.length === 1) {
      const activeSquare = selectedSquares[0];
      setSingleSelection(activeSquare.id);
      transformState = {
        pointerId: e.pointerId,
        squareId: activeSquare.id,
        mode: 'resize',
        hasSnapshot: false,
        startSquare: { ...activeSquare },
        handleX,
        handleY,
        anchorLocalX: (-handleX * activeSquare.width) / 2,
        anchorLocalY: (-handleY * activeSquare.height) / 2
      };
    } else {
      if (!selectionBounds) return;
      const startSquares = {};
      for (const square of selectedSquares) {
        startSquares[square.id] = { ...square };
      }
      transformState = {
        pointerId: e.pointerId,
        squareId: null,
        mode: 'resize-multi',
        hasSnapshot: false,
        selectedIds: selectedSquares.map((square) => square.id),
        startSquares,
        startBounds: { ...selectionBounds },
        handleX,
        handleY
      };
    }
    if (typeof e.currentTarget.setPointerCapture === 'function') {
      e.currentTarget.setPointerCapture(e.pointerId);
    }
  }

  function handleRotatePointerDown(e) {
    e.stopPropagation();
    e.preventDefault();
    if (e.button !== 0) return;
    const { x, y } = clientToCanvasPercent(e.clientX, e.clientY);
    if (selectedSquares.length === 0) return;
    if (selectedSquares.length === 1) {
      const activeSquare = selectedSquares[0];
      setSingleSelection(activeSquare.id);
      transformState = {
        pointerId: e.pointerId,
        squareId: activeSquare.id,
        mode: 'rotate',
        hasSnapshot: false,
        startSquare: { ...activeSquare },
        startRotation: activeSquare.rotation,
        startPointerAngle: pointerAngleFromSquare(activeSquare, x, y)
      };
    } else {
      if (!selectionBounds) return;
      const startSquares = {};
      for (const square of selectedSquares) {
        startSquares[square.id] = { ...square };
      }
      const centerX = selectionBounds.x + selectionBounds.width / 2;
      const centerY = selectionBounds.y + selectionBounds.height / 2;
      transformState = {
        pointerId: e.pointerId,
        squareId: null,
        mode: 'rotate-multi',
        hasSnapshot: false,
        selectedIds: selectedSquares.map((square) => square.id),
        startSquares,
        centerX,
        centerY,
        startPointerAngle: (Math.atan2(y - centerY, x - centerX) * 180) / Math.PI
      };
    }
    if (typeof e.currentTarget.setPointerCapture === 'function') {
      e.currentTarget.setPointerCapture(e.pointerId);
    }
  }

  function handleGlobalPointerMove(e) {
    if (!started || !canvasWrap) return;
    if (showPixelSpace) return;
    altPressed = e.altKey;
    const rect = canvasWrap.getBoundingClientRect();
    mouseCanvasX = ((e.clientX - rect.left) / rect.width) * 100;
    mouseCanvasY = ((e.clientY - rect.top) / rect.height) * 100;
    if (creationState && e.pointerId === creationState.pointerId) {
      creationState = { ...creationState, currentX: mouseCanvasX, currentY: mouseCanvasY };
    }
    if (transformState && e.pointerId === transformState.pointerId) {
      if (transformState.mode === 'resize') {
        const targetSquare = squares.find((s) => s.id === transformState.squareId);
        if (!targetSquare) return;
        const baseSquare = transformState.startSquare;
        const local = worldToSquareLocal(baseSquare, mouseCanvasX, mouseCanvasY);
        const anchorX = transformState.anchorLocalX;
        const anchorY = transformState.anchorLocalY;
        const deltaX = local.x - anchorX;
        const deltaY = local.y - anchorY;
        const side = Math.max(
          MIN_SQUARE_SIZE,
          Math.abs(deltaX),
          Math.abs(deltaY)
        );
        const constrainedCornerX = anchorX + (Math.sign(deltaX) || transformState.handleX) * side;
        const constrainedCornerY = anchorY + (Math.sign(deltaY) || transformState.handleY) * side;
        const newWidth = side;
        const newHeight = side;
        const centerLocalX = (constrainedCornerX + anchorX) / 2;
        const centerLocalY = (constrainedCornerY + anchorY) / 2;
        const centerWorld = squareLocalToWorld(baseSquare, centerLocalX, centerLocalY);
        const newX = centerWorld.x - newWidth / 2;
        const newY = centerWorld.y - newHeight / 2;
        const hasChanged =
          Math.abs(newX - targetSquare.x) > 0.02 ||
          Math.abs(newY - targetSquare.y) > 0.02 ||
          Math.abs(newWidth - targetSquare.width) > 0.02 ||
          Math.abs(newHeight - targetSquare.height) > 0.02;
        if (!hasChanged) return;
        if (!transformState.hasSnapshot) {
          pushHistorySnapshot();
        }
        squares = squares.map((s) => (
          s.id === transformState.squareId
            ? { ...s, x: newX, y: newY, width: newWidth, height: newHeight }
            : s
        ));
        transformState = { ...transformState, hasSnapshot: true };
      } else if (transformState.mode === 'rotate') {
        const targetSquare = squares.find((s) => s.id === transformState.squareId);
        if (!targetSquare) return;
        const startSquare = transformState.startSquare;
        const currentPointerAngle = pointerAngleFromSquare(startSquare, mouseCanvasX, mouseCanvasY);
        const delta = currentPointerAngle - transformState.startPointerAngle;
        let newRotation = transformState.startRotation + delta;
        if (e.shiftKey) {
          newRotation = Math.round(newRotation / 45) * 45;
        }
        newRotation = normalizeRotation(newRotation);
        if (Math.abs(newRotation - targetSquare.rotation) <= 0.05) return;
        if (!transformState.hasSnapshot) {
          pushHistorySnapshot();
        }
        squares = squares.map((s) => (
          s.id === transformState.squareId
            ? { ...s, rotation: newRotation }
            : s
        ));
        transformState = { ...transformState, hasSnapshot: true };
      } else if (transformState.mode === 'resize-multi') {
        const startBounds = transformState.startBounds;
        const anchorX = transformState.handleX === 1
          ? startBounds.x
          : startBounds.x + startBounds.width;
        const anchorY = transformState.handleY === 1
          ? startBounds.y
          : startBounds.y + startBounds.height;
        const deltaX = mouseCanvasX - anchorX;
        const deltaY = mouseCanvasY - anchorY;
        const constrainedCornerX = anchorX + (Math.sign(deltaX) || transformState.handleX) * Math.max(MIN_SQUARE_SIZE, Math.abs(deltaX));
        const constrainedCornerY = anchorY + (Math.sign(deltaY) || transformState.handleY) * Math.max(MIN_SQUARE_SIZE, Math.abs(deltaY));
        const startCornerX = transformState.handleX === 1
          ? startBounds.x + startBounds.width
          : startBounds.x;
        const startCornerY = transformState.handleY === 1
          ? startBounds.y + startBounds.height
          : startBounds.y;
        const baseVectorX = startCornerX - anchorX;
        const baseVectorY = startCornerY - anchorY;
        const baseLengthSquared = baseVectorX * baseVectorX + baseVectorY * baseVectorY;
        if (baseLengthSquared <= 0.0001) return;
        const pointerVectorX = constrainedCornerX - anchorX;
        const pointerVectorY = constrainedCornerY - anchorY;
        let uniformScale = (
          pointerVectorX * baseVectorX +
          pointerVectorY * baseVectorY
        ) / baseLengthSquared;
        const minScale = Math.max(
          startBounds.width > 0 ? MIN_SQUARE_SIZE / startBounds.width : 0,
          startBounds.height > 0 ? MIN_SQUARE_SIZE / startBounds.height : 0
        );
        uniformScale = Math.max(minScale, uniformScale);
        if (Math.abs(uniformScale - 1) <= 0.002) return;
        if (!transformState.hasSnapshot) {
          pushHistorySnapshot();
        }
        const selectedIdSet = new Set(transformState.selectedIds);
        squares = squares.map((square) => {
          if (!selectedIdSet.has(square.id)) return square;
          const startSquare = transformState.startSquares[square.id];
          if (!startSquare) return square;
          const startLeft = startSquare.x;
          const startRight = startSquare.x + startSquare.width;
          const startTop = startSquare.y;
          const startBottom = startSquare.y + startSquare.height;
          const transformedLeft = anchorX + (startLeft - anchorX) * uniformScale;
          const transformedRight = anchorX + (startRight - anchorX) * uniformScale;
          const transformedTop = anchorY + (startTop - anchorY) * uniformScale;
          const transformedBottom = anchorY + (startBottom - anchorY) * uniformScale;
          return {
            ...square,
            x: Math.min(transformedLeft, transformedRight),
            y: Math.min(transformedTop, transformedBottom),
            width: Math.max(MIN_SQUARE_SIZE, Math.abs(transformedRight - transformedLeft)),
            height: Math.max(MIN_SQUARE_SIZE, Math.abs(transformedBottom - transformedTop))
          };
        });
        transformState = { ...transformState, hasSnapshot: true };
      } else if (transformState.mode === 'rotate-multi') {
        const centerX = transformState.centerX;
        const centerY = transformState.centerY;
        const currentPointerAngle = (Math.atan2(mouseCanvasY - centerY, mouseCanvasX - centerX) * 180) / Math.PI;
        let delta = currentPointerAngle - transformState.startPointerAngle;
        if (e.shiftKey) {
          delta = Math.round(delta / 45) * 45;
        }
        if (Math.abs(delta) <= 0.05) return;
        if (!transformState.hasSnapshot) {
          pushHistorySnapshot();
        }
        const selectedIdSet = new Set(transformState.selectedIds);
        squares = squares.map((square) => {
          if (!selectedIdSet.has(square.id)) return square;
          const startSquare = transformState.startSquares[square.id];
          if (!startSquare) return square;
          const startCenter = getSquareCenter(startSquare);
          const offset = rotatePoint(
            startCenter.x - centerX,
            startCenter.y - centerY,
            delta
          );
          const nextCenterX = centerX + offset.x;
          const nextCenterY = centerY + offset.y;
          return {
            ...square,
            x: nextCenterX - startSquare.width / 2,
            y: nextCenterY - startSquare.height / 2,
            rotation: normalizeRotation(startSquare.rotation + delta)
          };
        });
        transformState = { ...transformState, hasSnapshot: true };
      }
      return;
    }
    if (!dragState || e.pointerId !== dragState.pointerId) return;
    let draggedSquare = squares.find((s) => s.id === dragState.squareId);
    if (!draggedSquare) return;
    const currentStart = dragState.startPositions[dragState.squareId];
    if (!currentStart) return;
    let newX = currentStart.x + (mouseCanvasX - dragState.startPointerX);
    let newY = currentStart.y + (mouseCanvasY - dragState.startPointerY);

    if (e.altKey && dragState.copySourceId === null && dragState.squareId === draggedSquare.id) {
      dragState = {
        ...dragState,
        copySourceId: draggedSquare.id,
        copySourceIds: [...dragState.selectedIds],
        copyStartX: draggedSquare.x,
        copyStartY: draggedSquare.y
      };
    }

    // Alt-drag duplicates the source selection on first movement.
    const isCopyDrag = dragState.copySourceId !== null;
    if (!isCopyDrag && !dragState.moved) {
      const hasMovedEnoughToDrag =
        Math.abs(newX - dragState.copyStartX) > DRAG_ACTIVATION_THRESHOLD ||
        Math.abs(newY - dragState.copyStartY) > DRAG_ACTIVATION_THRESHOLD;
      if (!hasMovedEnoughToDrag) return;
    }
    if (isCopyDrag && dragState.squareId === dragState.copySourceId) {
      const hasMovedEnough =
        Math.abs(newX - dragState.copyStartX) > DRAG_ACTIVATION_THRESHOLD ||
        Math.abs(newY - dragState.copyStartY) > DRAG_ACTIVATION_THRESHOLD;
      if (!hasMovedEnough) return;
      if (!dragState.hasSnapshot) {
        pushHistorySnapshot();
      }
      const sourceIds = dragState.copySourceIds.length > 0
        ? dragState.copySourceIds
        : [dragState.copySourceId];
      const sourceIdSet = new Set(sourceIds);
      const duplicatedIdMap = {};
      const duplicatedSquares = [];
      for (const square of squares) {
        if (!sourceIdSet.has(square.id)) continue;
        const duplicatedSquare = { ...square, id: nextId++ };
        duplicatedIdMap[square.id] = duplicatedSquare.id;
        duplicatedSquares.push(duplicatedSquare);
      }
      if (duplicatedSquares.length === 0) return;
      squares = [...squares, ...duplicatedSquares];
      const duplicatedSelectionIds = sourceIds
        .map((sourceId) => duplicatedIdMap[sourceId])
        .filter((id) => id !== undefined);
      selectedSquareIds = duplicatedSelectionIds;
      selectionAnchorId = duplicatedIdMap[selectionAnchorId] ?? duplicatedIdMap[dragState.squareId] ?? null;
      const duplicatedSquareId = duplicatedIdMap[dragState.squareId] ?? duplicatedSelectionIds[0];
      draggedSquare = duplicatedSquares.find((square) => square.id === duplicatedSquareId) ?? duplicatedSquares[0];
      const nextStartPositions = { ...dragState.startPositions };
      for (const square of duplicatedSquares) {
        nextStartPositions[square.id] = { x: square.x, y: square.y };
      }
      dragState = {
        ...dragState,
        squareId: draggedSquare.id,
        selectedIds: duplicatedSelectionIds,
        startPositions: nextStartPositions,
        moved: true,
        hasSnapshot: true
      };
      newX = nextStartPositions[draggedSquare.id].x + (mouseCanvasX - dragState.startPointerX);
      newY = nextStartPositions[draggedSquare.id].y + (mouseCanvasY - dragState.startPointerY);
    }

    // Hold Shift while dragging to constrain movement to one axis.
    if (e.shiftKey) {
      const axisAnchorX = isCopyDrag ? dragState.copyStartX : currentStart.x;
      const axisAnchorY = isCopyDrag ? dragState.copyStartY : currentStart.y;
      const axisDeltaX = newX - axisAnchorX;
      const axisDeltaY = newY - axisAnchorY;
      if (Math.abs(axisDeltaX) >= Math.abs(axisDeltaY)) {
        newY = axisAnchorY;
      } else {
        newX = axisAnchorX;
      }
    }

    const deltaX = newX - dragState.startPositions[dragState.squareId].x;
    const deltaY = newY - dragState.startPositions[dragState.squareId].y;
    const hasMoved = Math.abs(deltaX) > 0.05 || Math.abs(deltaY) > 0.05;
    if (!hasMoved) return;
    if (!dragState.hasSnapshot) {
      pushHistorySnapshot();
    }
    const dragSelectedIdSet = new Set(dragState.selectedIds);
    squares = squares.map((s) => {
      if (!dragSelectedIdSet.has(s.id)) return s;
      const start = dragState.startPositions[s.id];
      if (!start) return s;
      return { ...s, x: start.x + deltaX, y: start.y + deltaY };
    });
    dragState = { ...dragState, moved: true, hasSnapshot: true };
  }

  function handleGlobalPointerUp(e) {
    altPressed = e.altKey;
    if (creationState && e.pointerId === creationState.pointerId) {
      addSquareFromDrag(
        creationState.startX,
        creationState.startY,
        creationState.currentX,
        creationState.currentY
      );
      creationState = null;
    }
    if (transformState && e.pointerId === transformState.pointerId) {
      transformState = null;
    }
    if (dragState && e.pointerId === dragState.pointerId) {
      dragState = null;
    }
  }

  function handleKeydown(e) {
    if (showPixelSpace) {
      if (e.code === 'Escape') {
        e.preventDefault();
        closePixelSpace();
      } else if (e.code === 'Equal' || e.code === 'NumpadAdd') {
        e.preventDefault();
        if (!pixelSpaceWrap) return;
        const rect = pixelSpaceWrap.getBoundingClientRect();
        applyPixelSpaceZoomAtClientPoint(
          rect.left + rect.width / 2,
          rect.top + rect.height / 2,
          1.12
        );
      } else if (e.code === 'Minus' || e.code === 'NumpadSubtract') {
        e.preventDefault();
        if (!pixelSpaceWrap) return;
        const rect = pixelSpaceWrap.getBoundingClientRect();
        applyPixelSpaceZoomAtClientPoint(
          rect.left + rect.width / 2,
          rect.top + rect.height / 2,
          0.89
        );
      } else if (e.code === 'Digit0') {
        e.preventDefault();
        centerPixelSpaceCamera();
      } else if (e.code === 'KeyF') {
        e.preventDefault();
        focusSelectedPixelDrawing();
      } else if (e.code === 'KeyL') {
        e.preventDefault();
        fitPixelSpaceToDrawings();
      } else if (e.code === 'KeyR') {
        e.preventDefault();
        loadPixelSpaceDrawings();
      } else if (e.code === 'ArrowLeft' || e.code === 'ArrowRight' || e.code === 'ArrowUp' || e.code === 'ArrowDown') {
        e.preventDefault();
        const step = e.shiftKey ? PIXEL_SPACE_KEYBOARD_PAN_STEP * 2 : PIXEL_SPACE_KEYBOARD_PAN_STEP;
        const panX = e.code === 'ArrowLeft' ? step : (e.code === 'ArrowRight' ? -step : 0);
        const panY = e.code === 'ArrowUp' ? step : (e.code === 'ArrowDown' ? -step : 0);
        setPixelSpaceCamera({
          x: pixelSpaceCamera.x + panX,
          y: pixelSpaceCamera.y + panY,
          scale: pixelSpaceCamera.scale
        }, { syncTarget: true });
      }
      return;
    }
    if (e.key === 'Alt') {
      altPressed = true;
    }
    const isPrimaryModifierPressed = e.metaKey || e.ctrlKey;
    const normalizedKey = e.key.toLowerCase();
    if (isPrimaryModifierPressed && normalizedKey === 'c') {
      e.preventDefault();
      copySelectedSquares();
      return;
    }
    if (isPrimaryModifierPressed && normalizedKey === 'v') {
      e.preventDefault();
      pasteCopiedSquares();
      return;
    }
    const isUndoRedoShortcut = (e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'z';
    if (isUndoRedoShortcut) {
      e.preventDefault();
      if (e.shiftKey) {
        redoDrawing();
      } else {
        undoDrawing();
      }
      return;
    }
    if (e.code === 'Space') {
      e.preventDefault();
      if (started) nextWord();
    }
    if ((e.key === 'Delete' || e.key === 'Backspace') && selectedSquareIds.length > 0) {
      e.preventDefault();
      deleteSelectedSquares();
    }
    if (e.code === 'Escape' && creationState) {
      creationState = null;
      transformState = null;
    }
  }

  function handleKeyup(e) {
    if (e.key === 'Alt') {
      altPressed = false;
    }
  }

  function handleWindowBlur() {
    altPressed = false;
  }

  $: draggingSquareId = dragState ? dragState.squareId : null;
  $: selectedSquares = squares.filter((s) => selectedSquareIds.includes(s.id));
  $: selectedSquare = selectedSquares.length === 1 ? selectedSquares[0] : null;
  $: selectionBounds = selectedSquares.length ? getSelectionBounds(selectedSquares) : null;
  $: selectionOverlay = (() => {
    if (!selectionBounds) return null;
    if (selectedSquare) {
      return {
        x: selectedSquare.x,
        y: selectedSquare.y,
        width: selectedSquare.width,
        height: selectedSquare.height,
        rotation: selectedSquare.rotation
      };
    }
    return {
      ...selectionBounds,
      rotation: 0
    };
  })();
  $: timerDisplay = `${Math.floor(timeLeft / 60)}:${String(timeLeft % 60).padStart(2, '0')}`;
  $: cursorToolIcon = (() => {
    if (transformState?.mode === 'resize') return 'resize';
    if (transformState?.mode === 'rotate') return 'rotate';
    if (hoverTransformMode) return hoverTransformMode;
    return null;
  })();

  function handleTransformHoverStart(mode) {
    hoverTransformMode = mode;
  }

  function handleTransformHoverEnd(mode) {
    if (hoverTransformMode === mode) hoverTransformMode = null;
  }

</script>

<svelte:window
  on:pointermove={handleGlobalPointerMove}
  on:pointerup={handleGlobalPointerUp}
  on:pointercancel={handleGlobalPointerUp}
  on:keydown={handleKeydown}
  on:keyup={handleKeyup}
  on:blur={handleWindowBlur}
/>

{#if showPixelSpace}
  <div class="pixel-space-screen">
    <div class="pixel-space-hud">
      <div class="pixel-space-hud-row">
        <button class="pixel-space-btn" on:click={closePixelSpace}>Close</button>
        <button class="pixel-space-btn" on:click={centerPixelSpaceCamera}>Recenter</button>
        <button class="pixel-space-btn" on:click={fitPixelSpaceToDrawings}>Fit</button>
        <button class="pixel-space-btn" on:click={loadPixelSpaceDrawings} disabled={pixelSpaceLoading}>Reload</button>
      </div>
      <div class="pixel-space-meta">
        <span>{pixelSpaceDrawings.length} drawings</span>
        <span>Zoom {(pixelSpaceCamera.scale * 100).toFixed(0)}%</span>
        {#if pixelSpaceLastLoadedAt}
          <span>Synced {new Date(pixelSpaceLastLoadedAt).toLocaleTimeString()}</span>
        {/if}
        {#if pixelSpaceLoadedFromCache}
          <span>Cache preview</span>
        {/if}
      </div>
      {#if pixelSpaceSelectedDrawing}
        <div class="pixel-space-selected">
          <strong>{pixelSpaceSelectedDrawing.word || 'Untitled'}</strong>
          <span>by {pixelSpaceSelectedDrawing.username}</span>
          <button class="pixel-space-btn pixel-space-btn-compact" on:click={focusSelectedPixelDrawing}>Focus</button>
        </div>
      {/if}
      {#if pixelSpaceStatus}
        <p class="pixel-space-status">{pixelSpaceStatus}</p>
      {/if}
      {#if pixelSpaceError}
        <p class="pixel-space-error">{pixelSpaceError}</p>
      {/if}
    </div>
    <div
      class="pixel-space-wrap"
      bind:this={pixelSpaceWrap}
      on:pointerdown={handlePixelSpacePointerDown}
      on:pointermove={handlePixelSpacePointerMove}
      on:pointerup={handlePixelSpacePointerUp}
      on:pointercancel={handlePixelSpacePointerUp}
      on:wheel={handlePixelSpaceWheel}
    >
      {#if pixelSpaceLoading && pixelSpaceDrawings.length === 0}
        <div class="pixel-space-loading">Loading drawings...</div>
      {/if}
      <div class="pixel-space-world" style="transform: {pixelSpaceTransform};">
        {#each pixelSpaceDrawings as drawing (drawing.id)}
          <button
            type="button"
            class="pixel-space-tile"
            class:pixel-space-tile-selected={pixelSpaceSelectedDrawingId === drawing.id}
            style="transform: translate({drawing.worldX - PIXEL_TILE_SIZE / 2}px, {drawing.worldY - PIXEL_TILE_SIZE / 2}px);"
            on:pointerdown|stopPropagation
            on:click={(e) => handlePixelSpaceTileClick(e, drawing)}
            aria-label={`View drawing ${drawing.word || 'Untitled'} by ${drawing.username}`}
          >
            <div class="pixel-space-canvas">
              {#each drawing.squares as square (`${drawing.id}-${square.id}`)}
                <div
                  class="square square-result pixel-space-square"
                  style="
                    left: {square.x}%;
                    top: {square.y}%;
                    width: {square.width}%;
                    height: {square.height}%;
                    transform: rotate({square.rotation}deg);
                  "
                />
              {/each}
            </div>
            <footer class="pixel-space-tile-meta">
              <span class="pixel-space-tile-word">{drawing.word || 'Untitled'}</span>
              <span class="pixel-space-tile-user">{drawing.username}</span>
            </footer>
          </button>
        {/each}
      </div>
    </div>
  </div>
{:else if !started}
  <div class="start-screen-wrap">
    <h1 class="start-title">Gestalt Exercises</h1>
    <section class="start-section">
      <h2 class="start-heading">Controls</h2>
      <dl class="start-controls">
        <dt>Create a square</dt>
        <dd>Drag on empty canvas to draw a new black square.</dd>

        <dt>Drag / duplicate</dt>
        <dd>Drag a square (or selection) to move it. Hold <kbd>Shift</kbd> while dragging to lock movement to horizontal or vertical. Hold <kbd>Alt</kbd> while dragging to copy.</dd>

        <dt>Undo / Redo</dt>
        <dd><kbd>Ctrl</kbd>/<kbd>Cmd</kbd>+<kbd>Z</kbd> to undo, <kbd>Ctrl</kbd>/<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>Z</kbd> to redo.</dd>

        <dt>Next word</dt>
        <dd><kbd>Space</kbd> to skip to the next word and save the current composition.</dd>

      </dl>
    </section>
    <div class="start-btn-row">
      <button class="start-btn" on:click={() => start('timed')}>
        Game Mode
      </button>
      <button class="start-btn start-btn-secondary" on:click={() => start('free')}>
        Free Play
      </button>
      <button class="start-btn start-btn-secondary" on:click={openPixelSpace}>
        Drawing Space
      </button>
    </div>
  </div>
{:else if roundFinished}
  <div class="end-screen">
    <h2 class="end-title">Your words & compositions</h2>
    <div class="result-list">
      {#each roundResults as result}
        <div class="result-item">
          <div class="result-word">{result.word}</div>
          <div class="canvas-wrap result-canvas">
            <div class="corner corner-tl" aria-hidden="true" />
            <div class="corner corner-tr" aria-hidden="true" />
            <div class="corner corner-bl" aria-hidden="true" />
            <div class="corner corner-br" aria-hidden="true" />
            <div class="canvas">
              {#each result.squares as square (square.id)}
                <div
                  class="square square-result"
                  style="
                    left: {square.x}%;
                    top: {square.y}%;
                    width: {square.width}%;
                    height: {square.height}%;
                    transform: rotate({square.rotation}deg);
                  "
                />
              {/each}
            </div>
          </div>
        </div>
      {/each}
    </div>
    <button class="play-again-btn" on:click={playAgain}>
      <RotateCcw size={16} aria-hidden="true" />
      <span>Play again</span>
    </button>
    <button class="play-again-btn" on:click={openPixelSpace}>
      <span>View Drawing Space</span>
    </button>
  </div>
{:else}
  <div class="play-screen">
    <div class="word-row">
      <div class="word-display">{currentWord}</div>
      {#if gameMode === 'timed'}
        <div class="timer" aria-live="polite">
          <Timer size={14} aria-hidden="true" />
          <span>{timerDisplay}</span>
        </div>
      {/if}
    </div>
    <div class="play-actions">
      <button class="play-again-btn" on:click={submitCurrentDrawing} disabled={submitBusy}>
        {submitBusy ? 'Submitting...' : 'Submit drawing'}
      </button>
      <button class="play-again-btn" on:click={openPixelSpace}>
        Drawing Space
      </button>
    </div>
    {#if submitStatus}
      <p class="submit-status">{submitStatus}</p>
    {/if}
    <div
      class="canvas-wrap canvas-wrap--with-corners"
      bind:this={canvasWrap}
      role="presentation"
    >
      <div
        class="canvas-border-square"
        bind:this={borderSquareRef}
        aria-hidden="true"
      />
      <div
        class="canvas"
        class:tool-cursor-active={Boolean(cursorToolIcon)}
        role="application"
        aria-label="Composition canvas; drag to create squares"
        on:pointerdown={beginCreationDrag}
      >
        {#if previewSquare}
          <div
            class="square square-preview"
            style="
              left: {previewSquare.x}%;
              top: {previewSquare.y}%;
              width: {previewSquare.width}%;
              height: {previewSquare.height}%;
              transform: rotate({previewSquare.rotation}deg);
            "
          />
        {/if}
        {#each squares as square (square.id)}
          <div
            class="square"
            class:square-dragging={draggingSquareId === square.id}
            class:square-copy-ready={altPressed}
            class:square-selected={selectedSquareIds.includes(square.id)}
            style="
              left: {square.x}%;
              top: {square.y}%;
              width: {square.width}%;
              height: {square.height}%;
              transform: rotate({square.rotation}deg);
            "
            on:pointerdown={(e) => handleSquarePointerDown(e, square)}
          />
        {/each}
        {#if selectionOverlay}
          <div
            class="selection-overlay"
            style="
              left: {selectionOverlay.x}%;
              top: {selectionOverlay.y}%;
              width: {selectionOverlay.width}%;
              height: {selectionOverlay.height}%;
              transform: rotate({selectionOverlay.rotation}deg);
            "
          >
            {#each CORNER_HANDLES as handle (handle.key)}
              <div
                class="transform-handle resize-handle"
                class:resize-alt={(handle.x === 1 && handle.y === -1) || (handle.x === -1 && handle.y === 1)}
                style="--hx: {handle.x}; --hy: {handle.y};"
                on:pointerenter={() => handleTransformHoverStart('resize')}
                on:pointerleave={() => handleTransformHoverEnd('resize')}
                on:pointerdown={(e) => handleResizePointerDown(e, handle.x, handle.y)}
              />
            {/each}
            {#each CORNER_HANDLES as handle (handle.key)}
              <div
                class="transform-handle rotate-handle"
                style="--hx: {handle.x}; --hy: {handle.y};"
                on:pointerenter={() => handleTransformHoverStart('rotate')}
                on:pointerleave={() => handleTransformHoverEnd('rotate')}
                on:pointerdown={handleRotatePointerDown}
              />
            {/each}
          </div>
        {/if}
        {#if cursorToolIcon}
          <div
            class="cursor-tool-icon"
            style="left: {mouseCanvasX}%; top: {mouseCanvasY}%;"
            aria-hidden="true"
          >
            {#if cursorToolIcon === 'resize'}
              <MoveDiagonal2 size={13} />
            {:else}
              <RotateCw size={13} />
            {/if}
          </div>
        {/if}
      </div>
    </div>
    <p class="skip-hint">
      <SkipForward size={12} aria-hidden="true" />
      <span>{gameMode === 'timed' ? 'Space to skip' : 'Space for new word'}</span>
    </p>
  </div>
{/if}
