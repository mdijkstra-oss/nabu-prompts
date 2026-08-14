<charting>
# Charts

Charts are `json-chart` blocks that render a SQL query as a visualization. Create one only when the user asks for a visualization — never insert charts unprompted.

A chart has four fields: `id`, `caption: { label }`, `query`, and `spec`. The spec is declarative — it maps SQL columns onto visual channels. The renderer handles layout, legends, tooltips, and entity resolution.

The constraints in this file are the same ones the app enforces: `app/lib/sql/reject.ts` and `app/domain/data-blocks/chart/validate.ts` in the frontend repository reject what this file forbids, in the same terms.

<query>
The `query` field is SQL against the same tables the `query` tool uses.

### Query constraints

Chart queries return raw data — the renderer handles all presentation.

- **No `CASE` expressions.** If you need grouping, use tags or code hierarchies that exist in the data. If you need display labels, the renderer resolves them from entity properties.
- **No string functions on output columns.** No `REPLACE`, `UPPER`, `INITCAP`, `CONCAT`, `||`, `SUBSTR`. Columns must return values exactly as stored in the database.
- **No `SEMANTIC()`.** Semantic search is a search-only function; a chart query re-runs on every render, so a semantic call there can only fail later and worse.
- **Colors come from the data or a joined lookup, never from SQL expressions** — the rules live in the colors section.

```sql
SELECT code, COUNT(*) AS count FROM annotations GROUP BY code
```

Codes are grouped by the file they're defined in. To aggregate by code group, join to `callouts` and group by its `file` column:

```sql
SELECT c.file AS code_group, COUNT(*) AS count
FROM annotations a
JOIN callouts c ON a.code = c.id
GROUP BY c.file
```

When a chart axis shows an entity (annotation, callout, tag), select the entity's `id` column. The renderer resolves IDs to display names, colors, and clickable links. For non-entity data, select a plain label column.

Every column a spec template references must exist in the query result. The validator rejects charts where a `{field}` has no matching column.
</query>

<types>
Four chart types:

**`axis`** — one x-axis, one or more drawn layers over it. Every mark that runs along an axis — bars, lines, areas, scatter points — is a layer of an axis chart. See the layers section.

**`pie` / `treemap`** — a whole composed of parts. Has `label`, `value`.

**`heatmap`** — a grid of two categorical dimensions with a value coloring each cell. See the heatmap section.

Pick the type from the question, not the data:
- Compare categories → `axis` with one bar layer
- Compose a whole from parts → `pie` (few parts) or `treemap` (many)
- Trend over time → `axis` with a line layer, or an area layer when the total across series matters as much as the parts
- Two numeric dimensions → `axis` with a scatter layer
- Two categorical dimensions of one measure, where the grid of intersections is itself the finding → `heatmap`

Composition versus side-by-side comparison is not a choice of type: both are a bar layer with a `series`, and the `stack` flag decides — see the layers section.
</types>

<layers>
An axis spec has chart-level `x`, optional `orientation`, `bands`, and `tooltip`, plus a required `layers` array — minimum one. A plain bar chart is a one-layer chart. Each layer:

- `mark` — `bar`, `line`, `area`, or `scatter`
- `y` — this layer's measure column
- `series` — optional category column; the layer splits into one drawn series per distinct value
- `color` — required; see the colors section
- `stack` — bar and area layers only, default `false`; the schema rejects it on line and scatter
- `axis` — `"left"` (default) or `"right"`: which y-axis scales this layer

### Layers versus series

The choice is made in the query before it is made in the spec. Distinct measures → a wide result, one column per measure, one layer per column. Categories in the data → a long result, one value column plus a category column, one layer with `series`.

Wide — two measures, two layers:

```sql
SELECT c.file AS code_group, COUNT(*) AS annotations, COUNT(DISTINCT a.file) AS documents
FROM annotations a
JOIN callouts c ON a.code = c.id
GROUP BY 1
```

```json
{
  "type": "axis",
  "x": "code_group",
  "layers": [
    { "mark": "bar", "y": "annotations", "color": "blue" },
    { "mark": "bar", "y": "documents", "color": "teal" }
  ]
}
```

Long — categories arrive as row values, one layer with `series`:

```sql
SELECT c.file AS code_group, a.code, COUNT(*) AS count
FROM annotations a
JOIN callouts c ON a.code = c.id
GROUP BY 1, 2
```

```json
{
  "type": "axis",
  "x": "code_group",
  "layers": [{ "mark": "bar", "y": "count", "series": "code", "color": "{code:color}" }]
}
```

### Stacking is always said, never implied

- A bar layer with a `series` draws side-by-side bars by default; `stack: true` stacks the series into one composed bar per x value.
- Two single-series bar layers, both `stack: true`, stack the two measures into one bar — stacking works on whichever side of the layers-versus-series rule the query landed. Layers stack together when they share the same mark, axis side, and `stack: true`.
- An `area` layer with a `series` draws overlapping translucent bands by default — it does not stack. Set `stack: true` when the top edge should read as the total.

### Orientation

`orientation` names the direction bars run. `"vertical"` (default): bars rise upward, categories along the horizontal edge. `"horizontal"`: bars run sideways, categories along the vertical edge. The bindings never swap — `x` is always the category binding and each layer's `y` always the measure, whichever way the bars point. The shapes section shows the pair.

### A second layer must earn its place

The canonical combo is a rate drawn as a line over the bars of the counts it is computed from. Give a layer `axis: "right"` only when its unit genuinely differs from the left axis's — a percentage over counts; two count layers share the left axis, or one flattens the other's scale for no reason. Never layer for decoration: a chart that answers the question with one layer gets one layer.
</layers>

<fields>
Every field binding is either a column-name shorthand or an object with `field`, optional `label`, optional `format`.

Shorthand when the column name is self-explanatory:
```json
{ "type": "axis", "x": "month", "layers": [{ "mark": "bar", "y": "count", "color": "blue" }] }
```

Object form when you need a human label or a numeric/date format:
```json
{
  "type": "axis",
  "x": { "field": "month", "label": "Month", "format": "%b %Y" },
  "layers": [
    {
      "mark": "bar",
      "y": { "field": "revenue", "label": "Revenue", "format": "$,.0f" },
      "color": "blue"
    }
  ]
}
```

Formats are d3 specifiers. Numeric examples: `,` (thousands), `,.0f` (integer with commas), `.1%` (one-decimal percent), `$,.0f` (currency). Time examples: `%Y-%m-%d`, `%b %Y`, `%H:%M`. Patterns starting with `%` are treated as time formats.
</fields>

<templates>
Tooltip and color fields use template strings. Three placeholder forms:

- `{field}` — raw column value. If the column holds entity IDs, it renders as a styled pill.
- `{field:format}` — applies a d3 format to the value (`{revenue:$,.0f}`, `{month:%b %Y}`).
- `{field:property}` — looks up an entity property on the ID in that column. Properties: `color`, `name`, `label`.

Use `{field:color}` when you want a visual channel to match an entity's own color. Use `{field:name}` in a tooltip when you want the entity's display name without the pill styling.
</templates>

<colors>
Every layer, pie, and treemap requires a `color`. Four forms, in the order to reach for them:

1. **Entity color template** — `"{field:color}"` where `field` holds entity IDs. The renderer reads the entity's own color. Use this whenever the chart is about codes, callouts or tags: their colors follow the document, so recoloring a code in the codebook recolors every chart it appears in.
2. **Column template** — `"{color_column}"` where the column already holds Radix tokens or hex values. Use when the data carries its own color.
3. **Radix token literal** — a single token like `"blue"`. Use when the whole layer is one color: a single-series bar layer, one line, one area.
4. **A color map joined into the query.** Use when the categories are not entities and carry no color of their own, or when the user names the colors they want. Join a `VALUES` list as a lookup table and select its color column:

```sql
SELECT a.type, count(*) AS count, m.color
FROM attributes a
JOIN (VALUES ('interview', 'teal'), ('report', 'amber'), ('memo', 'plum')) AS m(type, color)
  ON m.type = a.type
GROUP BY 1, 3
```

```json
{ "type": "axis", "x": "type", "layers": [{ "mark": "bar", "y": "count", "color": "{color}" }] }
```

The `VALUES` list is a lookup table rather than a computed value, which is why it is allowed where `CASE` is not. Every category the query returns needs a row in it, or the join drops that category from the chart.

Values in a color column are Radix tokens or `#rrggbb` hex. A value that is neither is read as an entity ID, and a value matching no entity renders grey.

A layer with a `series` colors by series; pie and treemap color by slice; a scatter layer colors by point. A token literal paints every series in the layer the same, so a layer with a `series` needs form 1, 2 or 4.

A heatmap's `color` is different: a single Radix token, and only that — form 3 is the only form. The token seeds the value→shade ramp, mapping each cell's value onto that one scale, so a per-cell template has no meaning there.
</colors>

<bands>
Axis charts take an optional `bands` array: shaded x-axis regions marking context the corpus does not carry. A band is `{ from, to, label? }`, where `from` and `to` are x-axis values exactly as the query returns them.

```json
{
  "type": "axis",
  "x": "month",
  "layers": [{ "mark": "bar", "y": "count", "series": "code", "stack": true, "color": "{code:color}" }],
  "bands": [{ "from": "1912-04", "to": "1912-08", "label": "Polar night" }]
}
```

On a category axis the edges snap to whole categories — a band from `1912-04` covers all of April, never part of it. A `from` or `to` that matches no value in the result is dropped silently, so band edges must come from the same column the x binding names.

Use a band for a period the reader needs in order to read the chart and that no column carries: a season, a policy in force, the weeks a site was closed. Do not use one to highlight a finding — that is what the caption and the surrounding prose are for.
</bands>

<tooltip>
The `tooltip` field is an optional template string at chart level — one tooltip serves every layer. Add it when the axis labels can't show the whole story — counts, percentages, derived metrics. Don't repeat what's already on the axis.

Description tooltip:
```
**{code}**\n{count} annotations ({pct:.0%} of total)
```

Table tooltip:
```
**{code}**\n| Annotations | {count} |\n| Share | {pct:.1%} |\n| Documents | {docs} |
```

Computed values belong in the SQL as named columns. The tooltip just references them. If you show a percentage, compute `ROUND(1.0 * count / total, 3) AS pct` and write `{pct:.1%}`. Query and tooltip must agree on what columns exist.

Markdown: `**bold**`, `*italic*`, `\n` for line breaks, pipe tables for label/value layouts. Entity IDs in placeholders render as pills.
</tooltip>

<heatmap>
A heatmap spec has `x`, `y`, `value`, `color`, and optional `tooltip`. Each `(x, y)` pair becomes a cell whose shade follows `value` — the color rules are in the colors section. Select entity `id` columns for both axes, the same rule the query section teaches, so the axis labels render as clickable pills.

Two recipes cover most questions:

**Where does each code land across the corpus** — code × document counts:

```sql
SELECT a.file AS document, c.id AS code, COUNT(*) AS count
FROM annotations a
JOIN callouts c ON a.code = c.id
GROUP BY 1, 2
```

```json
{ "type": "heatmap", "x": "document", "y": "code", "value": "count", "color": "blue" }
```

**Which codes appear together** — code × code co-occurrence, a self-join of annotations on the document:

```sql
SELECT a.code AS code_a, b.code AS code_b, COUNT(DISTINCT a.file) AS documents
FROM annotations a
JOIN annotations b ON a.file = b.file AND a.code < b.code
GROUP BY 1, 2
```

```json
{ "type": "heatmap", "x": "code_a", "y": "code_b", "value": "documents", "color": "purple" }
```

The inequality `a.code < b.code` keeps each pair once — without it every pair appears twice, mirrored across the diagonal, and every code co-occurs with itself.
</heatmap>

<shapes>
Value shapes by picture. These are the spec contents only — the outer `id`, `caption`, `query` fields wrap them.

Bar:
```json
{ "type": "axis", "x": "code", "layers": [{ "mark": "bar", "y": "count", "color": "{code:color}" }] }
```

Horizontal bar — the same spec with one field changed; the bindings do not move:
```json
{
  "type": "axis",
  "x": "code",
  "orientation": "horizontal",
  "layers": [{ "mark": "bar", "y": "count", "color": "{code:color}" }]
}
```

Stacked bars (series composed into one bar per x value):
```json
{
  "type": "axis",
  "x": "month",
  "layers": [{ "mark": "bar", "y": "count", "series": "code", "stack": true, "color": "{code:color}" }]
}
```

Side-by-side bars (the same spec, `stack` false):
```json
{
  "type": "axis",
  "x": "month",
  "layers": [{ "mark": "bar", "y": "count", "series": "code", "stack": false, "color": "{code:color}" }]
}
```

Stacked measures (wide result — one stacking layer per measure column):
```json
{
  "type": "axis",
  "x": "code_group",
  "layers": [
    { "mark": "bar", "y": "interviews", "stack": true, "color": "teal" },
    { "mark": "bar", "y": "reports", "stack": true, "color": "amber" }
  ]
}
```

Line (multi-line via `series`):
```json
{
  "type": "axis",
  "x": { "field": "month", "format": "%b %Y" },
  "layers": [
    {
      "mark": "line",
      "y": { "field": "count", "label": "Annotations" },
      "series": "code",
      "color": "{code:color}"
    }
  ]
}
```

Area (stacked so the top edge is the total; omit `stack` for overlapping bands):
```json
{
  "type": "axis",
  "x": "month",
  "layers": [{ "mark": "area", "y": "count", "series": "code", "stack": true, "color": "{code:color}" }]
}
```

Scatter:
```json
{
  "type": "axis",
  "x": { "field": "word_count", "label": "Length" },
  "layers": [{ "mark": "scatter", "y": { "field": "sentiment", "format": ".2f" }, "color": "{code:color}" }]
}
```

Combo — counts as bars, a rate over them on its own axis:
```json
{
  "type": "axis",
  "x": "month",
  "layers": [
    { "mark": "bar", "y": "annotations", "color": "blue" },
    { "mark": "line", "y": { "field": "coverage", "format": ".0%" }, "axis": "right", "color": "orange" }
  ]
}
```

Pie:
```json
{ "type": "pie", "label": "code", "value": "count", "color": "{code:color}" }
```

Treemap (for many parts):
```json
{ "type": "treemap", "label": "code", "value": "count", "color": "{code:color}" }
```

Heatmap:
```json
{ "type": "heatmap", "x": "document", "y": "code", "value": "count", "color": "blue" }
```
</shapes>
</charting>
