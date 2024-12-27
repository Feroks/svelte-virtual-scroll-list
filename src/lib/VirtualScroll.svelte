<script>
    import Virtual, {isBrowser} from "./virtual.js"
    import Item from "./Item.svelte"
    import {onDestroy, onMount} from "svelte"

    /**
     * @typedef {Object} Props
     * @property {string} [key] - Unique key for getting data from `data`
     * @property {Array<any>} data - Source for list
     * @property {number} [keeps] - Count of rendered items
     * @property {number} [estimateSize] - Estimate size of each item, needs for smooth scrollbar
     * @property {boolean} [isHorizontal] - Scroll direction
     * @property {number|undefined} [start] - scroll position start index
     * @property {number|undefined} [offset] - scroll position offset
     * @property {boolean} [pageMode] - Let virtual list using global document to scroll through the list
     * @property {number} [topThreshold] - The threshold to emit `top` event, attention to multiple calls.
     * @property {number} [bottomThreshold] - The threshold to emit `bottom` event, attention to multiple calls.
     * @property {import('svelte').Snippet} [header]
     * @property {import('svelte').Snippet<[any]>} [children]
     * @property {import('svelte').Snippet} [footer]
     * @property {(param: *) => void} [onscroll]
     * @property {() => void} [ontop]
     * @property {() => void} [onbottom]
     */

    /** @type {Props} */
    let {
        key = "id",
        data,
        keeps = 30,
        estimateSize = 50,
        isHorizontal = false,
        start = undefined,
        offset = $bindable(undefined),
        pageMode = false,
        topThreshold = 0,
        bottomThreshold = 0,
        header,
        children,
        footer,
        onscroll,
        ontop,
        onbottom
    } = $props();

    let displayItems = $state([])
    let paddingStyle = $state()
    let directionKey = isHorizontal ? "scrollLeft" : "scrollTop"
    let range = null
    let virtual = new Virtual({
        slotHeaderSize: 0,
        slotFooterSize: 0,
        keeps: keeps,
        estimateSize: estimateSize,
        buffer: Math.round(keeps / 3), // recommend for a third of keeps
        uniqueIds: getUniqueIdFromDataSources(),
    }, onRangeChanged)
    let root = $state()
    let shepherd = $state()

    /**
     * @type {(id: number) => number}
     */
    export function getSize(id) {
        return virtual.sizes.get(id)
    }

    /**
     * Count of items
     * @type {() => number}
     */
    export function getSizes() {
        return virtual.sizes.size
    }

    /**
     * @type {() => number}
     */
    export function getOffset() {
        if (pageMode && isBrowser()) {
            return document.documentElement[directionKey] || document.body[directionKey]
        } else {
            return root ? Math.ceil(root[directionKey]) : 0
        }
    }

    /**
     * @type {() => number}
     */
    export function getClientSize() {
        const key = isHorizontal ? "clientWidth" : "clientHeight"
        if (pageMode && isBrowser()) {
            return document.documentElement[key] || document.body[key]
        } else {
            return root ? Math.ceil(root[key]) : 0
        }
    }

    /**
     * @type {() => number}
     */
    export function getScrollSize() {
        const key = isHorizontal ? "scrollWidth" : "scrollHeight"
        if (pageMode && isBrowser()) {
            return document.documentElement[key] || document.body[key]
        } else {
            return root ? Math.ceil(root[key]) : 0
        }
    }

    /**
     * @type {() => void}
     */
    export function updatePageModeFront() {
        if (root && isBrowser()) {
            const rect = root.getBoundingClientRect()
            const {defaultView} = root.ownerDocument
            const offsetFront = isHorizontal ? (rect.left + defaultView.pageXOffset) : (rect.top + defaultView.pageYOffset)
            virtual.updateParam("slotHeaderSize", offsetFront)
        }
    }

    /**
     * @type {(offset: number) => void}
     */
    export function scrollToOffset(offset) {
        if (!isBrowser()) return
        if (pageMode) {
            document.body[directionKey] = offset
            document.documentElement[directionKey] = offset
        } else if (root) {
            root[directionKey] = offset
        }
    }

    /**
     * @type {(index: number) => void}
     */
    export function scrollToIndex(index) {
        if (index >= data.length - 1) {
            scrollToBottom()
        } else {
            const offset = virtual.getOffset(index)
            scrollToOffset(offset)
        }
    }

    /**
     * @type {() => void}
     */
    export function scrollToBottom() {
        if (shepherd) {
            const offset = shepherd[isHorizontal ? "offsetLeft" : "offsetTop"]
            scrollToOffset(offset)

            // check if it's really scrolled to the bottom
            // maybe list doesn't render and calculate to last range,
            // so we need retry in next event loop until it really at bottom
            setTimeout(() => {
                if (getOffset() + getClientSize() + 1 < getScrollSize()) {
                    scrollToBottom()
                }
            }, 3)
        }
    }

    onMount(() => {
        if (start) {
            scrollToIndex(start)
        } else if (offset) {
            scrollToOffset(offset)
        }

        if (pageMode) {
            updatePageModeFront()

            document.addEventListener("scroll", onScroll, {
                passive: false,
            })
        }
    })

    onDestroy(() => {
        virtual.destroy()
        if (pageMode && isBrowser()) {
            document.removeEventListener("scroll", onScroll)
        }
    })

    function getUniqueIdFromDataSources() {
        return data.map((dataSource) => dataSource[key])
    }

    function onItemResized(event) {
        const {id, size, type} = event
        if (type === "item")
            virtual.saveSize(id, size)
        else if (type === "slot") {
            if (id === "header")
                virtual.updateParam("slotHeaderSize", size)
            else if (id === "footer")
                virtual.updateParam("slotFooterSize", size)

            // virtual.handleSlotSizeChange()
        }
    }

    function onRangeChanged(range_) {
        range = range_
        paddingStyle = paddingStyle = isHorizontal ? `0px ${range.padBehind}px 0px ${range.padFront}px` : `${range.padFront}px 0px ${range.padBehind}px`
        displayItems = data.slice(range.start, range.end + 1)
    }

    function onScroll(event) {
        const offset = getOffset()
        const clientSize = getClientSize()
        const scrollSize = getScrollSize()

        // iOS scroll-spring-back behavior will make direction mistake
        if (offset < 0 || (offset + clientSize > scrollSize) || !scrollSize) {
            return
        }

        virtual.handleScroll(offset)
        emitEvent(offset, clientSize, scrollSize, event)
    }

    function emitEvent(offset, clientSize, scrollSize, event) {
        onscroll?.({event, range: virtual.getRange()})

        if (virtual.isFront() && !!data.length && (offset - topThreshold <= 0)) {
            ontop?.();
        } else if (virtual.isBehind() && (offset + clientSize + bottomThreshold >= scrollSize)) {
            onbottom?.();
        }
    }

    $effect(() => {
        if (offset) {
            scrollToOffset(offset)
        }
    });
    $effect(() => handleKeepsChange(keeps));

    function handleKeepsChange(keeps) {
        virtual.updateParam("keeps", keeps)
        virtual.handleSlotSizeChange()
    }
    $effect.pre(() => { handleDataSourcesChange(data) });

    async function handleDataSourcesChange(data) {
        virtual.updateParam("uniqueIds", getUniqueIdFromDataSources())
        virtual.handleDataSourcesChange()
    }
</script>

<div bind:this={root} onscroll={onScroll} style="overflow-y: auto; height: inherit" class="virtual-scroll-root">
    {#if header}
        <Item onresize={onItemResized} type="slot" uniqueKey="header">
            {@render header?.()}
        </Item>
    {/if}
    <div style="padding: {paddingStyle}" class="virtual-scroll-wrapper">
        {#each displayItems as dataItem, dataIndex (dataItem[key])}
            <Item
                    onresize={onItemResized}
                    uniqueKey={dataItem[key]}
                    horizontal={isHorizontal}
                    type="item">
                {@render children?.({ data: dataItem, index: dataIndex, })}
            </Item>
        {/each}
    </div>
    {#if footer}
        <Item onresize={onItemResized} type="slot" uniqueKey="footer">
            {@render footer?.()}
        </Item>
    {/if}
    <div bind:this={shepherd} class="shepherd"
         style="width: {isHorizontal ? '0px' : '100%'};height: {isHorizontal ? '100%' : '0px'}"></div>
</div>
