<script>
    import { getContext } from "svelte"
    import { createEventDispatcher } from "svelte"

    export let columns;
    export let column;
    export let arrayName;
    export let onClick;

    const { API, notificationStore } = getContext("sdk");

    const dispatch = createEventDispatcher();
    // dispatch item to parent component
    function handleDragStart(item) {
        dispatch("item-dragged", item );
    }
    // deleted card
    function deleteItem(arrayName, itemId, tableId, revId, rowId) {
        // find the array in which the item to delete exists
        const array = columns[arrayName];
        console.log(array);
        // find the index of the item to delete
        const indexToDelete = array.findIndex((item) => item["Auto ID"] === itemId);
        if (indexToDelete !== -1) {
            // remove the item from the array
            array.splice(indexToDelete, 1);
            // update the columns object
            columns = {
                ...columns,
                [arrayName]: array,
            };
            // emit a custom event with the updated columns object
            dispatch("columnsUpdated", columns);
            API.deleteRow({
                tableId: tableId,
                rowId: rowId,
                revId: revId
            })
        }
    }
    // add event to allow for cards to be updated
    const handleClick = e => {
        if (onClick) {
            onClick({ cardId: e._id })
        }
    }
</script>
  
{#each column as item (item["Auto ID"])}
    <div class="kanban-cards" 
        draggable="true" on:dragstart={() => handleDragStart(item)}
        on:click={() => handleClick(item)}
        on:keydown={(event) => {
            if (event.key === 'Enter' || event.key === ' ') {
                handleClick(item);
            }
        }}
    >
        <header class="flexed">
            <h3>{item.Title.length > 35 ? item.Title.substring(0, 35) + '...' : item.Title}</h3>
            <button class="delete" on:click={() => deleteItem(arrayName, item["Auto ID"], item.tableId, item._rev, item._id)}>
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0" />
                </svg>
            </button>
        </header>
        <p>{item.Description.length > 50 ? item.Description.substring(0, 250) + '...' : item.Description }</p>
    </div>
{/each}
  
<style>
    .kanban-cards {
        background-color: white;
        margin: 0.5rem 0;
        padding: 0.5rem;
        border-radius: 0.5rem;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        cursor: move;
    }
    .flexed {
        display: flex;
        align-items: center;
        justify-content: space-between;
        position: relative;
        z-index: 10;
    }
    svg {
        height: 1rem;
        width: 1rem;
    }
    button {
        cursor: pointer;
    }
    .delete svg {
        color: red;
    }
</style>