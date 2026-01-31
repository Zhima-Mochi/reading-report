# Chapter 4 — Resource Scope and Hierarchy

## TL;DR (high-signal recall)
- Resource layout is the “schema” of an API: which resources exist, their fields, and how they relate.
- Relationships are bidirectional in meaning even when only one direction is stored explicitly.
- Relationships are stored only when they enable real functionality; unnecessary links add cost and confusion.
- References, many-to-many links, self-references, and hierarchical containment each fit different constraints.
- Choosing between separate resources and in-line data depends on atomic interaction needs.
- Anti-patterns include “resources for everything,” deep hierarchies, and in-lining everything.

## Resource layout
- Resource layout is the arrangement of resources and the relationships between them, similar to a database schema or ER model.
- Layout includes:
  - Which concepts are resources vs simple fields
  - Which fields carry relationships
  - The scope of uniqueness for identifiers
- A useful way to think about layout is “boxes and lines”:
  - Boxes are resource types.
  - Lines are relationships created by fields that reference other resources.

## Typical examples (from the chapter’s framing)
- A chat API commonly models `ChatRoom`, `User`, and `Message` resources, with messages referencing their authors.
- A commerce API commonly models `User`, `Address`, and `PaymentMethod`, with payment methods referencing billing addresses.

## Relationship types (what they mean in APIs)
### Reference relationships
- A reference is a field on one resource that points to another resource.
- References are conceptually similar to foreign keys.
- Even a one-direction field implies a reverse relationship in the domain model.

### Many-to-many relationships
- Both resource types can be linked to multiple instances of the other.
- Many-to-many is common and requires a deliberate representation strategy (later chapters cover concrete patterns).

### Self-references
- A resource can reference another resource of the same type.
- Self-references commonly represent parent-child trees or “previous/next” relationships within the same resource class.

### One-to-one and optional relationships
- One-to-one relationships exist when each side can have at most one counterpart.
- Optionality changes behavior and validation because a missing relationship can be valid rather than erroneous.

### Hierarchical relationships
- A hierarchy is a special kind of reference relationship that implies more than “A points to B.”
- The reference usually points upward (child → parent) and implies containment/ownership (like folders containing files).
- Hierarchies can repeat recursively (folders containing folders), sometimes indefinitely.
- Hierarchies typically carry expected behaviors:
  - Deleting a parent implies deleting contained children (cascading delete).
  - Access to a parent often implies access to contained children (permission inheritance).
- Example framing from earlier chapters: a `ChatRoom` “contains” many `Message` resources.

## Entity relationship diagrams (ERDs)
- ERDs make cardinality and optionality explicit in a way that is hard to communicate in prose.
- ERDs help prevent accidental layout drift, such as turning a one-to-many relationship into a many-to-many relationship by embedding lists everywhere.

## Choosing the right relationship
### Relationship necessity
- Not every domain connection becomes an API relationship.
- A relationship is stored when it enables a user workflow, navigation, integrity rule, or authorization boundary.
- Avoid connecting every resource “because it might be useful” since each relationship adds complexity and long-term maintenance cost.

### References vs in-line data
- References keep a single source of truth but can require extra calls for clients.
- In-line data reduces round trips but creates staleness and update ambiguity when the in-lined concept is shared elsewhere.
- The decision centers on atomic interaction:
  - A concept that must be addressed, secured, or updated independently is a strong candidate for a separate resource.
  - A concept that only exists within its parent and never needs independent lifecycle can remain in-line as a data type.

### Hierarchy and scope
- Hierarchy encodes containment and scope, often improving clarity when the domain has true ownership.
- Hierarchy becomes a liability when it becomes deep, because deep paths are hard to understand, hard to refactor, and hard to secure consistently.
- Hierarchies often imply cascading lifecycle and authorization inheritance; be explicit about these rules.
- Hierarchy must match real domain constraints, especially around whether children can move between parents.
- Hierarchy decisions also determine uniqueness scope:
  - A child identifier can be globally unique.
  - A child identifier can be unique only within its parent.
  - Parent-scoped identifiers create additional constraints when the child can move between parents.

## Anti-patterns
### Resources for everything
- Over-resource modeling explodes surface area and forces clients into overly chatty workflows.
- It increases authorization and lifecycle complexity without necessarily improving clarity.

### Deep hierarchies
- Deep nesting increases coupling between resources and makes restructuring a breaking change.
- It encourages path-based modeling for query/navigation problems that are better solved with references or list-and-filter behaviors.

### In-line everything
- In-lining shared concepts creates staleness and unclear update rules.
- It forces the API to answer “which copy is authoritative” and “how does an update propagate,” which are avoidable problems.

## Quick recall (answers)
- Resource layout is the API’s structural model of resources and relationships.
- Relationships are stored only when they serve a concrete purpose for users or system constraints.
- Separate resources enable atomic lifecycle and independent access control; in-line data favors convenience but risks staleness.
- Deep hierarchies are avoided because they reduce evolvability and increase client complexity.
