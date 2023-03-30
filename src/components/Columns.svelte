<script>
  import { getContext } from "svelte";
  import Card from "./Card.svelte";

  // variables
  export let columns;
  export let draggedArrayName;
  export let droppable;
  export let tableStatuses;
  export let kanbanCardTitles;
  export let CardWidths;
  export let onClick;
  export let cardsTableId;
  export let fetchTables;

  let draggedColumn;
  let draggedItem;
  const component = getContext("component");
  const { API, notificationStore } = getContext("sdk");

  // reduce statuses the array for more mangable data
  // set emitted item from card component
  function handleDragStart(event) {
    draggedItem = event.detail;
  }
  // emit to refresh columns
  function handleColumnsUpdated(event) {
    columns = event.detail;
  }
  // move items kanban style
  function moveItem() {
    const reducedStatuses = tableStatuses.map(({ _id, Name, tableId }) => ({ _id, Name, tableId }));
    // find specific object within reduce statuses arr
    const findDroppedStatusObj = reducedStatuses.find(
      (status) => status.Name === draggedArrayName
    );
    // delete state field and update with new moved state info
    delete draggedItem[kanbanCardTitles];
    draggedItem[kanbanCardTitles] = {
      _id: findDroppedStatusObj._id,
      primaryDisplay: findDroppedStatusObj.Name,
    };
    // find the dragged item in the original array
    const originalArrayName = Object.keys(columns).find((arrayName) =>
      columns[arrayName].some(
        (item) => item["Auto ID"] === draggedItem["Auto ID"]
      )
    );
    const originalArray = columns[originalArrayName]; // Get the original column array
    const draggedIndex = originalArray.findIndex(
      (item) => item["Auto ID"] === draggedItem["Auto ID"]
    ); // get the dragged index
    // check if the dragged item is being dropped onto its original column
    if (originalArrayName === draggedArrayName) {
      return; // exit the function without making any changes
    }
    // remove the dragged item from the original array and add it to the dropped array
    const droppedArray = columns[draggedArrayName];
    const updatedDraggedItem = { ...draggedItem }; // make a copy of the dragged item to avoid modifying the original
    originalArray.splice(draggedIndex, 1); // remove dragged item from original Array
    droppedArray.push(updatedDraggedItem); // push drag item to new dragged array
    // update the columns object
    columns = {
      ...columns,
      [originalArrayName]: originalArray,
      [draggedArrayName]: droppedArray,
    };
    // save card in the backend after its moved.
    API.saveRow({
      _id: draggedItem._id,
      tableId: draggedItem.tableId,
      Title: draggedItem.Title,
      [kanbanCardTitles]: [draggedItem[kanbanCardTitles]],
    });
    notificationStore.actions.success(
      `Your card ${draggedItem.Title} has been successfully moved!`
    );
    draggedArrayName = null;
  }
  function deleteColumn(columnName) {
    const reducedStatuses = tableStatuses.map(({ _id, Name, tableId }) => ({ _id, Name, tableId }));
    // find specific object within reduce statuses arr
    const findDeletedColumn = reducedStatuses.find(
      (status) => status.Name === columnName
    ); 
    // prevent deleting of the Backlog column
    if (columnName === "Backlog") {
      return;
    }
    // confirm deletion of column
    const confirmed = confirm(
      `Are you sure you want to delete the ${columnName} column?`
    );
    if (confirmed) {
      // move any cards in the specified column to Backlog
      if (columns[columnName]) {
        let findBacklogStatus;
        if (!columns["Backlog"]) {
          columns["Backlog"] = [];
          // create backlog if it doesn't exist
          const backlogTable = API.saveRow({
            tableId: findDeletedColumn.tableId,
            Name: "Backlog",
          });
          backlogTable
            .then((data) => {
              // bulk move cards within the backend to backlog
              columns["Backlog"].forEach((card) => {
                API.saveRow({
                  _id: card._id,
                  tableId: card.tableId,
                  Title: card.Title,
                  [kanbanCardTitles]: [data],
                });
              });
            })
            .catch((error) => {
              console.error(error);
            });
        } else {
          // find backlog data within arr
          findBacklogStatus = reducedStatuses.find(
            (status) => status.Name === "Backlog"
          ); 
          // bulk move cards within the backend to backlog
          columns[columnName].forEach((card) => {
            API.saveRow({
              _id: card._id,
              tableId: card.tableId,
              Title: card.Title,
              [kanbanCardTitles]: [findBacklogStatus],
            });
          });
        }
        // delete column
        API.deleteRow({
          tableId: findDeletedColumn.tableId,
          rowId: findDeletedColumn._id,
          revId: findDeletedColumn._rev,
        });
        columns["Backlog"] = columns["Backlog"].concat(columns[columnName]);
      }
      // delete the specified column
      delete columns[columnName];
    }
    // Refresh the columns
    fetchTables();
    columns = columns;
    notificationStore.actions.success(
      `Your column ${columnName} has been successfully deleted!`
    );
  }
  // add new card to specific column
  async function addCard(arrayName) {
    const reducedStatuses = tableStatuses.map(({ _id, Name, tableId }) => ({ _id, Name, tableId }));
    // find specific object within reduce statuses arr
    const findAddStatus = reducedStatuses.find(
      (status) => status.Name === arrayName
    ); 
    try {
      // save card in the backend after its moved.
      await API.saveRow({
        tableId: cardsTableId,
        Title: "New Card",
        Description: "",
        [kanbanCardTitles]: [findAddStatus],
      });
    } catch (error) {
      console.log(error);
    }
    // Call the fetchData function
    await fetchTables();
    // update current column details
    columns = columns;
    notificationStore.actions.success(
      `Your new card has been added to ${arrayName}!`
    );
  }
  function handleDragover(name) {
    draggedArrayName = name;
    droppable.classList.add("droppable");
  }
  function handleDragleave(name) {
    draggedArrayName = null;
    droppable.classList.remove("droppable");
  }
</script>

{#each Object.entries(columns) as [arrayName, column] (arrayName)}
  <div
    class="spectrum-Dropzone kanban-column"
    class:is-dragged={draggedArrayName === arrayName}
    on:dragover|preventDefault={handleDragover(arrayName)}
    on:dragleave|preventDefault={handleDragleave(arrayName)}
    on:drop|preventDefault={moveItem}
    style="min-width: {CardWidths}px;"
    role="region"
  >
    <div class="flexed">
      <h2
        class="spectrum-Heading spectrum-Heading--sizeM spectrum-Heading--regular spectrum-IllustratedMessage-heading"
      >
        {arrayName + " - " + column.length}
      </h2>
      <button
        disabled={arrayName === "Backlog"}
        on:click={deleteColumn(arrayName)}
        class="spectrum-ClearButton spectrum-ClearButton--sizeXL"
      >
        <div class="spectrum-ClearButton-fill">
          <svg
            class="spectrum-ClearButton-icon spectrum-Icon spectrum-UIIcon-Cross300"
            focusable="false"
            aria-hidden="true"
          >
            <use xlink:href="#spectrum-css-icon-Cross300" />
          </svg>
        </div>
      </button>
    </div>
    <div class="flexed">
      <div class="spectrum-Detail spectrum-Detail--sizeM">
        {column.length} cards
      </div>
      <button
        class="spectrum-ClearButton spectrum-ClearButton--sizeL"
        on:click={addCard(arrayName)}
      >
        <div class="spectrum-ClearButton-fill">
          <svg
            class="spectrum-Icon spectrum-ClearButton-icon spectrum-Icon--sizeL spectrum-icon-18-Add"
            focusable="false"
            aria-hidden="true"
          >
            <use xlink:href="#spectrum-icon-18-Add" />
          </svg>
        </div>
      </button>
    </div>
    <div class="h-full">
      <Card
        {columns}
        {column}
        {arrayName}
        {onClick}
        on:item-dragged={handleDragStart}
        on:columnsUpdated={handleColumnsUpdated}
      />
    </div>
  </div>
{/each}

<style>
  .kanban-column {
    flex: 1;
    margin: 0 0.5rem;
    /* background-color: #a4a4a4; */
    padding: 1rem;
    border-radius: 0.5rem;
  }
  .kanban-column:first-child {
    margin-left: 0;
  }
  .kanban-column:last-child {
    margin-right: 0;
  }

  .flexed {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .h-full {
    height: 100%;
    max-height: 500px;
    overflow-y: auto;
  }
  .spectrum-Dropzone {
    border-style: solid;
  }
</style>
