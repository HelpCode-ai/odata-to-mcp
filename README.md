# OData to MCP

**Turn any OData V2 or V4 service, SAP Gateway included, into MCP tools for Claude, ChatGPT and Copilot.** Powered by [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

OData to MCP turns any OData V2 or V4 service, SAP Gateway included, into MCP tools that Claude, ChatGPT, Copilot and Cursor can call, without code. AnythingMCP reads `$metadata`, so the model sees entity sets, keys and field labels before it queries. This repository runs the chain against the public Northwind service.

**Last verified:** 2026-09-27 against public OData services: Northwind V2 and V4 and TripPin V4 on services.odata.org (the OData engine's live test suite on all three, and this repository's install.sh + scripts/smoke.mjs against Northwind V4 on an image built from the release branch).  
**Adapter synced:** <!-- synced -->2026-09-27

Maintained by [helpcode.ai](https://helpcode.ai), the team that builds and maintains [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

## Try it in five minutes

Needs Docker 24+, openssl and Node 18+.

```bash
git clone https://github.com/HelpCode-ai/odata-to-mcp.git
cd odata-to-mcp
./scripts/install.sh
npm install && node scripts/smoke.mjs
```

`install.sh` starts AnythingMCP, creates an OData connector for the public Northwind V4 service (`https://services.odata.org/V4/Northwind/Northwind.svc`: customers, orders, products; read-only, no login) and an MCP API key. The service is on the internet, so there is no demo container and nothing to add to `SSRF_ALLOWED_HOSTS`. `smoke.mjs` lists the tools and asks for the latest orders shipped to Germany.

Every OData connector gets five built-in tools as soon as it is created:

| MCP tool | What it answers |
|---|---|
| `northwind_list_services` | Where the service lives; on SAP Gateway, a search of the V2 and V4 service catalog |
| `northwind_describe_service` | Entity sets with their labels and keys, flagged when they are analytical or parameterised |
| `northwind_describe_entity` | Fields with label, type and key, the currency or unit field of each amount, dimensions and measures, navigation properties |
| `northwind_query` | `select`, `filter`, `orderby`, `top` (up to 1000), `skip`, `expand`, V4 `apply` and `search`; field names are checked against the model, server paging is followed and rows come back flat |
| `northwind_get_entity` | One entity by key; a composite key as a JSON string such as `{"OrderID": 10248, "ProductID": 11}` |

The prefix is `config.odata.toolPrefix` (here `northwind`), or the connector name plus `_odata`. The built-ins only read and are annotated read-only for MCP clients. `$metadata` is cached for 24 hours.

## Use your own OData service

1. In the AnythingMCP UI, **Connectors → New connector → OData**. For one service, the base URL is its service root (`https://services.example.com/odata/v4/Sales`). For SAP, choose *SAP Gateway* and give only the host with its HTTPS port (`https://s4.example.com:44300`); services then come from SAP's catalog.
2. Pick the auth. The OData connector runs on the REST engine, so every REST method works: Basic, OAuth 2.0, API key, client certificates, a login token.
3. Optional: **Import Tools → OData $metadata** turns entity sets into named tools (`<set>_list`, `<set>_get`) next to the built-ins.
4. List the service paths the agent may call under *Allowed services* (`*` as wildcard), and assign the connector to an MCP server.

Through the API:

```bash
curl -s http://localhost:4000/api/connectors -H "Authorization: Bearer $TOKEN" -H "content-type: application/json" \
  -d '{"name":"Sales","type":"ODATA","baseUrl":"https://services.example.com/odata/v4/Sales","authType":"BASIC_AUTH","authConfig":{"username":"reader","password":"…"},"config":{"odata":{"toolPrefix":"sales"}}}'
```

| Setting (`config.odata`) | Meaning |
|---|---|
| `sap` | SAP Gateway mode: catalog discovery and SAP parameters. Implied by `sapClient`. |
| `sapClient`, `sapLanguage` | Sent as `sap-client` and `sap-language` on every request. |
| `services` | Allowed service paths, `*` as wildcard; other services are refused. |
| `version` | `v2` or `v4`, to override the detection from `$metadata`. |
| `maxRows` | Row cap per query, at most 1000. |
| `toolPrefix` | Prefix of the built-in tool names. |

**SAP S/4HANA, ECC and BW:** the *SAP S/4HANA (OData)* catalog adapter packages the SAP setup (built-ins with the `s4` prefix, ready tools for released APIs and a guide to SAP's filters and analytical views). See [sap-mcp-server](https://github.com/HelpCode-ai/sap-mcp-server).

## Connect Claude, ChatGPT, Copilot or Cursor

- **Claude (claude.ai, Desktop, mobile):** *Customize → Connectors → Add custom connector*, paste your MCP server URL and sign in. Claude connects from Anthropic's cloud, so the URL must be public HTTPS: your AnythingMCP Cloud URL, or your own instance behind TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http odata-to-mcp http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) and **VS Code / GitHub Copilot** (`.vscode/mcp.json`, key `servers` instead of `mcpServers`, plus `"type": "http"`):

  ```json
  { "mcpServers": { "odata-to-mcp": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** add the public HTTPS URL as a connector (app) in ChatGPT's settings. A `localhost` URL does not work there.

## Example prompts

- Which orders were shipped to Germany most recently, and to which customers?
- Show order 10248 with its lines and the product names.
- Which products are at or below their reorder level?
- Which customers are in Berlin, and what did they order?
- Which ten orders had the highest freight?
- Describe the Order_Details entity: what is its key, and how does it link to products?

More in [examples/prompts.md](examples/prompts.md).

## Security

- **The built-ins only read.** Imported or hand-written tools that POST, PATCH or DELETE are write tools; leave them out of the MCP server's role.
- **Allowed services** limit which service paths a connector calls, and server paging is only followed on the connector's own host.
- **The service decides what the user may see.** On SAP, the technical user's authorizations are checked on every call; they are the real boundary.
- **Credentials** stay in the connector, encrypted with AES-256-GCM; the model never sees them.
- **Audit log:** every call with input, output, duration and status.

## FAQ

### How do I turn an OData service into an MCP server?
Create an OData connector in AnythingMCP with the service root as base URL and the service's auth, then add the MCP server URL to your AI client. Five built-in tools let the model find the service, read its model and query it; no import and no code.

### Does it support OData V2 and V4?
Both. The version is detected from `$metadata`. V2 answers are requested as JSON, `/Date()/` values come back as ISO strings and decimals as strings, so amounts are not rounded. V4 `$apply` and `$search` are passed through where the service supports them.

### Does it work with SAP Gateway?
Yes, that is the first-class case. In SAP Gateway mode the connector searches SAP's V2 and V4 service catalog, sends `sap-client` and `sap-language`, reads SAP's annotations (labels, the currency or unit field of an amount, text fields, analytical and parameterised views) and fetches a CSRF token before a write. For S/4HANA on-premise and Private Cloud, the SAP S/4HANA (OData) catalog adapter adds ready tools for released APIs and a guide.

### Can the AI change data through OData?
Not through the built-ins, and not through the tools imported from `$metadata` (`<set>_list`, `<set>_get`): they only read. Tools you write by hand can POST, PATCH or DELETE; leave them out of the MCP server's role if the AI should only read.

### How is this different from importing an OpenAPI spec?
The connector reads `$metadata` itself, so there is no conversion step, and it knows OData: field names are checked before the call, server paging is followed, composite keys work, and the model sees labels and units instead of bare property names. REST APIs without OData stay with openapi-to-mcp.

### What about services on my internal network?
Self-host AnythingMCP where it can reach them and add their hostnames to `SSRF_ALLOWED_HOSTS`. SAP Gateway ports are usually internal.

## Troubleshooting

| Problem | Fix |
|---|---|
| "Unknown field … Did you mean …" | The model guessed a field name. The query is checked against `$metadata` before it is sent; let it call `describe_entity` first. |
| A query is refused for a missing filter or parameter | The entity set requires a filter or is a parameterised view. `describe_entity` names them; pass them in `filter` and `parameters`. |
| `403` from SAP Gateway | The technical user lacks `S_SERVICE` for that service, or the business authorization. Fix the role in SAP. |
| New fields are missing | `$metadata` is cached for 24 hours. Call `describe_service` with `refresh: true`. |
| "SSRF" or "blocked host" error | The service is on a private network. Add its hostname to `SSRF_ALLOWED_HOSTS` on a self-hosted instance. |

## Related

- [openapi-to-mcp](https://github.com/HelpCode-ai/openapi-to-mcp): OpenAPI to MCP: turn any OpenAPI/Swagger or REST API into an MCP server for Claude & ChatGPT. Every endpoint a tool, no code.
- [sap-mcp-server](https://github.com/HelpCode-ai/sap-mcp-server): SAP MCP server: connect SAP Business One, S/4HANA (Cloud, on-premise via OData or HANA SQL) and Concur to Claude & ChatGPT.
- [soap-to-mcp](https://github.com/HelpCode-ai/soap-to-mcp): Turn any SOAP/WSDL web service into MCP tools for Claude & ChatGPT. Legacy SOAP APIs as AI tools, no code, self-hosted.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): the open-source MCP server and gateway this repository is built on.

## License

AGPL-3.0-only. The adapter definition in `adapter/` comes from AnythingMCP (AGPL-3.0).
