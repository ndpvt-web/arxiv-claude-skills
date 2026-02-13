---
name: "ci4a-semantic-component-interfaces"
description: "Build semantic component interfaces that expose UI components as structured tool primitives for AI agent automation. Use when: 'make my UI agent-friendly', 'add CI4A interfaces to components', 'create semantic wrappers for web components', 'build agent-accessible UI toolkit', 'expose component actions as tools', 'wrap Ant Design components for agent use'."
---

# CI4A: Semantic Component Interfaces for Agent-Driven Web Automation

This skill teaches Claude how to implement Component Interface for Agent (CI4A), a technique that wraps UI components in semantic interfaces so AI agents can interact with them through structured tool calls instead of fragile pixel-clicking or DOM scraping. Rather than forcing agents to reverse-engineer human-centric UIs, CI4A exposes each component's state, available actions, and parameter constraints as a machine-readable triplet, cutting interaction steps by ~57% and raising task success rates from ~70% to 86.3% on the WebArena benchmark.

## When to Use

- When building a web application that AI agents will operate (form filling, data entry, testing)
- When wrapping an existing component library (Ant Design, MUI, Radix) to be agent-accessible
- When creating browser automation that breaks on DOM changes and needs a stable semantic layer
- When an agent needs to interact with complex widgets like date pickers, cascaders, or data tables without coordinate-based clicking
- When designing a hybrid agent that should prefer high-level semantic actions but fall back to low-level DOM operations
- When refactoring a frontend to support automated QA or RPA workflows driven by LLMs

## Key Technique

**The Semantic Triplet.** CI4A encapsulates each UI component as a triplet `<S, T, M>`:
- **S (Semantic State View):** A direct mapping of the component's internal data model, bypassing the rendering layer. Instead of parsing a DatePicker's rendered calendar grid (O(N) DOM nodes), the agent reads `{ field: "check-in", value: "2025-12-31", format: "YYYY-MM-DD" }` in O(1). This works by tapping into React state/props or Vue reactive objects directly.
- **T (Executable Toolset):** Atomic operation primitives that encapsulate state mutation logic. For a DatePicker, this is `setValue(date)`. For a Table, it includes `sort(column, direction)` and `filter(column, value)`. These wrap either the component's public API or internal state manipulation into standard function calls.
- **M (Interaction Metadata):** A structured contract defining parameter types, valid ranges, and constraints. A DatePicker's metadata specifies the date format (`YYYY-MM-DD`), min/max bounds, and disabled dates -- computed at runtime so the agent never sends invalid input.

**The Global Registry.** Each CI4A-wrapped component registers itself on mount via a `data-cid` DOM attribute and exposes its triplet through a global registry at `window.__ci4a__`. The registry maps component instance keys to their `<S, T, M>` triplets. On unmount, the registration is removed. This gives agents a single entry point to discover all interactive components on the current page and their available operations.

**Semantic-First Execution.** The hybrid agent (called Eous in the paper) follows a strict priority: check the CI4A registry for a semantic tool matching the intent; invoke it if constraints are satisfied; fall back to atomic WebDriver operations (`click`, `type`, `scroll`) only when no semantic tool exists. This reduces a 12-step date selection workflow (open picker, navigate months, click day, confirm) to a single `call("datepicker", "setValue", "2025-12-31")`.

## Step-by-Step Workflow

1. **Identify target components.** Inventory the UI components agents will interact with. Prioritize complex widgets (date pickers, cascaders, tables, multi-step forms) where atomic DOM operations are most fragile.

2. **Define the semantic state view (S) for each component.** Extract the minimal set of properties an agent needs to understand the component's current state. Strip away rendering concerns -- an agent doesn't need to know a dropdown's pixel position, only its current value, options list, and whether it's disabled.

   ```typescript
   // DatePicker state view
   interface DatePickerState {
     field: string;        // e.g., "check-in-date"
     value: string | null; // e.g., "2025-12-31"
     format: string;       // e.g., "YYYY-MM-DD"
     disabled: boolean;
   }
   ```

3. **Define the executable toolset (T) for each component.** Wrap each meaningful user interaction as a named function. Keep tools atomic -- one action per tool. Name them with clear verbs (`setValue`, `selectOption`, `toggleExpand`, `sortColumn`).

   ```typescript
   // DatePicker toolset
   const tools = {
     setValue: (date: string) => { /* set via component API */ },
     clear: () => { /* reset to null */ },
   };
   ```

4. **Define interaction metadata (M) for each tool.** Specify parameter types, valid enumerations, numeric ranges, and format constraints. Compute these at runtime from the component's current props so they reflect dynamic state (e.g., disabled dates).

   ```typescript
   // DatePicker metadata (computed at runtime)
   const metadata = {
     setValue: {
       params: { date: { type: "string", format: "YYYY-MM-DD" } },
       constraints: { min: "2025-01-01", max: "2026-12-31" },
       disabledDates: ["2025-12-25"] // from props
     }
   };
   ```

5. **Implement the component transceiver.** Embed three modules in each wrapped component:
   - **Auto-Register:** On mount, generate a unique key `K`, set `data-cid=K` on the DOM root, and register `<S, T, M>` in the global registry. On unmount, deregister.
   - **Props Listener:** Reactively monitor props/state changes, filter noise via a whitelist, and keep the semantic state view current.
   - **Dispatcher:** Route `callTool` invocations to the correct event handler or internal state setter.

6. **Expose the global registry.** Create a `window.__ci4a__` object with two methods:
   - `getStatus(key?)` -- returns the semantic triplet(s) for one or all registered components.
   - `callTool(key, toolName, params)` -- invokes a tool on the target component after validating params against metadata.

   ```typescript
   window.__ci4a__ = {
     getStatus: (key?: string) => { /* return <S, T, M> from registry */ },
     callTool: (key: string, tool: string, params: Record<string, any>) => {
       const entry = registry.get(key);
       if (!entry) throw new Error(`Component ${key} not found`);
       const meta = entry.metadata[tool];
       validateParams(params, meta); // enforce constraints
       return entry.tools[tool](params);
     }
   };
   ```

7. **Build the hybrid view for agent perception.** When the agent observes the page, traverse the DOM tree. For each node with a `data-cid` attribute, replace the subtree with a compact semantic description node containing the component's `<S, T, M>`. This dramatically shrinks the context window the agent must process.

8. **Implement semantic-first action selection in the agent.** The agent's action space is `A = A_semantic ∪ A_atomic`. On each step:
   - Query `window.__ci4a__.getStatus()` for available semantic tools.
   - Match the current intent to a semantic tool. If a match exists and metadata constraints are satisfiable, invoke it.
   - Otherwise, fall back to atomic operations (click, type, scroll).

9. **Validate and test each CI4A interface.** For every wrapped component, write tests that invoke each tool with valid and invalid parameters, verify state transitions, and confirm metadata constraints are enforced at runtime.

10. **Iterate on the component coverage.** Start with the highest-value components (forms, tables, navigation) and expand. Track which agent failures are caused by missing CI4A coverage and prioritize wrapping those components next.

## Concrete Examples

**Example 1: Wrapping an Ant Design DatePicker**

User: "Make our DatePicker agent-friendly so our LLM agent can set dates without clicking through the calendar."

Approach:
1. Create a CI4A wrapper around the Ant Design DatePicker.
2. Expose state: current value, format, disabled dates.
3. Expose tool: `setValue(date)` with format validation from metadata.
4. Register on mount, deregister on unmount.

Output:
```tsx
import React, { useEffect, useRef } from 'react';
import { DatePicker } from 'antd';
import dayjs from 'dayjs';

function CI4ADatePicker({ field, format = 'YYYY-MM-DD', disabledDate, ...props }) {
  const cid = useRef(`datepicker-${field}`);

  useEffect(() => {
    const registry = (window as any).__ci4a__ ??= { _components: {} };
    registry._components[cid.current] = {
      state: () => ({
        field,
        value: props.value ? dayjs(props.value).format(format) : null,
        format,
        disabled: !!props.disabled,
      }),
      tools: {
        setValue: (date: string) => {
          const parsed = dayjs(date, format, true);
          if (!parsed.isValid()) throw new Error(`Invalid date format, expected ${format}`);
          props.onChange?.(parsed, date);
        },
        clear: () => props.onChange?.(null, ''),
      },
      metadata: {
        setValue: {
          params: { date: { type: 'string', format } },
        },
        clear: { params: {} },
      },
    };
    return () => { delete registry._components[cid.current]; };
  }, [props.value, props.disabled]);

  return <DatePicker {...props} data-cid={cid.current} format={format} />;
}
```

**Example 2: Wrapping a Data Table with Sort and Filter**

User: "Our agent needs to sort and filter the user table without clicking column headers."

Approach:
1. Wrap the Ant Design Table to expose current sort/filter state.
2. Expose tools: `sort(column, order)`, `filter(column, values)`, `goToPage(page)`.
3. Metadata enumerates sortable columns and valid filter values from the data.

Output:
```tsx
useEffect(() => {
  const registry = (window as any).__ci4a__ ??= { _components: {} };
  registry._components[cid] = {
    state: () => ({
      columns: columns.map(c => c.dataIndex),
      sortedBy: currentSort,
      filters: currentFilters,
      page: currentPage,
      pageSize,
      totalRows: data.length,
    }),
    tools: {
      sort: ({ column, order }) => {
        setSortState({ column, order }); // triggers re-render
      },
      filter: ({ column, values }) => {
        setFilterState(prev => ({ ...prev, [column]: values }));
      },
      goToPage: ({ page }) => {
        setPagination(prev => ({ ...prev, current: page }));
      },
    },
    metadata: {
      sort: {
        params: {
          column: { type: 'string', enum: sortableColumns },
          order: { type: 'string', enum: ['ascend', 'descend'] },
        },
      },
      filter: {
        params: {
          column: { type: 'string', enum: filterableColumns },
          values: { type: 'array', items: { type: 'string' } },
        },
      },
      goToPage: {
        params: {
          page: { type: 'number', min: 1, max: Math.ceil(data.length / pageSize) },
        },
      },
    },
  };
  return () => { delete registry._components[cid]; };
}, [data, currentSort, currentFilters, currentPage]);
```

**Example 3: Agent Invoking CI4A Tools**

User: "Write the agent loop that uses CI4A to fill out a travel booking form."

Output:
```typescript
async function agentStep(intent: string, page: Page) {
  // 1. Query registry for available components
  const status = await page.evaluate(() => {
    const reg = (window as any).__ci4a__._components;
    return Object.fromEntries(
      Object.entries(reg).map(([k, v]: any) => [k, { state: v.state(), metadata: v.metadata }])
    );
  });

  // 2. LLM selects tool based on intent + available components
  const plan = await llm.plan(intent, status);
  // plan = { cid: "datepicker-checkin", tool: "setValue", params: { date: "2025-12-31" } }

  // 3. Invoke semantic tool if available
  if (plan.cid && status[plan.cid]) {
    await page.evaluate(({ cid, tool, params }) => {
      (window as any).__ci4a__._components[cid].tools[tool](params);
    }, plan);
  } else {
    // 4. Fall back to atomic operations
    await page.click(plan.selector);
  }
}
```

## Best Practices

- **Do:** Compute metadata constraints at runtime from current props/state. A date picker's disabled dates change -- stale metadata causes agent errors.
- **Do:** Keep tool granularity atomic. One tool = one state transition. Combine at the agent planning layer, not the interface layer.
- **Do:** Use the `data-cid` attribute convention consistently so the agent can discover components via a single DOM traversal.
- **Do:** Include the component's `field` or `label` in the state view so the agent can match it to task instructions by name, not position.
- **Avoid:** Exposing rendering internals (CSS classes, pixel coordinates, animation states) in the semantic state view. These add noise without aiding agent decisions.
- **Avoid:** Creating tools for non-interactive state. If a component is read-only, expose only `getStatus`, not dummy mutation tools.
- **Avoid:** Coupling CI4A logic to a specific agent framework. The `window.__ci4a__` registry should be framework-agnostic -- any agent (Playwright-based, Puppeteer-based, browser extension) can consume it.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| `Component not found` from `callTool` | Component unmounted or `cid` stale | Re-query `getStatus()` before each action to get current registry |
| Invalid parameter rejected by metadata | Agent hallucinated a date format or enum value | Return the metadata constraints in the error message so the agent can self-correct |
| Tool invocation has no visible effect | Event handler not wired, or state update batched | Verify the dispatcher routes to the actual `onChange`/`onSort` handler; add a post-invocation state check |
| Registry grows unbounded | Components mount without unmounting (memory leak) | Ensure cleanup in `useEffect` return / `onUnmounted` hook |
| Agent loops on the same action | Semantic tool call succeeds but agent doesn't perceive state change | Re-read `getStatus()` after each tool call and include updated state in the next LLM prompt |

## Limitations

- **Requires component instrumentation.** CI4A only works for components you explicitly wrap. Third-party widgets, iframes, or legacy jQuery UI lack coverage and require fallback to atomic operations.
- **One library at a time.** The paper implemented CI4A for Ant Design only. Porting to MUI, Radix, or Shadcn requires re-implementing the transceiver for each library's internal API patterns.
- **Not suitable for open-web browsing.** CI4A is designed for controlled applications where you own the frontend. It cannot wrap arbitrary websites the agent visits.
- **Metadata maintenance cost.** Runtime metadata computation adds logic to every component. For rapidly changing component APIs, keeping metadata accurate requires discipline.
- **Agent model dependency.** The technique amplifies capable models (GPT-5 went from 26.4% to 86.3%) but still depends on the LLM's ability to select the right tool and parameters from the semantic description.

## Reference

[CI4A: Semantic Component Interfaces for Agents Empowering Web Automation](https://arxiv.org/abs/2601.14790v1) -- Qiu et al., 2026. Focus on Section 3 (the semantic triplet formalization), Section 4 (the hybrid agent architecture and dynamic action space), and Table 1 (performance comparisons showing 86.3% success rate with 57.5% fewer steps).