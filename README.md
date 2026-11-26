# figcheck

Did the numbers in that draft actually come from the filing? figcheck is a local forwarding client for the hosted financial number-check MCP service: it answers `tools/list` locally from the captured tool definition and forwards every call to the service, returning the answers exactly as the service sends them back.

The one tool scans a piece of financial writing for $CASHTAG-style ticker mentions (for example `$AAPL`) and checks the numeric claims in the text against SEC EDGAR's own XBRL company-facts data for each company's most recent 10-K — EPS, book value per share, revenue, market cap, total assets, and total liabilities. Claims that do not match the filed figures are flagged, so a writer can fix a stale or invented number before publishing. There is no write, signup, or send tool in this surface.

## The tool

- **`check_financial_numbers`** — Check the draft in `text` (with `$TICKER` cashtags for each company mentioned) against the filed figures. `text` is required.

## Install and run

As an MCP server via npm:

```bash
npx -y @reneturcios/figcheck mcp
```

MCP client configuration:

```json
{
  "mcpServers": {
    "figcheck": {
      "command": "npx",
      "args": ["-y", "@reneturcios/figcheck", "mcp"]
    }
  }
}
```

As a container:

```bash
docker run -i ghcr.io/cloudcorprecords/figcheck
docker run --rm ghcr.io/cloudcorprecords/figcheck check_financial_numbers '{"text":"$AAPL revenue was $391 billion last year."}'
```

## CLI

One binary serves both the command line and the MCP server. The tool is a subcommand taking a single JSON string with that tool's arguments:

```bash
figcheck check_financial_numbers '{"text":"$AAPL revenue was $391 billion last year."}'
figcheck --version
```

Exit codes: `0` when the forwarded call succeeds, `1` when the upstream result is an error, and `2` for usage problems such as an unknown subcommand or a malformed JSON argument.

## Configuration

- `FIGCHECK_MCP_URL` — Overrides the upstream endpoint. The default is the pinned hosted endpoint, `https://www.edgethirteen.com/api/mcp/numbercheck`. (The override variable is a reconstruction added by this rebuild.)

The upstream registry record declares no headers, so no authentication header is ever sent; the tool surface was captured keyless.

Upstream failures — a JSON-RPC error from the service, a non-2xx response, an unreadable body, or a request that fails outright — are reported to the caller as an error result whose text starts with `figcheck error:`.

## How forwarding works

The first forwarded call opens a session: an `initialize` request (protocol version `2025-03-26`), followed by a `notifications/initialized` notification, then the `tools/call` itself. If the service answers the initialize request with an `Mcp-Session-Id` header, that value is carried on every later request, and the open session is reused for the rest of the process. Replies are accepted either as plain JSON or as server-sent events. Successful results pass through verbatim, including any `isError` flag and any extra fields the service includes.

## Development

```bash
npm install
npm run build   # tsc -> dist/ (dist/cli.js is the bin)
npm test        # tsx --test test/*.test.ts — stubbed upstream, no network
npm run dev     # run the CLI from source
```

The test suite checks the served tool list against the captured surface (the 1 tool, deep-equal, with no `title` key served — the capture carries none — and the `required` list exactly as captured), exercises the session handshake and its reuse, verifies that no `Authorization` or `X-API-Key` header is ever sent under any environment, and verifies that arguments go upstream untouched and results come back untouched.

## License

MIT — see `LICENSE`. Copyright (c) 2026 Rene Turcios.
