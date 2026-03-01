<script>
  import './app.css';

  const WORDS = [
    'Balance', 'Rhythm', 'Emphasis', 'Unity', 'Contrast',
    'Movement', 'Pattern', 'Proximity', 'Alignment', 'Space',
    'Hierarchy', 'Scale', 'Repetition', 'Harmony', 'Tension'
  ];

  const WORDS_PER_ROUND = 5;
  const MINUTES_PER_WORD = 1;
  const SECONDS_TOTAL = MINUTES_PER_WORD * 60;

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
  let timeLeft = SECONDS_TOTAL;
  let timerId = null;

  // Two-step placement: click center, then click corner
  const PREVIEW_SIZE_MIN = 5;
  const PREVIEW_SIZE_MAX = 50;
  let canvasWrap = null;
  let mouseCanvasX = 50;
  let mouseCanvasY = 50;
  let placingPhase = 'center'; // 'center' | 'corner'
  let pendingCenter = null; // { x, y } when phase is 'corner'
  let shiftKeyDown = false;

  $: currentWord = roundWords[wordIndex] ?? '';

  function snapTo45(rotation) {
    return Math.round(rotation / 45) * 45;
  }

  // Preview in corner phase: square from pendingCenter to mouse (Shift = snap to 45°)
  $: previewInCornerPhase = placingPhase === 'corner' && pendingCenter;
  $: previewSquare = (() => {
    if (!previewInCornerPhase) return null;
    const cx = pendingCenter.x;
    const cy = pendingCenter.y;
    const dx = mouseCanvasX - cx;
    const dy = mouseCanvasY - cy;
    const dist = Math.sqrt(dx * dx + dy * dy) || 0.1;
    const side = Math.min(PREVIEW_SIZE_MAX, Math.max(PREVIEW_SIZE_MIN, dist * Math.SQRT2));
    let rotation = (Math.atan2(dy, dx) * 180 / Math.PI) - 45;
    if (shiftKeyDown) rotation = snapTo45(rotation);
    return { x: cx - side / 2, y: cy - side / 2, size: side, rotation };
  })();

  function start() {
    started = true;
    roundFinished = false;
    roundWords = shuffle([...WORDS]).slice(0, WORDS_PER_ROUND);
    roundResults = [];
    wordIndex = 0;
    squares = [];
    placingPhase = 'center';
    pendingCenter = null;
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

  function addSquareFromCenterAndCorner(cx, cy, px, py, snap45 = false) {
    const dx = px - cx;
    const dy = py - cy;
    const dist = Math.sqrt(dx * dx + dy * dy) || 0.1;
    const side = Math.min(PREVIEW_SIZE_MAX, Math.max(PREVIEW_SIZE_MIN, dist * Math.SQRT2));
    let rotation = (Math.atan2(dy, dx) * 180 / Math.PI) - 45;
    if (snap45) rotation = snapTo45(rotation);
    const x = cx - side / 2;
    const y = cy - side / 2;
    squares = [...squares, {
      id: nextId++,
      x, y, size: side, rotation
    }];
  }

  function nextWord() {
    roundResults = [...roundResults, { word: roundWords[wordIndex], squares: squares.map(s => ({ ...s })) }];
    wordIndex += 1;
    squares = [];
    if (wordIndex >= roundWords.length) {
      roundFinished = true;
      if (timerId) clearInterval(timerId);
      timerId = null;
    } else if (started) {
      resetTimer();
    }
  }

  function playAgain() {
    roundFinished = false;
    started = false;
  }

  function isInCenter(square, mx, my) {
    const left = square.x + square.size * 0.25;
    const right = square.x + square.size * 0.75;
    const top = square.y + square.size * 0.25;
    const bottom = square.y + square.size * 0.75;
    return mx >= left && mx <= right && my >= top && my <= bottom;
  }

  function getSquareAtCenter(mx, my) {
    for (let i = squares.length - 1; i >= 0; i--) {
      if (isInCenter(squares[i], mx, my)) return squares[i];
    }
    return null;
  }

  function deleteSquare(id) {
    squares = squares.filter(s => s.id !== id);
  }

  function clientToCanvasPercent(clientX, clientY) {
    if (!canvasWrap) return { x: 0, y: 0 };
    const rect = canvasWrap.getBoundingClientRect();
    return {
      x: ((clientX - rect.left) / rect.width) * 100,
      y: ((clientY - rect.top) / rect.height) * 100
    };
  }

  function placeSquareFromCornerClick(clientX, clientY, shiftKey = false) {
    const { x, y } = clientToCanvasPercent(clientX, clientY);
    addSquareFromCenterAndCorner(pendingCenter.x, pendingCenter.y, x, y, shiftKey);
    pendingCenter = null;
    placingPhase = 'center';
  }

  function handleCanvasPointerDown(e) {
    if (e.target.classList.contains('canvas')) {
      const { x, y } = clientToCanvasPercent(e.clientX, e.clientY);
      if (placingPhase === 'center') {
        pendingCenter = { x, y };
        placingPhase = 'corner';
      } else {
        placeSquareFromCornerClick(e.clientX, e.clientY, e.shiftKey);
      }
    }
  }

  function handleWindowClick(e) {
    if (placingPhase !== 'corner' || !pendingCenter || !canvasWrap) return;
    if (canvasWrap.contains(e.target)) return;
    placeSquareFromCornerClick(e.clientX, e.clientY, e.shiftKey);
  }

  function handleSquareClick(e, square) {
    e.stopPropagation();
    if (e.button !== 0) return;
    if (placingPhase === 'corner') {
      placingPhase = 'center';
      pendingCenter = null;
      return;
    }
    const rect = e.currentTarget.closest('.canvas-wrap').getBoundingClientRect();
    const x = ((e.clientX - rect.left) / rect.width) * 100;
    const y = ((e.clientY - rect.top) / rect.height) * 100;
    if (isInCenter(square, x, y)) {
      deleteSquare(square.id);
    }
  }

  function handleGlobalPointerMove(e) {
    if (!started || !canvasWrap) return;
    const rect = canvasWrap.getBoundingClientRect();
    mouseCanvasX = ((e.clientX - rect.left) / rect.width) * 100;
    mouseCanvasY = ((e.clientY - rect.top) / rect.height) * 100;
  }

  function handleKeydown(e) {
    shiftKeyDown = e.shiftKey;
    if (e.code === 'Space') {
      e.preventDefault();
      if (started) nextWord();
    }
    if (e.code === 'Escape' && placingPhase === 'corner') {
      placingPhase = 'center';
      pendingCenter = null;
    }
  }

  function handleKeyup(e) {
    shiftKeyDown = e.shiftKey;
  }

  $: hoveredForDeleteId = (() => {
    const sq = getSquareAtCenter(mouseCanvasX, mouseCanvasY);
    return sq ? sq.id : null;
  })();
  $: timerDisplay = `${Math.floor(timeLeft / 60)}:${String(timeLeft % 60).padStart(2, '0')}`;
</script>

<svelte:window
  on:pointermove={handleGlobalPointerMove}
  on:keydown={handleKeydown}
  on:keyup={handleKeyup}
  on:click={handleWindowClick}
/>

{#if !started}
  <div class="start-screen" on:click={start} on:keydown={(e) => e.code === 'Enter' && start()} role="button" tabindex="0">
    Start
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
                    width: {square.size}%;
                    height: {square.size}%;
                    transform: rotate({square.rotation}deg);
                  "
                />
              {/each}
            </div>
          </div>
        </div>
      {/each}
    </div>
    <button class="play-again-btn" on:click={playAgain}>Play again</button>
  </div>
{:else}
  <div class="play-screen">
    <div class="timer" aria-live="polite">{timerDisplay}</div>
    <div class="word-display">{currentWord}</div>
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
        role="application"
        aria-label="Composition canvas; click to set center, then click to set corner"
        on:pointerdown={handleCanvasPointerDown}
      >
        {#if placingPhase === 'center'}
          <div
            class="center-marker"
            style="left: {mouseCanvasX}%; top: {mouseCanvasY}%;"
          />
        {:else if previewSquare}
          <div
            class="center-marker center-marker-fixed"
            style="left: {pendingCenter.x}%; top: {pendingCenter.y}%;"
          />
          <div
            class="square square-preview"
            style="
              left: {previewSquare.x}%;
              top: {previewSquare.y}%;
              width: {previewSquare.size}%;
              height: {previewSquare.size}%;
              transform: rotate({previewSquare.rotation}deg);
            "
          />
        {/if}
        {#each squares as square (square.id)}
          <div
            class="square"
            class:square-delete-hover={hoveredForDeleteId === square.id}
            role="button"
            tabindex="0"
            aria-label="Square; hover center then click to delete"
            style="
              left: {square.x}%;
              top: {square.y}%;
              width: {square.size}%;
              height: {square.size}%;
              transform: rotate({square.rotation}deg);
            "
            on:click={(e) => handleSquareClick(e, square)}
            on:keydown={(e) => e.key === 'Enter' && deleteSquare(square.id)}
          />
        {/each}
      </div>
    </div>
    <p class="skip-hint">Space to skip</p>
  </div>
{/if}
