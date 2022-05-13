# Notion Page Tree
> Fetch nested Notion pages from the root page/database/block.

---

- [Notion Page Tree](#notion-page-tree)
	- [Why Would I Want This?](#why-would-i-want-this)
		- [🙌 Use the official Notion API.](#-use-the-official-notion-api)
		- [🕸 Fetch nested children pages.](#-fetch-nested-children-pages)
		- [🛠 Handle API Errors gracefully.](#-handle-api-errors-gracefully)
	- [Other Features](#other-features)
		- [💾 It saves fetch results to your local disk.](#-it-saves-fetch-results-to-your-local-disk)
		- [🥞 It builds basic page server.](#-it-builds-basic-page-server)
		- [🔎 It builds basic page search indexes.](#-it-builds-basic-page-search-indexes)
	- [Usage](#usage)
		- [`.env` File Configuration](#env-file-configuration)
		- [Create Instance](#create-instance)
		- [Fetcher and Server Setup (sample)](#fetcher-and-server-setup-sample)
	- [How It Works (Flowchart)](#how-it-works-flowchart)

---

## Why Would I Want This?
### 🙌 Use the official Notion API.
- Popular `/loadpagechunk/` endpoint is not public and may not be stable in future updates.
- Official API can be integrated with private key, so you can keep your database private.

### 🕸 Fetch nested children pages.
- Pages inside non-page blocks are also fetched.
- Max search-depth can be set in your preference.

### 🛠 Handle API Errors gracefully.
- Maximum fetch concurrency is set to avoid `rate_limited` error.
- On `rate_limited` error, it stops and waits for some minutes.
- Other errors are automatically retried. Max retry count can be set in your preference.

---

## Other Features
### 💾 It saves fetch results to your local disk.
- Set parameter `private_file_path` to your custom path.

### 🥞 It builds basic page server.
- `/page/:id/` endpoint for retrieving page and its childrens' id.
- `/tree/:id/` endpoint for retrieving all nested pages from the page.
  
### 🔎 It builds basic page search indexes.
- Uses lunr.js.
- Page's properties and chilren are converted into plain text for building search index.
- `/search?keyword=` endpoint for searching page properties and retrieving page ids.
- `/suggestion?keyword=` endpoint for looking for search index's tokens.

---

## Usage
### `.env` File Configuration

Write directly on `<package_root>/.env`
```text
NOTION_ENTRY_ID = <root page/database/block's id>
NOTION_ENTRY_KEY = <root's integration key>
NOTION_ENTRY_TYPE = <page/database/block>
```

### Create Instance
Interactively Ask for parameters.
```js
const notionPageTree = new NotionPageTree({
	fetchIntervalMinutes: 5,
	private_file_path: path.resolve('./sample/')
});
```

### Fetcher and Server Setup (sample)
  
Run `yarn serve` to see fetcher running in action.

Sample database is here https://notion.so/2345a2ce3cdd48f183cca1f6d1ae25ca

`./sample/index.ts`
```js
import NotionPageTree from 'notion-page-tree';
import path from 'path';

(async function main() {
	// Create main class instance.
	const notionPageTree = new NotionPageTree({
		private_file_path: path.resolve('./sample/'),
		createFetchQueueOptions: {
			maxConcurrency: 3, // Current official rate limit is 3 requests per second. Notion api will throw error when you increase this value.
			maxRetry: 2, // "rate_limited" error will not be retried and process will be exited immediately.
			maxRequestDepth: 3, // Depth applied to all the entities.
			maxBlockDepth: 2, // Depth applied to blocks (relative to nearest parent page).
			databaseQueryFilter: {
				// Database query filter.
				property: 'isPublished',
				checkbox: {
					equals: true
				}
			}
		}
	});

	notionPageTree.parseCachedDocument();
	// Look for cached documents in `private_file_path`, assign to Page Data Variables.

	const requestParameters = await notionPageTree.setRequestParameters({
		forceRewrite: false,
		prompt: true
	});
	console.log(
		'Requesting',
		requestParameters.entry_type,
		'with id',
		requestParameters.entry_id
	);
	// Set environment variables that are needed for Notion API.

	const server = notionPageTree.setupServer({ port: 8888 });
	// Setup servers for listing and searching pages. (will respond 503 if pages are undefined)

	// await notionPageTree.fetchOnce();
	// Fetch pages asynchronously, then assign results to variables.

	notionPageTree.startFetchLoop(1000 * 10); // 10 seconds
	// Create asynchronouse fetch loop, which waits for some milliseconds between each fetch.

	setTimeout(() => {
		// after 30 seconds
		console.log('stopping fetch loop');
		notionPageTree.stopFetchLoop();
		// Stopping fetch loop after current fetch resolves.
		console.log('closing servers');
		server.close();
		// Stopping servers immediately.
	}, 1000 * 30);
})();

```

## How It Works (Flowchart)
<div style="overflow-x: scroll !important; white-space: pre-wrap !important;">
<pre style="display: inline-block;">
<code>/******************************************************************************************************************************************************************************************************************************************************************************************************************************\
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
*    ┏━━━━━━━Main Routine━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                                                                                                                                                *
*    ┃                                                                                                                                                                        ┃                                                                                                                                                *
*    ┃                                                                                                                                                                        ┃                                                                                                                                                *
*    ┃                                          ┌───────────────────────┐                                                           ┌───────────────────────┐                 ┃                                                                                                                                                *
*    ┃                                     ┌───▶│    page_collection    │                                        ┌clearTimeout()───▶│ Request Promise Timer │                 ┃                                                                                                                                                *
*    ┃                                     │    └───────────────────────┘                                        │                  └───────────────────────┘                 ┃                                                                                                                                                *
*    ┃                                     │                                                  Λ                  │                                                            ┃                                                                                                                                                *
*    ┃                                     │    ┌───────────────────────┐                    ╱ ╲                 │                  ┌───────────────────────┐                 ┃                                                                                                                                                *
*    ┃                                     ├───▶│       page_tree       │                   ╱   ╲                ├clearTimeout()───▶│  Request Ready Timer  │                 ┃                                                                                                                                                *
*    ┃                                     │    └───────────────────────┘                  ╱     ╲               │                  └───────────────────────┘                 ┃                                                                                                                                                *
*    ┃                                     │                                              ╱       ╲              │                                                            ┃                                                                                                                                                *
*    ┃                                     │    ┌───────────────────────┐                ╱  check  ╲             │                  ┌─────────────────┐                       ┃                                                                                                                                                *
*    ┃                                     ├───▶│ Request Promise Queue │──────┐        ╱  routine  ╲            │   ┌───update ───▶│ page_collection │                       ┃                                                                                                                                                *
*    ┃        ┌───────────────────────┐    │    └───────────────────────┘      │       ╱      ↻      ╲           │   │              └─────────────────┘                       ┃                                                                                                                                                *
*    ┃        │    Fetcher Routine    │────┤                                   ├─────▶▕  promise = 0  ▏──┬─true──┼───┤                                                        ┃                                                                                                                                                *
*    ┃        └───────────────────────┘    │    ┌───────────────────────┐      │       ╲  ready = 0  ╱   │       │   │              ┌─────────────────┐                       ┃                                                                                                                                                *
*    ┃                    ▲                ├───▶│  Request Ready Queue  │──────┘        ╲     ↻     ╱    │       │   └───update────▶│    page_tree    │                       ┃                                                                                                                                                *
*    ┃                    │                │    └───────────────────────┘                ╲   0ms   ╱     │       │                  └─────────────────┘                       ┃                                                                                                                                                *
*    ┃                    │                │                                              ╲       ╱      │       │                                                            ┃                                                                                                                                                *
*    ┃                    │                │    ┌───────────────────────┐                  ╲     ╱       │       │                                                            ┃                                                                                                                                                *
*    ┃                    │                ├───▶│ Request Promise Timer │                   ╲   ╱        │       └──────────────────▶ wait for some minutes ─ ─ ┐             ┃                                                                                                                                                *
*    ┃                    │                │    └───────────────────────┘                    ╲ ╱         │                                                                    ┃                                                                                                                                                *
*    ┃                    │                │                                                  V          │                                                      │             ┃                                                                                                                                                *
*    ┃                    │                │    ┌───────────────────────┐                     ▲          │                                                                    ┃                                                                                                                                                *
*    ┃                    │                └───▶│  Request Ready Timer  │                     └──false───┘                                                      │             ┃                                                                                                                                                *
*    ┃                    │                     └───────────────────────┘                                                                                                     ┃                                                                                                                                                *
*    ┃                    │                                                                                                                                     │             ┃                                                                                                                                                *
*    ┃                    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  create new fetcher routine ◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─              ┃                                                                                                                                                *
*    ┃                                                                                                                                                                        ┃                                                                                                                                                *
*    ┗━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛                                                                                                                                                *
*                                                                                                                                                                                                                                                                                                                              *
*                         │                                                                                                                                                                                                                                                                                                    *
*                                                                                                                                                                                                                                                                                                                              *
*                         │                                                                                                                                                                                                                                                                                                    *
*                                                                                                                                                                                                                                                                                                                              *
*     ┏━━━Fetcher Routine━┻━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    *
*     ┃                                                                                                                                                                                                  ┌─.plaintext += plaintext────────────────────────────────────────────────────┐                                   ┃    *
*     ┃                                                                                                                                                                                                  │                                                                            │                                   ┃    *
*     ┃                                                                                                                                                                     ╔══════════════════════════╗ │                                      ╔══════════════════════╗              │                                   ┃    *
*     ┃                                                                                                                       ┌──false───┐                                  ║  Promise (resolved)      ║ ├─.children.push()┐                    ║Connector             ║              │                                   ┃    *
*     ┃                                                                                    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓ │          │                                  ║┌───────────────────────┐ ║ │                 │                    ║ ┌──────────────────┐ ║              │                                   ┃    *
*     ┃                                                                                    ┃   Request Promise Queue        ┃ │          │                                  ║│parentToAssign: Entity │◀──┘       ┌──────────────────┐           ║ │toAssigned: itself│ ║              │                                   ┃    *
*     ┃                                                                                    ┃ ╔══════════════════════════╗   ┃ │          Λ                                  ║└───────────────────────┘ ║   ┌────▶│ is Page/Database │──create──▶║ └──────────────────┘ ║──────────────│─────────────────┐                 ┃    *
*     ┃                                                                                    ┃ ║  Promise (pending)       ║   ┃ │         ╱ ╲                                 ║┌───────────────────────┐ ║   │     └──────────────────┘           ║┌────────────────────┐║              │                 │                 ┃    *
*     ┃       ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                                          ┃ ║ ┌───────────────────────┐║   ┃ │        ╱   ╲                                ║│parentToRequest: Entity│ ║   │                      │             ║│toRequested: itself │║              │                 │                 ┃    *
*     ┃       ┃  New Requests                 ┌──────────────────────────────────────────────╬▶│parentToAssign: Entity │║   ┃ │       ╱     ╲            ┌──────────────┐   ║└───────────────────────┘ ║   │                      │             ║└────────────────────┘║              │                 │                 ┃    *
*     ┃       ┃ ╔═════════════════════════╗   │ ┃ ┌──────────────────────────────────────┐ ┃ ║ └───────────────────────┘║...┃ │      ╱       ╲       ┌──▶│ is Fulfilled │──▶║┌───────────────────────┐ ║   │                      │             ╚══════════════════════╝              │                 │                 ┃    *
*     ┃       ┃ ║┌───────────────────────┐║   │ ┃ │                                      │ ┃ ║ ┌───────────────────────┐║   ┃ │     ╱  set    ╲      │   └──────────────┘   ║│       children:       │ ║   │                      │                                                   │                 │                 ┃    *
*     ┃       ┃ ║│parentToAssign: Entity │╠───┘ ┃ │   ┌─────────────┐    List Block      └─╋─╬▶│parentToRequest: Entity│║   ┃ │    ╱ interval  ╲     │                      ║│   QueryablePromise    │─╬───┤                      │                                         ┌──────────────────┐        │                 ┃    *
*     ┃       ┃ ║└───────────────────────┘║     ┃ ├──▶│is Block/Page│───▶Children()─────┐  ┃ ║ └───────────────────────┘║   ┃ │   ╱      ↻      ╲    │                      ║│       Entity[]        │ ║   │                      │       extractPlainText                  │   Concatenated   │        │                 ┃    *
*     ┃       ┃ ║┌───────────────────────┐║ ... ┃ │   └─────────────┘                   │  ┃ ║ ┌───────────────────────┐║   ┃─┴─▶▕  promise > 0  ▏───┤                      ║└───────────────────────┘ ║   │                      │    ┌─▶   FromBlock    ────▶.reduce()───▶│    Plain Text    │        │                 ┃    *
*     ┃       ┃ ║│parentToRequest: Entity│╠───────┘                                 .push()┃ ║ │       children:       │║   ┃     ╲ isSetteled  ╱    │   ┌──────────────┐   ║┌───────────────────────┐ ║   │                      │    │                                    └──────────────────┘        │                 ┃    *
*     ┃       ┃ ║└───────────────────────┘║     ┃ │   ┌─────────────┐      Query        ┌────╬▶│   QueryablePromise    │║   ┃      ╲     ↻     ╱     └──▶│ is Rejected  │   ║│     retry: number     │ ║   │                      │    │                                                                │                 ┃    *
*     ┃       ┃ ║┌───────────────────────┐║     ┃ └──▶│ is Database │───▶Database()─────┘  ┃ ║ │       Entity[]        │║   ┃       ╲   0ms   ╱          └──────────────┘   ║└───────────────────────┘ ║   │     ┌────────────────│─┐  │                                                                │                 ┃    *
*     ┃       ┃ ║│     retry: number     │║     ┃     └─────────────┘                      ┃ ║ └───────────────────────┘║   ┃        ╲       ╱                    │         ╚══════════════════════════╝   └────▶│     is Block   │ │──┤                     ┌──────────────────┐                       │                 ┃    *
*     ┃       ┃ ║└───────────────────────┘║     ┃                                          ┃ ║ ┌───────────────────────┐║   ┃         ╲     ╱                     │                                              └────────────────│─┘  │                     │  is in Traverse  │                       │                 ┃    *
*     ┃       ┃ ╚═════════════════════════╝     ┃                                          ┃ ║ │     retry: number     │║   ┃          ╲   ╱                      │               set maxConcurrency to 0                         │    │               ┌────▶│  Exclusion List  │           ╔═══════════╬══════════╗      ┃    *
*     ┃       ┗━━━━━━━━━━━━━━▲━━━━━━━━━━━━━━━━━━┛                                          ┃ ║ └───────────────────────┘║   ┃           ╲ ╱                       ▼               empty promise_queue                             │    │               │     └──────────────────┘           ║Connector  │          ║      ┃    *
*     ┃                      │                                                             ┃ ╚══════════════════════════╝   ┃            V                  rate_limited──true──▶ move promises to ready_queue                    │    └─▶.filter()────┤     ┌──────────────────┐           ║ ┌─────────┴────────┐ ║      ┃    *
*     ┃                      │                                                             ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛                                     │               wait for some minutes                           │                    │     │is NOT in Traverse│           ║ │toAssigned: PARENT│ ║      ┃    *
*     ┃                      │                                                                                                                                    │               set maxConcurrency to 3                         │                    └────▶│  Exclusion List  │──create──▶║ └─────────┬────────┘ ║      ┃    *
*     ┃                      │                                                                                      ╔═════════════════════════╗                 false                                                             │                          └──────────────────┘           ║┌──────────┴─────────┐║      ┃    *
*     ┃                      │                                                                                      ║┌───────────────────────┐║                   │                                                               │                                                         ║│toRequested: itself │║      ┃    *
*     ┃            .splice(concurrency -                                                                            ║│parentToAssign: Entity │║                   │                                                               │                                                         ║└──────────┬─────────┘║      ┃    *
*     ┃               promise.length)                                                                               ║└───────────────────────┘║                   ▼                                                               │                                                         ╚═══════════╬══════════╝      ┃    *
*     ┃                      │                                                                                      ║┌───────────────────────┐║               retry <                                                             │                                                                     │                 ┃    *
*     ┃                      │                                                                               ┌──────║│parentToRequest: Entity│║◀───true───── retryCount ──false──▶ console.error                                  │                                                                     │                 ┃    *
*     ┃                      │                                                                               │      ║└───────────────────────┘║                                                                                   │                                                                     │                 ┃    *
*     ┃                      │                                                                               │      ║┌───────────────────────┐║                                                                                   │                                                                     │                 ┃    *
*     ┃                      │                         ┌──false───┐                                          │      ║│      retry: +=1       │║                                                                                   │                                                                     │                 ┃    *
*     ┃                      │                         │          │                                          │      ║└───────────────────────┘║                                                                                   │                                                                     │                 ┃    *
*     ┃                      │                         │          │                                     .unshift()  ╚═════════════════════════╝                                                                                   │                                                                     │                 ┃    *
*     ┃                      │                         Λ          │                                          ▼                                                                                                                    │                                                                     │                 ┃    *
*     ┃                      │                        ╱ ╲         │            ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                       ╱   ╲        │            ┃  Request Ready Queue                                      ┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                      ╱     ╲       │            ┃ ╔═════════════════════════╗ ╔═════════════════════════╗   ┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                     ╱       ╲      │            ┃ ║┌───────────────────────┐║ ║┌───────────────────────┐║   ┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                    ╱  check  ╲     │            ┃ ║│parentToAssign: Entity │║ ║│parentToAssign: Entity │║   ┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                   ╱  routine  ╲    │            ┃ ║└───────────────────────┘║ ║└───────────────────────┘║   ┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      │                  ╱      ↻      ╲   │            ┃ ║┌───────────────────────┐║ ║┌───────────────────────┐║...┃                                                                                      │                                                                     │                 ┃    *
*     ┃                      └──────true───────▕  ready > 0    ▏◀─┴────────────┃ ║│parentToRequest: Entity│║ ║│parentToRequest: Entity│║   ┃◀────────────.push()──────────────────────────────────────────────────────────────────│─────────────────────────────────────────────────────────────────────┘                 ┃    *
*     ┃                                         ╲ promise < 3 ╱                ┃ ║└───────────────────────┘║ ║└───────────────────────┘║   ┃                                                                                      │                                                                                       ┃    *
*     ┃                                          ╲           ╱                 ┃ ║┌───────────────────────┐║ ║┌───────────────────────┐║   ┃                                                                                      │                                                                                       ┃    *
*     ┃                                           ╲    ↻    ╱                  ┃ ║│       retry: 0        │║ ║│       retry: 0        │║   ┃                                                                                      │                                                                                       ┃    *
*     ┃                                            ╲ 250ms ╱                   ┃ ║└───────────────────────┘║ ║└───────────────────────┘║   ┃                                                                                      │                                                                                       ┃    *
*     ┃                                             ╲     ╱                    ┃ ╚═════════════════════════╝ ╚═════════════════════════╝   ┃                                                                                      │                                                                                       ┃    *
*     ┃                                              ╲   ╱                     ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛                                                                                      │                                                                                       ┃    *
*     ┃                                               ╲ ╱                                                    ▲                                                                                                                    │                                                                                       ┃    *
*     ┃                                                V                                                     │                    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━┓                                                                    │                                                                                       ┃    *
*     ┃                                                                                                   (init)                  ┃  Page Collection         ┃                                                                    │                                                                                       ┃    *
*     ┃                                                                                                      │                    ┃ ┌────╦═════════════════╗ ┃                                                                    │                                                                                       ┃    *
*     ┃                                                                                                      │                    ┃ │ id ║  page: Entity   ║ ┃                                                                    │                                                                                       ┃    *
*     ┃                                                                                                      │                    ┃ └────╩═════════════════╝ ┃                                                                    │                                                                                       ┃    *
*     ┃                                                                                            ┌──────────────────┐           ┃ ┌────╦═════════════════╗ ┃                                                                    │                                                                                       ┃    *
*     ┃                                                                                            │       ROOT       │───(init)─▶┃ │ id ║  page: Entity   ║ ┃◀────────────assign with key────────────────────────────────────────┘                                                                                       ┃    *
*     ┃                                                                                            └──────────────────┘           ┃ └────╩═════════════════╝ ┃                                                                                                                                                            ┃    *
*     ┃                                                                                                      │                    ┃ ┌────╦═════════════════╗ ┃                                                                                                                                                            ┃    *
*     ┃                                                                                                                           ┃ │ id ║  page: Entity   ║ ┃                                                                                                                                                            ┃    *
*     ┃                                                                                                      │                    ┃ └────╩═════════════════╝ ┃                                                                                                                                                            ┃    *
*     ┃                                                                                                                           ┃           ....           ┃                                                                                                                                                            ┃    *
*     ┃                                                                                                      │                    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━┛                                                                                                                                                            ┃    *
*     ┃                                                                                                                                         │                                                                                                                                                                         ┃    *
*     ┃                                                                                                      └ ─ ─ ─ ─ ─ ─ ─ ┬ ─ ─ ─ ─ ─ ─ ─ ─ ─                                                                                                                                                                          ┃    *
*     ┃                                                                                                                                                                                                                                                                                                                   ┃    *
*     ┃                                                                                                                      │                                                                                                                                                                                            ┃    *
*     ┃                                                                                                          ╔══════════════════════╗                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║ Entity               ║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│     id: string     │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│   depth: number    │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│       type:        │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│ 'page'|'database'| │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│      'block'       │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│     metadata:      │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│GetBlockResponseWith│║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│      Metadata      │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│     plainText:     │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│extractPlainText(...│║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│     children)      │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║┌────────────────────┐║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║│ children: Entity[] │║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ║└────────────────────┘║                                                                                                                                                                                 ┃    *
*     ┃                                                                                                          ╚══════════════════════╝                                                                                                                                                                                 ┃    *
*     ┃                                                                                                                                                                                                                                                                                                                   ┃    *
*     ┃                                                                                                                                                                                                                                                                                                                   ┃    *
*     ┃                                                                                                                                                                                                                                                                                                                   ┃    *
*     ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    *
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
*                                                                                                                                                                                                                                                                                                                              *
\******************************************************************************************************************************************************************************************************************************************************************************************************************************/</code></pre>
</div>