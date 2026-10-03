# Cruise Itinerary for Claude, ChatGPT and other MCP clients

![Cruise Itinerary](assets/icon.svg)

Ask an AI assistant about cruises and get answers from real data: which sailings call at a port, what a ship's itinerary looks like day by day, how a sailing's fare has moved week by week, how much ship capacity reaches a port each month, and how cruise prices are trending overall.

This repository holds:

- the connection details for the **Cruise Itinerary remote MCP server** at `https://cruise-itinerary.com/mcp`,
- a **Claude plugin** that adds that server plus a `cruise-research` skill, which teaches Claude the data's caveats,
- `server.json`, the server's entry for the official MCP Registry.

The server is run by [Cruise Itinerary](https://cruise-itinerary.com) (Agile Travel Group, Inc., operated as Cruiseable). The data is the same as its REST API: [API docs](https://cruise-itinerary.com/docs), [MCP docs](https://cruise-itinerary.com/docs#mcp).

## What you can ask

- "Which sailings call at Juneau in July 2027, cheapest balcony first?"
- "Show me the day-by-day itinerary of that sailing."
- "Has the fare on this sailing gone up or down since September?"
- "How many berths of ship capacity reach Cozumel each month next winter?"
- "What does the Cruise Price Index say about fares this month?"

## Tools

All tools are read-only (`readOnlyHint: true`). Each returns the same JSON as the matching REST endpoint, as `structuredContent` with an `outputSchema`.

| Tool | What it answers | Minimum plan |
|---|---|---|
| `resolve_entity` | A port, ship or cruise line name, however spelt, to the id the other tools take | Free |
| `cruise_price_index` | The weekly Cruise Price Index: latest 12 weeks (Free), or a full series by line, cabin and segment (Business) | Free / Business |
| `search_sailings` | Upcoming sailings by ship, line, port, departure port, region, dates, length and fare, with the latest weekly fares | Developer |
| `get_itinerary` | A sailing's dated day-by-day itinerary, or a cruise's stop list | Developer |
| `get_price_history` | Every weekly fare snapshot of one sailing | Business |
| `port_month_capacity` | Ship-days, distinct ships and berths at one port, month by month | Business |

A free account (an email address, no card) includes 500 calls a month. Every successful tool call counts as one API call. A tool above your plan answers with a short message and a link to [pricing](https://cruise-itinerary.com/pricing).

## Setup

### Claude (claude.ai, Desktop, mobile)

Add it as a connector: **Customize → Connectors → Add custom connector**, name it Cruise Itinerary, and paste `https://cruise-itinerary.com/mcp`. Claude detects the OAuth sign-in and asks you to sign in when you add it: enter your email, open the link we send, then approve the connection.

To get the `cruise-research` skill as well, install this repository as a plugin (below) or from the Claude directory once it is listed.

### Claude Code

As a plugin (server and skill):

```
/plugin marketplace add BigBalli/cruise-itinerary-mcp
/plugin install cruise-itinerary@cruise-itinerary
```

Or the server only:

```
claude mcp add --transport http cruise-itinerary https://cruise-itinerary.com/mcp
```

Then run `/mcp` and choose cruise-itinerary to sign in.

### ChatGPT

Add a custom app (connector) with the URL `https://cruise-itinerary.com/mcp` and OAuth authentication. Until the listing appears in the ChatGPT directory, this needs developer mode.

### Cursor, VS Code, Windsurf and other clients

Point the client at the URL. Clients that support MCP OAuth sign in in the browser:

```json
{
  "mcpServers": {
    "cruise-itinerary": {
      "url": "https://cruise-itinerary.com/mcp"
    }
  }
}
```

Clients without OAuth can send an API key from your [dashboard](https://cruise-itinerary.com/account) as a header instead. Keep the key out of shared files.

```json
{
  "mcpServers": {
    "cruise-itinerary": {
      "url": "https://cruise-itinerary.com/mcp",
      "headers": { "Authorization": "Bearer <your API key>" }
    }
  }
}
```

## How sign-in works

The server follows the [MCP authorization specification](https://modelcontextprotocol.io/specification/latest/basic/authorization): OAuth 2.1 with PKCE (S256), protected resource metadata at `https://cruise-itinerary.com/.well-known/oauth-protected-resource/mcp`, and an authorization server at `https://cruise-itinerary.com` that supports Client ID Metadata Documents and dynamic client registration. Sign-in is by email link; a new address gets a free account. You approve each app on a consent page that shows which app is asking and where it returns you. Access tokens last an hour and refresh tokens rotate on use. Connecting never shares your API keys, email address or billing with the app, and the app can only read.

Every request needs an account, `initialize` included: without a token the server answers `401` with a `WWW-Authenticate` header pointing at the protected resource metadata, which is how clients know to start sign-in.

## Read the data the right way

- Fares are weekly snapshots captured every Monday since 6 September 2026, not live or bookable prices. `meta.as_of` gives the load date.
- A missing (null) fare means the cabin was not offered or was sold out; the data does not say which.
- Itinerary days marked `estimated` were placed from the distance between ports and can be off by a day. `exact` days come from the cruise line's published itinerary.
- Port capacity is monthly only, and berths are ship capacity, not passengers.
- Coverage is strongest for North American cruise lines. TUI Cruises/Marella, AIDA, Saga, Fred. Olsen and Hapag-Lloyd are not covered.

## Data and privacy

The plugin contains no code that runs on your machine. It declares one remote MCP server, `https://cruise-itinerary.com/mcp`, and one skill made of instructions. When your assistant calls a tool, the tool name and its arguments (names of ports, ships or lines, dates, ids) go to cruise-itinerary.com with your access token. They are used to answer the call, count it toward your plan and enforce rate limits. Nothing else is sent, and nothing is sent anywhere else.

- Privacy policy: https://cruise-itinerary.com/privacy
- Terms: https://cruise-itinerary.com/terms
- Support: hello@cruise-itinerary.com

## License

MIT, see [LICENSE](LICENSE). The data served by the API is covered by the [Cruise Itinerary terms](https://cruise-itinerary.com/terms), not by this license.
