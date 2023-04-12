<script>
    import { getContext, createEventDispatcher } from 'svelte';
  
    export let tableStatuses;
    const dispatch = createEventDispatcher();
  
    const { API, notificationStore } = getContext('sdk');
  
    let dragIndex = null;
    let dropIndex = null;
    $: reactiveTableStatuses = tableStatuses;

    function handleDragStart(event, index) {
      event.dataTransfer.effectAllowed = 'move';
      dragIndex = index;
    }
    function handleDragOver(event, index) {
        event.preventDefault();
        if (index === dragIndex) return;
        dropIndex = index;
    }
    async function handleDrop(event) {
        if (dragIndex !== null && dropIndex !== null) {
            tableStatuses.splice(
                dropIndex,
                0,
                tableStatuses.splice(dragIndex, 1)[0]
            );
            dispatch('tableStatusesUpdated', reactiveTableStatuses);
            reactiveTableStatuses = tableStatuses;
        }
        dragIndex = null;
        dropIndex = null;
    }
</script>
  
<div on:drop={handleDrop}>
    {#each reactiveTableStatuses as status, index}
        <div
        class="status-item"
        draggable="true"
        on:dragstart={(event) => handleDragStart(event, index)}
        on:dragover={(event) => handleDragOver(event, index)}
        on:dragend={() => {
            dragIndex = null;
            dropIndex = null;
        }}
        >
            {status.Title}
        </div>
    {/each}
</div>
  
<style>
    .status-item {
        padding: 10px;
        margin-bottom: 10px;
        border: 1px solid #ccc;
        background-color: #f8f8f8;
        cursor: pointer;
    }
</style>