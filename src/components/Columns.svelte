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
  export let updateStatusOptions;
  export let primaryField;
  export let descriptionField;

  let draggedItem;
  const component = getContext("component");
  const { API, notificationStore } = getContext("sdk");

  function getCardTitle(card) {
    const value =
      card?.[primaryField] ??
      card?.Title ??
      card?.Name ??
      card?._id ??
      "";
    return String(value);
  }

  // set emitted item from card component
  function handleDragStart(event) {
    draggedItem = event.detail;
  }
  // emit to refresh columns
  function handleColumnsUpdated(event) {
    columns = event.detail;
  }
  // move items kanban style
  async function moveItem(event, dropArrayName) {
    const targetArrayName = dropArrayName || draggedArrayName;
    if (!targetArrayName || !columns[targetArrayName]) {
      return;
    }
    if (!draggedItem) {
      return;
    }
    // Single select fields are stored as plain strings.
    draggedItem[kanbanCardTitles] = targetArrayName;
    // find the dragged item in the original array
    const originalArrayName = Object.keys(columns).find((arrayName) =>
      columns[arrayName].some(
        (item) => item._id === draggedItem._id
      )
    );
    if (!originalArrayName) {
      return;
    }
    const originalArray = columns[originalArrayName]; // Get the original column array
    const draggedIndex = originalArray.findIndex(
      (item) => item._id === draggedItem._id
    ); // get the dragged index
    // check if the dragged item is being dropped onto its original column
    if (originalArrayName === targetArrayName) {
      return; // exit the function without making any changes
    }
    // remove if issues arise with doc conflicts etc, this just makes it really snappy for moving
      // remove the dragged item from the original array and add it to the dropped array
      const droppedArray = columns[targetArrayName];
      if (!droppedArray) {
        return;
      }
      const updatedDraggedItem = { ...draggedItem }; // make a copy of the dragged item to avoid modifying the original
      originalArray.splice(draggedIndex, 1); // remove dragged item from original Array
      droppedArray.push(updatedDraggedItem); // push drag item to new dragged array
      // update the columns object
      columns = {
        ...columns,
        [originalArrayName]: originalArray,
        [targetArrayName]: droppedArray,
      };
    // 
    // save card in the backend after its moved.
    try {
      await API.saveRow({
        _id: draggedItem._id,
        tableId: draggedItem.tableId,
        [primaryField]: getCardTitle(draggedItem),
        [kanbanCardTitles]: draggedItem[kanbanCardTitles],
      });
      await fetchTables();
    } catch (error) {
      console.log(error);
    }
    notificationStore.actions.success(
      `Your card ${getCardTitle(draggedItem)} has been successfully moved!`
    );
    draggedArrayName = null;
    draggedItem = null;
  }
  async function deleteColumn(columnName) {
    // refresh tables before deleting
    try {
      await fetchTables();
    } catch (error) {
      console.log('Failed to fetch tables before deletions', error);
    }
    const reducedStatuses = tableStatuses.map(({ Title }) => ({ Title }));
    if (columnName === "Backlog") {
      return;
    }
    const statusTitles = reducedStatuses.map((status) => status.Title);
    const nextStatusTitles = statusTitles.filter((title) => title !== columnName);
    if (!nextStatusTitles.includes("Backlog")) {
      nextStatusTitles.unshift("Backlog");
    }

    try {
      await updateStatusOptions(nextStatusTitles);
      await Promise.all((columns[columnName] || []).map(async (card) => {
        await API.saveRow({
          _id: card._id,
          tableId: card.tableId,
          [primaryField]: getCardTitle(card),
          [kanbanCardTitles]: "Backlog",
        });
      }));
      await fetchTables();
    } catch (error) {
      console.log("There was an error during deletion", error);
      notificationStore.actions.error(
        `Unable to delete ${columnName}.`
      );
      return;
    }
    notificationStore.actions.success(
      `Your column ${columnName} has been successfully deleted!`
    );
  }
  // add new card to specific column
  async function addCard(arrayName) {
    try {
      // save card in the backend after its moved.
      const payload = {
        tableId: cardsTableId,
        [primaryField]: "New Card",
        [kanbanCardTitles]: arrayName,
      };
      if (descriptionField) {
        payload[descriptionField] = "";
      }
      await API.saveRow(payload);
    } catch (error) {
      console.log(error);
    }
    // Call the fetchData function
    await fetchTables();
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
    on:drop|preventDefault={(event) => moveItem(event, arrayName)}
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
        {primaryField}
        {descriptionField}
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
