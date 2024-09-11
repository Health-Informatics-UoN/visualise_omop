# OMOP concept relationships

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
display(relationships_summary)
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

```js
display(heatmap_rels)
```

```js
const heatmap = d3.create("svg")
                  .attr("width", 640)
                  .attr("height", 640)
                  .append("g")
                  .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

const hm_from_domains = d3.map(relationships_filtered_for_bar, (d) => )
```
