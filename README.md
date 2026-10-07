# WeedCrawler docs

Public reference documentation for WeedCrawler data products. Written for data consumers and for the AI assistants they work with.

## What is here

| Document | What it covers |
|---|---|
| [BBFYB data feed, data dictionary](bbfyb-gold/data-dictionary.md) | The GOLD star schema delivered in Snowflake and Microsoft Fabric: every table and column, coverage by province, freshness and revision rules, conventions, worked SQL examples, FAQ and changelog. |
| [Connecting Microsoft Fabric to the feed](bbfyb-gold/fabric-client-guide.md) | Step by step setup of the OneLake shortcuts to the Iceberg tables: lakehouse, connection, one shortcut per table folder, verification, what to expect from the feed, troubleshooting. |

An [`llms.txt`](llms.txt) at the root lists the same documents in the format AI tools look for.

## Pointing Claude (or another assistant) at it

Raw Markdown URL of the data dictionary:

```
https://raw.githubusercontent.com/Weed-Crawler/docs/main/bbfyb-gold/data-dictionary.md
```

- **Claude Code**: add the line below to your project's `CLAUDE.md`, or paste the URL in the prompt. Claude fetches it when it needs the schema.

  ```
  The BBFYB data dictionary is at https://raw.githubusercontent.com/Weed-Crawler/docs/main/bbfyb-gold/data-dictionary.md. Read it before writing SQL against the GOLD schema.
  ```

- **claude.ai**: download the file and add it to a Project's knowledge, or paste the URL in the conversation.
- **Any tool that reads a repo**: clone this repository. It only contains documentation.

## Keeping up to date

These files are generated from WeedCrawler's private repositories and republished whenever the source changes. The version and changelog are at the top and bottom of each document. Do not edit here: changes would be overwritten on the next publish.

## About WeedCrawler and BBFYB

[WeedCrawler](https://weedcrawler.ca) observes Canadian recreational cannabis retail every day: store menus, prices, inventory levels and sales movements across thousands of stores. [Best Bang For Your Bud](https://bbfyb.com) (BBFYB) is the consumer-facing side of the same data. The GOLD feed is the analysis-ready version licensed producers and analysts query directly.

Questions: hi@weedcrawler.ca
