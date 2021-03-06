#### [Why another GraphqQL Server?](https://github.com/sikanhe/gqtx/blob/master/WHY.md)

## Getting Started

<p align="center">
<code>yarn add gqtx</code>
</p>

## Type-safety without manual work

`gqtx` is a thin layer on top of `graphql-js` for writing a type-safe GraphQL server in TypeScript. It provides you with a set of helper functions to create an intermediate representation of a GraphQL schema, and then converts that schema to a raw `graphql-js` schema. So you get to use everything from the reference implementation of GraphQL, but with way more typesafety. 

If a schema compiles, the following holds:

- The type of a field agrees with the return type of the resolver.
- The arguments of a field agrees with the accepted arguments of the resolver.
- The source of a field agrees with the type of the object to which it belongs.
- The return type of the resolver will not be input types (InputObject)
- The arguments of a field will not be abstract types (Interface, Union)
- The context argument for all resolver functions in a schema agree.

Most importantly, we achieve all this *without* having to:

- Set up code generation tools
- Write SDL and having your schema patially defined in code and in a DSL file
- Require special compiler magic such as `reflect-metadata` and decorators

### What does it look like?

```ts
import t, { buildGraphQLSchema } from 'gqtx';

enum Role {
  Admin,
  User,
}

type User = {
  id: number;
  role: Role;
  name: string;
};

const users: User[] = [
  { id: 1, role: Role.Admin, name: 'Sikan' },
  { id: 2, role: Role.User, name: 'Nicole' },
];

const RoleEnum = t.enumType({
  name: 'Role',
  description: 'A user role',
  values: [{ name: 'Admin', value: Role.Admin }, { name: 'User', value: Role.User }],
});

const UserType = t.objectType<User>({
  name: 'User',
  description: 'A User',
  fields: () => [
    t.defaultField('id', t.NonNull(t.ID)),
    t.defaultField('role', t.NonNull(RoleEnum)),
    // `defaultField` is the safe vesion of a default resovler
    // field. In this case, field 'name' must exist on `User`
    // and its type must be `string`
    t.defaultField('name', t.NonNull(t.String)),
  ],
});

const Query = t.queryType({
  fields: [
    t.field('userById', {
      type: UserType,
      args: {
        id: { type: t.NonNullInput(t.ID) },
      },
      resolve: (_, args) => {
        // `args` is automatically inferred as { id: string }
        const user = users.find(u => u.id === args.id);
        // Also ensures we return an `User | null` type
        return user || null;
      },
    }),
  ],
});

const schema = buildGraphQLSchema({
  query: Query,
});
```

#### Use your favorate server option to serve the schema!

```ts
import express from 'express';
import graphqlHTTP from 'express-graphql';

const app = express();

app.use(
  '/graphql',
  graphqlHTTP({
    schema,
    graphiql: true,
  })
);

app.listen(4000);
```

## To Recap

- We created an intermediate representation of a GraphQL schema via the helper functions exported by this library.
- Then, we converted the schema to a real graphql-js schema by calling `buildGraphQLSchema` at server startup time.
- Used existing express middleware `express-graphql` to server our schema with `graphiql` explorer
- That's it! We get a fully typesafe server with almost zero type annotation needed
