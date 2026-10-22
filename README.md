# Datablist API setup

Set up the [Datablist Enrichments API](https://platform.datablist.com/docs/enrichments/quickstart) with an AI assistant. Paste this into Claude, ChatGPT or Cursor:

> Set up the Datablist API for me with this skill: https://github.com/b1b1h-m1k3r/datablist-api-setup/blob/main/skills/datablist-api-setup/SKILL.md

The assistant creates a `Datablist` folder with a `.env` file, asks you to paste your API key from https://app.datablist.com/account/api into it (never into the chat), lets you pick an enrichment from the catalog and runs it on your data. It also works for Make, n8n and Zapier.

The instructions for the assistant are in [`skills/datablist-api-setup/SKILL.md`](skills/datablist-api-setup/SKILL.md).
