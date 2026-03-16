<script>
  import { ArrowUpRight } from 'lucide-svelte';
  import { tick } from 'svelte';

  export let startInlineDraw;
  export let username = '';
  export let updateUsername = () => {};
  export let pixelSpaceVerbWeight;
  export let pixelSpaceFormWeight;
  export let verbWords = [];
  export let focusPixelSpaceWord;
  export let pixelSpaceDimWord;
  export let currentDrawWord = '';
  export let drawPanelOpen = false;

  export let pixelSpaceWrap;
  export let handlePixelSpacePointerDown;
  export let handlePixelSpacePointerMove;
  export let handlePixelSpacePointerUp;
  export let handlePixelSpaceWheel;
  export let handlePixelSpaceBackgroundClick;

  export let pixelSpaceLoading;
  export let pixelSpaceDrawings;
  export let pixelSpaceTransform;
  export let PIXEL_TILE_SIZE;
  export let pixelSpaceSelectedDrawingId;
  export let handlePixelSpaceTileClick;
  export let handlePixelSpaceTileHoverStart;
  export let handlePixelSpaceTileHoverMove;
  export let handlePixelSpaceTileHoverEnd;
  export let pixelSpaceHull;

  export let pixelSpaceHoveredDrawing;
  export let pixelSpaceTooltipX;
  export let pixelSpaceTooltipY;

  let showVerbList = false;
  let showClusterControls = false;
  const USERNAME_PLACEHOLDER = 'your name here';
  let nameSizerEl = null;
  let nameInputWidth = 0;
  $: nameSizerText = (username ?? '').length > 0 ? username : USERNAME_PLACEHOLDER;
  $: nameSizerText, tick().then(() => { if (nameSizerEl) nameInputWidth = nameSizerEl.offsetWidth; });
  $: navFocusedWord = drawPanelOpen && currentDrawWord ? currentDrawWord : pixelSpaceDimWord;
  $: pixelSpaceUniqueUsers = new Set(
    pixelSpaceDrawings
      .map((drawing) => String(drawing?.username ?? '').trim())
      .filter((username) => username.length > 0)
  ).size;

  function handleBackgroundKeydown(e) {
    if (e.key !== 'Enter' && e.key !== ' ') return;
    e.preventDefault();
    handlePixelSpaceBackgroundClick();
  }

  function toggleVerbList() {
    showVerbList = !showVerbList;
  }

  function toggleClusterControls() {
    showClusterControls = !showClusterControls;
  }

  function handleSpanActionKeydown(event, callback) {
    if (event.key !== 'Enter' && event.key !== ' ') return;
    event.preventDefault();
    callback();
  }

  function handleVerbWeightInput(event) {
    pixelSpaceVerbWeight = Number(event.currentTarget.value);
  }

  function handleFormWeightInput(event) {
    pixelSpaceFormWeight = Number(event.currentTarget.value);
  }

  function handleUsernameInput(event) {
    updateUsername(event.currentTarget.value);
  }

  function handleStartDraw() {
    if (drawPanelOpen || !username.trim()) return;
    startInlineDraw();
  }
</script>

<div
  class="pixel-space-screen"
  class:pixel-space-screen--with-draw-panel={drawPanelOpen}
>
  <div class="play-nav-text pixel-space-nav-text">
    <p class="play-nav-line pixel-space-nav-primary-line">
      <span
        class="play-nav-highlight play-nav-highlight-white play-nav-highlight-button pixel-space-nav-title"
        role="button"
        tabindex="0"
        on:click={toggleVerbList}
        on:keydown={(event) => handleSpanActionKeydown(event, toggleVerbList)}
      >
        Drawing Is a Verb
      </span>
      {#if showVerbList}
        {#each verbWords as word}
          <span
            class="play-nav-highlight play-nav-highlight-white play-nav-highlight-button pixel-space-nav-verb-item"
            class:pixel-space-word-focus={navFocusedWord === word}
            role="button"
            tabindex="0"
            on:click={() => focusPixelSpaceWord(word)}
            on:keydown={(event) => handleSpanActionKeydown(event, () => focusPixelSpaceWord(word))}
          >
            {word}
          </span>
        {/each}
      {:else if pixelSpaceDimWord !== null && !drawPanelOpen}
        <span class="play-nav-highlight pixel-space-word-focus">{pixelSpaceDimWord}</span>
      {/if}
      &
      <span class="play-nav-highlight play-nav-highlight-black">
        <a
          class="play-nav-inline-link"
          href="https://www.moma.org/calendar/galleries/5739?"
          target="_blank"
          rel="noreferrer"
        >
          <span>black is the densest color material</span>
          <ArrowUpRight size={11} aria-hidden="true" />
        </a>
      </span>
      <span class="play-nav-highlight play-nav-highlight-white pixel-space-nav-name-chip">
        <span class="pixel-space-nav-name-sizer" bind:this={nameSizerEl} aria-hidden="true">{nameSizerText}</span>
        <input
           class="pixel-space-nav-name-input"
           type="text"
           value={username}
           placeholder={USERNAME_PLACEHOLDER}
           style={nameInputWidth ? `width: ${nameInputWidth}px;` : ''}
           maxlength="40"
           on:input={handleUsernameInput}
           aria-label="Your name"
         />
      </span>
      <button
        type="button"
        class="play-nav-highlight play-nav-highlight-button pixel-space-nav-go"
        class:play-nav-highlight-disabled={!username.trim() || drawPanelOpen}
        on:click={handleStartDraw}
        disabled={!username.trim() || drawPanelOpen}
      >
        draw
      </button>
      <span class="play-nav-highlight play-nav-highlight-white">
        {pixelSpaceDrawings.length} drawings, by {pixelSpaceUniqueUsers} users
      </span>
      <span
        class="play-nav-highlight play-nav-highlight-white play-nav-highlight-button pixel-space-nav-title"
        role="button"
        tabindex="0"
        on:click={toggleClusterControls}
        on:keydown={(event) => handleSpanActionKeydown(event, toggleClusterControls)}
      >
        clustered
      </span>
      {#if showClusterControls}
        <span class="pixel-space-nav-cluster-label">by</span>
        <span class="play-nav-highlight play-nav-highlight-white pixel-space-nav-slider-chip">
          <span>verb</span>
          <input
            class="pixel-space-nav-slider"
            type="range"
            min="0"
            max="1"
            step="0.01"
            value={pixelSpaceVerbWeight}
            on:input={handleVerbWeightInput}
            aria-label="Verb clustering weight"
          />
        </span>
        &
        <span class="play-nav-highlight play-nav-highlight-white pixel-space-nav-slider-chip">
          <span>form</span>
          <input
            class="pixel-space-nav-slider"
            type="range"
            min="0"
            max="1"
            step="0.01"
            value={pixelSpaceFormWeight}
            on:input={handleFormWeightInput}
            aria-label="Form clustering weight"
          />
        </span>
      {/if}
    </p>
  </div>
  <div
    class="pixel-space-wrap"
    bind:this={pixelSpaceWrap}
    role="button"
    tabindex="0"
    aria-label="Pixel drawing space"
    on:pointerdown={handlePixelSpacePointerDown}
    on:pointermove={handlePixelSpacePointerMove}
    on:pointerup={handlePixelSpacePointerUp}
    on:pointercancel={handlePixelSpacePointerUp}
    on:wheel={handlePixelSpaceWheel}
    on:click={handlePixelSpaceBackgroundClick}
    on:keydown={handleBackgroundKeydown}
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
          class:pixel-space-tile-dimmed={pixelSpaceDimWord !== null && (drawing.word ?? '') !== pixelSpaceDimWord}
          class:pixel-space-tile-focused={pixelSpaceDimWord !== null && (drawing.word ?? '') === pixelSpaceDimWord}
          style="transform: translate({drawing.worldX - PIXEL_TILE_SIZE / 2}px, {drawing.worldY - PIXEL_TILE_SIZE / 2}px);"
          on:pointerdown
          on:click={(e) => handlePixelSpaceTileClick(e, drawing)}
          on:pointerenter={(e) => handlePixelSpaceTileHoverStart(e, drawing)}
          on:pointermove={(e) => handlePixelSpaceTileHoverMove(e, drawing)}
          on:pointerleave={handlePixelSpaceTileHoverEnd}
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
        </button>
      {/each}
      {#if pixelSpaceHull}
        <svg class="pixel-space-hull-overlay" aria-hidden="true">
          <path class="pixel-space-hull-path" d={pixelSpaceHull.path} />
        </svg>
      {/if}
    </div>
  </div>
  {#if pixelSpaceHoveredDrawing}
    <div
      class="pixel-space-cursor-tooltip"
      style="left: {pixelSpaceTooltipX + 12}px; top: {pixelSpaceTooltipY + 12}px;"
    >
      <div class="pixel-space-cursor-tooltip-title">{pixelSpaceHoveredDrawing.word || 'Untitled'}</div>
      <div class="pixel-space-cursor-tooltip-by">by {pixelSpaceHoveredDrawing.username}</div>
    </div>
  {/if}
</div>
