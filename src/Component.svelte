<script>
  import { getContext, onMount } from "svelte";
  import KanbanColumns from "./components/Columns.svelte";
  import {
    Input,
    Label,
    Layout,
    ModalContent,
    Modal,
    Skeleton,
  } from "@budibase/bbui";

  // variables
  export let dataProvider;
  export let table;
  export let kanbanCardTitles;
  export let onClick;
  export let CardWidths;
  export let title;

  let tableStatuses = [];
  let columns = {};
  let droppable;
  let columnsLoaded = false;
  let kanbanColumns;
  let modal;
  let newColumnTitle;

  const cardsTableId = dataProvider?.datasource.tableId;
  const statusTableId = table?.tableId;

  const { styleable, API, notificationStore } = getContext("sdk");
  const component = getContext("component");
  // fetch cards table
  function fetchCardsTable() {
    return API.fetchTableData(cardsTableId)
      .then(function (data) {
        dataProvider.rows = data;
        return data;
      })
      .catch((err) => {
        console.log(err); // logging the error to the console.
      });
  }
  async function fetchTables() {
    try {
      await fetchCardsTable();
      API.fetchTableData(statusTableId).then(function (data){
          tableStatuses = data; // set table status after looping of data
          data.forEach((status) => {
            columns[status.Name] = [];
          });
          dataProvider.rows.forEach((card) => {
            const state = card[kanbanCardTitles][0].primaryDisplay;
            columns[state].push(card);
          });
      }).catch((err) => {
        console.log(err);
      }).finally(function () {
        columnsLoaded = true;
      });
    } catch(error){
      console.log("Unable to fetch cards table");
    }
  }
  // on page load
  onMount(() => {
    // define a Promise that resolves when the variable is populated
    function waitForVariable(dataProvider) {
      return new Promise(function (resolve) {
        var checkVariable = function () {
          if (dataProvider) {
            resolve();
          } else {
            setTimeout(checkVariable, 100); // check variable every 100ms
          }
        };
        checkVariable();
      });
    }
    // wait for the variable to become populated, then run the process
    waitForVariable(dataProvider)
      .then(function () {
        fetchTables();
      })
      .catch(function (err) {
        console.log(err); // logging the error to the console.
      });
  });
  // add new column
  async function addColumn() {
    if (newColumnTitle && !(newColumnTitle in columns)) {
      // check if column already exists, if it does don't let them create.
      const newColumnObject = [];
      const newColumns = { [newColumnTitle]: newColumnObject, ...columns };
      try {
        await API.saveRow({
          // create a new column on the backend
          tableId: statusTableId,
          Name: newColumnTitle,
        });
        columns = newColumns;
        fetchTables(); // reload main data after creating new column row
      } catch (error) {
        console.log(error);
      }
      notificationStore.actions.success(
        `New column: ${newColumnTitle} has been successfully created!`
      );
    } else {
      notificationStore.actions.error(
        "Sorry you have either tried to add a row without a name, or tried to create a duplicate column."
      );
    }
  }
  // call the moveItem() function in the child component
  function handleDrop(event) {
    childComponent.moveItem();
  }
</script>

<div use:styleable={$component.styles} class="container">
  {#if !dataProvider || !table}
    <div class="placeholder">
      Please provide a Data Provider and a Relationship Table
    </div>
  {:else if columnsLoaded}
    <div class="header">
      <h2>{title}</h2>
      <div>
        <button
          on:click={modal.show()}
          class="spectrum-Button spectrum-Button--sizeM spectrum-Button--cta"
          >Add Column</button
        >
      </div>
    </div>
    <div
      class="kanban-board"
      on:drop|preventDefault={() => handleDrop}
      bind:this={droppable}
    >
      <KanbanColumns
        {columns}
        {droppable}
        {tableStatuses}
        {kanbanCardTitles}
        {onClick}
        {cardsTableId}
        {CardWidths}
        {fetchTables}
        bind:this={kanbanColumns}
      />
    </div>
    <p>You have a total of {Object.keys(columns).length} columns</p>

    <Modal bind:this={modal}>
      <ModalContent title="Add Column" size="S" onConfirm={addColumn}>
        <form>
          <Layout noPadding gap="S">
            <div class="form-row">
              <Label>Column name</Label>
              <Input on:change bind:value={newColumnTitle} />
            </div>
          </Layout>
        </form>
      </ModalContent>
    </Modal>
  {:else}
    <Skeleton />
  {/if}
</div>

<style>
  *:focus {
    outline: none;
  }
  .kanban-board {
    display: flex;
    overflow: scroll;
    -ms-overflow-style: none; /* IE and Edge */
    scrollbar-width: none; /* Firefox */
    width: 100%;
  }
  .kanban-board::-webkit-scrollbar {
    display: none;
  }
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin: 0.5rem;
  }
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
</style>
