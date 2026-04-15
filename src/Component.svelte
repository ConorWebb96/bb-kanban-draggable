<script>
  import { getContext } from "svelte";
  import KanbanColumns from "./components/Columns.svelte";
  import SortColumns from "./components/ColumnsSort.svelte";
  import {
    Input,
    Label,
    Layout,
    ModalContent,
    Modal,
  } from "@budibase/bbui";

  // variables
  export let dataProvider;
  export let kanbanCardTitles;
  export let onClick;
  export let CardWidths;
  export let title;

  let tableStatuses = [];
  let columns = {};
  let droppable;
  let columnsLoaded = false;
  let isLoading = false;
  let loadError = null;
  let kanbanColumns;
  let modal;
  let orderModal;
  let newColumnTitle;
  let updatedTableStatuses = [];
  let searchQuery = "";
  let cardsTableId;
  let primaryField = "Title";
  let descriptionField = null;
  let hasInitialised = false;
  let lastFetchKey = null;

  function getCardTitle(card) {
    const value =
      card?.[primaryField] ??
      card?.Title ??
      card?.Name ??
      card?._id ??
      "";
    return String(value);
  }

  function getCardDescription(card) {
    if (descriptionField && card?.[descriptionField] != null) {
      return String(card[descriptionField]);
    }
    if (card?.Description != null) {
      return String(card.Description);
    }
    return "";
  }

  $: filteredColumns = searchQuery.length > 3
  ? Object.fromEntries(Object.entries(columns).map(([key, value]) => {
      const filteredCards = value.filter(card => {
        const regex = new RegExp(searchQuery, 'gi');
        const titleValue = getCardTitle(card);
        const descriptionValue = getCardDescription(card);
        const match = (titleValue.match(regex) || []).length > 0 ||
          (descriptionValue.match(regex) || []).length > 0;
        return match;
      });
      return [key, filteredCards];
    }))
  : columns;

  const { styleable, API, notificationStore } = getContext("sdk");
  const component = getContext("component");
  const DEFAULT_COLUMN_TITLE = "Backlog";
  const LOG_PREFIX = "[bb-kanban-draggable]";

  const logDebug = (...args) => {
    console.log(LOG_PREFIX, ...args);
  };

  function getProviderRows(provider) {
    if (!provider) {
      return [];
    }
    if (Array.isArray(provider)) {
      return provider;
    }
    if (Array.isArray(provider.rows)) {
      return provider.rows;
    }
    if (Array.isArray(provider.rows?.rows)) {
      return provider.rows.rows;
    }
    return [];
  }

  $: providerRows = getProviderRows(dataProvider);
  $: cardsTableId =
    dataProvider?.datasource?.tableId ||
    providerRows?.[0]?.tableId ||
    null;
  $: {
    const schema = dataProvider?.schema || {};
    primaryField =
      dataProvider?.primaryDisplay ||
      (schema.Title ? "Title" : schema.Name ? "Name" : "Title");
    descriptionField = schema.Description ? "Description" : null;
  }

  // Reactive initialisation/refetch instead of mount-time polling.
  $: {
    if (dataProvider && kanbanCardTitles) {
      const fetchKey = `${cardsTableId || "no-table"}::${kanbanCardTitles}::${providerRows.length}`;
      if (!hasInitialised || lastFetchKey !== fetchKey) {
        hasInitialised = true;
        lastFetchKey = fetchKey;
        columnsLoaded = false;
        logDebug("Triggering fetchTables via reactive initialiser", {
          fetchKey,
        });
        fetchTables();
      }
    }
  }

  function normalizeStatusValue(value) {
    if (!value) {
      return null;
    }
    if (typeof value === "string") {
      return value;
    }
    if (Array.isArray(value) && value.length > 0) {
      return value[0]?.primaryDisplay || value[0]?.Title || value[0]?._id || null;
    }
    if (typeof value === "object") {
      return value.primaryDisplay || value.Title || value._id || null;
    }
    return null;
  }

  async function getStatusFieldInclusion() {
    if (!cardsTableId) {
      throw new Error("No tableId could be resolved from data provider.");
    }
    const tableDefinition = await API.fetchTableDefinition(cardsTableId);
    const statusFieldDefinition = tableDefinition?.schema?.[kanbanCardTitles];
    logDebug("Status field definition", {
      cardsTableId,
      kanbanCardTitles,
      statusFieldType: statusFieldDefinition?.type,
      constraints: statusFieldDefinition?.constraints,
    });
    const inclusion = statusFieldDefinition?.constraints?.inclusion;
    return Array.isArray(inclusion) ? [...inclusion] : [];
  }

  async function updateStatusOptions(inclusion) {
    if (!cardsTableId) {
      throw new Error("No tableId could be resolved from data provider.");
    }
    const tableDefinition = await API.fetchTableDefinition(cardsTableId);
    if (!tableDefinition?.schema?.[kanbanCardTitles]) {
      throw new Error(`Field ${kanbanCardTitles} could not be found in table schema`);
    }
    const dedupedInclusion = [...new Set(inclusion.filter(Boolean))];
    tableDefinition.schema[kanbanCardTitles] = {
      ...tableDefinition.schema[kanbanCardTitles],
      constraints: {
        ...(tableDefinition.schema[kanbanCardTitles].constraints || {}),
        inclusion: dedupedInclusion,
      },
    };
    logDebug("Updating status options", {
      cardsTableId,
      kanbanCardTitles,
      inclusion: dedupedInclusion,
    });
    await API.saveTable(tableDefinition);
  }

  // fetch cards table
  async function fetchCardsTable() {
    // If no datasource.tableId exists, use the already supplied provider rows.
    if (!cardsTableId) {
      const rows = getProviderRows(dataProvider);
      logDebug("No cardsTableId found. Falling back to provider rows only.", {
        providerShape: dataProvider ? Object.keys(dataProvider) : null,
        rowsCount: rows.length,
      });
      return rows;
    }
    try {
      const data = await API.fetchTableData(cardsTableId);
      const rows = Array.isArray(data) ? data : data?.rows || [];
      dataProvider.rows = rows;
      return rows;
    } catch (err) {
      console.log(err);
      return [];
    }
  }
  async function fetchTables() {
    isLoading = true;
    loadError = null;
    try {
      logDebug("fetchTables start", {
        kanbanCardTitles,
        cardsTableId,
        primaryField,
        descriptionField,
        providerShape: dataProvider ? Object.keys(dataProvider) : null,
      });
      const rows = await fetchCardsTable();
      const inclusion = await getStatusFieldInclusion();
      const effectiveInclusion = inclusion.length ? inclusion : [DEFAULT_COLUMN_TITLE];
      tableStatuses = effectiveInclusion.map((statusTitle, index) => ({
        Title: statusTitle,
        Order: index + 1,
      }));
      const newColumns = {};
      tableStatuses.forEach((status) => {
        newColumns[status.Title] = [];
      });
      rows.forEach((card) => {
        const state = normalizeStatusValue(card[kanbanCardTitles]);
        if (state) {
          if (!newColumns[state]) {
            newColumns[state] = [];
          }
          newColumns[state].push(card);
        } else {
          if (!newColumns[DEFAULT_COLUMN_TITLE]) {
            newColumns[DEFAULT_COLUMN_TITLE] = [];
          }
          newColumns[DEFAULT_COLUMN_TITLE].push(card);
        }
      });
      columns = newColumns; // update columns object
      logDebug("fetchTables success", {
        rowsCount: rows.length,
        statusCount: tableStatuses.length,
        columnKeys: Object.keys(columns),
      });
    } catch(error){
      console.log("Unable to fetch cards table", error);
      loadError = error?.message || "Unable to load kanban data.";
      logDebug("fetchTables failed", {
        message: error?.message,
        cardsTableId,
        kanbanCardTitles,
      });
    } finally {
      columnsLoaded = true;
      isLoading = false;
      logDebug("fetchTables finalised", {
        columnsLoaded,
        isLoading,
        loadError,
      });
    }
}

  // add new column
  async function addColumn() {
    // check if column already exists, if it does don't let them create.
    if (newColumnTitle && !(newColumnTitle in columns)) {
      try {
        await updateStatusOptions([
          ...tableStatuses.map((status) => status.Title),
          newColumnTitle,
        ]);
        await fetchTables();
      } catch (error) {
        console.log(error);
        notificationStore.actions.error(
          `Unable to add column "${newColumnTitle}". Make sure the selected field is a single select column.`
        );
        return;
      }
      notificationStore.actions.success(
        `New column: ${newColumnTitle} has been successfully created!`
      );
    } else {
      notificationStore.actions.error(
        "Sorry you have either tried to add a row without a Title, or tried to create a duplicate column."
      );
    }
  }
  function handleTableStatusesUpdated(event) {
    updatedTableStatuses = event.detail;
  }
  async function refreshColumns() {
    try {
      const reorderedStatuses =
        updatedTableStatuses.length > 0
          ? updatedTableStatuses.map((status) => status.Title)
          : tableStatuses.map((status) => status.Title);
      await updateStatusOptions(reorderedStatuses);
      await fetchTables();
      notificationStore.actions.success(
          `Your column orders have been successfully rearranged.`
      );
    } catch (error) {
      console.error(error);
      notificationStore.actions.error("Unable to reorder columns.");
    }
  }
  // call the moveItem() function in the child component
  function handleDrop(event) {
    childComponent.moveItem();
  }
</script>
<div use:styleable={$component.styles} class="container">
  {#if !dataProvider || !kanbanCardTitles}
    <div class="placeholder">
      Please provide a Data Provider and a Single Select field
    </div>
  {:else}
    {#if loadError}
      <div class="placeholder">{loadError}</div>
    {/if}
    {#if isLoading}
      <div class="placeholder">Loading board...</div>
    {/if}
    <div class="header">
        <h2>{#if title}{title}{/if}</h2>
        <div class="header">
          <input type="text" bind:value={searchQuery} placeholder="Search..." class="customInput spectrum-Textfield-input" />
          <button
            on:click={modal.show()}
            class="spectrum-Button spectrum-Button--sizeM spectrum-Button--cta"
            style="margin-right: 0.5rem; min-width: fit-content;"
            >Add Column</button
          >
          {#if tableStatuses.length > 1}
            <button style="border: none; cursor: pointer; background-color: transparent;" on:click={orderModal.show()}>
              <svg xmlns="http://www.w3.org/2000/svg" height="18" viewBox="0 0 18 18" width="18" class="customSvgColour"><rect id="Canvas" fill="fillCurrent" opacity="0" width="18" height="18" /><path class="fill" d="M16.45,7.8965H14.8945a5.97644,5.97644,0,0,0-.921-2.2535L15.076,4.54a.55.55,0,0,0,.00219-.77781L15.076,3.76l-.8365-.836a.55.55,0,0,0-.77781-.00219L13.4595,2.924,12.357,4.0265a5.96235,5.96235,0,0,0-2.2535-.9205V1.55a.55.55,0,0,0-.55-.55H8.45a.55.55,0,0,0-.55.55V3.106a5.96235,5.96235,0,0,0-2.2535.9205l-1.1-1.1025a.55.55,0,0,0-.77781-.00219L3.7665,2.924,2.924,3.76a.55.55,0,0,0-.00219.77781L2.924,4.54,4.0265,5.643a5.97644,5.97644,0,0,0-.921,2.2535H1.55a.55.55,0,0,0-.55.55V9.55a.55.55,0,0,0,.55.55H3.1055a5.967,5.967,0,0,0,.921,2.2535L2.924,13.4595a.55.55,0,0,0-.00219.77782l.00219.00218.8365.8365a.55.55,0,0,0,.77781.00219L4.5405,15.076,5.643,13.9735a5.96235,5.96235,0,0,0,2.2535.9205V16.45a.55.55,0,0,0,.55.55H9.55a.55.55,0,0,0,.55-.55V14.894a5.96235,5.96235,0,0,0,2.2535-.9205L13.456,15.076a.55.55,0,0,0,.77782.00219L14.236,15.076l.8365-.8365a.55.55,0,0,0,.00219-.77781l-.00219-.00219L13.97,12.357a5.967,5.967,0,0,0,.921-2.2535H16.45a.55.55,0,0,0,.55-.55V8.45a.55.55,0,0,0-.54649-.55349ZM11.207,9A2.207,2.207,0,1,1,9,6.793H9A2.207,2.207,0,0,1,11.207,9Z" /></svg>
            </button>
          {/if}
        </div>
    </div>
    <div
      class="kanban-board"
      on:drop|preventDefault={() => handleDrop}
      bind:this={droppable}
    >
      <KanbanColumns
        columns={filteredColumns}
        {droppable}
        {tableStatuses}
        {kanbanCardTitles}
        {onClick}
        {cardsTableId}
        {CardWidths}
        {fetchTables}
        {updateStatusOptions}
        {primaryField}
        {descriptionField}
        bind:this={kanbanColumns}
      />
    </div>
    <p>You have a total of {Object.keys(columns).length} columns</p>

    <Modal bind:this={modal}>
      <ModalContent title="Add Column" size="S" onConfirm={addColumn}>
        <form>
          <Layout noPadding gap="S">
            <div class="form-row">
              <Label>Column Title</Label>
              <Input on:change bind:value={newColumnTitle} />
            </div>
          </Layout>
        </form>
      </ModalContent>
    </Modal>
    <Modal bind:this={orderModal}>
      <ModalContent title="Columns Positions" size="XL" onConfirm={refreshColumns}>
        <SortColumns
          {tableStatuses}
          on:tableStatusesUpdated={handleTableStatusesUpdated}
        />
      </ModalContent>
    </Modal>
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
  .customInput {
    display: block;
    border-radius: 4px;
    /* border: 1px solid rgb(220 220 220); */
    padding: 6px;
    margin-right: 0.5rem;
  }
  .spinner {
    width: 1.5rem;
    height: 1.5rem;
    border-top-color: #444;
    border-left-color: #444;
    animation: spinner 400ms linear infinite;
    border-bottom-color: transparent;
    border-right-color: transparent;
    border-style: solid;
    border-width: 2px;
    border-radius: 50%;  
    box-sizing: border-box;
    display: inline-block;
    vertical-align: middle;
  }
  .spinner-large {
    width: 5rem;
    height: 5rem;
    border-width: 6px;
  }
  @keyframes spinner {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  .customSvgColour {
    fill: var(--spectrum-global-color-gray-900);
  }
</style>
