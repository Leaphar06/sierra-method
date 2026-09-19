---
template:
  id: https://www.modelware.io/sierra/system-analysis/connections
  name: "Connections"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Connections

View internal connections between system components.

```diagram
---
stylesheet:
  - selector: node
    style:
      stroke: red
      height: 100
      padding: 40
      layout:
        type: dagre
        ranksep: 150
        nodesep: 40
      label:
        refY: 10
        fill: vblack

  - selector: node.component
    style:
      fill: lightgreen

  - selector: node.subComponent
    style:
      fill: yellow

  - selector: edge
    style:
      stroke: red
      router:
        name: manhattan
        args:
          padding: 10
      label:
        font-size: 10
        fill: vblack
        text-anchor: middle
      label-body:
        fill: none

  - selector: port
    style:
      fill: var(--oml-static-background, #ffffff)
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX : <http://opencaesar.io/diagram#>

CONSTRUCT {
  ?component a :Node ;
             :class "component" .
   
  ?port a :Port ;
            :class "port" ;
            :parent ?component .

  ?subComponent a :Node ;
             :parent ?component ;
             :class "subComponent" .

  ?subPort a :Port ;
            :class "port" ;
            :parent ?subComponent .
 
  ?connection a :Edge ;
            :source ?srcPort ;
            :target ?tgtPort ;
            :text ?itemLabel .
}
WHERE {
  ?component component:hasPort ?port ;
     base:contains ?subComponent .

  ?subComponent component:hasPort ?subPort .

  ?connection a component:Connection ;
     oml:hasSource ?srcPort ;
     oml:hasTarget ?tgtPort ;
     component:transfers ?item .

    BIND(REPLACE(STR(?item), "^.*[#/]", "") AS ?itemLabel)
}
```

## Ports

The ports associated with each component are shown here for reference when wiring the connections below.

```table-editor
---
columns: { this: { label: "Port" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:PortShape
    a sh:NodeShape ;
    sh:targetClass component:Port ;
    sh:property [
        sh:path [ sh:inversePath component:hasPort ] ;
        sh:name "Component" ;
        sh:class component:Component ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path component:direction ;
        sh:name "Direction" ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    .
```

## Connections

Define connections between component ports. Each connection links a source port to a target port and can optionally carry an item, such as a signal, material, or energy.

```table-editor
---
columns: { this: { label: "Connection" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix oml: <http://opencaesar.io/oml#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:ConnectionShape
    a sh:NodeShape ;
    sh:targetClass component:Connection ;
    sh:sparql [
        sh:message "Sibling components must connect an Out port to an In port — two ports of the same direction cannot be wired together." ;
                sh:select """
            PREFIX base: <https://www.modelware.io/sierra/base#>
            PREFIX component: <https://www.modelware.io/sierra/component#>
            PREFIX oml: <http://opencaesar.io/oml#>
            SELECT $this WHERE {
                $this oml:hasSource ?src ;
                      oml:hasTarget ?tgt .
                ?src component:direction ?srcDir .
                ?tgt component:direction ?tgtDir .
                ?srcComp component:hasPort ?src ;
                         base:isContainedBy ?parent .
                ?tgtComp component:hasPort ?tgt ;
                         base:isContainedBy ?parent .
                FILTER(?srcDir = ?tgtDir)
            }
        """ ;
    ] ;
    sh:property [
        sh:path oml:hasSource ;
        sh:name "Source" ;
        sh:class component:Port ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        oml:localReference true ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path oml:hasTarget ;
        sh:name "Target" ;
        sh:class component:Port ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        oml:localReference true ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path component:transfers ;
        sh:name "Item" ;
        sh:class base:Item ;
        sh:maxCount 1 ;
        sh:order 3 ;
    ] ;
    .
```