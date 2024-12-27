<script>
    import {flip} from "svelte/animate"
    import {VirtualScroll} from "$lib"
    import {createSequenceGenerator, randomInteger} from "../mock"
    import TestItem from "./TestItemHorizontal.svelte"

    const getItemId = createSequenceGenerator()
    const getNotificationId = createSequenceGenerator()

    let items = $state([])
    addItems(1000)

    let list = $state()
    let notifications = $state({})

    function addItems(count = 10) {
        let new_items = []
        for (let i = 0; i < count; i++)
            new_items.push({uniqueKey: getItemId(), width: randomInteger(40, 200)})
        items = [...items, ...new_items]
    }

    function addNotification(e) {
        const id = getNotificationId()
        notifications[id] = e
        setTimeout(() => {
            delete notifications[id]
            notifications = notifications
        }, 5000)
    }
</script>
<div class="vs">
    <VirtualScroll
            bind:this={list}
            data={items}
            key="uniqueKey"
            isHorizontal={true}
            onbottom={() => addNotification("bottom")}
            ontop={() => addNotification("top")}
    >
        {#snippet header()}
            <div >
                This is a header set via slot
            </div>
        {/snippet}
        {#snippet children({ data })}
            <TestItem {...data}/>
        {/snippet}
        {#snippet footer()}
            <div >
                This is a footer set via slot
            </div>
        {/snippet}
    </VirtualScroll>
</div>
<button onclick={() => list.scrollToOffset(0)}>To top</button>
<button onclick={list.scrollToBottom}>To bottom</button>
<div>
    {#each Object.entries(notifications) as [id, action] (id)}
        <div animate:flip>{action} </div>
    {/each}
</div>


<style>
    .vs {
        height: 300px;
    }

    .vs :global(.virtual-scroll-item) {
        padding: 0 4px;
    }

    .vs :global(.virtual-scroll-wrapper) {
        display: flex;
        flex-direction: row;
    }
    .vs :global(.virtual-scroll-root){
        display: flex;
        flex-direction: row;
    }
</style>
