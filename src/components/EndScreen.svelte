<script>
  import { RotateCcw } from 'lucide-svelte';

  export let roundResults;
  export let playAgain;
  export let openPixelSpace;
</script>

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
