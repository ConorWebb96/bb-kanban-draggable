<script>
  import { getContext, onMount } from "svelte";
  import KabanColumns from "./components/columns.svelte";
  // variables
  export let dataProvider;
  export let table;
  export let kabanCardTitles;
  export let onClick;
  let tableStatuses = [];
  let columns = {};
  let droppable;
  let columnsLoaded = false;
  let kabanColumns;
  const ticketsTableId = dataProvider.datasource.tableId;
  const statusTableId = table.tableId;

  const { styleable, API, notificationStore } = getContext("sdk");
  const component = getContext("component");
  // fetch tables function
  function fetchTables() {
    API.fetchTableData(statusTableId).then(function(data) { // get the promised statusTable data
      tableStatuses = data; // set table status after looping of data
      data.forEach((status) => {
        columns[status.Name] = [];
      })
      dataProvider.rows.forEach((card) => {
        const state = card[kabanCardTitles][0].primaryDisplay;
        columns[state].push(card);
      });
    }).catch(err => {
        console.log(err) // logging the error to the console.
    }).finally(function() {
      columnsLoaded = true; // set columns loaded to true after this opperation has been completed
    });
  };
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
    waitForVariable(dataProvider).then(function () {
      fetchTables();
    }).catch(function (err) {
      console.log(err); // logging the error to the console.
    });
  });
  // add new column
  async function addColumn() {
    const newColumnTitle = prompt("Enter column title");
    if (newColumnTitle && !(newColumnTitle in columns)) { // check if column already exists, if it does don't let them create.
      const newColumnObject = [];
      const newColumns = { [newColumnTitle]: newColumnObject, ...columns };
      try {
        await API.saveRow({ // create a new column on the backend 
          tableId: statusTableId,
          Name: newColumnTitle
        });
        columns = newColumns;
        fetchTables(); // reload main data after creating new column row
      } catch(error) {
        console.log(error);
      }
      notificationStore.actions.success(`New column: ${newColumnTitle} has been successfully created!`);
    }else {
      notificationStore.actions.error("Sorry you have either tried to add a row without a name, or tried to create a duplicate column.");
    }
  }
  // call the moveItem() function in the child component
  function handleDrop(event) {
    childComponent.moveItem(); 
  }
</script>

<div use:styleable={$component.styles} class="container">
    {#if columnsLoaded}
      <div class="flexed space-x-5">
          <span stlye="display: block;">You have a total of {Object.keys(columns).length} columns</span>
          <button on:click={addColumn} class="ml-5">Add Column</button>
      </div>
      <div class="kanban-board" on:drop|preventDefault={() => handleDrop} bind:this={droppable}>
          <KabanColumns 
              {columns} 
              {droppable} 
              {tableStatuses}
              {kabanCardTitles}
              {onClick}
              bind:this={kabanColumns}
          />
      </div>
    {:else}
      <p>Sorry but your content hasn't been loaded</p>
    {/if}
</div>

<style>
  .flexed {
    display: flex;
    align-items: center;
  }
  *:focus {
    outline: none;
  }
  .kanban-board {
    display: flex;
    overflow: scroll;
    -ms-overflow-style: none;  /* IE and Edge */
    scrollbar-width: none;  /* Firefox */
    width: 100%;
  }
  .kanban-board::-webkit-scrollbar {
    display: none;
  }
  button {
    max-width: fit-content!important;
    margin: 0.5rem 0;
  }
  button {
    cursor: pointer;
  }
  .ml-5 {
    margin-left: 1rem;
  }
</style>