# Database Design

The design layer of a relational database: the formal data model, the algebra that defines what queries mean, conceptual modeling with Entity-Relationship diagrams, the translation of a conceptual schema into a logical (relational) one, and normalization theory.

This document covers *what a correct schema is and how to arrive at one*. For the systems view (choosing and operating real stores at scale) see `Data Systems.md`. For how a DBMS (database management system) executes against a schema (buffer management, access methods, concurrency, recovery, query processing) see `Database Internals.md`. For the SQL realization of these constructs see `SQL.md`.

---

## Table of Contents

1. The Relational Model
2. Integrity Constraints and Keys
3. Relational Algebra
4. Conceptual Design — the Entity-Relationship Model
5. ER Integrity Constraints
6. Logical Design — ER to Relational
7. Normalization

---

## 1. The Relational Model

A mathematical **relation** over domains `D1, D2, ..., Dn` is a subset of the Cartesian product `D1 x D2 x ... x Dn`. Each element of the relation is an **n-tuple**. Two consequences follow from "subset of a Cartesian product":

- The number of tuples is the **cardinality** of the relation; `n` is its **degree**.
- A relation is a *set*, so tuples are unordered and no two tuples are identical. Components, however, are positional.

In the relational *model* a relation is presented as a **table**: each column is an **attribute** with a name and an associated **domain** (the set of values it may take); each row is a tuple. Naming the columns removes the reliance on position — a tuple is accessed by attribute name rather than index.

```
                         attribute
                            |
                            v
        +------+-------+----------+----------+
        | Home | Away  | GoalsHome| GoalsAway|
        +------+-------+----------+----------+
tuple ->| Juve | Lazio |    3     |    1     |
        | Lazio| Milan |    2     |    0     |
        | Juve | Roma  |    1     |    2     |
        +------+-------+----------+----------+
```

**Notation.** `t[A]` (equivalently `t.A`) denotes the value of tuple `t` on attribute `A`. For a set of attributes `X`, `t[X]` is the sub-tuple of values on those attributes.

**NULL.** A relational value may be absent. The model does not distinguish *why* a value is missing: `NULL` may mean unknown, inapplicable, or simply withheld. This single undifferentiated marker has consequences throughout the algebra (section 3) and the constraint language.

---

## 2. Integrity Constraints and Keys

An **integrity constraint** is a condition expressed at the schema level that every instance of the database must satisfy. It characterizes the database states that correctly represent the modeled world; states violating it are rejected. Constraints divide by whether they concern one relation or several.

### Keys

A **key** of a relation is a non-empty set of attributes that identifies its tuples. The notion is built in three steps:

- **Superkey** — a set of attributes `K` such that no two tuples agree on all of `K`. A relation may have many superkeys (any superset of a superkey is a superkey).
- **Key** — a *minimal* superkey: a superkey from which no attribute can be removed while remaining a superkey.
- **Primary key** — a key chosen as the principal identifier, with the additional requirement that it admits **no NULL values**. Every minimal key is a *candidate key*; the primary key is the candidate key chosen as principal. A relation has exactly one primary key and zero or more other candidate keys.

Every relation has at least one key, because a relation is a set of distinct tuples — at worst, the full set of attributes is a superkey. The primary key is what other relations reference.

### Intra-relational constraints

Hold within a single relation.

- **Tuple constraint** — a condition on the values of each tuple independently of other tuples (e.g., `GoalsHome >= 0`, or `net = gross - tax`).
- **Key constraint** — designates a set of attributes as a key (below).

### Inter-relational constraints

Relate values across relations.

- **Referential integrity (foreign key)** — a non-empty sequence of attributes `X` of relation `R1` is a *foreign key* referencing a key `Y` of relation `R2` if every combination of values on `X` appearing in `R1` also appears on `Y` in some tuple of `R2`. The foreign key forces `R1`'s references to point at existing `R2` tuples.

```
Infractions
+--------+--------+---------+------+---------+
| Code   | Date   | Officer | Prov | Number  |
+--------+--------+---------+------+---------+
| 34321  | 1/2/95 | 3987    | MI   | 39548K  |   (Prov, Number) must exist in Cars
| 53524  | 4/3/95 | 3295    | TO   | E39548  |
+--------+--------+---------+------+---------+

Cars
+------+---------+---------+-------+
| Prov | Number  | Surname | Name  |
+------+---------+---------+-------+
| MI   | 39548K  | Rossi   | Mario |
| TO   | E39548  | Rossi   | Mario |
+------+---------+---------+-------+
```

- **Inclusion constraint** — the generalization of a foreign key: the projection of `R1` on `X` is contained in the projection of `R2` on `Y`, without requiring `Y` to be a key. A foreign key is an inclusion constraint whose target is a key.

---

## 3. Relational Algebra

A **relational algebra** is a set of values (relations) and operations closed over that set: every operation takes relations and yields a relation, so operations compose into **expressions**. Edgar Codd defined two equivalent query languages — relational algebra (procedural: a query is a sequence of operations) and relational calculus (declarative: a query is a first-order formula). By **Codd's theorem** they have equal expressive power — taking the calculus restricted to its *safe*, domain-independent fragment, since unrestricted calculus can express infinite results. SQL is a hybrid drawing on both.

The algebra rests on six primitive operations — three set-theoretic (union, difference, cartesian product), two unary (selection, projection), and rename — with intersection and the joins derived from them. (In a positional algebra rename is unnecessary; under named attributes it is primitive, since no other operator can change an attribute name.)

### Set operations

Because relations are sets of tuples, union, intersection, and difference apply — but only between **union-compatible** relations (same degree, same attributes/domains).

- **Union** `R ∪ S` — tuples in `R` or `S`; duplicates collapse to one.
- **Intersection** `R ∩ S` — tuples in both. Derivable: `R ∩ S = R - (R - S)`.
- **Difference** `R - S` — tuples in `R` not in `S`. Not symmetric.

```
            R              S
        .---------.   .---------.
        |  a      |   |      d  |
        |     .---+---+---.     |
        |     |   b , c   |     |     middle region = R ∩ S
        |     '---+---+---'     |
        '---------'   '---------'

  R = {a, b, c}     S = {b, c, d}

  R ∪ S = {a, b, c, d}   union        : in either circle, overlap counted once
  R ∩ S = {b, c}         intersection : only the overlap
  R - S = {a}            difference   : R minus overlap (asymmetric: S - R = {d})
```

### Cartesian product

`R x S` pairs every tuple of `R` with every tuple of `S`. If `R` is m-ary and `S` is n-ary, `R x S` is `(m+n)`-ary with `|R| x |S|` tuples. It requires no shared attributes and is rarely useful alone — it is the substrate on which joins are built (a join is a product followed by a selection).

```
    R         S            R x S   (every tuple of R paired with every of S)
 +-----+   +-----+       +-----+-----+
 |  a  |   |  x  |       |  a  |  x  |
 |  b  | X |  y  |   =   |  a  |  y  |
 +-----+   +-----+       |  b  |  x  |
  |R|=2     |S|=2        |  b  |  y  |
                         +-----+-----+    |R x S| = |R| * |S| = 4
```

### Rename

`REN A1<-B1, ..., An<-Bn (R)` produces a relation identical to `R` with attribute `Ai` renamed `Bi`. Its purpose is to make set operations and joins apply to schemas that differ only in attribute names, and to disambiguate a relation joined with itself.

### Selection

`SEL_condition (R)` (also `σ_condition (R)`) returns the tuples of `R` satisfying `condition`. The condition is a Boolean expression over comparisons (`=, <, >, !=, <=, >=`) of attributes and constants, combined with `AND` / `OR`. The result has the same schema as `R`; the tuple count can only shrink.

```
SEL Salary > 50 AND Branch = 'Milan' (Employee)
```

### Projection

`PROJ_attribute-list (R)` (also `π_attribute-list (R)`) keeps only the listed attributes, dropping the rest. Because the result is a set, **duplicate tuples are removed** — so a projection can have fewer tuples than the input. Selection and projection are *orthogonal*: selection chooses rows, projection chooses columns.

```
                  PROJ keeps these columns ->  drops the rest, dedups
                  +------+----------+--------+
                  | Name |  Salary  | Branch |
                  +------+----------+--------+
  SEL keeps  -->  | Anna |    60    | Milan  |    row satisfies Salary > 50
  matching        | Bob  |    40    | Milan  |    row dropped
  rows       -->  | Carl |    80    | Rome   |    row satisfies Salary > 50
                  +------+----------+--------+

  Selection  (σ) = horizontal cut: choose ROWS     -> schema unchanged
  Projection (π) = vertical   cut: choose COLUMNS  -> duplicates removed
```

### Joins

A **join** correlates tuples of different relations.

- **Natural join** `R ⋈ S` — combines tuples that agree on their *common* attributes. With `X` the attributes of `R` only, `Z` common, `Y` of `S` only, the result over `XZY` contains a tuple wherever an `R` tuple and an `S` tuple share their `Z` values. Cardinality is bounded by `0 <= |R ⋈ S| <= |R| x |S|`. If `R` and `S` share *no* attributes, the natural join degenerates to the Cartesian product.

```
  R (X, Z)        S (Z, Y)               R ⋈ S  over (X, Z, Y)
 +----+----+     +----+----+            +----+----+----+
 | x1 | z1 |     | z1 | y1 |            | x1 | z1 | y1 |
 | x2 | z1 |     | z2 | y2 |    =>      | x2 | z1 | y1 |
 | x3 | z9 |     +----+----+            +----+----+----+
 +----+----+
        \___ match on common attribute Z ___/   (x3,z9) dangles: no z9 in S -> dropped
```

```
join complete   — every tuple of both relations contributes
join incomplete — one or more tuples do not contribute (dangling)
join empty      — no tuple contributes
```

- **Theta join** `R ⋈_condition S` — a Cartesian product followed by a selection: `SEL_condition (R x S)`. It is the tool for relations with no common attributes, or for non-equality conditions. When the condition is equality it is an **equi-join**.

- **Outer joins** preserve tuples that an inner join would discard (the *dangling* tuples), padding the missing side with `NULL`:
  - **Left** keeps all tuples of the left operand;
  - **Right** keeps all tuples of the right operand;
  - **Full** keeps the dangling tuples of both.

```
Employee              Dept
 Name | DeptId         DeptId | DeptName
 -----+-------         -------+---------
 Anna |   D1           D1     | Sales
 Bob  |  NULL          D2     | IT      (no employees)

Left outer join (Employee ⟕ Dept) — keep every Employee row:
 Name | DeptId | DeptName
 -----+--------+---------
 Anna |   D1   | Sales
 Bob  |  NULL  | NULL        <- Bob kept, Dept side padded with NULL

Full outer join (Employee ⟗ Dept) — keep unmatched rows from both:
 Name | DeptId | DeptName
 -----+--------+---------
 Anna |   D1   | Sales
 Bob  |  NULL  | NULL        <- unmatched Employee
 NULL |   D2   | IT          <- unmatched Dept
```

### NULL in the algebra

Algebraic conditions evaluate only over non-null values, so a condition on an attribute that is `NULL` discards that tuple. Consequently `SEL_(A>n) (R) ∪ SEL_(A<=n) (R) != R` in general — tuples with `A = NULL` fall out of both. To reference nulls explicitly the conditions `IS NULL` and `IS NOT NULL` are used. The full three-valued logic (`UNKNOWN` propagation, aggregate handling) is given in `SQL.md`.

```
SEL (Age > 40) (Employee)            -- drops rows where Age IS NULL
SEL (Age > 40) OR (Age IS NULL) (Employee)
```

### Expression equivalence

Two expressions are **equivalent** if they produce the same result on every database instance. Equivalences (e.g., pushing a selection through a join) are the basis of query optimization — the optimizer rewrites an expression into an equivalent, cheaper one. The execution algorithms behind these operators are covered in `Database Internals.md`.

---

## 4. Conceptual Design — the Entity-Relationship Model

Conceptual design produces a description of *what* data the application needs, independent of any DBMS. The **Entity-Relationship (ER)** model is the standard notation. It separates two levels:

- **Intensional level** — the schema: the structure (entity and relationship types).
- **Extensional level** — an instance: the actual tuples populating that structure. `instances(I, X)` denotes the set of objects of construct `X` in instance `I`.

### Entities

An **entity** is a class of objects with autonomous existence and shared properties (e.g., `Employee`, `Course`). Drawn as a rectangle. In any instance `I`, an entity `E` has an *extension* `instances(I, E)` — its set of occurrences.

### Attributes

An **attribute** of an entity is a local property associating each occurrence with a value from a **domain**. Formally an attribute `A` of entity `E` over domain `D` is a total function `A : instances(I, E) -> D` — every occurrence has exactly one value. Drawn as a circle on the entity.

A **composite attribute** is defined over a structured domain and groups sub-attributes (e.g., `Address` composed of `Street`, `Number`, `ZIP`).

```
   ENTITY with ATTRIBUTES                 COMPOSITE ATTRIBUTE
                                         (groups sub-attributes)
    (EmpId)      (Name)
        \         /                       (Street)(Number)(ZIP)
       +-----------+                           \    |    /
       | Employee  |                           ( Address )
       +-----------+                                |
        /         \                           +-----------+
   (Salary)     (Branch)                      | Employee  |
                                              +-----------+

   rectangle = entity    circle = attribute (one value per occurrence)
```

### Relationships

A **relationship** is a link defined over two or more entities; the number of participating entities is its **degree**. In an instance, a relationship `R` over entities `E` and `F` has extension `instances(I, R) ⊆ instances(I, E) x instances(I, F)` — generalized for n entities to a subset of the n-way product. Because the extension is a *set* of tuples of occurrences, **no two identical occurrences of a relationship can exist** (the same combination of participants appears at most once). Drawn as a diamond between entities.

Multiple distinct relationships may connect the same entities (e.g., `Employee` *Residence* `City` and `Employee` *WorkSite* `City`).

```
  RELATIONSHIP (binary), with its own attribute and cardinalities

  +---------+ (0,N)          (1,N) +--------+
  | Student |------<  Exam  >------| Course |
  +---------+          |           +--------+
                       |
                    (Grade)   <- attribute of the LINK, not of either entity

  rectangle = entity    diamond = relationship
  (min,max) on each side bounds how many links one occurrence may take part in:
     each Student sits in 0..N exams ;  each Course is taken in 1..N exams
```

### Relationship attributes

A relationship may carry its own **attribute** — a property of the *link*, not of either participant (e.g., a `Grade` on the `PassedExam` relationship between `Student` and `Course`). Because the attribute is not intrinsic to the relationship's identity, it cannot by itself distinguish two occurrences — distinguishing them requires reification.

#### Reification
**Reification** turns a relationship into an entity, so that a former relationship attribute becomes a property of a first-class object that *can* distinguish occurrences. The single relationship `R` between `E` and `F` is replaced by a new entity `R` plus two binary relationships connecting it to `E` and `F`. An occurrence is then a triple including the reified attribute, lifting the "no duplicate pairs" restriction.

```
  BEFORE — Grade is an attribute of the relationship

  +---------+                              +--------+
  | Student |-------<  PassedExam  >-------| Course |
  +---------+              |               +--------+
                        (Grade)

  the relationship is identified by its participants, so the pair
  (Student, Course) may appear AT MOST ONCE -> a retake (same student,
  same course, different grade) has nowhere to live.

  AFTER — PassedExam reified into an entity

  +---------+         +-----------+         +--------+
  | Student |-< Sat >-|   Exam    |-< For >-| Course |
  +---------+         +-----------+         +--------+
                        /       \
                    (Grade)   (Date)   <- the new entity has its own identity,
                                        so many exams may link the same
                                        Student and Course (retakes allowed).
```

### Roles

When a relationship involves the same entity more than once, each participation is given a distinct **role** to indicate the part each occurrence plays (e.g., a `Succession` relationship over `Sovereign` with roles `Successor` and `Predecessor`).

```
  ROLES (the same entity participates twice)

      predecessor           successor
              \                /
               <  Succession  >
                   |      |
                +-----------+
                | Sovereign |    one entity, two roles name the two parts
                +-----------+
```

### ISA (subset)

An **ISA** relationship states that a *child* entity is a subset of a *parent* entity: every occurrence of the child is also an occurrence of the parent, inheriting all its attributes, relationship participations, and constraints. `instances(I, E1) ⊆ instances(I, E2)` for `E1 ISA E2`. The child may add its own properties. Each entity has **at most one parent** (no multiple inheritance); an entity may have several children, which *may* share occurrences. ISA is transitive.

```
       +--------+
       | Person |   parent
       +--------+
           /_\   every Student IS-A Person: inherits all attributes,
            |    relationship participations and constraints; may add its own
       +---------+
       | Student |  child  (child extension ⊆ parent extension)
       +---------+
```

### Generalizations

A **generalization** groups several child entities under one parent by a single classification criterion; each child is a subset of the parent and inherits its attributes, relationship participations, and constraints. As with ISA, each entity has at most one parent. A generalization is classified along two independent dimensions:

- **Coverage** — *total (complete)* when the union of the children's occurrences equals the parent's, `instances(I, E1) ∪ ... ∪ instances(I, En) = instances(I, F)`; *partial (incomplete)* when a parent occurrence may belong to no child.
- **Disjointness** — *exclusive (disjoint)* when siblings share no occurrence; *overlapping* when one occurrence may belong to several children.

```
   TOTAL generalization                PARTIAL generalization
       +--------+                          +--------+
       | Person |                          | Person |
       +--------+                          +--------+
           ^                                   ^
          /_\  total, disjoint                /_\  partial, disjoint
         /   \                               /   \
   +-----+   +-------+              +--------+   +--------+
   | Man |   | Woman |              | Student|   | Teacher|
   +-----+   +-------+              +--------+   +--------+

  every Person is Man or Woman        a Person may be neither
  (both diagrams show disjoint children; they contrast TOTAL vs PARTIAL coverage)
```

A plain ISA hierarchy is the degenerate single-child case; a generalization adds coverage and disjointness constraints over a set of siblings sharing one criterion.

### ISA and generalizations between relationships

The same constructs apply to relationships, under conditions: both relationships must have the **same degree** and the **same roles**, and for each role the entity in the child relationship must be a child (via ISA) of the entity in that role in the parent. Example: a `Manages` relationship being a sub-relationship of `Works` over `Person`/`Department`.

---

## 5. ER Integrity Constraints

Rules expressed in the ER schema that every instance must respect.

### Cardinality constraints on relationships

A cardinality `(min, max)` on the participation of entity `E` in relationship `R` through a role bounds how many times each occurrence of `E` may appear in `R`. For each `e` in `instances(I, E)`, the number of occurrences of `R` having `e` in that role lies in `[min, max]`.

```
Employee (1,5) ---- Assignment ---- (0,50) Task
each employee has 1..5 tasks; each task is assigned 0..50 times
```

- An **absent** explicit cardinality defaults to `(0, N)` (optional, unbounded).
- Cardinalities `(1,1)` / `(0,1)` / `(0,N)` / `(1,N)` give the familiar one-to-one, optional, and one/many-to-many shapes.
- Under ISA, a child inherits the parent's cardinality; it may only be made **more** restrictive.

### Cardinality constraints on attributes

By default an attribute has cardinality `(1,1)` — exactly one value. This can be changed:

- **Optional** `(0,1)` — the value may be absent (`NULL`).
- **Multivalued** `(1,N)` or `(0,N)` — multiple values per occurrence (e.g., several phone numbers).

Multivalued attributes are flagged for elimination in logical design (they have no direct relational counterpart — see section 6).

### Identification constraints

An **identifier** is a set of properties (attributes and/or relationship roles) that uniquely identifies the occurrences of an entity: no two occurrences agree on all of them. Two kinds:

- **Internal** — formed only from the entity's own attributes (e.g., `TaxCode` of `Person`).
- **External** — formed from roles in relationships (plus possibly some attributes), so the entity borrows part of its identity from another entity through a relationship; an entity identified this way is called a **weak entity**. Every attribute and role in an identifier must have cardinality `(1,1)`.

```
Student --(1,1)-- Enrollment --(0,N)-- University
identifier of Student = Matricola (enrollment number) + the university it enrolled in
```

An entity may have **several** identifiers. A weak entity is one whose identifier is external.

### Identification constraints on relationships

A relationship is implicitly identified by all the roles participating in it — no two occurrences can have the same participants. An explicit identifier may additionally restrict this. A `(1,1)` participation of an entity in a relationship implies an implicit identifier of that relationship on the entity.

### External constraints

Constraints not expressible in the diagram are recorded in the **data dictionary**, stated in a suitable formalism (mathematical logic) or natural language, to be enforced by assertions or application code.

---

## 6. Logical Design — ER to Relational

Logical design translates the conceptual (ER) schema into a **logical schema** in the data model of the target DBMS (here relational), accounting for the expected **application load** — data volume, and operation characteristics (batch vs interactive, frequency). Optimization targets execution time and storage. The process has two stages: restructuring the ER schema, then translating it.

### 6.1 Restructuring the ER schema

Reshapes the schema so every construct becomes translatable, and tunes for the load.

0. **Schema preparation** — restate all constraints, including implicit ones.
1. **Redundancy analysis** — decide which redundancies (e.g., derived values) to keep for efficiency and which to remove.
2. **Eliminate multivalued attributes** — the relational model permits one value per attribute, so a multivalued attribute becomes a separate binary relationship to a new entity.
3. **Eliminate composite attributes** — flatten into their components, or promote to an entity (as for multivalued).
4. **Eliminate ISA and generalizations** — not representable directly. Options: collapse the hierarchy into the parent, or into the children, or replace it with relationships. Disjointness/coverage become explicit constraints.
5. **Select primary identifiers** — where an entity (or relationship) has several identifiers, choose one by: essentiality, simplicity (few attributes), most-frequent use; for entities, prefer internal identifiers, and for external ones prefer those derived from ISA, possibly adding a surrogate code. Watch for cycles when choosing external identifiers.
6. **Reformulate external constraints** against the restructured schema.
7. **Reformulate operations and load**.

### 6.2 Direct translation

Translates the restructured ER schema into relations. Notation:

```
EntityName(attr1, attr2, attr3*, ...)         underline = primary key
    foreign key: EntityName[attr...] ⊆ OtherEntity[attr...]
    inclusion:   EntityName[attr...] ⊆ OtherEntity[attr...]
    key:         additional identifiers (not the primary key)
    constraint:  external constraints not otherwise translatable

  underline -> primary key        asterisk (*) -> attribute may be NULL
```

**Translating an entity.** Each entity becomes a table. Each entity attribute becomes a column. The principal identifier becomes the primary key; other identifiers become key constraints. Optional `(0,1)` attributes are marked nullable (`*`).

```
      ER entity                      Relational table
    (EmpId)  (Name)
        \     /
     +-----------+    ===>    Employee( EmpId, Name, Branch* )
     | Employee  |                     -----              ^
     +-----------+                     PK (underline)     (0,1) attr -> nullable (*)
         |
      (Branch)
```

**Translating a relationship** (one not merged into an entity table):

- Each relationship attribute becomes a column of the relationship's table.
- Each participating role becomes a column, with a **foreign key** to the participant's table.
- The principal identifier becomes the primary key.

**Merging (accorpamento).** A relationship with a `(1,1)` participation is usually *merged* into the table of the `(1,1)` entity rather than given its own table: the related entity's key is added as a foreign key inside that entity's table. The decision is driven by cardinalities:

- `(1,1)` participation -> add a **foreign key** in the entity's table.
- `(0,1)` participation -> same, but the foreign-key column is nullable, or an **inclusion** constraint is added.
- `(0,N)` / `(1,N)` on both sides (many-to-many) -> the relationship needs its **own table** with foreign keys to both participants forming its key.

```
  MANY-TO-MANY -> own table              (1,1) PARTICIPATION -> merge (no own table)

  +---------+(0,N)   (0,N)+--------+    +----------+(1,1)    (0,N)+------------+
  | Student |---< Exam >--| Course |    | Employee |--< WorksIn >-| Department |
  +---------+  (Grade)    +--------+    +----------+              +------------+

   Student(Matr, ...)                    Employee(EmpId, Name, Dept*)
   Course(Code, ...)                              Dept is FK -> Department
   Exam(Matr, Code, Grade)               Department(DeptId, ...)
        \__ FK __/  PK = (Matr, Code)    (the (1,1) relationship folds into the
        Matr -> Student  Code -> Course   entity's table as a FK column)
```

**Translating ISA / generalization.** Collapsing into the parent yields one table with the union of attributes, the child-only attributes nullable, plus a constraint encoding disjointness/coverage (for a total generalization, every parent row participates in exactly one child; for partial, at most one). Collapsing into children duplicates the parent's attributes into each child table.

```
  ER hierarchy to translate ('id' is the primary key throughout):

        +--------+
        | Person |   id, name
        +--------+
           /_\         Employee and Student are kinds of Person
          /   \
   +----------+  +---------+
   | Employee |  | Student |
   |  salary  |  | enrolYr |
   +----------+  +---------+

  Three ways to turn this into tables:

  (1) Collapse INTO PARENT  -> one table holding everyone
      Person( id, name, salary*, enrolYr*, type )
                        '------- child-only, NULL    '-- discriminator:
                         for rows of the other kind       'Employee' / 'Student'
      + simple, no joins         - many NULLs; needs a CHECK for disjointness

  (2) Collapse INTO CHILDREN -> one table per subtype (parent attrs copied down)
      Employee( id, name, salary )
      Student ( id, name, enrolYr )
      + no NULLs                 - parent attrs duplicated; a plain Person
                                   (neither kind) has nowhere to live

  (3) Keep PARENT + child tables, linked by the shared key
      Person  ( id, name )
      Employee( id, salary )   id -> Person     (id is both PK and FK)
      Student ( id, enrolYr )  id -> Person
      + no duplication, no NULLs - reading a whole subtype needs a join
```

The result may need a final **logical restructuring** for efficiency:

- A **cost model** estimates operation cost from page counts (pages occupied by a table, and tuples per page), so packing more tuples per page — fewer pages — speeds selections and unindexed joins. The formulas and the physical mechanics behind them are in `Database Internals.md`.
- **Decomposition** (vertical to enable normalization and narrow rows; horizontal to separate ranges) and **merging** (to eliminate frequent joins) trade these costs, each reformulating the affected constraints. The original tables remain recoverable through views.

---

## 7. Normalization

Normalization decomposes relations so that **each fact lives in exactly one place**. It is a formal correctness criterion for a relational schema, applied after translation (or used to validate it).

### 7.1 Why it exists: the three anomalies

When one fact is stored in many rows, the copies drift apart. A table that keeps a user's email on every order row exhibits three failure modes:

- **Update anomaly** — changing the email requires updating every row that repeats it. Miss one and the database contradicts itself.
- **Insertion anomaly** — a fact cannot be recorded until an unrelated fact exists. A new product cannot be stored until someone orders it, because the product name only lives on an order row.
- **Deletion anomaly** — removing one fact destroys another. Deleting a user's last order also erases the only copy of their email.

Normalization eliminates all three by giving every fact exactly one home, so no second copy can fall out of sync.

```
Unnormalized:
+----------+--------+----------+------------+----------+
| order_id | user   | email    | product    | qty      |
+----------+--------+----------+------------+----------+
| 1        | alice  | a@ex.com | widget     | 2        |
| 2        | alice  | a@ex.com | gadget     | 1        |
+----------+--------+----------+------------+----------+
              ^^^^^^^^^^^^^^^^^
              email duplicated; one change touches many rows

Normalized (3NF):
users:                    orders:                  order_items:
+----+-------+----------+ +----+---------+-------+ +----+----------+---------+-----+
| id | name  | email    | | id | user_id | total | | id | order_id | product | qty |
+----+-------+----------+ +----+---------+-------+ +----+----------+---------+-----+
| 1  | alice | a@ex.com | | 1  | 1       | 20    | | 1  | 1        | widget  | 2   |
+----+-------+----------+ | 2  | 1       | 10    | | 2  | 2        | gadget  | 1   |
                          +----+---------+-------+ +----+----------+---------+-----+
```

### 7.2 Functional dependencies

The formal basis is the **functional dependency (FD)**. Write `X -> Y` ("X determines Y") when, for every pair of tuples agreeing on attributes `X`, they also agree on `Y` — fixing `X` fixes `Y`. A `product_id` fixes its `product_name`, so `product_id -> product_name`.

```
  X -> Y   "X determines Y" : any two rows equal on X are also equal on Y

      product_id | product_name
      -----------+-------------
         P1      |   widget        same product_id  =>  same product_name
         P1      |   widget
         P2      |   gadget
      ^^^^^^^^^^                   left side = the DETERMINANT
      product_id -> product_name
```

- A **determinant** is the left side of an FD.
- A **superkey** is a set of attributes that functionally determines the entire row; a **candidate key** is a minimal such set — no attribute can be dropped while it still determines the row.

A normal form is a rule restricting *which* FDs a table may contain. Each form bans the kind of dependency that lets one fact be stored in many rows.

### 7.3 The normal forms

Through 3NF the whole rule reduces to one sentence:

> Every non-key column must depend on **the key, the whole key, and nothing but the key.**

Each clause is one normal form:

- **"the key"** — every row is identified by a primary key. (**1NF**)
- **"the whole key"** — a column must not depend on only *part* of a composite key. (**2NF**)
- **"nothing but the key"** — a column must not depend on another non-key column. (**3NF**)
- Beyond the mnemonic, **BCNF** (Boyce-Codd Normal Form) extends 3NF so the rule holds for *every* column, not just non-key ones.

Whenever a column is fixed by something other than the row's own key, it is really describing a different thing and belongs in a different table; leaving it in place duplicates a fact and reintroduces the anomalies.

```
  Normal forms are NESTED — each is strictly stronger than the previous:

  +------------------------------------------------------+
  | 1NF   a key exists, every cell holds a single value  |
  | +--------------------------------------------------+ |
  | | 2NF   no column depends on PART of the key       | |
  | | +----------------------------------------------+ | |
  | | | 3NF   no column depends on a NON-KEY column  | | |
  | | | +------------------------------------------+ | | |
  | | | | BCNF  every determinant is a superkey    | | | |
  | | | |       (rule holds for every column)      | | | |
  | | | +------------------------------------------+ | | |
  | | +----------------------------------------------+ | |
  | +--------------------------------------------------+ |
  +------------------------------------------------------+
   a table in BCNF is automatically in 3NF, then 2NF, then 1NF
```

**1NF — a key exists, and every cell holds one value.** No repeating groups, no lists packed into a column, no meaning attached to row order. A `tags` column holding `"red,large,sale"` violates 1NF: a query for "everything tagged `sale`" must parse strings instead of matching a value, and no index helps. The fix is one row per value in a `product_tags` table.

**2NF — depend on the *whole* key.** Only relevant when the key spans several columns. If a column is fixed by only part of the key, it describes that part, not the row.

```
order_items(order_id, product_id, qty, product_name)
            \_________ key _________/

  qty           depends on order_id AND product_id   -> the whole key    OK
  product_name  depends on product_id alone           -> part of the key  VIOLATION

product_name describes the product, not this order line, so it repeats on
every line for that product (update anomaly), and a never-ordered product
has nowhere to live (insertion anomaly).

Fix: product_name moves to a products table keyed by product_id.
```

**3NF — depend on *nothing but* the key.** Even with a single-column key (so 2NF holds automatically), a column can depend on another *non-key* column rather than the key itself.

```
employees(emp_id, dept_id, dept_name)
          \_key_/

  dept_id    depends on emp_id    -> the key                 OK
  dept_name  depends on dept_id   -> another non-key column  VIOLATION

emp_id reaches dept_name only by way of dept_id (a "transitive" dependency),
so dept_name repeats for every employee in the department.

Fix: dept_name moves to a departments table keyed by dept_id.
```

2NF and 3NF are the same correction from two angles: a column whose real determinant is not the table's key duplicates, so move it to the table where its determinant *is* the key.

**BCNF — the rule holds for every column.** 3NF leaves one rare gap: a *prime* attribute (one that is part of some candidate key) is allowed to depend on something that is not a whole key. BCNF closes it by requiring that anything which fixes another column be a **superkey** itself. It only matters when a table has two overlapping candidate keys — for example a class schedule where both `(room, time)` and `(instructor, time)` identify a row and `instructor` fixes `room`. Here `instructor` is not a superkey, so BCNF rejects the table; but `room` is a prime attribute, so 3NF tolerates the dependency — that is exactly the gap. Tables already in 3NF are almost always in BCNF; this is a corner case.

**4NF and 5NF** extend the same idea to multi-valued facts. Storing two independent one-to-many facts in one table (a person's phone numbers and their email addresses) forces a row for every combination; 4NF separates them. 5NF covers the rarer case where a table is the spurious result of joining others. Neither is targeted deliberately in practice — one table per distinct relationship avoids both.

### Lossless-join decomposition

A decomposition of relation `R` into `R1, ..., Rn` is **lossless** (non-additive) if joining the parts always reconstructs exactly `R` — `R1 ⋈ ... ⋈ Rn = R` on every instance. A lossy decomposition reintroduces *spurious* tuples that were never in `R`, so losslessness is the minimum correctness requirement of any normalizing decomposition.

For a binary split, the test is local to the shared attributes: decomposing `R` into `R1` and `R2` is lossless iff their common attributes form a key of at least one side —

```
(R1 ∩ R2) -> R1   or   (R1 ∩ R2) -> R2
```

A second, weaker goal is **dependency preservation**: every FD of `R` should be enforceable on a single `Ri`, so no FD requires a join to check. BCNF decompositions are always lossless but not always dependency-preserving; 3NF can always be reached by a decomposition that is both lossless and dependency-preserving. This is the practical reason 3NF, not BCNF, is the usual target.

### 7.4 In practice

"Normalize to 3NF absent a specific reason not to." 3NF removes the anomalies that matter for transactional workloads while keeping the schema queryable. Higher forms are rarely needed.

**Denormalization** — deliberate duplication for read performance — is the far more common deliberate deviation. It trades write complexity and storage for fewer joins per read, and is appropriate when the same join runs on every read of a hot path, writes are rare relative to reads, and eventual consistency between copies is acceptable. The relational source of truth stays normalized; denormalized copies belong in materialized views, search indexes, and analytical projections (see `Data Systems.md`).
