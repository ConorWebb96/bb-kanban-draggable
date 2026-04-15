# Bb-kanban-draggable
This is the read me for the Kanban draggable plugin.

> :warning: **Warning:** As of the latest update `1.3.1` it will break any existing versions of the plugin if you update. The relationship table aspect has been removed and its been changed to use Budibase single-option field instead for statuses.

# Description
With this plugin you are able to create and delete tickets, create and delete columns, and manage ticket states using a single select field in the same table.

## Features
* Creation of new columns
* Drag cards between columns, quickly progress them through the pipeline
* Filtering of the cards within the columns. Filtering works off Title and Description
* Card creation with a single click
* Deleting columns, creates a Backlog row automatically and moves all cards into it.
* Specific card deleting on a single click.
* Total number of columns 
* Total number of cards within each column visible
* Reordering functionality of the columns.

## Instructions
* Create a table with the following fields.
    * **Title** make sure the toggle `use as table display column` is on
    * **Description**
    * **Status** - Single Select field (this powers the Kanban columns)
    * **Image** - Optional field not required

### Additional Notes
* Column headings come from the configured single select field options.
* Column creation, deletion, and reordering updates that options list for you.

## Demo
![Kanban Demo](https://user-images.githubusercontent.com/126772285/234514184-a4913b9e-4539-4a1a-a8a0-a659906d7260.gif)

## Disclaimer
* Live loading when interacting with other Budibase components hasn't been built in. This means that sidepanel/formblock amends will need a page refresh in order to see the changes.

## App export
* An example app export with everything setup can be found here;
[kanban-export-1691144265536.tar.gz](https://github.com/ConorWebb96/bb-kanban-draggable/files/12259729/kanban-export-1691144265536.tar.gz)
