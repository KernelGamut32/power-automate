# Lab 06 - Build a Scheduled Flow That Summarizes List Items

- Create a new SharePoint site using a name of your choice
- Use "Team site" and the "Standard team" template
- Create a new SharePoint list called "Orders"
- Add the following columns to the "Orders" list:
  - Customer Name (Single line of text)
  - Owner Name (Single line of text)
  - Amount (Number)
  - Order Status (Choice: New, Approved, Declined)
  - Use "New" as the default for Order Status
- Create a new "Scheduled cloud flow"
- Use a trigger that executes the flow once per day
- Add a "Get items" action that retrieves all orders from the Orders list where the Order Status is New or Approved and where the Owner Name equals your name
- Use a "Select" data operation to project the data to a new format with different names for the target columns you're interested in pulling
- Use a "Create HTML table" data operation to generate an HTML table from the selected results
- Add a step that sends an email with the generated HTML table as the message body
- Create a few items in the list with yourself as the owner (using different statuses)
- Test the scheduled flow and confirm that the expected data is sent to your email address
