# Magento MCP Server

**Connect Magento to Claude, ChatGPT and Copilot: products, stock, orders and customers as MCP tools.** Powered by [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

Magento MCP Server gives Claude, ChatGPT, Copilot and Cursor 12 tools for Magento: products, stock, orders and customers. 7 tools read and 5 can change data. It runs on AnythingMCP: one click on AnythingMCP Cloud, or self-hosted with Docker. Credentials are stored encrypted and every call is audited.

**Last verified:** 2026-09-26 against the Magento 2 REST API (production traffic on AnythingMCP Cloud: 19 successful tool calls from 3 workspaces in the last 90 days).  
**Adapter synced:** <!-- synced -->2026-09-26

Maintained by [@keysersoft](https://github.com/keysersoft), an AnythingMCP maintainer. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Quick start (AnythingMCP Cloud)

1. Sign in at [cloud.anythingmcp.com](https://cloud.anythingmcp.com) and open the [install link](https://cloud.anythingmcp.com/connectors/store?install=magento).
2. Enter `MAGENTO_BASE_URL`, `MAGENTO_ACCESS_TOKEN` (see [Authentication](#authentication)).
3. Copy the URL of your MCP server under **MCP Servers** and add it to your AI client ([below](#connect-claude-chatgpt-copilot-or-cursor)).

AnythingMCP Cloud is the same open-source code, operated by helpcode.ai in Frankfurt, Germany.

## Self-hosted (Docker)

Needs Docker 24+, openssl and Node 18+.

```bash
git clone https://github.com/keysersoft/magento-mcp-server.git
cd magento-mcp-server
./scripts/install.sh
```

`install.sh` writes `.env` with fresh secrets, starts AnythingMCP, creates the first admin, installs the connector if `MAGENTO_BASE_URL` and `MAGENTO_ACCESS_TOKEN` are set in `.env` and creates an MCP API key. Without credentials it prints the install link instead: `http://localhost:3000/connectors/store?install=magento`. Then check the whole chain:

```bash
npm install && node scripts/smoke.mjs
```

## Connect Claude, ChatGPT, Copilot or Cursor

- **Claude (claude.ai, Desktop, mobile):** *Customize → Connectors → Add custom connector*, paste your MCP server URL and sign in. Claude connects from Anthropic's cloud, so the URL must be public HTTPS: your AnythingMCP Cloud URL, or your own instance behind TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http magento-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) and **VS Code / GitHub Copilot** (`.vscode/mcp.json`, key `servers` instead of `mcpServers`, plus `"type": "http"`):

  ```json
  { "mcpServers": { "magento-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** add the public HTTPS URL as a connector (app) in ChatGPT's settings. A `localhost` URL does not work there.

## Tools

12 tools, generated from [`adapter/magento.json`](adapter/magento.json). **read** tools cannot change anything in the source system.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | What it does | Access |
|---|---|---|
| `magento_search_products` | Search products with Magento's search criteria DSL. | read |
| `magento_get_product` | Fetch a single product by SKU. | read |
| `magento_create_product` | Create a product. | write |
| `magento_update_product` | Update a product by SKU (PUT). | write |
| `magento_delete_product` | Permanently delete a product by SKU. | write |
| `magento_update_stock` | Update inventory stock for a SKU at a source. | write |
| `magento_list_categories` | List the category tree (or below a rootCategoryId). | read |
| `magento_search_orders` | Search orders. | read |
| `magento_get_order` | Fetch one order by ID with billing/shipping addresses, items[], status history. | read |
| `magento_cancel_order` | Cancel an order (only valid in pending/processing status). | write |
| `magento_search_customers` | Search customers with the same DSL. | read |
| `magento_get_customer` | Fetch one customer by ID. | read |
<!-- tools:end -->

## Example prompts

- Which orders from the last 24 hours are still pending?
- Show order 000000123 with its items and shipping address.
- Which products in the category "Doors" are out of stock?
- What is the price and stock of SKU DR-1002?
- Find the customer with e-mail anna@example.com and list her last orders.
- Which products were updated this week?

More in [examples/prompts.md](examples/prompts.md).

## Authentication

This connector uses the Magento 2 REST API (devdocs.magento.com).

**Setup**:
1. As Magento admin → **System → Integrations → Add New Integration**.
2. Name + email, then on the API tab pick scopes: at minimum Catalog (Products + Categories + Stock), Customers, Sales (Orders + Invoices + Shipments).
3. **Activate** the integration → grant access. Magento generates an **Access Token** (this is the long-lived bearer token).
4. Set:
   - `MAGENTO_BASE_URL` = your storefront base URL (e.g. `https://shop.example.com`)
   - `MAGENTO_ACCESS_TOKEN` = the integration access token
   - Optionally `MAGENTO_STORE_VIEW` if you want a specific store view (default 'default' or use 'all')

**Authentication**: `Authorization: Bearer ${MAGENTO_ACCESS_TOKEN}`.

**Path prefix**: `/rest/{store_view}/V1/...`. The adapter uses `/rest/default/V1/` as default — if you need to target a different store view, replace baseUrl accordingly or pass `?storeCode=` (Magento accepts both).

**Search criteria DSL**: Magento has a unique multi-key search syntax for list endpoints:
```
?searchCriteria[filter_groups][0][filters][0][field]=status
&searchCriteria[filter_groups][0][filters][0][value]=1
&searchCriteria[filter_groups][0][filters][0][condition_type]=eq
&searchCriteria[pageSize]=20
&searchCriteria[currentPage]=1
```
It's verbose but powerful. The adapter exposes the most common filters as flat params and lets you pass arbitrary criteria via `searchCriteria_raw` for advanced cases.

**SKU as primary key**: most product endpoints use SKU (not ID) in the URL: `/V1/products/{sku}`.

**Rate limits**: not enforced by Magento core — depends on hosting. On 429 back off.

**Out of scope here**: GraphQL endpoint (separate), Inventory MSI deep features, B2B/Commerce-only features (shared catalogs, company hierarchies), CMS pages/blocks, sales rules editing.

## Security

- **Read or write is your choice.** 7 of the tools only read; `magento_create_product`, `magento_update_product`, `magento_delete_product`, `magento_update_stock`, `magento_cancel_order` can change data. Assign the connector to an MCP server whose role whitelists only the tools you want, and the rest are invisible to that client.
- **Credentials** are encrypted with AES-256-GCM and never shown to the model.
- **Response mapping** drops or reshapes fields per tool before they reach the model, e.g. bank details or personal data.
- **Audit log:** every call is recorded with input, output, duration and status, in your own database when self-hosted.
- **SSO, RBAC and SCIM** are included in the self-hosted build.

## FAQ

### Is there a Magento MCP server?
Yes, this one. It connects Magento 2 and Adobe Commerce to Claude, ChatGPT and Copilot through AnythingMCP: 12 tools for products, stock, categories, orders and customers.

### What do I need to connect it?
Your store's base URL and an admin access token from an integration (**System → Extensions → Integrations**). The Authentication section has the details.

### Can the AI change data in my store?
Yes, if you let it: five tools write (create, update and delete products, update stock, cancel orders). Give the MCP server a role that whitelists only the read tools to start.

### Does it work with Adobe Commerce Cloud?
It uses the standard Magento 2 REST API, which Adobe Commerce exposes too.

### Does it work with ChatGPT and Copilot?
Yes. The same MCP server works in ChatGPT (with a public HTTPS URL such as AnythingMCP Cloud), GitHub Copilot in VS Code, Cursor and Claude Code.

## Troubleshooting

| Problem | Fix |
|---|---|
| `401` / `403` from the vendor | The credentials are wrong or lack rights. Re-enter them on the connector page; the import runs a test call and shows the result. |
| Tools missing in the AI client | The connector is not assigned to the MCP server the client uses. Check **MCP Servers**, then run `node scripts/smoke.mjs`. |
| The host is on your internal network | Self-host AnythingMCP on that network and add the hostname to `SSRF_ALLOWED_HOSTS`, or the outbound guard blocks the call. |
| Works locally, fails on AnythingMCP Cloud | The system must be reachable from the internet with a valid TLS certificate. |

## Related

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): the open-source MCP server and gateway this repository is built on.

## License

AGPL-3.0-only. The adapter definition in `adapter/` comes from AnythingMCP (AGPL-3.0).
