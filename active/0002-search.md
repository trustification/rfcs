# Search API

Define a pattern for a search API, adding concrete definitions for searching "packages" and "vulnerabilties".

## Optional: Glossary 

<dl>
    <dt>Package</dt>
    <dd>A package from an SBOM, either the main SBOM package, or a component</dd>
    <dt>Vulnerability</dt>
    <dd>A vulnerability report from a security advisory</dd>
</dl>

## Motivation

Right now we mostly perform lookups, a user would need to know the exact information (id), to retrieve the information.
That is not always reasonable and prevents the user from exploring the data.

In order to support such use cases, we need to define a way to search for data. More explicitly, we need to for
searching packages and vulnerabilities now, maybe for other things later.

The design should be:

1. Extensible for future additions
2. Suitable for manual as well as programmatic use
3. Reasonably simple for implementing on the backend side
4. Support creating a UI/frontend to improve usability

## Design

The basic idea of the search API is it provide a simple, known, query language that is easy to understand and easy to
implement. The goal is not to re-create a full-blown query language as SQL, but provide more flexibility than a few
HTTP query parameters.

This idea builds on the query syntax from GitHub (an others).

### Query language

The basic idea is to use query string such as: `primary qualifier:value is:predicate`.

Where `primary` is performing a full text search of "relevant" (primary) data. The default definition of "relevant"
depends on the type of data that is being searched for. This can be overridden using `in:qualifier` to search in
specific data. Multiple `in` statement will be additive.

`qualifier:filter` searches for `value` on the selected `qualifier`. Searching for qualifiers is an exact match, unless
additional filter operators (like `>` or `*..100` are being used).

Qualifiers may use additional `:` (colons) to sub-scope the filter. For example: `qualifier:type:jar` would search
for the PURL qualifier (`qualifier`) named `type` that has the value of `jar`. 

`is:predicate` simply checks if a `predicate` is `true`.

Qualifier and predicates can be inverted by prefixing with a `-` (minus). For example: `-is:predicate` or
`-qualifier:value`.

Strings can be wrapped with double quotes to allow using spaces, commas, and other control characters. For
example: `cve:"Foo, Bar"` 

### Filter operators

Depending on the type of the qualifier, it is possible to add additional filter operators:

#### String

None.

#### Numeric / Date / Timestamp

The following operators are available:  

* `>` – Greater than the provided value
* `>=` – Greater or equal than the provided value
* `<` – Less than the provided value
* `<=` – Less or equal than the provided value
* `..` – Between the provided values (may use `*` as wildcard)

Examples:

* `cvss:>5`: A CVSS greater than five
* `created:2020-01-01..2021-01-01`: Created between 1st of January 2020 (inclusive) and 1st of January 2021 (exclusive)
* `fixed:2022-10-01T15:00:00Z..*`: Fixed after 1st of October 2022, 3pm UTC (inclusive). 

#### Versions

Using semantic versioning filters.

### Sorting

Each resource which can be search defined a default/primary sort order. This can be influenced as part of the search
string using the format `sort:qualifier` to search ascending by `qualifier` or prefix with a `-` (minus) to sort 
descending (for example: `-sort:qualifier`).

Multiple sort statements are possible, these will be applied in the order of appearance. Conflicting statements will
be silently ignored and the first statement will be used.

### Package and VEX definitions

NOTE: Both example are basic minimum definitions. As the search API is considered to be easily extensible, adding
more options shouldn't be a big problem if the data exists and can be indexed.

#### Packages

The assumption is that SBOMs did get parsed and packages from `.metadata.component` or `.components` did get
extracted and indexed.

Qualifiers (primary, qualifier, sort):

| Name               | Type    | As Primary   | Description                                             |
|--------------------|---------|--------------|---------------------------------------------------------|
| `purl`             | String  | ✔️           | The full Package URL                                    |
| `type`             | String  |              | The `type` part of the Package URL                      |
| `namespace`        | String  |              | The `namespace` part of the Package URL                 |
| `name`             | String  | ✔️ (default) | The `name` part of the Package URL                      |
| `version`          | Version |              | The `version` part of the Package URL                   |
| `description`      | String  | ✔️           | The `.description` field                                |
| `digest`           | String  |              | Any digest (SHA, SHA-256, …) matches the provided value |
| `qualifier:<name>` | String  |              | Match the value of the PURL qualifier `<name>`          |

Predicates:

| Name        | Description                             |
|-------------|-----------------------------------------|
| `library`   | The field `.type` equals to `library`   |
| `framework` | The field `.type` equals to `framework` |

Examples:

* `postgres`: Search for all packages which contain `postgres` in their name
* `log4j in:description`: Search for all packages which contain `log4j` in their description.
* `postgres type:maven`: Search for all Maven packages which contain `postgres` in their name


##### Vulnerabilities

Qualifiers (primary, qualifier, sort):

| Name            | Type      | As Primary   | Description                         |
|-----------------|-----------|--------------|-------------------------------------|
| `title`         | String    | ✔️ (default) | The document title                  |
| `id`            | String    | ✔️ (default) | The ID of the security advisory     |
| `release`       | Timestamp |              | The current release date            |
| `initial`       | Timestamp |              | The initial release date            |
| `discovery`     | Timestamp |              | The discovery date                  |
| `cve`           | String    | ✔️ (default) | The CVE ID                          |
| `cvss`, `cvss3` | Numeric   |              | The CVSS 3 score                    |
| `package`       | String    | ✔️           | The URL of the package this affects |

Predicates:

| Name       | Description                                 |
|------------|---------------------------------------------|
| `final`    | The status is `final`                       |
| `critical` | The CVSS 3 score is in the "critical" range |
| `high`     | The CVSS 3 score is in the "height" range   |
| `medium`   | The CVSS 3 score is in the "medium" range   |
| `low`      | The CVSS 3 score is in the "low" range      |

Examples:

* `RHSA-2023`: All vulnerabilities which have a report ID, CVE ID or title containing "RHSA-2023"
* `is:criticial release:2022-01-01..*`: All critical vulnerabilities released after January 1st 2022

### Limitations

Due to the syntax, the following list of strings cannot be used as a name for qualifiers:

* `is`
* `in`
* `sort`

### Paging

The search APIs should support "paging". The idea being to be able to fetch chunks of data (pages), instead of
pulling in the full result list. This should solve two issues:

1. Prevent the both sides of the API from being overloading due to a large result set
2. Allow the user to "step through" pages of data (prev/next navigation)
3. Allow the implementation of a "too many results" mechanism

Paging is controlled by two query parameters:

* `start` - The number of items to skip in the start of the result set
* `length` - The maximum number of items to return in the result set

## Alternatives

### `AND` / `OR` / `NOT`

By default, search expressions are additive (AND). It would be possible to extend this with an explicit `AND` and
additional `OR`.

GitHub also has `NOT` to invert the primary search (like `foo NOT bar`, searching for `foo` but not `bar` in the
primary text search).

### Selective Sorting

The basic idea is to have all qualifiers sortable. This might not be feasible for some fields, so we might
actually limit this (like description fields).

### Implementations

TODO: If there is a suitable implementation already, we might just use this and adapt the query syntax to it.
