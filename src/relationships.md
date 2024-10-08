# OMOP concept relationships

```js
import {sankey, sankeyLinkHorizontal} from "npm:d3-sankey"
```
OMOP concepts have relationships to one another. There are many different types of relationships.
It's helpful to look at the nature of the concepts in different relationships.

<details>
    <summary>If you think the vocabularies have changed</summary>
    <p>
        This SQL can be used to build <tt>./data/relationships_summary.csv</tt>
        <br/>
        <br/>
        <tt>SELECT
            c1.domain_id as from_domain,
            c1.vocabulary_id as from_vocab,
            c1.standard_concept as from_standard,
            c2.domain_id as to_domain,
            c2.vocabulary_id as to_vocab,
            c2.standard_concept as to_standard,
            cr.relationship_id as relationship,
            COUNT(c1.concept_id)
       FROM omop.concept_relationship cr
       JOIN omop.concept c1 ON cr.concept_id_1 = c1.concept_id
       JOIN omop.concept c2 ON cr.concept_id_2 = c2.concept_id
       GROUP BY
           from_domain,
           from_vocab,
           from_standard,
           to_domain,
           to_vocab,
           to_standard,
           relationship;
        </tt>
    </p>
</details>

The first thing to look at is how many relationships there are, and what the type of relationship is.
<details>
    <summary>Note</summary>
    <p>There are a lot of redundant relationship IDs, as many relationships are inverses of one another. For example, "Maps to" and "Mapped from" are the same, but with the concepts swapped. For our purposes, these are unnecessary</p>
</details>
Below is a bar chart of the counts of each relationship ID. You can filter by domain.

```js
const non_redundant_rels = [
    "Maps to",
    "Is a",
    "SPL - RxNorm",
    "Has Module",
    "Has status",
    "Concept replaces",
    "Has method",
    "ATC - RxNorm",
    "Has finding site",
    "Component of",
    "Has property",
    "Has asso morph",
    "Concept same_as from",
    "Drug class of drug",
    "Concept poss_eq from",
    "Has interprets",
    "Has access",
    "Concept was_a from",
    "ATC - RxNorm sec up",
    "Active ing of",
    "Has pathology",
    "Acc device used by",
    "Device used by",
    "Causative agent of",
    "Dose form of",
    "Source - RxNorm eq",
    "Subst used by",
    "SNOMED - RxNorm eq",
    "Dir device of",
    "Intent of",
    "Due to of",
    "Maps to value",
    "Has interpretation",
    "Has occurrence",
    "Basis str subst of",
    "Prec ingredient of",
    "Plays role",
    "Dir subst of",
    "Has disposition",
    "Has relat context",
    "Has temporal context",
    "Focus of",
    "Has clinical course",
    "ATC - RxNorm pr lat",
    "Using finding method",
    "Has inherent",
    "Has finding context",
    "Asso finding of",
    "ATC - RxNorm pr up",
    "Followed by",
    "Occurs after",
    "Modification of",
    "Has route",
    "Using finding inform",
    "Spec active ing of",
    "Asso with finding",
    "Disp dose form of",
    "SNOMED - ATC eq",
    "ATC - RxNorm sec lat",
    "Proc device of",
    "Specimen subst of",
    "Concept alt_to from",
    "Has count of ing",
    "Has dev intend site",
    "During",
    "Affected by process",
    "Comp material of",
    "Has prod character",
    "Is sterile",
    "Manifestation of",
    "Has absorbability",
    "Temp related to",
    "Has state of matter",
    "Indir device of",
    "Specimen identity of",
    "Process output of",
    "Precondition of",
    "Coating material of",
    "Has severity",
    "Has variant",
    "Filling of",
    "Surf character of",
    "Before"
    ]
```

```js
const file_input = FileAttachment("./data/relationships_summary.csv").csv();
```

```js
const relationships_summary = file_input.filter((row) => non_redundant_rels.includes(row.relationship));
```

```js
const domains = file_input.reduce((agg, row) => {
        const existing_domain = agg.find(element => element == row.from_domain)
        if (!existing_domain){
            agg.push(row.from_domain)
        }
        return agg
    }, [])
```

```js
const domain_filter = view(
    Inputs.form([
        Inputs.checkbox(domains, {multiple: true, label: "Source domain", value: domains}),
        Inputs.checkbox(domains, {multiple: true, label: "Destination domain", value: domains})
        ],
        {template: inputs => html`
        <div>
            <details><summary>Filter domains</summary>${inputs}</details>
    `}))
```

```js
const relationships_filtered_for_bar = relationships_summary.filter(
    (row) => (domain_filter[0].includes(row.from_domain)) & (domain_filter[1].includes(row.to_domain))
);
const relationship_grouped_bars = d3.rollups(relationships_filtered_for_bar, v => d3.sum(v, d => d.count), d => d.relationship)
                                    .sort((a,b) => b[1]-a[1])
```

```js
const bar_top_n = view(Inputs.range([0, relationship_grouped_bars.length], {value: 10, step: 1, label: "Show top"}))
```


```js
const top_rel_bar = relationship_grouped_bars.slice(0, bar_top_n);

display(
    Plot.plot({
        marginLeft: d3.max(top_rel_bar, (d) => d[0].length)*6,
        marks: [
            Plot.ruleX([0]),
            Plot.barX(top_rel_bar, {y: ([key]) => key, x: ([, value]) => value, sort: {y: "x", reverse: true}})
        ]}
    )
)
```

```js
const heatmap_rels = d3.rollups(relationships_filtered_for_bar,
    v => d3.sum(v, d => d.count),
    d => d.from_domain,
    d => d.to_domain
    )
    .reduce(
        (agg, group) => {
        group[1].forEach(
            (e) => {
                agg.push({from_domain: group[0], to_domain: e[0], count: e[1]})
            }
        )
        return agg
    }, []
    );
```

That's a nice little summary of the overall counts. What are the domains being mapped to and from?

Below is a heatmap for this. It's filtered on the same domains as the bar chart.

```js
const width = 640;
const height = 640;

const hm_from_domains = Array.from(new Set(heatmap_rels.map(d => d.from_domain)));
const hm_to_domains = Array.from(new Set(heatmap_rels.map(d => d.to_domain)));

const margin = {
    top: 60,
    right: 30,
    bottom: Math.max(...(hm_from_domains.map(el => el.length)))*6,
    left: Math.max(...(hm_to_domains.map(el => el.length)))*6
};

const heatmap = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto;");

const g = heatmap.append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

const hm_x = d3.scaleBand()
    .range([0, width - margin.left - margin.right])
    .domain(hm_from_domains)
    .padding(0.05);

g.append("g")
    .attr("transform", `translate(0,${height - margin.top - margin.bottom})`)
    .call(d3.axisBottom(hm_x))
    .selectAll("text")
    .attr("transform", "rotate(-45)")
    .style("text-anchor", "end");

const hm_y = d3.scaleBand()
    .range([height - margin.top - margin.bottom, 0])
    .domain(hm_to_domains)
    .padding(0.05);

g.append("g")
    .call(d3.axisLeft(hm_y));

// Build color scale
const myColor = d3.scaleSequential()
    .interpolator(d3.interpolateRgbBasis(["purple", "green"]))
    .domain([0, d3.max(heatmap_rels, d => d.count)]);

// Create a tooltip
const tooltip = d3.select(document.createElement("div"))
    .style("opacity", 0)
    .attr("class", "tooltip")
    .style("background-color", "white")
    .style("border", "solid")
    .style("border-width", "2px")
    .style("border-radius", "5px")
    .style("padding", "5px")
    .style("position", "absolute");

document.body.appendChild(tooltip.node());

// Three functions that change the tooltip when users hover / move / leave a cell
const mouseover = function(event, d) {
    tooltip.style("opacity", 1);
    d3.select(this)
        .style("stroke", "black")
        .style("opacity", 1);
}

const mousemove = function(event, d) {
    tooltip.html(`From: ${d.from_domain}<br>To: ${d.to_domain}<br>Count: ${d.count}`)
        .style("left", (event.pageX + 10) + "px")
        .style("top", (event.pageY - 10) + "px")
        .style("color", "black");
}

const mouseleave = function(event, d) {
    tooltip.style("opacity", 0);
    d3.select(this)
        .style("stroke", "none")
        .style("opacity", 0.8);
}

// Add the squares
g.selectAll()
    .data(heatmap_rels)
    .enter()
    .append("rect")
        .attr("x", d => hm_x(d.from_domain))
        .attr("y", d => hm_y(d.to_domain))
        .attr("rx", 4)
        .attr("ry", 4)
        .attr("width", hm_x.bandwidth())
        .attr("height", hm_y.bandwidth())
        .style("fill", d => myColor(d.count))
        .style("stroke-width", 4)
        .style("stroke", "none")
        .style("opacity", 0.8)
    .on("mouseover", mouseover)
    .on("mousemove", mousemove)
    .on("mouseleave", mouseleave);

// Add title to graph
heatmap.append("text")
    .attr("x", width / 2)
    .attr("y", margin.top / 2)
    .attr("text-anchor", "middle")
    .style("font-size", "22px")
    .style("stroke", "#FFFFFF")
    .text("Heatmap of Domain Relationships");

display(heatmap.node());
```

I'm sorry choosing domains is a pain.

The heatmap shows what is being mapped, but not the nature of the relationships. The diagram below shows a more detailed view of the nature of relationships. The domain being mapped from is shown on the left, and the domains it is mapped to is shown on the right. The stripes running left to right show are thick proportional to the count of relationships, coloured by the type of relationship. You can hover over the stripe to see what the relationship is.

```js
const m_graph_source = view(Inputs.select(domains))
```

```js
const m_graph_filtered = relationships_summary.filter(el => el.from_domain == m_graph_source)
const m_graph_rollup = d3.rollups(m_graph_filtered, v => d3.sum(v, d => d.count), d => d.to_domain, d => d.relationship)

const m_graph_nodes = [
    {name: "Source: " + m_graph_source},
    ...m_graph_rollup.map(e => {return {name: e[0]}})
]

const m_graph_links = m_graph_rollup.reduce(
    (agg, e) => {
        const links = e[1].map(rel => {
            return {
                source: "Source: " + m_graph_source,
                target: e[0],
                value: rel[1],
                type: rel[0],
                color: "red",
            }
        })
        return [...agg, ...links]
        return agg
    }, []
)

const m_graph_data = {
    nodes: m_graph_nodes,
    links: m_graph_links
}
```

```js
// Set up dimensions
const sankey_width = 640;
const sankey_height = 400;
const sankey_margin = {top: 10, right: 120, bottom: 10, left: 10};

const sankey_color = d3.scaleOrdinal(d3.schemeCategory10);

// Set up the sankey generator
const sankeyGenerator = sankey()
    .nodeId(d => d.name)
    .nodeWidth(15)
    .nodePadding(10)
    .extent([[sankey_margin.left, sankey_margin.top], [sankey_width - sankey_margin.right, sankey_height - sankey_margin.bottom]]);

// Generate the sankey data
const {nodes, links} = sankeyGenerator({
    nodes: m_graph_data.nodes.map(d => Object.assign({}, d)),
    links: m_graph_data.links.map(d => Object.assign({}, d))
  });

const svg = d3.create("svg")
    .attr("viewBox", [0, 0, sankey_width, sankey_height])
    .attr("width", sankey_width)
    .attr("height", sankey_height)
    .attr("style", "max-width: 100%; height: auto;");

// Add the links
const link = svg.append("g")
    .selectAll("path")
    .data(links)
    .join("path")
      .attr("d", sankeyLinkHorizontal())
      .attr("fill", "none")
      .attr("stroke", d => sankey_color(d.type))
      .attr("stroke-width", d => Math.max(1, d.width))
      .attr("opacity", 0.5);

link.append("title")
    .text(d => `${d.source.name} → ${d.target.name}\n${d.value} ${d.type}`);

// Add the nodes
const node = svg.append("g")
    .selectAll("rect")
    .data(nodes)
    .join("rect")
      .attr("x", d => d.x0)
      .attr("y", d => d.y0)
      .attr("height", d => d.y1 - d.y0)
      .attr("width", d => d.x1 - d.x0)
      .attr("fill", "#bbb");

node.append("title")
    .text(d => `${d.id}\n${d.value}`);

// Add the labels
const label = svg.append("g")
    .attr("font-family", "sans-serif")
    .attr("font-size", 10)
    .selectAll("text")
    .data(nodes)
    .join("text")
      .attr("x", d => d.x0 < sankey_width / 2 ? d.x1 + 6 : d.x0 - 6)
      .attr("y", d => (d.y1 + d.y0) / 2)
      .attr("dy", "0.35em")
      .attr("text-anchor", d => d.x0 < sankey_width / 2 ? "start" : "end")
      .text(d => d.name);

// I had a legend here, but there are often too many relationship IDs for it to be useful, so tooltips it is

display(svg.node());
```

I had thought to show more domains on the left, but that would be too chaotic.

<!--
TODO: There are a few million relationships in OMOP's relationships table. It wouldn't be unfeasible to visualise this as a graph. It would be a bit big, but maybe?
-->
