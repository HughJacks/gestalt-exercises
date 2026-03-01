<script>
  import { onMount } from 'svelte';
  import { MoveDiagonal2, RotateCcw, RotateCw, SkipForward, Timer } from 'lucide-svelte';
  import './app.css';

  const WORDS = [
    'Balance', 'Rhythm', 'Emphasis', 'Unity', 'Contrast',
    'Movement', 'Pattern', 'Proximity', 'Alignment', 'Space',
    'Hierarchy', 'Scale', 'Repetition', 'Harmony', 'Tension'
  ];

  const WORDS_PER_ROUND = 5;
  const MINUTES_PER_WORD = 1;
  const SECONDS_TOTAL = MINUTES_PER_WORD * 60;
  const MIN_SQUARE_SIZE = 2;
  const DRAG_ACTIVATION_THRESHOLD = 0.2;
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

  let canvasWrap = null;
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

  $: currentWord = roundWords[wordIndex] ?? '';

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

  function start() {
    started = true;
    roundFinished = false;
    roundWords = shuffle([...WORDS]).slice(0, WORDS_PER_ROUND);
    roundResults = [];
    wordIndex = 0;
    squares = [];
    resetHistory();
    creationState = null;
    dragState = null;
    transformState = null;
    clearSelection();
    resetTimer();
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

  function nextWord() {
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
  }

  function playAgain() {
    start();
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

    // During copy drag, hold Shift to keep movement in line from the source.
    if (isCopyDrag && e.shiftKey) {
      const deltaX = newX - dragState.copyStartX;
      const deltaY = newY - dragState.copyStartY;
      if (Math.abs(deltaX) >= Math.abs(deltaY)) {
        newY = dragState.copyStartY;
      } else {
        newX = dragState.copyStartX;
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

  onMount(() => {
    start();
  });
</script>

<svelte:window
  on:pointermove={handleGlobalPointerMove}
  on:pointerup={handleGlobalPointerUp}
  on:pointercancel={handleGlobalPointerUp}
  on:keydown={handleKeydown}
  on:keyup={handleKeyup}
  on:blur={handleWindowBlur}
/>

{#if roundFinished}
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
  </div>
{:else}
  <div class="play-screen">
    <div class="word-row">
      <div class="word-display">{currentWord}</div>
      <div class="timer" aria-live="polite">
        <Timer size={14} aria-hidden="true" />
        <span>{timerDisplay}</span>
      </div>
    </div>
    <div
      class="canvas-wrap"
      bind:this={canvasWrap}
      role="presentation"
    >
      <div class="corner corner-tl" aria-hidden="true" />
      <div class="corner corner-tr" aria-hidden="true" />
      <div class="corner corner-bl" aria-hidden="true" />
      <div class="corner corner-br" aria-hidden="true" />
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
      <span>Space to skip</span>
    </p>
  </div>
{/if}
