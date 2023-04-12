# Bb-kanban-draggable
This is the read me for the Kanban draggable plugin.

# Description
With this plugin you are able to create and delete tickets, Create and delete columns and manage ticket states.

## Instructions
* Create a table with the following fields.
    * Title, Description, Image - Optional field not required

* Create an additional table with the following fields
    * Title and a relationship link between the Card/Ticket and the State field - (one State to many Cards/Tickets.)

    ### Additional Notes
    * There is ordering functionality for the columns.
        * You will need to add an Order column for the State table, this column should be a Number field.
        * Make sure that these are incremented by 1 as the inbuilt functionality works in this way.