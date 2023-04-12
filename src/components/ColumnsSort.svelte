<script>
    import { getContext, onMount } from 'svelte';
  
    export let tableStatuses;
    export let fetchTables;
  
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
            try {
                const promises = reactiveTableStatuses.map((status, index) => {
                    return API.saveRow({
                        ...status,
                        Order: index + 1,
                    }).catch((error) => {
                        console.error(error);
                    });
                });
                await Promise.all(promises);
                await fetchTables();
                notificationStore.actions.success(
                    `Your column orders have been successfully rearranged.`
                );
            }catch (error) {
                console.log('Something went wrong with reordering the columns ', error)
            }
        }
        dragIndex = null;
        dropIndex = null;
    }
</script>
  
<div on:drop={handleDrop}>
    {#each tableStatuses as status, index}
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