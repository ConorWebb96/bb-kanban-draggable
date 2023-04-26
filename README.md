# Bb-kanban-draggable
This is the read me for the Kanban draggable plugin.

# Description
With this plugin you are able to create and delete tickets, Create and delete columns and manage ticket states.

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
    * **Image** - Optional field not required

* Create an additional table with the following fields - this table is for the column headings and linking cards to their stage in the pipeline.
    * **Title** make sure the toggle `use as table display column` is on
    * Relationship link between the Card/Ticket and the State table - (one State to many Cards/Tickets.)

    ### Additional Notes
    * There is ordering functionality for the columns.
        * You will need to add an Order column for the State table, this column should be a Number field.
        * Make sure that these are incremented by 1 as the inbuilt functionality works in this way.
        * Optionally you can create the columns purely through the frontend and it will increment everything on its own.

## Demo
![Kanban Demo](https://user-images.githubusercontent.com/126772285/234514184-a4913b9e-4539-4a1a-a8a0-a659906d7260.gif)

## Disclaimer
* Live loading when interacting with other Budibase components hasn't been built in. This means that sidepanel/formblock amends will need a page refresh in order to see the changes.

## App export
* An example app export with everything setup can be found here;
[Kanban Board-export-1682492725334.tar.gz](https://github.com/ConorWebb96/bb-kanban-draggable/files/11329969/Kanban.Board-export-1682492725334.tar.gz)
