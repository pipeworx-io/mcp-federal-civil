# Federal civil litigation

Federal civil litigation in US district courts — everything not captioned `United States v. ...`. The largest categories are prisoner civil rights and habeas petitions, then civil rights employment, product liability and contract.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1866+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

No API key.

## What this is, and what it is not

**Docket metadata**: that a case exists, its parties, court, docket number,
filing and termination dates, and assigned judge. It contains **no filings** —
no complaints, motions, briefs or orders. Those are RECAP, a separate corpus
that is not keyless.

An agent with this data knows a case was brought. It does not know what anyone
argued. One narrow exception, below: the TEXT of a docket entry's description is
searchable without a token even though the documents themselves are not, and
that is what makes removal detectable.

## Removed cases: `removed_from`

A case removed from state to federal court is two records, and nothing on the
federal one says so. `njd 2:26-cv-05788` is NS Pharma's removal of New Jersey
Superior Court `BER-C-000117-26`, and its `cause` reads
`09:1 U.S. Arbitration Act` — the federal statute invoked, not where the case
came from. Both tools therefore look up the originating court by default and
return it on the row:

```json
"removed_from": {
  "state_court": "Superior Court of New Jersey, Chancery Division: Bergen County",
  "state_docket_number": "BER-C-000117-26",
  "notice_filed": "2026-05-20",
  "notice_url": "https://www.courtlistener.com/docket/73371682/1/capricor-therapeutics-inc-v-nippon-shinyaku-co-ltd/",
  "source": "Notice of Removal filed on the federal docket (CourtListener RECAP filing text)"
}
```

**Its absence is not a negative.** No `removed_from` means no Notice of Removal
was found, and the filing text comes from RECAP, which covers only dockets
somebody has purchased or uploaded. Every response that ran the lookup says so
in `removal_note`, and `removal_lookup` carries the counts — including how many
notices were found but carried no parseable court and number, which are
reported rather than emitted as half a field. A throttled lookup surfaces as
`removal_lookup.error`, so "could not ask" never reads as "not removed".

**The clause is often not there, and that is the real limit.** The lookup finds
the Notice of Removal reliably; whether that notice's text names the state court
is up to whoever docketed it. Measured live on 2026-10-11 across four dockets
carrying a notice, ONE wrote the parseable form (`... from Superior Court of New
Jersey, Chancery Division: Bergen County, case number BER-C-000117-26`) and
three did not — two read only `NOTICE OF REMOVAL by <party> (Filing fee ...)`
and one had no description at all. A small sample, so take it as "expect
misses", not as a rate. Those three are counted in
`removal_lookup.notices_unparsed` rather than having a court inferred from, say,
an attachment titled "PCCP Court Docket" — that inference is exactly the
plausible-but-fabricated link this field exists to avoid.

A structured originating-court field does exist on CourtListener's
`/api/rest/v4/dockets/` endpoint and would not have this limit, but it answers
HTTP 401: it needs a Free Law Project account token. See the pack header for
that trade-off.

Set `check_removal: false` to skip it and save one upstream request.

## Tools

| Tool | Returns |
|---|---|
| `federal_civil_search` | Dockets matching a party name |
| `federal_civil_recent` | Recent dockets, newest first |

## Caveats

- **Quarterly snapshot.** Every response carries `snapshot_date`.
- **`date_terminated` is 78.3% filled**, so a missing date means *not recorded*,
  never *still open*. Responses say which.
- **`nature_of_suit` is sparse** — 14.8% corpus-wide, 30.5% in district courts.
  This can tell you a company is in federal court; it often cannot tell you what
  for.
- Party search matches a substring of the case caption, so a distinctive party
  name works better than a full caption.
- **The removal lookup is capped at 20 dockets per call**, in chunks of 10,
  because the upstream filing search is throttled to 5 requests/minute and
  about 125/day for a token-less caller. When a response returns more rows than
  that, `removal_lookup.not_checked` says how many were left unexamined — treat
  those as unknown rather than as not removed.

## Attribution

Data is CourtListener / Free Law Project, published under the Public Domain
Mark. Free Law Project funds and maintains the corpus. If this pack is useful
to you, <https://free.law/donate/> is where that goes.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "federal-civil": {
      "url": "https://gateway.pipeworx.io/federal-civil/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/federal-civil/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1866+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/federal_civil_search \
  -H 'Content-Type: application/json' \
  -d '{"party":"Monsanto","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/federal_civil_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "federal-civil": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-federal-civil"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-federal-civil
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Federal Civil data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
