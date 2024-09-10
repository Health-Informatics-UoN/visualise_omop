---
toc: false
---

<div class="hero">
  <h1>Visualising OMOP</h1> 
</div>

It's hard to get an idea of what the structure of the OMOP vocabularies are, so here's a nice interactive treemap to help.
I've split the database in a slightly artificial hierarchy that runs:

1. Domain
2. Vocabulary
3. Concept class
4. Standard/Non-standard/Classification concept

I know that vocabularies can contain concepts with several domains, but a treemap needs a hierarchy, and this is the best way to split things by my conception of the database.

Click on it! When you click, it expands the box you've selected! To go back up a level, click the white bar at the top.
  
```js
const concepts = FileAttachment("./data/all_concepts.csv").csv();
```

```js
const concept_hierarchy = concepts.reduce((acc, row) => {
    const existing_domain = acc.find(element => element.name == row.domain_id);
    if (existing_domain) {
        const existing_vocab = existing_domain["children"].find(element => element.name == row.vocabulary_id);
        if(existing_vocab) {
            const existing_concept_class = existing_vocab["children"].find(element => element.name == row.concept_class_id);
            if (existing_concept_class) {
                existing_concept_class["children"].push({
                    "name": row.standard_concept === "S" ? "Standard concept": row.standard_concept === "C" ? "Classification": "Non-standard concept",
                    "value": row.count
                })
            } else {
                existing_vocab["children"].push({
                    "name": row.concept_class_id,
                    "children": [
                        {
                            "name": row.standard_concept === "S" ? "Standard concept": row.standard_concept === "C" ? "Classification": "Non-standard concept",
                            "value": row.count
                        }

                    ]
                })
            }
        } else {
            existing_domain["children"].push({
                "name": row.vocabulary_id,
                "children": [
                    {
                        "name": row.concept_class_id,
                        "children": [
                            {
                                "name": row.standard_concept === "S" ? "Standard concept": row.standard_concept === "C" ? "Classification": "Non-standard concept",
                                "value": row.count
                            }
                        ]
                    }
                ]
            })
        }
    } else {
        acc.push({
            "name": row.domain_id,
            "children": [
                {
                    "name": row.vocabulary_id,
                    "children": [
                        {
                            "name": row.concept_class_id,
                            "children": [
                                {
                                    "name": row.standard_concept === "S" ? "Standard concept" : row.standard_concept === "C" ? "Classification" : "Non-standard concept",
                                    "value": row.count
                                }
                            ]
                        }
                    ]
                }
            ]
        })
    }
    return acc
    }, [])
```

```js
  // I took this pretty much whole from https://observablehq.com/@d3/zoomable-treemap
  // Specify the chart’s dimensions.
  const width = 900;
  const height = 600;

  // This custom tiling function adapts the built-in binary tiling function
  // for the appropriate aspect ratio when the treemap is zoomed-in.
  function tile(node, x0, y0, x1, y1) {
    d3.treemapBinary(node, 0, 0, width, height);
    for (const child of node.children) {
      child.x0 = x0 + child.x0 / width * (x1 - x0);
      child.x1 = x0 + child.x1 / width * (x1 - x0);
      child.y0 = y0 + child.y0 / height * (y1 - y0);
      child.y1 = y0 + child.y1 / height * (y1 - y0);
    }
  }
  var count = 0;
  
  function Id(id) {
    this.id = id;
    this.href = new URL(`#${id}`, location) + "";
  }

  Id.prototype.toString = function() {
    return "url(" + this.href + ")";
  };

  function uid(name) {
    return new Id("O-" + (name == null ? "" : name + "-") + ++count);
  }

  // Compute the layout.
  const hierarchy = d3.hierarchy({
      "name": "OMOP",
      "children": concept_hierarchy
  })
    .sum(d => d.value)
    .sort((a, b) => b.value - a.value);
  const root = d3.treemap().tile(tile)(hierarchy);

  // Create the scales.
  const x = d3.scaleLinear().rangeRound([0, width]);
  const y = d3.scaleLinear().rangeRound([0, height]);

  // Formatting utilities.
  const format = d3.format(",d");
  const name = d => d.ancestors().reverse().map(d => d.data.name).join("/");

  // Create the SVG container.
  const svg = d3.create("svg")
      .attr("viewBox", [0.5, -30.5, width, height + 30])
      .attr("width", width)
      .attr("height", height + 30)
      .attr("style", "max-width: 100%; height: auto;")
      .style("font", "10px sans-serif");

  // Display the root.
  let group = svg.append("g")
      .call(render, root);

  function render(group, root) {
    const node = group
      .selectAll("g")
      .data(root.children.concat(root))
      .join("g");

    node.filter(d => d === root ? d.parent : d.children)
        .attr("cursor", "pointer")
        .on("click", (event, d) => d === root ? zoomout(root) : zoomin(d));

    node.append("title")
        .text(d => `${name(d)}\n${format(d.value)}`);

    node.append("rect")
        .attr("id", d => (d.leafUid = uid("leaf")).id)
        .attr("fill", d => d === root ? "#fff" : d.children ? "#ccc" : "#ddd")
        .attr("stroke", "#fff");

    node.append("clipPath")
        .attr("id", d => (d.clipUid = uid("clip")).id)
      .append("use")
        .attr("xlink:href", d => d.leafUid.href);

    node.append("text")
        .attr("clip-path", d => d.clipUid)
        .attr("font-weight", d => d === root ? "bold" : null)
      .selectAll("tspan")
      .data(d => (d === root ? name(d) : d.data.name).split(/(?=[A-Z][^A-Z])/g).concat(format(d.value)))
      .join("tspan")
        .attr("x", 5)
        .attr("y", (d, i, nodes) => `${(i === nodes.length - 1) * 0.3 + 1.1 + i * 0.7}em`)
        .attr("fill-opacity", (d, i, nodes) => i === nodes.length - 1 ? 0.7 : null)
        .attr("font-weight", (d, i, nodes) => i === nodes.length - 1 ? "normal" : null)
        .text(d => d);

    group.call(position, root);
  }

  function position(group, root) {
    group.selectAll("g")
        .attr("transform", d => d === root ? `translate(0,-30)` : `translate(${x(d.x0)},${y(d.y0)})`)
      .select("rect")
        .attr("width", d => d === root ? width : x(d.x1) - x(d.x0))
        .attr("height", d => d === root ? 30 : y(d.y1) - y(d.y0));
  }

  // When zooming in, draw the new nodes on top, and fade them in.
  function zoomin(d) {
    const group0 = group.attr("pointer-events", "none");
    const group1 = group = svg.append("g").call(render, d);

    x.domain([d.x0, d.x1]);
    y.domain([d.y0, d.y1]);

    svg.transition()
        .duration(750)
        .call(t => group0.transition(t).remove()
          .call(position, d.parent))
        .call(t => group1.transition(t)
          .attrTween("opacity", () => d3.interpolate(0, 1))
          .call(position, d));
  }

  // When zooming out, draw the old nodes on top, and fade them out.
  function zoomout(d) {
    const group0 = group.attr("pointer-events", "none");
    const group1 = group = svg.insert("g", "*").call(render, d.parent);

    x.domain([d.parent.x0, d.parent.x1]);
    y.domain([d.parent.y0, d.parent.y1]);

    svg.transition()
        .duration(750)
        .call(t => group0.transition(t).remove()
          .attrTween("opacity", () => d3.interpolate(1, 0))
          .call(position, d))
        .call(t => group1.transition(t)
          .call(position, d.parent));
  }
  const concept_treemap = display(svg.node());
```
---

<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
