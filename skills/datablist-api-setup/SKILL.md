---
name: datablist-api-setup
description: Set up the Datablist Enrichments API or Sources API for a user and run any Datablist enrichment or source, on their computer or in Make, n8n or Zapier. Use when the user asks to set up, install or run the Datablist API, a Datablist enrichment or a Datablist source.
---

# Set up the Datablist Enrichments API

Instructions for an AI assistant (Claude, ChatGPT, Cursor or similar) that helps a user set up and run any Datablist enrichment through the Enrichments API. The user is usually not a developer. Do the work yourself and tell them in one short line what you are doing. If you cannot run commands on their computer, give them one copy-paste step at a time and wait for their reply before the next one.

The user only does two things by hand: copy their API key from the Datablist app, and paste it where you tell them. Everything else is your job.

If the user asked for Make, n8n or Zapier, skip to "Set it up in Make, n8n or Zapier" at the end.

Enrichments add data to rows the user already has (a list of names, websites, profiles). Sources create new records from settings alone, without input rows: Instagram followers, companies using a technology, Google search results, the URLs of a sitemap. If the user named a source (for example the Instagram Followers Scraper, Find Companies by Technologies, Bulk Google Search or the Sitemap Scraper), or wants to build a new list rather than enrich one, do steps 1 and 2, then follow "Run a source" instead of steps 3 and 4.

Full API reference: https://platform.datablist.com/docs/enrichments/quickstart (enrichments) and https://platform.datablist.com/docs/sources/quickstart (sources)

## 1. Create the Datablist folder and the .env file

1. Create a folder named `Datablist` in the user's home folder (`~/Datablist`, on Windows `%USERPROFILE%\Datablist`).
2. In it, create a file named `.env` with exactly one line: `DATABLIST_API_KEY=`
3. Open `.env` in the user's text editor (macOS `open -e .env`, Windows `notepad .env`, Linux `xdg-open .env`), or tell them its full path.
4. Ask the user to do this, in these words or very close to them. The part about the `=` sign matters most.

   > 1. Log in to Datablist and open https://app.datablist.com/account/api
   > 2. Create or copy your Personal API Key.
   > 3. In the `.env` file I opened, click right after the `=` sign and paste your key on the same line. No space, no quotes. It should look like `DATABLIST_API_KEY=abc123yourkey`
   > 4. Save the file and reply "done" here.

5. Tell them clearly: never paste the API key into this chat. If they do anyway, do not repeat it, and tell them to replace the key at the link above.

API access needs the Datablist Growth plan or higher. Enrichments use credits from the user's Datablist account.

## 2. Check the key without showing it

Never print, log, echo or commit the key. Only check that it is set, for example:

```sh
grep -Eq '^DATABLIST_API_KEY=[^[:space:]"]+$' ~/Datablist/.env && echo "Key found" || echo "Key missing or wrong format"
```

The usual mistakes: the key on a new line instead of after the `=`, a space after the `=`, quotes around the key, or the file not saved. Tell the user which one it is and ask them to fix and save. You may fix the format yourself as long as you never print the key.

Load the key into the shell before each run:

- macOS and Linux: `set -a; . ~/Datablist/.env; set +a`
- Windows PowerShell: `Get-Content $HOME\Datablist\.env | ForEach-Object { $k, $v = $_ -split '=', 2; if ($k) { Set-Item "env:$k" $v } }`

## 3. Pick the enrichment

1. List the enrichments. This needs no key: `curl https://api.datablist.com/v1/enrichments`
2. Show the user each enrichment's name and description, and ask which one they need. If they came from a page about one enrichment (for example Company Name to Domain), suggest that one.
3. Fetch its schema: `curl https://api.datablist.com/v1/enrichments/ENRICHMENT_ID`. It lists the settings, the input fields (`inputSchema`, with the required ones), the outputs and the cost.
4. Ask the user for their data: a short list typed in the chat, or the path of a CSV file and which column holds each required input. Tell them the run uses credits before you start.

## 4. Run it

Every call goes to `https://api.datablist.com` with the header `X-API-Key: $DATABLIST_API_KEY`. Write a small script in `~/Datablist` (Python or shell, whatever runs on their computer) that does this:

1. Create a task: `POST /v1/enrichments/tasks` with a JSON body

   ```json
   {
     "enrichmentId": "ENRICHMENT_ID",
     "uniqueId": "a-new-unique-id-per-run",
     "settings": { },
     "inputs": [ { "inputId": "row-1", "data": { "REQUIRED_FIELD": "value" } } ]
   }
   ```

   `settings` and the fields in `data` come from the schema. Send 1 to 10,000 inputs per task, with your own `inputId` per row. The answer is `202 Accepted` with the task `@id`.
2. Wait: call `GET /v1/enrichments/tasks/{taskId}` every few seconds until the status is `completed`, `failed` or `stopped`.
3. Read the results: `GET /v1/enrichments/tasks/{taskId}/results?limit=100`, and follow `nextCursor` until it is null. Each result has the `inputId`, a `status`, the `outputs`, `creditsUsed` and an `error` field.

Do not resend the same POST after a timeout: look up the uniqueId in `GET /v1/enrichments/tasks` first. Results are kept for 90 days.

## 5. Show the result

Show a table with the input and the main outputs. Point out rows without a result or with a low confidence, if the enrichment returns one. For a CSV, write a new file next to the original with the output columns added, never overwrite their file, and give them its full path.

If something fails, read the error, fix what you can (key not loaded, wrong field name, missing required input) and explain the rest in plain words.

## Run a source

1. List the sources. This needs no key: `curl https://api.datablist.com/v1/sources`. Fetch the chosen one: `curl https://api.datablist.com/v1/sources/SOURCE_ID`. It lists the settings (`settingsSchema`, with allowed values), the outputs and the cost. There is no input schema.
2. Ask the user for the settings in plain words (which profiles, which technology and country, which searches, which sitemap) and turn them into the JSON the schema expects. Lists such as profiles or search queries are JSON arrays, one element per item. Tell them the run uses credits and give an estimate from the cost before you start.
3. Create a task: `POST /v1/sources/tasks` with

   ```json
   {
     "sourceId": "SOURCE_ID",
     "uniqueId": "a-new-unique-id-per-run",
     "settings": { }
   }
   ```

   The answer is `202 Accepted` with the task `@id`. Add `"maxCredits": "500"` (a string) for a best-effort credit budget if the user wants one.
4. Wait: call `GET /v1/sources/tasks/{taskId}` every few seconds until the status is `completed`, `failed` or `stopped`. Failed and stopped tasks can still have results.
5. Read the results: `GET /v1/sources/tasks/{taskId}/results?limit=100`, following `nextCursor` until it is null. Each result is one flat JSON object with the output fields, not wrapped in `inputId` or `outputs`. The task shows `credits.used`.
6. Save the results as a CSV in `~/Datablist` and show the first rows in a table, with the full path of the file.

Do not resend the same POST after a timeout: look up the uniqueId in `GET /v1/sources/tasks` first. Sources and enrichments share about 20 active tasks per account.

## Set it up in Make, n8n or Zapier

The user wants an automation, not a script on their computer. Guide them step by step and tell them exactly what to put in every field.

1. Ask them to store their API key from https://app.datablist.com/account/api in the tool's credentials (a Make connection, an n8n Header Auth credential, or the Zap's connection settings) as the header `X-API-Key`. Never ask for the key in the chat.
2. Ask which enrichment or source they need. For an enrichment, ask which of their fields to send, list the enrichments from `https://api.datablist.com/v1/enrichments` and read the schema as in step 3 above. For a source, list `https://api.datablist.com/v1/sources` and read the settings as in "Run a source".
3. Build three HTTP steps: create the task (`POST https://api.datablist.com/v1/enrichments/tasks` with the JSON body from step 4, or `POST https://api.datablist.com/v1/sources/tasks` with the body from "Run a source"), wait and check the task until it is `completed`, then read the results (`GET .../results?limit=100`).
4. Map each result's `outputs` to their sheet, CRM or next step, matched by `inputId`. Source results have no `inputId`: map each result object to a new row.
