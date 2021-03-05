# Relay - Pagination and Relationship Properties

## Problem

`@neo4j/graphql` does not facilitate relationship properties which are key to giving data in a Neo4j database intent. Additionally, we have no thorough solution for pagination.

A Relay implementation will address both of these concerns.

## Usage example

### Type definitions

Relationship definitions on the nodes also point to a properties interface on either end

```graphql
type Movie @node {
  title: String!
  actors: [Actor!]!
    @relationship(type: "ACTED_IN", properties: ActedIn, direction: "IN")
}

type Actor {
  id: ID! @id(property: "_id", autogenerate: false)
  name: String!
  movies: [Movie!]!
    @relationship(type: "ACTED_IN", properties: ActedIn, direction: "OUT")
}

interface ActedIn {
  screenTime: Int!
}
```

#### Notes

- Much like node types, property interfaces can be named however users see fit. If the same relationship type can be used for different node combinations, they can name them differently to distinguish between them.
- As per the current implementation, all types will be assumed to be node types and will have the relevant queries and mutations generated for them.
- All nodes types will automatically have a field `id` of type `ID!` added to them, and this will be fully managed by `@neo4j/graphql`, mapped to a property `id` in each Neo4j node. Users can specify the `id` field themselves with an `@id` directive as per the example above if they wish for it to be mapped to a different property or wish to manage the ID themselves.
- All of this functionality will be opt in on `Neo4jGraphQL` construction. A `boolean` argument `relay` will be passed in with the type definitions.
- Furthermore, if the `boolean` argument `relay` is `true` and any of the following are used in the type definitions, an error will be thrown:
  - A field with name `id` which does not have the type `ID!` in any type
  - A `@relationship` directive "pair" where the `properties` argument does not have the same value in each directive
  - The `@id` directive used against any field other than `id` with type `ID!`

### Schema changes

If you want to check out the whole schema then it can be found [here](schema.md).

#### Node interface

The `Node` interface required for Relay is autogenerated when `relay: true` is passed into the constructor for `Neo4jGraphQL`

```graphql
interface Node {
  id: ID!
}
```

#### Changes to node types

Any type decorated with the `@node` directive will automatically implement the `Node` interface and have an `id` field added. Relay `Connection` fields are added in addition to direct relationship fields, with arguments for how many results to return and after which position.

```graphql
type Actor implements Node {
  id: ID!
  name: String!
  movies: [Movie!]!
  moviesConnection(first: Int, after: String): ActorMoviesConnection!
}

type Movie implements Node {
  id: ID!
  title: String!
  actors: [Actor!]!
  actorsConnection(first: Int, after: String): MovieActorsConnection!
}
```

#### New types for relationship properties and relationships

Relationship types are automatically generated, one for each direction of a relationship. They are synonymous with the Relay concept of edges, with a string field `cursor` and a field `node` which is the type at the other end of the relationship. The remaining fields will be from the properties interface which was specified in the `@relationship` directive, and the relationship type will implement that interface.

```graphql
interface ActedIn {
  screenTime: Int!
}

type ActorMovieRelationship implements ActedIn {
  cursor: String!
  node: Movie!
  screenTime: Int!
}

type MovieActorRelationship implements ActedIn {
  cursor: String!
  node: Actor!
  screenTime: Int!
}
```

#### Connection types

The type `PageInfo` will be automatically generated if `relay: true` is passed into the constructor arguments for `Neo4jGraphQL`. Connection types will be automatically generated for all relationships to facilitate cursor-based pagination.

```graphql
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type ActorMoviesConnection {
  edges: [ActorMovieRelationship!]!
  pageInfo: PageInfo!
}

type MovieActorsConnection {
  edges: [MovieActorRelationship!]!
  pageInfo: PageInfo!
}
```

#### Input types for relationship properties

Two new types will exist for each set of relationship properties - one for creating a relationship and one for updating the properties on an existing relationship. Two types are needed to respect if properties are mandatory on creation:

```graphql
input ActedInCreateInput {
  screenTime: Int!
}

input ActedInUpdateInput {
  screenTime: Int
}
```

#### Changes to nested creates

There are new input types to facilitate the setting of relationship properties when performing nested creates:

```graphql
input ActorMovieCreateInput {
  properties: ActedInCreateInput!
  node: MovieCreateInput!
}

input MovieActorCreateInput {
  properties: ActedInCreateInput!
  node: ActorCreateInput!
}
```

Now where nested create operations and `RelationInput` input types used the `ActorCreateInput` / `MovieCreateInput` types directly, they will use these in their place.

#### Changes to connecting nodes

`*ConnectionFieldInput` types now contain a `properties` field for setting relationship properties on new relationships.

```graphql
input ActorConnectFieldInput {
  where: ActorWhere
  properties: ActedInCreateInput
  connect: ActorConnectInput
}

input MovieConnectFieldInput {
  where: MovieWhere
  properties: ActedInCreateInput
  connect: MovieConnectInput
}
```

#### Updating relationship properties

Updating relationship properties doesn't fit well into any of the existing operations we have available on our mutations, so we will make a new one!

```graphql
input MovieUpdateConnectionFieldInput {
  where: MovieWhere
  properties: ActedInUpdateInput
}

input ActorUpdateConnectionInput {
  movies: [MovieUpdateConnectionFieldInput]
}

input ActorUpdateConnectionFieldInput {
  where: ActorWhere
  properties: ActedInUpdateInput
}

input MovieUpdateConnectionInput {
  actors: [ActorUpdateConnectionFieldInput]
}

input ActorMoviesUpdateFieldInput {
  ...
  updateConnection: [MovieUpdateConnectionFieldInput]
}

input MovieActorsUpdateFieldInput {
  ...
  updateConnection: [ActorUpdateConnectionFieldInput]
}

type Mutation {
  updateActors(
    ...
    updateConnection: ActorUpdateConnectionInput
  ): UpdateActorsMutationResponse!
  updateMovies(
    ...
    updateConnection: MovieUpdateConnectionInput
  ): UpdateMoviesMutationResponse!
}
```

#### Node query

A standard Relay `node` query will be added, allowing all nodes to be queried for a single ID:

```graphql
type Query {
  node(id: ID!): Node
}
```

### Example mutations

#### Creating a movie and an actor connected together

```graphql
mutation CreateMovieAndActor(
  $title: String!
  $name: String!
  $screenTime: Int!
) {
  createMovies(
    input: {
      title: $title
      actors: {
        create: [
          { properties: { screenTime: $screenTime }, node: { name: $name } }
        ]
      }
    }
  ) {
    movies {
      title
      actorsConnection {
        edges {
          node {
            name
          }
          screenTime
        }
      }
    }
  }
}
```

#### Creating a movie and connecting to an existing actor

```graphql
mutation CreateMovieAndConnectActor(
  $title: String!
  $name: String!
  $screenTime: Int!
) {
  createMovies(
    input: {
      title: $title
      actors: {
        connect: [
          { where: { name: $name }, properties: { screenTime: $screenTime } }
        ]
      }
    }
  ) {
    movies {
      title
      actorsConnection {
        edges {
          node {
            name
          }
          screenTime
        }
      }
    }
  }
}
```

#### Updating a relationship property

This will take advantage of a new `updateConnection` operation (name is far from final). Unlike creating new relationships with properties, updating them doesn't really seem to fit sensibly into any of the existing operations.

```graphql
mutation UpdateScreenTime($title: String, $name: String, $screenTime: Int) {
  updateMovies(
    where: { title: $title }
    updateConnection: {
      actors: [
        { where: { name: $name }, properties: { screenTime: $screenTime } }
      ]
    }
  ) {
    movies {
      title
      actorsConnection {
        edges {
          node {
            name
          }
          screenTime
        }
      }
    }
  }
}
```

## Out of scope (for this initial implementation)

- Nested relationship property updates
- Filtering by relationship properties
- Sorting on relationship properties