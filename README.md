# fastify-zod

## Why?

`fastify` is awesome and arguably the best Node http server around.

`zod` is awesome and arguably the best TypeScript modeling / validation library around.

Unfortunately, `fastify` and `zod` don't work together very well. [`fastify` suggests using `@sinclair/typebox`](https://www.fastify.io/docs/latest/TypeScript/#typebox), which is nice but is nowhere close to `zod`. This library allows you to use `zod` as your primary source of truth for models with nice integration with `fastify`, `fastify-swagger` and OpenAPI `typescript-fetch` generator.

## Goals

- Use `zod` to validate input and output with type-safety in `fastify`
- Generate OpenAPI spec with `fastify-swagger` out of the box
- Generate OpenAPI client with `typescript-fetch` generator with human-friendly model names based on generated spec

## Examples

See the [fastify-zod-example](./fastify-zod-example) package for a full example, including generating an OpenAPI spec document and an OpenAPI client.

## Installation

You will need of course `fastify` and `zod`, but also `zod-to-json-schema` and `fastify-swagger`:

```
$ npm install --save fastify zod zod-to-json-schema fastify-swagger fastify-zod
```

## Usage

Define your models using `zod` as usual:

```ts
const TodoItemId = z.object({
  id: z.string().uuid(),
});
type TodoItemId = z.infer<typeof TodoItemId>;

const TodoItem = TodoItemId.extend({
  label: z.string(),
  dueDate: z.date().optional(),
  state: z.union([
    z.literal(`todo`),
    z.literal(`in progress`),
    z.literal(`done`),
  ]),
});

type TodoItem = z.infer<typeof TodoItem>;

const TodoItems = z.array(TodoItem);
type TodoItems = z.infer<typeof TodoItems>;
```

Then generate and register the schemas generated by this library:

```ts
import swagger from "fastify-swagger";
import { buildJsonSchemas, withRefResolver } from "..";

const { schemas, $ref } = buildJsonSchemas({
  TodoItemId,
  TodoItem,
  TodoItems,
});

for (const schema of schemas) {
  app.addSchema(schema);
}

app.register(
  swagger,
  withRefResolver({
    routePrefix: `/openapi`,
    exposeRoute: true,
    staticCSP: true,
    openapi: {
      info: {
        title: `Zod Fastify Test Server`,
        description: `API for Zod Fastify Test Server`,
        version: `0.0.0`,
      },
    },
  })
);
```

You can now define routes that will work nicely with `zod` for validation:

```ts
app.get<{
  Reply: TodoItems;
}>(
  `/item`,
  {
    schema: {
      operationId: `getTodoItems`,
      response: {
        200: $ref(`TodoItems`),
      },
    },
  },
  async () => /* ... */,
);

app.post<{
  Body: TodoItem;
  Reply: TodoItems;
}>(
  `/item`,
  {
    schema: {
      operationId: `postTodoItem`,
      body: $ref(`TodoItem`),
      response: {
        200: $ref(`TodoItems`),
      },
    },
  },
  async ({ body: nextItem }) => /* ... */,
);
```

You can also generate an OpenAPI spec document:

```ts
import { writeFile } from "fs/promises";
app
  .inject({
    method: "get",
    url: "/openapi/yaml",
  })
  .then((res) => writeFile("spec.yml", res.body));
```

And generate the corresponding client:

```
$ openapi-generator-cli generate
```