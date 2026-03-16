<script>
  import { MoveDiagonal2, RotateCw, Timer } from 'lucide-svelte';

  export let currentWord;
  export let wordIndex = 0;
  export let wordsPerRound = 5;
  export let gameMode;
  export let timerDisplay;
  export let submitCurrentDrawing;
  export let canCloseDrawPanel = false;
  export let closeDrawPanel = () => {};

  export let canvasWrap;
  export let borderSquareRef;

  export let cursorToolIcon;
  export let beginCreationDrag;
  export let previewSquare;
  export let squares;
  export let draggingSquareId;
  export let altPressed;
  export let selectedSquareIds;
  export let handleSquarePointerDown;
  export let selectionOverlay;
  export let CORNER_HANDLES;
  export let handleTransformHoverStart;
  export let handleTransformHoverEnd;
  export let handleResizePointerDown;
  export let handleRotatePointerDown;
  export let mouseCanvasX;
  export let mouseCanvasY;

  let showMoreInstructions = false;

  function toggleMoreInstructions() {
    showMoreInstructions = !showMoreInstructions;
  }
</script>

<div class="play-screen">
  <div class="play-nav-text play-screen-nav">
    <p class="play-nav-line">
      {#if canCloseDrawPanel}
        <button
          class="play-nav-highlight play-nav-highlight-button play-nav-highlight-inverted"
          on:click={closeDrawPanel}
          aria-label="Close drawing panel"
        >
          X
        </button>
      {/if}
      <span class="play-nav-highlight play-nav-highlight-current-word">{currentWord}</span>
      {#if gameMode === 'timed'}
        <span class="play-nav-drawing-count">{wordIndex + 1}/{wordsPerRound}</span>
      {/if}
      {#if gameMode === 'timed'}
        <span class="play-nav-highlight play-nav-highlight-inverted play-nav-timer-chip" aria-live="polite">
          <Timer size={11} aria-hidden="true" />
          <span>{timerDisplay}</span>
        </span>
      {/if}
      <span class="play-nav-highlight play-nav-highlight-yellow">click</span>
      <span class="play-nav-highlight play-nav-hint-plain">&amp;</span>
      <span class="play-nav-highlight play-nav-highlight-yellow">drag</span>
      <span class="play-nav-highlight play-nav-hint-plain">to add a</span>
      <span class="play-nav-highlight play-nav-highlight-inverted">square</span>
      <span class="play-nav-plain-text">&amp;</span>
      <button
        class="play-nav-highlight play-nav-highlight-button play-nav-highlight-yellow"
        on:click={toggleMoreInstructions}
        aria-expanded={showMoreInstructions}
        aria-label="Toggle drawing instructions"
      >
        more
      </button>
      <button
        class="play-nav-highlight play-nav-highlight-button play-nav-highlight-inverted"
        on:click={submitCurrentDrawing}
      >
        Submit drawing
      </button>
    </p>
    {#if showMoreInstructions}
      <p class="play-nav-line play-nav-more-line">
        <span class="play-nav-highlight play-nav-highlight-yellow">drag square</span>
        <span class="play-nav-plain-text">to move</span>
        <span class="play-nav-highlight play-nav-highlight-yellow">shift + drag</span>
        <span class="play-nav-plain-text">to align</span>
        <span class="play-nav-highlight play-nav-highlight-yellow">⌥ + drag</span>
        <span class="play-nav-plain-text">to duplicate</span>
        <span class="play-nav-highlight play-nav-highlight-yellow">⌘ + z</span>
        <span class="play-nav-plain-text">to undo</span>
        <span class="play-nav-highlight play-nav-highlight-yellow">⌘ + shift + z</span>
        <span class="play-nav-plain-text">to redo</span>
      </p>
    {/if}
  </div>
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
</div>
