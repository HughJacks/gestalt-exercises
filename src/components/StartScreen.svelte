<script>
  export let start;
  export let openPixelSpace;
</script>

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
