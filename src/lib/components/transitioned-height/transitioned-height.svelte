<script lang="ts">
  import setTabIndexRecursively from '$lib/utils/set-tab-index-recursive';
  import { onMount } from 'svelte';
  import { cubicInOut } from 'svelte/easing';
  import { tweened } from 'svelte/motion';

  // ___________
  // PROPS

  /** Force a height of 0, and optionally apply `negativeMarginWhileCollapsed`. */
  export let collapsed = false;

  /**
   * If true, all content height changes are transitioned. If false, only collapsing and expanding
   * the content is transitioned.
   */
  export let transitionHeightChanges = false;

  /**
   * Force a negative margin while collapsed. This is useful when you use `transitionedHeight`
   * in the context of some layout where there's further content below.
   */
  export let negativeMarginWhileCollapsed: string | undefined = undefined;

  /**
   * If true, when `collapsed`, all content will be removed from the tab index, and restored
   * when the section is expanded again.
   */
  export let removeFromTabIndexWhileCollapsed = true;

  // ____________
  // VARS

  let contentContainerElem: HTMLDivElement;

  let firstRender = true;
  onMount(() => (firstRender = false));

  /*
    During SSR, the server cannot determine the actual DOM node height of the content, so
    if the content is not collapsed, we set the height to `fit-content`.

    Further, if `transitionHeightChanges` is false, fitContent is set to true unless we're
    currently transitioning between collapsed and expanded states.
  */
  let fitContent = !collapsed;

  let containerHeight = tweened(0);

  let animating = false;

  let zeroHeight = collapsed;

  // ____________
  // LOGIC

  let previouslyCollapsed = collapsed;

  async function updateHeight(newHeightValue: number) {
    const collapsedChanged = previouslyCollapsed !== collapsed;

    const shouldTransition = transitionHeightChanges || collapsedChanged;

    if (collapsed && !collapsedChanged) return;

    // Set fitContent to false, so that we can smoothly animate the container height.
    if (shouldTransition) {
      fitContent = false;
      animating = true;
    }

    containerHeight.set(
      newHeightValue,
      shouldTransition ? { duration: 300, easing: cubicInOut } : undefined,
    );

    if (shouldTransition && !collapsed) {
      // Set fitContent back to true 300ms after the transition has ended.
      setTimeout(() => {
        fitContent = true;
        animating = false;
      }, 300);
    }

    zeroHeight = newHeightValue === 0;

    previouslyCollapsed = collapsed;
  }

  let sizeObserver: ResizeObserver | undefined;
  onMount(() => {
    sizeObserver = new ResizeObserver((v) => updateHeight(v[0].contentRect.height));
    sizeObserver.observe(contentContainerElem);

    return () => sizeObserver?.disconnect();
  });

  $: {
    if (!firstRender) {
      if (collapsed) {
        updateHeight(0);
      } else {
        updateHeight(contentContainerElem.getBoundingClientRect().height);
        setTabIndexRecursively(contentContainerElem, '0');
      }
    }
  }

  $: {
    if (contentContainerElem && removeFromTabIndexWhileCollapsed) {
      if (collapsed) {
        setTabIndexRecursively(contentContainerElem, '-1');
      } else {
        setTabIndexRecursively(contentContainerElem, '0');
      }
    }
  }

  $: heightStyleString = fitContent ? 'fit-content' : `${$containerHeight}px`;
</script>

<div
  class="transitioned-height"
  class:animating
  class:zero-height={zeroHeight}
  style:margin-bottom={negativeMarginWhileCollapsed && zeroHeight === true
    ? negativeMarginWhileCollapsed
    : undefined}
  style:height={heightStyleString}
>
  <div class="inner" bind:this={contentContainerElem}>
    <slot />
  </div>
</div>

<style>
  .transitioned-height {
    width: 100%;
    transition: margin-bottom 0.3s;
    position: relative;
  }

  .animating,
  .zero-height {
    overflow: hidden;
  }
</style>
