<script>
  import { createEventDispatcher, getContext } from "svelte";

  export let columns;
  export let column;
  export let arrayName;
  export let onClick;
  export let primaryField;
  export let descriptionField;

  const { API, notificationStore } = getContext("sdk");

  const dispatch = createEventDispatcher();
  function getCardTitle(item) {
    const value =
      item?.[primaryField] ??
      item?.Title ??
      item?.Name ??
      item?._id ??
      "";
    return String(value);
  }
  function getCardDescription(item) {
    if (descriptionField && item?.[descriptionField] != null) {
      return String(item[descriptionField]);
    }
    if (item?.Description != null) {
      return String(item.Description);
    }
    return "";
  }
  // dispatch item to parent component
  function handleDragStart(item) {
    dispatch("item-dragged", item);
  }
  // deleted card
  function deleteItem(arrayName, rowId, tableId, revId) {
    // find the array in which the item to delete exists
    const array = columns[arrayName];
    // find the index of the item to delete
    const indexToDelete = array.findIndex((item) => item._id === rowId);
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
        revId: revId,
      });
      notificationStore.actions.success(
        `Your card has been successfully deleted!`
      );
    }
  }
  // add event to allow for cards to be updated
  const handleClick = (item) => {
    if (onClick) {
      onClick({ cardId: item._id });
    }
  };
</script>

{#each column as item (item._id)}
  <div
    class="spectrum-Card spectrum-Card--sizeM kanban-cards "
    draggable="true"
    on:dragstart={() => handleDragStart(item)}
    on:click={() => handleClick(item)}
    on:keydown={(event) => {
      if (event.key === "Enter" || event.key === " ") {
        handleClick(item);
      }
    }}
  >
    {#if item.Image && item.Image[0]}
      <img
        alt="{getCardTitle(item)} card thumbnail"
        class="spectrum-Asset-image"
        src={item.Image[0].url}
        style="max-width: 100%; object-fit: cover;"
      />
    {/if}
    <div class="spectrum-Card-body">
      <div class="spectrum-Card-header flexed">
        <div
          class="spectrum-Card-title spectrum-Heading spectrum-Heading--sizeXS"
        >
          {getCardTitle(item).length > 35
            ? getCardTitle(item).substring(0, 35) + "..."
            : getCardTitle(item)}
        </div>
        <button
          on:click={() =>
            deleteItem(
              arrayName,
              item._id,
              item.tableId,
              item._rev
            )}
          class="spectrum-ClearButton spectrum-ClearButton--sizeL"
        >
          <div class="spectrum-ClearButton-fill">
            <svg
              class="spectrum-ClearButton-icon spectrum-Icon spectrum-UIIcon-Cross200"
              focusable="false"
              aria-hidden="true"
            >
              <use xlink:href="#spectrum-css-icon-Cross200" />
            </svg>
          </div>
        </button>
      </div>
      <div class="spectrum-Card-content">
        {#if getCardDescription(item)}
          {getCardDescription(item).length > 50
            ? getCardDescription(item).substring(0, 250) + "..."
            : getCardDescription(item)}
        {/if}
      </div>
    </div>
  </div>
{/each}

<style>
  .kanban-cards {
    margin: 0.5rem 0;
    border-radius: 0.5rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    cursor: move;
    width: 100%;
  }
  .flexed {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  button {
    cursor: pointer;
  }
  .spectrum-Card-content {
    height: 100%!important;
    align-items: center;
    text-align: left;
  }
</style>
