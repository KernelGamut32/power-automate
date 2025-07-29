# Lab 08 - Build an Approval Flow for a SharePoint List

- Create a new SharePoint site using a name of your choice
- Use "Team site" and the "Standard team" template
- Create a new SharePoint list called "Orders
- Add the following columns to the "Orders" list:
  - Customer Name (Single line of text)
  - Owner Name (Single line of text)
  - Amount (Number)
  - Order Status (Choice: New, Approved, Declined)
  - Use "New" as the default for Order Status
- Create a new "Automated cloud flow"
- Use a trigger for when a new item gets added to the SharePoint list where Order Status is "New" and Owner Name is your name
- Add an approval step (Approve/Reject - First to respond) that routes the approval request to your email address
- On approval, update the status on the SharePoint list item to "Approved"
- On rejection, update the status on the SharePoint list item to "Declined"
- Test by creating a new order item in the list with your name as the owner; approve it, and observe results
- Also test by creating a new order item in the list with you name as the owner; reject it, and observe results
