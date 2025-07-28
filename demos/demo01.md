# Demo 01 - Text Sentiment & Summarization Using AI

Demonstration of capabilities in Power Automate using AI Builder to assess sentiment of text input along with text summarization.

- Login to <https://teams.microsoft.com/v2/>
- Create a new team called "Positive Feedback"; add a channel called "General"
- Create a new team called "Detractor Feedback"; add a channel called "General"
- Login to <https://make.powerautomate.com>
- Select "Create" and choose "Automated cloud flow"
- Provide a name for the flow (e.g., "Process Feedback")
- For trigger, search for and select "When a new email arrives (V3)"
- Click "Create"
- Edit the "When a new email arrives (V3)" step to specify "Feedback" for the "Subject filter"
- Specify "Inbox" for the Folder
- Click "+ New step" and search for "AI Builder"
- Choose the "Analyze sentiment" action
- If prompted, provide a name for "Connection", choose "Oauth" for "Authentication Type", and click "Sign In"
- Choose the language and for "Text", use Dynamic Content, referencing the body from the "When a new email arrives" step
- For summarization, you can use an account created at <https://platform.openai.com/docs/overview>
- Click "+ New step" and search for "HTTP"
- Set properties as follows:

  - Set URI to <https://api.openai.com/v1/chat/completions>
  - Select POST or method
  - For headers, create a header with the key "Authorization" and a value of "Bearer [API key from Open API]"
  - Add a header using a key of "Content-Type" and a value of "application/json"
  - For body, use:

```json
{
  "model": "gpt-3.5-turbo",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that summarizes text."
    },
    {
      "role": "user",
      "content": "Summarize the following: [input text]"
    }
  ],
  "temperature": 0.5
}
```

- For [input text], use Dynamic Content, referencing the body from the "When a new email arrives" step
- Click "+ New step" and search for "Compose"; select it
- For "Inputs" use something like:

```text
Sentiment: [Overall text sentiment from previous step]

Summary: body('HTTP')['choices'][0]['message']['content']
```

- Click "+ New step" and search for "Condition"; select it
- Use the "Overall text sentiment" checking for equal to "positive"
- In the "True" branch, click "+ New step" and search for/select "Post message in a chat or channel"
- If required, sign in
- Use "Channel" for "Post in"
- For "Team", select "Positive Feedback"
- For "Channel", select "General"
- For "Message", use the "Outputs" from the previous "Compose" step
- For the "False" branch, click "+ New step" and search for/select "Post message in a chat or channel"
- If required, sign in
- Use "Channel" for "Post in"
- For "Team", select "Detractor Feedback"
- For "Channel", select "General"
- For "Message", use the "Outputs" from the previous "Compose" step
- Click "Save" and then test it by sending an email with an overtly positive body
- Also, do a couple of negative tests - send an email with an overtly negative body and and email with a neutral body
- Verify that expected results are posted to channels in the correct teams
- Review the associated results of each
