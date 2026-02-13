---
name: structured-context-engineering-file-native
description: |
  Structure database schemas and structured data as file-native context for LLM agent operations.
  Applies evidence-based format selection, domain-partitioned file layouts, and architecture-aware
  context engineering to maximize accuracy when agents generate SQL, query APIs, or navigate
  large structured systems. Use this skill when:
  - "Generate SQL against a database with many tables"
  - "Set up schema context for an agent pipeline"
  - "Organize database metadata files for LLM consumption"
  - "Help me structure context for a text-to-SQL system"
  - "Scale my agent to handle thousands of tables"
  - "Choose the best format for feeding schema info to an LLM"
---

# Structured Context Engineering for File-Native Agentic Systems

This skill enables Claude to engineer structured context — database schemas, API definitions, configuration metadata — so that LLM agents can accurately operate over them at scale. Based on a systematic study of 9,649 experiments across 11 models and 4 formats (McMillan, 2026), the technique replaces naive schema-dumping with principled decisions about format selection, file architecture, and domain partitioning. The core insight: **model capability is the dominant accuracy factor** (21 percentage-point gap between frontier and open-source tiers), and architectural choices must be tailored to the model tier rather than following universal rules.

## When to Use

- When building a text-to-SQL pipeline and deciding how to feed schema information to the LLM
- When an agent needs to navigate a database with hundreds or thousands of tables
- When choosing between inline context (schema in the prompt) vs. file-based context (schema in files the agent reads)
- When selecting a serialization format (YAML, Markdown, JSON, or compact notations) for structured metadata
- When designing a multi-file schema layout that an agentic system will grep/search through
- When optimizing token usage for large-schema SQL generation tasks
- When deploying an LLM agent against a production database and needing evidence-based configuration

## Key Technique

**Context engineering** is the practice of structuring the information an LLM agent consumes to maximize task accuracy. For structured data operations like SQL generation, this means deciding (a) what format to serialize schemas in, (b) whether to inject schemas inline into the prompt or store them in files the agent navigates, and (c) how to partition large schemas so the agent can find relevant tables without exceeding context limits.

The paper's central finding is that **architecture choice is model-dependent**. File-based context retrieval — where schemas live in files that the agent reads using tools like `grep`, `cat`, and `ls` — improves accuracy by +2.7% (p=0.029) for frontier models (Claude, GPT, Gemini) but degrades accuracy for many open-source models (aggregate -7.7%, p<0.001). This is because file-native navigation requires the model to formulate search queries, interpret grep output, and synthesize information across multiple reads — capabilities that scale with model sophistication. Meanwhile, **format choice (YAML vs. Markdown vs. JSON vs. TOON) has no statistically significant effect on aggregate accuracy** (chi-squared=2.45, p=0.484), though individual open-source models show format-specific sensitivities.

For scaling beyond what fits in a single context window, **domain-partitioned schemas** organize tables into topical directories (e.g., `schemas/billing/`, `schemas/inventory/`, `schemas/users/`). The agent first reads a domain index to identify relevant partitions, then drills into specific files. This approach scales to 10,000+ tables while maintaining high navigation accuracy for frontier models.

## Step-by-Step Workflow

### 1. Audit the structured data landscape

Inventory all tables, views, or API endpoints the agent will operate on. Count them. If fewer than ~50 tables, inline context may suffice. If 50-500, consider single-file or small multi-file layouts. If 500+, domain partitioning is required.

### 2. Select architecture based on model tier

- **Frontier models (Claude, GPT-4+, Gemini):** Use file-based context. Store schemas in files the agent navigates with search tools. This yields a measurable accuracy improvement.
- **Open-source or smaller models:** Prefer inline context (schema directly in the prompt). File navigation adds cognitive load these models handle poorly. If you must use files, test thoroughly.

### 3. Choose a serialization format

Since format has no significant aggregate effect, optimize for your operational constraints:

- **YAML:** Human-readable, easy to edit, moderate token count. Good default for maintainability.
- **Markdown:** Familiar to most LLMs from training data. Use for documentation-heavy schemas with descriptions.
- **JSON:** Machine-parseable, widely supported. Use when schemas are auto-generated from tooling.
- **Compact/TOON:** Token-efficient for very large schemas, but may increase grep output density and confuse less capable models. Only use with frontier models when token budget is tight.

### 4. Serialize schemas with consistent structure

For each table, include: table name, column names, column types, primary keys, foreign key relationships, and a one-line description. Keep the structure identical across all tables so the agent can pattern-match reliably.

```yaml
# schemas/billing/invoices.yaml
table: invoices
description: "Customer invoices with line items and payment status"
columns:
  - name: invoice_id
    type: INTEGER
    primary_key: true
  - name: customer_id
    type: INTEGER
    foreign_key: customers.customer_id
  - name: amount
    type: DECIMAL(10,2)
  - name: status
    type: VARCHAR(20)
    values: [draft, sent, paid, overdue]
  - name: created_at
    type: TIMESTAMP
```

### 5. Build domain-partitioned directory layout (for large schemas)

Organize schemas into topical directories with an index file:

```
schemas/
  _index.yaml          # Maps domains to descriptions
  billing/
    _tables.yaml       # Lists all tables in this domain
    invoices.yaml
    payments.yaml
    refunds.yaml
  inventory/
    _tables.yaml
    products.yaml
    warehouses.yaml
    stock_levels.yaml
  users/
    _tables.yaml
    customers.yaml
    addresses.yaml
    preferences.yaml
```

The `_index.yaml` file is the agent's entry point:

```yaml
domains:
  billing: "Invoices, payments, refunds, and financial transactions"
  inventory: "Products, warehouses, stock levels, and supply chain"
  users: "Customer accounts, addresses, and preferences"
```

### 6. Equip the agent with navigation tools

Give the agent access to file operations: `ls` to list directories, `grep` to search for table/column names, and `cat`/`read` to load specific schema files. Define a navigation protocol:

1. Agent reads `_index.yaml` to identify relevant domains
2. Agent reads `_tables.yaml` in the relevant domain(s)
3. Agent reads individual schema files for tables it needs
4. Agent generates the SQL query

### 7. Test with representative queries and measure accuracy

Run at least 20-30 representative queries spanning different domains, join patterns, and complexity levels. Measure exact-match and execution accuracy. Compare against an inline-context baseline.

### 8. Monitor token overhead from grep density

Compact formats or novel notations may produce dense grep output that the agent over-reads, wasting tokens. Monitor total tokens consumed per query, not just input tokens. If a format causes the agent to issue many extra search operations, switch to a more familiar format.

### 9. Iterate on partition boundaries

If the agent frequently needs tables from multiple domains in a single query, consider reorganizing domains so commonly co-queried tables are grouped together, or create cross-domain index files listing common joins.

## Concrete Examples

**Example 1: Setting up file-native context for a medium database (100 tables)**

User: "I have a PostgreSQL database with about 100 tables for an e-commerce platform. I need an LLM agent to generate SQL queries from natural language. How should I structure the schema context?"

Approach:
1. Export schema metadata using `pg_dump --schema-only` or query `information_schema`
2. Since this targets a frontier model (Claude), use file-based architecture for the +2.7% accuracy gain
3. Group 100 tables into ~8-12 domains (orders, products, users, shipping, analytics, etc.)
4. Serialize each table as a YAML file with columns, types, keys, and a description
5. Create `_index.yaml` and per-domain `_tables.yaml` index files
6. Configure the agent to navigate: read index → identify domains → read table schemas → generate SQL

Output directory structure:
```
schemas/
  _index.yaml
  orders/
    _tables.yaml
    orders.yaml
    order_items.yaml
    order_status_history.yaml
  products/
    _tables.yaml
    products.yaml
    categories.yaml
    product_reviews.yaml
  ...
```

Agent workflow for query "Show me total revenue by category last month":
```
1. Read _index.yaml → identifies "orders" and "products" domains
2. Read orders/_tables.yaml → finds orders, order_items
3. Read products/_tables.yaml → finds products, categories
4. Read order_items.yaml, orders.yaml, products.yaml, categories.yaml
5. Generates:
   SELECT c.name, SUM(oi.quantity * oi.unit_price) AS revenue
   FROM order_items oi
   JOIN orders o ON oi.order_id = o.order_id
   JOIN products p ON oi.product_id = p.product_id
   JOIN categories c ON p.category_id = c.category_id
   WHERE o.created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
     AND o.created_at < DATE_TRUNC('month', CURRENT_DATE)
   GROUP BY c.name
   ORDER BY revenue DESC;
```

**Example 2: Choosing architecture for an open-source model**

User: "I'm using Llama 3 70B to generate SQL against our 30-table database. Should I use file-based or inline context?"

Approach:
1. With 30 tables and an open-source model, inline context is strongly preferred
2. The paper shows open-source models lose ~7.7% accuracy on average with file-based context
3. Serialize all 30 table schemas into a single document placed directly in the system prompt
4. Use Markdown or YAML format (test both — open-source models can have format-specific sensitivities)
5. Keep total schema under ~4,000 tokens to leave room for the query and response

Output (inline prompt structure):
```
You are a SQL assistant. Below is the database schema.

## Tables

### orders
| Column | Type | Key |
|--------|------|-----|
| order_id | INTEGER | PK |
| customer_id | INTEGER | FK → customers.customer_id |
| total | DECIMAL(10,2) | |
| created_at | TIMESTAMP | |

### customers
| Column | Type | Key |
|--------|------|-----|
| customer_id | INTEGER | PK |
| name | VARCHAR(100) | |
| email | VARCHAR(255) | |

... (remaining 28 tables)

Generate a SQL query for the following question:
```

**Example 3: Scaling to 10,000 tables for a data warehouse**

User: "Our enterprise data warehouse has ~8,000 tables across dozens of business units. How do we make this work with an LLM agent?"

Approach:
1. This requires domain-partitioned file architecture — no single prompt can hold 8,000 schemas
2. Use a frontier model (mandatory at this scale for navigation accuracy)
3. Create a two-level hierarchy: business unit → functional area → table files
4. Build a searchable index with table descriptions and common column names
5. Implement a navigation protocol with explicit search steps

Directory layout:
```
schemas/
  _index.yaml              # Top-level: 15 business units
  _search_index.yaml       # Flattened table name → domain mapping
  finance/
    _index.yaml            # Sub-domains: AP, AR, GL, treasury
    accounts_payable/
      _tables.yaml
      ap_invoices.yaml
      ap_vendors.yaml
      ...
    general_ledger/
      _tables.yaml
      gl_accounts.yaml
      gl_journal_entries.yaml
      ...
  supply_chain/
    _index.yaml
    procurement/
      ...
    warehousing/
      ...
```

Navigation protocol for the agent:
```
1. Grep _search_index.yaml for keywords from the user's question
2. Identify 1-3 relevant business units and sub-domains
3. Read the domain _tables.yaml files to narrow to specific tables
4. Load only the 3-8 table schemas actually needed for the query
5. Generate SQL using only the loaded schemas as context
```

## Best Practices

**Do:** Match your architecture to your model tier. File-based for frontier models, inline for open-source.

**Do:** Keep schema serialization format consistent across your entire corpus. Mixing formats within a project confuses the navigation agent.

**Do:** Include foreign key relationships explicitly in every table schema. These are critical for the agent to generate correct JOINs.

**Do:** Create a flattened search index (`_search_index.yaml`) for large schemas so the agent can quickly locate tables by keyword without traversing the directory tree.

**Do:** Test with your actual model and your actual queries before committing to an architecture. The paper shows significant model-specific variation.

**Avoid:** Assuming a compact token-saving format will be faster. Dense grep output from compact formats can cause agents to issue more tool calls, increasing total cost.

**Avoid:** Using file-based context with smaller open-source models without thorough testing. The accuracy penalty is real and statistically significant.

**Avoid:** Putting all 500+ table schemas in a single file. Even with file-based context, individual files should contain one table's schema for precise retrieval.

## Error Handling

- **Agent navigates to wrong domain:** Improve `_index.yaml` descriptions. Add keywords and example queries for each domain. Consider adding a `_search_index.yaml` with table-to-domain mappings.
- **Agent misses tables needed for JOINs:** Add a `related_tables` field to each schema file listing commonly joined tables. Create cross-reference files for frequent multi-domain joins.
- **Token budget exceeded on large schemas:** Reduce schema verbosity (drop column descriptions for obvious fields). Split the query into sub-queries if it spans too many domains.
- **Grep returns too many matches:** Make table/column names more distinctive. Add domain prefixes to table names in the search index. Limit grep scope to `_tables.yaml` files first.
- **Format-specific failures with open-source models:** If accuracy drops with one format, try another. The paper found individual models (especially open-source) can be sensitive to specific formats even though the aggregate effect is null.

## Limitations

- Findings are calibrated to SQL generation. Other structured operations (API calls, config management) may not transfer directly, though the architectural principles likely generalize.
- The +2.7% file-based accuracy gain for frontier models is statistically significant but modest. If inline context already works well, migration cost may not be justified.
- Domain partitioning requires manual curation. Poorly chosen boundaries (co-queried tables split across domains) degrade navigation accuracy.
- Requires agent access to file-reading tools (grep, cat, ls). Prompt-only deployments cannot use file-native architecture.

## Reference

McMillan, D. (2026). *Structured Context Engineering for File-Native Agentic Systems: Evaluating Schema Accuracy, Format Effectiveness, and Multi-File Navigation at Scale.* arXiv:2602.05447v1. Key takeaway: model capability dominates format and architecture effects; tailor context architecture to model tier, not universal assumptions.