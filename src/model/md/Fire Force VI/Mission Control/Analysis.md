---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Fire Force Analysis Layer

This notebook audits the Mission Control model against Sierra method expectations. Every value on the page is computed live from the model. The prose explains what each result means rather than what today's numbers are, so it stays true as the model changes.

## 1. Conformance: are Connections wired at a level the method can check?

**Question.** Sierra's Connection rule (an Out port must connect to an In port) is only evaluated between sibling Components, and the Connections diagram only draws a Component with its direct children. Does every Connection join siblings, or a parent and its direct child, so both the rule and the diagram can see it?

```chart
---
type: bar
data:
  labels: Status
  datasets:
    - label: Connections
      data: Count
options:
  plugins:
    title:
      display: true
      text: Connection wiring conformance
    legend:
      display: false
  scales:
    y:
      beginAtZero: true
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

# One row = one status bucket. VALUES creates both, so an empty bucket shows as 0.
SELECT ?Status (COALESCE(?n, 0) AS ?Count)
WHERE {
  VALUES ?Status { "Conforms" "Violates" }
  OPTIONAL {
    SELECT ?Status (COUNT(DISTINCT ?conn) AS ?n)
    WHERE {
      ?conn a component:Connection ;
            oml:hasSource ?src ;
            oml:hasTarget ?tgt .
      ?from component:hasPort ?src .
      ?to component:hasPort ?tgt .
      OPTIONAL { ?from base:isContainedBy ?fromParent }
      OPTIONAL { ?to base:isContainedBy ?toParent }
      BIND(IF((BOUND(?fromParent) && BOUND(?toParent) && ?fromParent = ?toParent)
              || (BOUND(?toParent) && ?toParent = ?from)
              || (BOUND(?fromParent) && ?fromParent = ?to),
              "Conforms", "Violates") AS ?Status)
    }
    GROUP BY ?Status
  }
}
ORDER BY ?Status
```

```table
---
orderBy: ["Status desc", "Connection asc"]
stylesheet:
  - selector: cell[col === "Status" && value === "Violates"]
    target: value
    style:
      color: "#DC2626"
      font-weight: 600
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

SELECT ?Connection ?Status ?From ?FromContainer ?To ?ToContainer
WHERE {
  ?Connection a component:Connection ;
              oml:hasSource ?src ;
              oml:hasTarget ?tgt .
  ?From component:hasPort ?src .
  ?To component:hasPort ?tgt .
  OPTIONAL { ?From base:isContainedBy ?FromContainer }
  OPTIONAL { ?To base:isContainedBy ?ToContainer }
  BIND(IF((BOUND(?FromContainer) && BOUND(?ToContainer) && ?FromContainer = ?ToContainer)
          || (BOUND(?ToContainer) && ?ToContainer = ?From)
          || (BOUND(?FromContainer) && ?FromContainer = ?To),
          "Conforms", "Violates") AS ?Status)
}
```
## 2. Near miss: Stakeholders served by a single path

**Question.** The method's Mission / Stakeholder matrix counts Objective and Concern paths from a Mission to each Stakeholder and highlights more than one as well served. Which Stakeholders sit exactly one path below that threshold, so a single broken link would leave them unserved?

```table
---
orderBy: ["Stakeholder asc"]
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

# Same path the Mission / Stakeholder matrix counts.
SELECT ?Stakeholder ?Objective ?Concern
WHERE {
  {
    SELECT ?Stakeholder
    WHERE {
      ?m a mission:Mission ;
         mission:pursues ?o .
      ?o mission:isDerivedFrom ?c .
      ?c stakeholder:isExpressedBy ?Stakeholder .
    }
    GROUP BY ?Stakeholder
    HAVING (COUNT(*) = 1)
  }
  ?mission mission:pursues ?Objective .
  ?Objective mission:isDerivedFrom ?Concern .
  ?Concern stakeholder:isExpressedBy ?Stakeholder .
}
```

**Interpretation.** Each row is one Concern and one Objective away from being unrepresented. That is acceptable if the Stakeholder is genuinely narrow in scope, but it deserves a second look: another Objective derived from the same Concern, or another Concern elicited from the Stakeholder, would make coverage resilient to a single change.

## 3. Capabilities with no Process behind them

**Question.** Does every Capability have at least one Process describing how it is carried out?

**Evidence.**

```text
---
stylesheet:
  - selector: paragraph[index === 0]
    style: { font-weight: "bold" }
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>

SELECT ?Finding
WHERE {
  { SELECT (COUNT(DISTINCT ?c) AS ?total) WHERE { ?c a mission:Capability } }
  { SELECT (COUNT(DISTINCT ?c) AS ?orphans)
    WHERE { ?c a mission:Capability .
            FILTER NOT EXISTS { ?p process:describes ?c } } }
  BIND(CONCAT(STR(?orphans), " of ", STR(?total),
       " Capabilities have no Process recorded.") AS ?Finding)
}
```

```table
---
orderBy: ["Capability asc"]
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>

# One row = one Capability with no process:describes link in this scope.
SELECT ?Capability ?Description ?Objective
WHERE {
  ?Capability a mission:Capability .
  OPTIONAL { ?Capability base:description ?Description }
  OPTIONAL { ?Capability mission:isRequiredBy ?Objective }
  FILTER NOT EXISTS { ?process process:describes ?Capability }
}
```

**Interpretation.** A Capability without a Process is something the system is supposed to do with no modeled account of how. The Processes editor requires every Process to describe a Capability (process:describes, sh:minCount 1), but no shape checks the opposite direction, so every Capability and Process can be individually valid while most Capabilities have nothing behind them. This is a closed-world check over this page's scope: the accurate reading is "no Process is recorded," not "none exists." Because the relation already exists in the vocabulary, this is a process gap rather than a vocabulary gap. For each row, model a Process with its Activities and Flows, or record why the Capability does not need one.

## 4. Coverage: requirement priority against concern priority

This view is defined once in the Sierra method as a compose template and reused here and on the Coverage Review page.

```compose
template: https://www.modelware.io/sierra/operational-analysis/requirement-coverage
```

## 5. View graph: from Mission to Capability to Process

**Question.** Which Capabilities does the Mission depend on, and which of them are realized by a modeled Process?

```graph
---
layout: { mode: force, fit: true }
group: { byPredicate: true }
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>
PREFIX analysis: <http://example.org/fireforce/analysis#>

CONSTRUCT {
  ?mission analysis:needsCapability ?capability .
  ?capability analysis:realizedBy ?process .
  ?mission analysis:hasUnsupportedObjective ?orphan .
}
WHERE {
  {
    ?objective mission:isPursuedBy ?mission ;
               mission:requires ?capability .
    OPTIONAL { ?process process:describes ?capability }
  }
  UNION
  {
    ?orphan a mission:Objective ;
            mission:isPursuedBy ?mission .
    FILTER NOT EXISTS { ?orphan mission:requires ?anyCapability }
  }
}
```

**Interpretation.** This graph is not a copy of the model. CONSTRUCT collapses the Objective layer into a direct Mission to Capability edge and follows each Capability to the Process that realizes it. Capabilities with no outgoing edge are the dead ends from section 3, seen in context. An Objective with no Capability would hang off the Mission on its own edge type. The graph shows structural dependency, not whether a Process is sufficient.

## 6. Scripted analysis: physical part masses

**Question.** What is the total leaf mass once units are normalized, and which part masses look like data-entry errors?

SPARQL selects each physical part with its mass value and unit (the same pattern Sierra's Masses template uses). Python normalizes units, computes the statistics, and renders the result.

```python
import statistics

# 1. fetch: one row per physical part; values come back as strings
result = await query("""
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
SELECT ?part ?description ?value ?unit ?multiplier
WHERE {
  ?part a component:PhysicalPart ;
        component:mass ?q .
  ?q oml:value ?value .
  OPTIONAL { ?q oml:unit ?unit . OPTIONAL { ?unit oml:multiplier ?multiplier } }
  OPTIONAL { ?part base:description ?description }
}
""")
rows = result["rows"]

def frag(iri):
    return str(iri or "").rsplit("#", 1)[-1].rsplit("/", 1)[-1]

def fmt(x, d=2):
    return "n/a" if x is None else f"{x:.{d}f}"

FALLBACK = {"kg": 1.0, "g": 0.001}   # only used if a unit carries no multiplier

# 2. compute: normalize every mass to kg, then run the statistics
parts = {}
for r in rows:
    name = frag(r.get("part"))
    raw = float(r.get("value"))
    m = r.get("multiplier")
    mult = float(m) if m else FALLBACK.get(frag(r.get("unit")), 1.0)
    parts[name] = {"name": name, "desc": r.get("description") or "",
                   "raw": raw, "kg": raw * mult, "nonKg": abs(mult - 1.0) > 1e-9}
parts = list(parts.values())

n = len(parts)
if n == 0:
    display("<p><em>No physical part masses found in this scope.</em></p>")
else:
    masses = [p["kg"] for p in parts]
    total = sum(masses)
    naive_total = sum(p["raw"] for p in parts)   # what an unconverted sum would report
    non_kg = sum(1 for p in parts if p["nonKg"])
    mean = statistics.mean(masses)
    stdev = statistics.stdev(masses) if n > 1 else 0.0

    outliers = []
    for p in parts:
        z = (p["kg"] - mean) / stdev if stdev else 0.0
        if abs(z) > 3:
            outliers.append((p["name"], p["kg"], z))
    outliers.sort(key=lambda o: -abs(o[2]))

    # parts that share a description should weigh about the same
    groups = {}
    for p in parts:
        if p["desc"]:
            groups.setdefault(p["desc"], []).append(p)
    inconsistent = []
    for desc, members in groups.items():
        if len(members) < 2:
            continue
        med = statistics.median(x["kg"] for x in members)
        for x in members:
            if med and abs(x["kg"] - med) / med > 0.25:
                inconsistent.append((x["name"], desc, x["kg"], med, len(members)))

    # 3. render
    def table(head, body):
        th = "".join(f"<th>{h}</th>" for h in head)
        tr = "".join("<tr>" + "".join(f"<td>{c}</td>" for c in row) + "</tr>" for row in body)
        return f'<table class="oml-md-table"><thead><tr>{th}</tr></thead><tbody>{tr}</tbody></table>'

    html = (f"<p><b>Leaf mass total: {fmt(total)} kg</b> across {n} physical parts "
            f"(mean {fmt(mean)} kg, standard deviation {fmt(stdev)} kg).</p>")
    if non_kg:
        html += (f"<p>{non_kg} parts are recorded in units other than kg. Adding raw values "
                 f"without unit conversion would report {fmt(naive_total)} kg.</p>")
    html += "<h4>Statistical outliers (|z| &gt; 3)</h4>"
    html += (table(["Part", "Mass (kg)", "z"],
                   [(o[0], fmt(o[1]), fmt(o[2], 1)) for o in outliers])
             if outliers else "<p>None.</p>")
    html += "<h4>Identical parts with inconsistent mass</h4>"
    html += (table(["Part", "Description", "Mass (kg)", "Group median (kg)", "Group size"],
                   [(i[0], i[1], fmt(i[2]), fmt(i[3]), i[4]) for i in inconsistent])
             if inconsistent else "<p>None.</p>")
    display(html)
```

**Interpretation.** A part far outside the mass distribution, or one that weighs differently from parts with an identical description, is more likely a data-entry error than an engineering fact. The analysis flags it for the owner rather than correcting it. The total is derived at render time and never stored, and it applies each unit's multiplier, because a raw sum silently treats grams as kilograms.