<script>
    import {tick} from "svelte"
    import {VirtualScroll} from "$lib"
    import {createSequenceGenerator, randomInteger} from "../mock"
    import TestItem from "../TestItem.svelte"

    const getItemId = createSequenceGenerator()

    let items = $state([])
    addItems(true, 100)

    let list = $state()

    function addItems(top = true, count = 10) {
        let new_items = []
        for (let i = 0; i < count; i++)
            new_items.push({uniqueKey: getItemId(), height: randomInteger(20, 60)})
        if (top)
            items = [...new_items, ...items]
        else
            items = [...items, ...new_items]
    }
</script>
<div class="vs">
    <VirtualScroll
            bind:this={list}
            data={items}
            key="uniqueKey"
    >
        {#snippet header()}
            <div >
                This is a header
            </div>
        {/snippet}
        {#snippet children({ data })}
            <TestItem {...data}/>
        {/snippet}
        {#snippet footer()}
            <div >
                This is a footer
            </div>
        {/snippet}
    </VirtualScroll>
</div>
<button onclick={addItems}>Add 10 to top</button>
<button onclick={() => addItems(false)}>Add 10 to bottom</button>
<button onclick={list.scrollToBottom}>To bottom</button>
<button onclick={async () => {
        addItems(false, 1)
        await tick()
        list.scrollToBottom()
    }}>Add 1 and scroll to bottom
</button>
<button onclick={()=>items[15].height = randomInteger(10, 150)}>Random height for 15 item</button>


<style>
    .vs {
        height: 300px;
    }
</style>
