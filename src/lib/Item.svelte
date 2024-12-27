<script>
    import {onDestroy, onMount} from "svelte"

    /**
     * @typedef {Object} Props
     * @property {string} uniqueKey
     * @property {boolean} [horizontal]
     * @property {string} [type]
     * @property {import('svelte').Snippet} [children]
     * @property {(param: {id: string, size: number, type: string}) => void} [onresize]
     */

    /** @type {Props} */
    let {
        horizontal = false,
        uniqueKey,
        type = "item",
        children,
        onresize
    } = $props();

    let resizeObserver
    let itemDiv
    let previousSize

    const shapeKey = horizontal ? "offsetWidth" : "offsetHeight"

    onMount(() => {
        if (typeof ResizeObserver !== "undefined") {
            resizeObserver = new ResizeObserver(dispatchSizeChange)
            resizeObserver.observe(itemDiv)
        }
    })
    $effect(dispatchSizeChange)
    onDestroy(() => {
        if (resizeObserver) {
            resizeObserver.disconnect()
            resizeObserver = null
        }
    })

    function dispatchSizeChange() {
        const size = itemDiv ? itemDiv[shapeKey] : 0
        if (size === previousSize) return
        previousSize = size
        onresize?.({id: uniqueKey, size, type});
    }
</script>

<div bind:this={itemDiv} class="virtual-scroll-item">
    {@render children?.()}
</div>
