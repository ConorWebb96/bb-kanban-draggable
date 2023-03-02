<script>
    import { getContext } from "svelte"
    import Card from "./card.svelte";

    // variables
    export let columns;
    export let droppedArrayName;
    export let droppable;
    export let tableStatuses;
    export let kabanCardTitles;
    export let onClick;
    let draggedItem;
    const component = getContext("component");
    const { API, notificationStore } = getContext("sdk");
    // set emitted item from card component
    function handleDragStart(event) {
        draggedItem = event.detail;
    }
    // emit to refresh columns
    function handleColumnsUpdated(event) {
        columns = event.detail;
    }
    // move items kaban style
    function moveItem() {
        const reducedStatuses = tableStatuses.map(({ _id, Name }) => ({ _id, Name })); // reduce statuses the array for more mangable data
        const findDroppedStatusObj = reducedStatuses.find(status => status.Name === droppedArrayName); // find specific object within reduce statuses arr
        // delete state field and update with new moved state info
        delete draggedItem[kabanCardTitles];
        draggedItem[kabanCardTitles] = {
            _id: findDroppedStatusObj._id,
            primaryDisplay: findDroppedStatusObj.Name
        };
        // find the dragged item in the original array
        const originalArrayName = Object.keys(columns).find((arrayName) =>
            columns[arrayName].some((item) => item["Auto ID"] === draggedItem["Auto ID"])
        );
        const originalArray = columns[originalArrayName]; // Get the original column array
        const draggedIndex = originalArray.findIndex((item) => item["Auto ID"] === draggedItem["Auto ID"]); // get the dragged index
        // check if the dragged item is being dropped onto its original column
        if (originalArrayName === droppedArrayName) {
            return; // exit the function without making any changes
        }
        // remove the dragged item from the original array and add it to the dropped array
        const droppedArray = columns[droppedArrayName];
        const updatedDraggedItem = { ...draggedItem }; // make a copy of the dragged item to avoid modifying the original
        originalArray.splice(draggedIndex, 1); // remove dragged item from original Array
        droppedArray.push(updatedDraggedItem); // push drag item to new dragged array
        // update the columns object
        columns = {
            ...columns,
            [originalArrayName]: originalArray,
            [droppedArrayName]: droppedArray
        };
        // save card in the backend after its moved.
        API.saveRow({
            _id: draggedItem._id,
            tableId: draggedItem.tableId,
            Title: draggedItem.Title,
            [kabanCardTitles]: [draggedItem[kabanCardTitles]]
        });
        notificationStore.actions.success(`Your card ${draggedItem.Title} has been successfully moved!`);
    }
    function deleteColumn(columnName) {
        const reducedStatuses = tableStatuses.map(({ _id, Name, tableId }) => ({ _id, Name, tableId })); // reduce statuses the array for more mangable data
        const findDeletedColumn = reducedStatuses.find(status => status.Name === columnName); // find specific object within reduce statuses arr
        // prevent deleting of the Backlog column
        if (columnName === "Backlog") { 
            return;
        }
        // confirm deletion of column
        const confirmed = confirm(`Are you sure you want to delete the ${columnName} column?`);
        if (confirmed) {
            // move any cards in the specified column to Backlog
            if (columns[columnName]) {
                let findBacklogStatus;
                if (!columns["Backlog"]) {
                    columns["Backlog"] = [];
                    // create backlog if it doesn't exist
                    const backlogTable = API.saveRow({
                        tableId: findDeletedColumn.tableId,
                        Name: "Backlog"
                    });
                    backlogTable.then((data) => {
                        // bulk move cards within the backend to backlog
                        columns["Backlog"].forEach((card) => {
                            API.saveRow({
                                _id: card._id,
                                tableId: card.tableId,
                                Title: card.Title,
                                [kabanCardTitles]: [data]
                            });
                        })
                    }).catch((error) => {
                        console.error(error);
                    });
                } else {
                    findBacklogStatus = reducedStatuses.find(status => status.Name === 'Backlog'); // find backlog data within arr
                    // bulk move cards within the backend to backlog
                    columns[columnName].forEach((card) => {
                        API.saveRow({
                            _id: card._id,
                            tableId: card.tableId,
                            Title: card.Title,
                            [kabanCardTitles]: [findBacklogStatus]
                        });
                    })
                }
                // delete column
                API.deleteRow({
                    tableId: findDeletedColumn.tableId,
                    rowId: findDeletedColumn._id,
                    revId: findDeletedColumn._rev
                })
                columns["Backlog"] = columns["Backlog"].concat(columns[columnName]);
            }
            // delete the specified column
            delete columns[columnName];
        }
        // Refresh the columns
        columns = columns; 
        notificationStore.actions.success(`Your column ${columnName} has been successfully deleted!`);
    }
    function addCard() {

    }
</script>

{#each Object.entries(columns) as [arrayName, column] (arrayName)}
    <div class="kanban-column">
        <div class="flexed">
            <h2>{arrayName + ' - ' + column.length}</h2>
            {#if arrayName != "Backlog"}
                <button class="delete" on:click={() => deleteColumn(arrayName)}>
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                        <path stroke-linecap="round" stroke-linejoin="round" d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0" />
                    </svg>
                </button>
            {/if}
        </div>
        <div
            on:dragover|preventDefault={() => droppable.classList.add("droppable")}
            on:dragleave|preventDefault={() => droppable.classList.remove("droppable")}
            on:drop|preventDefault={() => {
                droppedArrayName = arrayName;
                moveItem();
            }}
            class="h-full"
        >
            <Card {columns} {column} {arrayName} {onClick} on:item-dragged={handleDragStart} on:columnsUpdated={handleColumnsUpdated} />
            <button class="ml-5" on:click={() => addCard(arrayName, column)}>Add Card</button>
        </div>
    </div>
{/each}

<style>
    .kanban-column {
        flex: 1;
        margin: 0 1rem;
        background-color: #a4a4a4;
        padding: 1rem;
        border-radius: 0.5rem;
        min-width: 350px;
        max-width: 350px;
    }
    .kanban-column:first-child {
        margin-left: 0;
    }
    .kanban-column:last-child {
        margin-right: 0;
    }
    button {
        margin: 0.5rem 0;
    }
    .flexed {
        display: flex;
        align-items: center;
        justify-content: space-between;
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
    .h-full {
        height: 100%;
        max-height: 500px;
    }
</style>