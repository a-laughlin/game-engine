# Game Engine

- [Game Engine](#game-engine)
  - [Definitions](#definitions)
  - [Further Rulebook Concepts](#further-rulebook-concepts)
  - [Spec](#spec)
  - [Solution](#solution)
    - [`GameEngine` CLI Reference](#gameengine-cli-reference)
    - [`GameEngine` CLI Usage Details](#gameengine-cli-usage-details)
      - [`init`](#init)
      - [`apply_transform` (most common use case)](#apply_transform-most-common-use-case)
      - [`create_transform`](#create_transform)
      - [`apply_next_transform`](#apply_next_transform)
      - [`apply_all_transforms`](#apply_all_transforms)
      - [`test_transform`](#test_transform)
    - [`GameEngine` CLI Implementation Details](#gameengine-cli-implementation-details)
      - [`$ GameEngine create_transform "example"` details](#-gameengine-create_transform-example-details)
    - [Game State Shape](#game-state-shape)
    - [Transformation DSL](#transformation-dsl)
    - [Rule (constraint type) DSL](#rule-constraint-type-dsl)
      - [Example Constraint Operator Composition](#example-constraint-operator-composition)


## Definitions
1. A "Graph" is a collection of concepts and relationships between them
2. A "Node" is a concept in a graph
3. An "Edge" is a relationship between two Nodes
4. An "Element" is a Node or Edge
5. A "Node Type" is a label for a category of Nodes.
6. An "Edge Type" is a label for a category of Edges
7. A "Source" is an Edge's origin Node
8. A "Sink" is an Edge's terminus Node
9. A "RPG" is a subset of all conceptual games called "Role Playing Games".
10. A "Rulebook" is a text (book, blog, napkin-back, etc.) that defines or modifies how to play a RPG.
11. A "Ruleset" is a directed acyclical Graph (DAG) projection of one or more rulebooks, containing references back to the originals.
12. A "Ruleset Node" is a single Rulebook concept (e.g. Creature, Combat, Rest).
13. A "Ruleset Edge" is an Edge between Ruleset Nodes whose type is based on a common linguistic relationships. Type examples include: synonym, hierarchy, containment, superset, condition, attribute, equality, inequality. Edge examples include: Creatures may wear Equipment, Equipment is a superset of Sword, Container may contain Noun.
14. A "Rule" is a composition of one or more Ruleset Elements. Its type is based on whatever is useful for discussion and program execution. Example types include Fact and Constraint.
15. A "Constraint" is a Rule whose component Ruleset Edges include at least one equality or inequality. Constraints' purpose is validating other Elements. e.g. "A rides B" might only be valid when: "A is Creature && B is Creature && A size <= B size".
16. "A Time Schema" refers to a to a cluster of time-semantic Rules (e.g. speed, distance, forward, back, groupings, orderings, duration).
17. "A Space Schema" refers to a cluster of space-semantic Rules (e.g. proximity, groupings, distance, direction, orderings, length, width)
18. "A Spacetime Schema" refers to one pairing of a space schema and a time schema, and any relationships specific to it. (e.g. velocity)
19. An "Instance" is a concrete version of a Rule. (e.g. Node: Creature -> Bob.) (e.g. Edge: A rides B -> Bob rides Bob's Horse.) Instances probably need references back to their Rule for validation.
20. "A Game" refers to a Ruleset paired with its Instances.
21. "A Game Element" references a single Ruleset Element or Instance in a Game.
22. "A Campaign" refers to all Instances in a Game.
23. "A SpaceTime" is a SpaceTime Schema Instance, including all points in its time and space. One or more SpaceTimes may exist in a Campaign. A SpaceTime may relate to many Campaign Elements.
24. "A Time Point" refers to an infinitely small slice of a SpaceTime's time paired with all of its space
25. "A Space Point" refers to an infinitely small area of a SpaceTime's space paired with all of its time
26. "SpaceTime Time" is the collection of all Time Points in a SpaceTime
27. "SpaceTime Space" is the collection of all Space Points in a SpaceTime
28. A "Space Point" refers to an infinitely small slice of SpaceTime Space paired with all of SpaceTime Time
29. A "SpaceTime Point" is a Space Point paired with a Time Point
30. A "SpaceTime Clock" is a discrete numerical value that corresponds to a SpaceTime's Time Points. Its unit is milliseconds. It enables SpaceTime Elements to change.
31. A "SpaceTime Offset" is a multiplier that indicates how a SpaceTime's clock changes relative to the Campaign Clock. For now, all offsets are 1 (100%).
32. "The Campaign Clock" is a monotonically increasing discrete numerical value in milliseconds. It provides a stable comparison for SpaceTime Clocks.
33. A "Tick" is a discrete span of time. It may differ according to the rulebook. For example a Tick may be 6 seconds in D&D Combat mode, and 1 day in D&D Downtime mode. Since the Campaign Clock is in Milliseconds, a 6-second Tick advances the Campaign Clock by 6000. For now, all SpaceTime clocks advance by the same amount.
34. A "Game Version" is a monotonically increasing discrete numerical value that increments with all Game State changes.
35. A "Game State" is a snapshot of a Game at a single Game Version. The current Game State is persisted to the game state file on every state change.
36. A "Transformation" is a list of one or more changes to Game Elements.
37. A "Game State Patch" is an undoable/replayable diff of each Game Element Change.
38. A "Narrative" is a multi-layered story that occurs over time. It exists in the GM's head and outside the Game State. The Narrative only affects the Game in that Narrative-progressing Transforms increment the Campaign Clock and Game Version, while Non-narrative-progressing Transforms, like rule overrides, only increment the Game Version.

## Further Rulebook Concepts
1. Rulebook concepts, meanings, and relationships may be explicit or implicit.
2. Rulesets may include continuous concepts like time and space, and discrete concepts like integers.
3. Discrete concepts and relationships may be ordered (e.g. ordinal, sequential) or unordered (e.g. categorical).
4. Discrete concepts and relationships may be grouped in mutually exclusive and inclusive ways. (e.g. Space: A geographic state is a distinct spatial entity. "The midwest" is a spatially contiguous group of states. "Square states" is a non-contiguous group.) (e.g. Time: "span" is an ordered contiguous group. "the good years" is an unordered non-contiguous group.).
5. Rulesets may define other relationships, like representations (map represents country), synonyms (big is like large), subsets (human is a subset of race) and attributes (bob is race human, or bob is large).
6. Some Game Elements may have a duration. In that case, they need a way to keep their time in sync with their SpaceTime.

## Spec
1. We want to play a campaign where:
    1. A game master (GM) controls the Game and facilitates all Transformations
    2. The GM submits Transformations via CLI
    3. The Game Engine keeps timered Campaign Elements in sync with the Campaign Clock.
    4. The Game State is always valid
    6. The GM can easily trace rule provenance back to its rulebook origin while playing.
2. Non-functional Requirements
    1. Testability - to prevent bugs
    2. Reliability
    3. Flexibility - Transformations and Constraints DSLs, whether from rules, the GM, or the world itself, should be agnostic to any particular Rulebook.
    4. Composability - for flexibility. Each transformation or constraint should be maximally composable to ensure they can express any ruleset and be programmatically enforced and executed.
3. Tech Stack
    1. The Game Engine Should be written with Python
    2. It should use the lates uv for package management.
    3. Read
    4. Testability - to prevent bugs
    5. Reliability
    6. Flexibility - Transformations and Constraints DSLs, whether from rules, the GM, or the world itself, should be agnostic to any particular Rulebook.
    7. Composability - for flexibility. Each transformation or constraint should be maximally composable to ensure they can express any ruleset and be programmatically enforced and executed.

## Solution

### `GameEngine` CLI Reference

- `$ GameEngine init path_to_rules path_to_campaign path_to_state` initializes a Game State object in a file at 'path_to_state', using the passed rules and campaign files
- `$ GameEngine apply_transform "example"` [most common case] creates transforms appropriate for the example and applies them to the game state. Shorthand for 'create_transform "example" && apply_all_transforms'.
- `$ GameEngine create_transform "example"` Creates all transforms appropriate for the 'example' scenario, prepends them to 'game_state.unresolved_transforms', and saves the game state file. Nothing else.
- `$ GameEngine apply_next_transform` pops and applies the last transform in the 'unresolved_transforms' queue to the game state.
- `$ GameEngine apply_all_transforms` repeatedly calls 'apply_next_transform' until the 'unresolved_transforms' queue is empty.
- `$ GameEngine test_transform "example"` for internal_testing. Identical to 'create_transform', but returns transforms as text instead of modifying 'unresolved_transforms'.
- `$ GameEngine test` for internal_testing. It tests everything.'.

See "`GameEngine` CLI Usage Details" for more.



### `GameEngine` CLI Usage Details

All game transforms are queued into and resolved from the `{unresolved_transforms:[]}` queue on the Game State object.

#### `init`
`$ GameEngine init path_to_rules path_to_campaign path_to_game_state`
Initializes a file at path_to_game_state. It contains a Game State object with appropriate rules from path_to_rules, campaign data from path_to_campaign, and emtpy queues of unresolved_transforms and game_state_patches.

Returns a useful error when rules or campaign files are absent, either are malformed, or path_to_state exists.

#### `apply_transform` (most common use case)

`$ GameEngine apply_transform "example"`

Shorthand for `GameEngine queue_transform "example" && GameEngine apply_all_transforms`

#### `create_transform`
`$ GameEngine create_transform "example"`
Creates all transforms appropriate for the example scenario, prepends them to `game_state.unresolved_transforms`, and saves the game state file.

Returns a useful error when the passed scenario does not match a valid Transformation DSL format.

Example scenarios include:
- **"6 seconds pass"** Only transforms game time and any other active timers in `game_state.campaign_edges`.
- **"Bob casts fireball on Bob"** In this case, creates transforms appropriate for bob to cast fireball at a point centered on himself. Created transforms are automatically expanded with related transforms like targeting and resolution paths, as well as any time advancements appropriate for the transform, game state, and ruleset (like consuming Bob's action, or a "Tick" of 6 seconds for a combat turn.)
- **"override ...example..."** Creates an override transform for existing ruleset_nodes and ruleset_edges. Unlike other transforms, override patches don't change campaign_edges. Instead they go to ruleset_nodes_overrides or ruleset_edges_overrides for rule lookups.

#### `apply_next_transform`
`$ GameEngine apply_next_transform`

Validates the last item in `game_state.unresolved_transforms` against the rules, pops it off the queue, creates a patch for it, applies the patch to the game state, queues the patch into game_state_patches, and saves the game state file.

#### `apply_all_transforms`
`$ GameEngine apply_all_transforms`
Calls apply_next_transform until the unresolved_transforms queue is empty.



#### `test_transform`
`$ GameEngine test_transform "example"`
\[for internal testing] Identical to "create_transform", but returns successful results as text. Does not modify 'unresolved_transforms' or save the game state file.

### `GameEngine` CLI Implementation Details

#### `$ GameEngine create_transform "example"` details

TBD on what steps the English DSL must go through to become its final representation, but the result will be an adjacency list, and may contain tuples like this:

"Bob hurls a fireball at the Orc" becomes...
```js
{
  unresolved_transforms:[ // as tuples
  // id     source    type        sink
    ['fb',  'bob',    'casts',    'fireball'],
    ['t',   'fb',     'targets',  'tile2'],
  ]
}
```

Problems in the transformation include:
1. Fireball requires a map-point-center to target, not a creature.
2. Any synonyms in source, type, and sink must be resolved to ruleset language. For example, "hurls" isn't a term in ruleset_nodes, nor is it defined as a synonym in ruleset_edges, but the game engine needs a way to resolve the term "hurls" to "casts". Options might include hard-interrupting the GM to ask them to add an override, or less-interruptive resolutions like suggesting some synonyms and asking for approval, or asking if the GM meant a term from the ruleset_nodes, like "casts".
3. All edges necessary to relate the concepts to appropriate campaign_edges, campaign_nodes, ruleset_nodes, and ruleset_edges must be created.
4. All edges appropriate to specify and resolve time and spatial resolution must be created.



### Game State Shape
A single canonical data source for rules, hombrew rules, narrative, canon, and campaign.

Example Game State
```js
campaign_state = {
  // Rule Elements
  rule_nodes:[
    {id:'orc'},
    {id:'orc_size',value:'large'}
  ],
  rule_edges:[
    {source:'orc', type:'attribute',sink:'orc_size'},
  ],

  // Homebrew Rule Elements
  homebrew_rule_nodes:[],
  homebrew_rule_edges:[],

  // Narrative Elements
  narrative_nodes:[
    {id:'orc1',description:'a sturdy looking fellow'},
    {id:'orc1_mom'},
    {id:'orc1_dad'},
    {id:'orc1_expected_injury',description:'heart shaped scar on his butt'}
  ],
  narrative_edges:[
    {source:'orc1',type:'mother',sink:'orc1_mom'},
    {source:'orc1',type:'father',sink:'orc1_dad'},
    {source:'orc1',type:'event',sink:'orc1_expected_injury'},
  ],

  // Canon Elements
  canon_nodes:[
    {id:'orc1_injury',description:'heart shaped scar on his butt from dragon',campaign_clock_timestamp:1},
    {id:'orc1_unexpected_heal',description:'butt scar healed',campaign_clock_timestamp:5},
  ],
  canon_edges:[
    {source:'orc1_injury',type:'narrative',sink:'orc1_expected_injury'},
    // ^ edge to narrative for further details - unlike orc1_unexpected_heal
  ],

  // Campaign (Instance) Elements
  campaign_nodes:[
    {id:'orc1'},
    {id:'campaign_clock',value:7}
  ],
  campaign_edges:[
    {source:'orc1',type:'rule',sink:'orc'},
    {source:'orc1',type:'narrative',sink:'orc1'},
    {source:'orc1',type:'narrative',sink:'orc1_mom'},
    {source:'orc1',type:'narrative',sink:'orc1_dad'},
    {source:'orc1',type:'canon',sink:'orc1_injury'},
  ],

  // Unresolved Patches
  // FIFO Queue of diff-like Game State Patch objects. When a GM provides a transform via CLI,
  // one or more Patch objects are created and prepended to this queue.
  unresolved_patches:[],

  // Resolved Patches
  // Chronological, undoable, replayable list of list of previously applied Game State Patches.
  // Undoing should shift the first item off this list, apply the reversion,
  // and possibly add it back to unresolved_patches.
  resolved_patches:[],
}
```


### Transformation DSL
The transformation DSL is the language GMs use to transform the Game State


| User Input DSL (linguistic) | 5e User Input Example 1 | 5e User Input Example 2 | User Input DSL (Graph) |
| --- | --- | --- | --- |
| subject verb object preposition object | Bob casts fireball at Orc | Bob throws rock into well | node edge node edge node |
| subject verb preposition object | Bob moves to North | Bob sits behind Rock |  node edge edge node |
| subject verb object | Bob attacks Orc1 | Bob moves North | node edge node |

| CRUD Operations |
| // edge transforms |
| [set, edge_id, new_sink_value] |
| [delete, edge_id] |
| [create, source, sink] |

| List Operations |
| ------------- |
| [map, transform, collection] |
| [filter, transform, collection] |
| [omit, transform, collection] |



### Rule (constraint type) DSL
For expressing constraints. It includes a string format for simple comparisons, similar to many CSP solvers. The Game Engine should apply a CSP solver to check them.

Constraints can get complex. For example, "Space conflict occurs when the temporal mode is Combat, and a creature occupies Map_Tile, and another creature enters Map_Tile during their turn, and both creatures are size small or greater, and each creature differs in size by <= 2, and the current time label is Turn End." That constraint implies many sub-constraints and transformations. Writing a function for each combination of constraints is infeasible.

This DSL supports complex constraints with operators that compose, much like "and", "or","during", and "differs" in the original english.

It is an intial take, and needs exploration to ensure it makes sense and enables the Game Engine to programmatically enforce the constraints with a CSP solver.

| Data format | String format |
| ------------- | --------------- |
| // relational: [name, rulebook_reference, value1, value2] | |
| \[eq, rulebook_reference, value1, value2] | value1==value2 |
| \[neq, rulebook_reference, value1, value2] | value1!=value2 |
| \[gte, rulebook_reference, value1, value2] | value1>=value2 |
| \[lte, rulebook_reference, value1, value2] | value1<=value2 |
| | |
| // logic: [name, rulebook_reference, constraints...] | |
| \[and, rulebook_reference, \[constraints...]] | constraint1 && constraint2 |
| \[or, rulebook_reference, \[constraints...]] | constraint1 \|\| constraint2 |
| \[xor, rulebook_reference, \[constraints...]] | |
| \[not, rulebook_reference, constraint] | !constraint1 |
| | |
| // list reductions: [name, rulebook_reference, constraint, collection] | |
| \[exists, rulebook_reference, constraint, collection] | |
| \[sum, rulebook_reference, collection] | |
| \[product, rulebook_reference, collection] | |
| \[count, rulebook_reference, collection] | |
| \[allEqual, rulebook_reference, collection] | |
| \[allDifferent, rulebook_reference, collection] | |
| | |
| // requirements/dependencies: [name, rulebook_reference, edge_source, collection] | |
| \[requires, rulebook_reference, edge_source, constraint] | |


#### Example Constraint Operator Composition

```txt
[requires, "PHB_REF_3", Occupies, [xor, [
   [exists, "PHB_REF_4", "source==TimeMode && type==Value && sink==NonCombat", "campaign_state_edges"],
   [and,"PHB_REF_5", [
      [exists, "PHB_REF_6", "source==TickPhase && type==Value && sink=='TurnEnd'", "campaign_state_edges"],
      [exists, "source==TimeMode && type==Value && sink==Combat", "campaign_state_edges"],
      [eq, 1, [count, [filter, "type==Occupies && source==Creature && sink==required_node_sink", "campaign_state_edges"]]]
   ]]
]]]
```
