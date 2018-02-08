### GraphQL server side schema generation

Lots of tools to generate GraphQL schemas already exists. What makes this different?

* Goals here does not include easy communication with backend databases / REST services. This is meant to generate code for the backend that can talk to _any_ service.
* What is a consideration is generating code that enables _strong_ typing using TypeScript. IE. for every GraphQL type an appropriate TypeScript type should be generated on the backend.
* Great error reporting. I don't want people working with this to be bothered by errors popping up from the GraphQL library. While the errors there are actually quite nice and descriptive, I want the errors to be given in the context of the files that the user themselves have written, not as a part of the generated files.

## Workflow

The workflow I have in mind is using GraphQL schema DDL to specify types. The ideal workflow here is that the definitions are spread throughout many files. Ie:

* schema/
    * MyEnum.graphql
    * MyType/
        * MyType.common.graphql
        * MyType.server.graphql
        * MyType.graphql

It is not my intention to enforce a specific structure or naming convention, other than that the files must end in `.graphql`.

Take, as an example, a simple enum type:

```graphql
enum MyEnum {
	Value1
	Value2
}
```

For this sort of type we'd want to generate a type like this:

```ts
const enum MyEnum {
	Value1 = 'Value1',
	Value2 = 'Value2',
}
```

Obviously there'd also be generated the appropriate GraphQL type on the backend (something like):

```ts
const MyEnumType = GraphQLEnumType({
	values: {
		Value1: {},
		Value2: {},
	},
});
```

Clearly, we see some stuff missing here. There's no descriptions and no server values. Something like this should express that desire:

```graphql
"""
My enum description
"""
enum MyEnum {
	"""
	Some value 1 description
	"""
	Value1 @server(value: "value1")

	"""
	Some value 2 description
	"""
	Value2 @server(value: "value2")
}
```

Which would then end up as:

```ts
/**
 * My enum description
 */
const enum MyEnum {
	/**
	 * Some value 1 description
	 */
	Value1 = 'value1'
	/**
	 * Some value 2 description
	 */
	Value2 = 'value2'
}
const MyEnumType = GraphQLEnumType({
	description: 'My enum description',
	values: {
		Value1: {
			description: 'Some value 1 description',
			value: 'value1',
		},
		Value2: {
			description: 'Some value 2 description',
			value: 'value2',
		},
	},
});
```

## More complex examples

Obviously generating enum types is kind of a joke. There's no references to other GraphQL types, and there's no actual runnable code involved.

### Object types

Handling object types is where things get real interesting and complicated. A naive approach is to simply generate the GraphQLObjectType along with a class to represent the server side value. However - this won't work. Usually the shapes of the server side data and the data exposed through GraphQL differ in significant ways (think of relations in SQL databases, which in the model might be a simple ID that is then loaded in the resolve function). Also we need to represent fields that just need a resolve function to be expressed in GraphQL - ie they have no server side representation, as well as fields used on the server that have no GraphQL representation.

One suggestion here is to differentiate where the fields should be generated using directives directly on the fields:

```graphql
type MyType {
	myServerField: String @generate(for: Server)
	myGraphQLField: String @generate(for: GraphQL)
	myField: String @generate(for: Both)
}
```

Another would be on the type declaration. Ie. no directive indicates the types should be present on both the server and in the GraphQL side:

```graphql
type MyType {
	myField: String
}

extend type MyType @generate(for: Server) {
	myServerField: String
}

extend type MyType @generate(for: GraphQL) {
	myGraphQLField: String
}
```

A class would be generated like so:

```ts
export class MyType {
	myField;
}
```

To indicate a resolve function - things again turn out to be somewhat more complicated than they seem. We need a file to `import` to know where the resolution logic resides. When declaring the type (`extend` or not) where there's resolutions inside that declaration, one would need to point to that file:

```graphql
type MyType @resolvers(file: "someFile.ts") @generate(for: GraphQL) {
	myResolvedField @resolve(function: "myResolvedField")
}
```

This does several things:

1. It only adds the `myResolvedField` to the GraphQLObject type, not the class representing the type in the backend.
2. It will create some named type (possibly just placed and named along side the `.graphql` file that declared the type).

**Note**: It should be an error to generate types purely for the GraphQL side that does not use `@resolve` (as in the example further up).

```ts
import { MyType } from 'somewhere';
import { Context } from 'somewhere';
export interface MyTypeResolver {
	myResolvedField(root: MyType, context: Context): string | null | Promise<string | null>;
}
```

Now it is expected that `someFile.ts` exports a value `MyType` that implements `MyTypeResolver`, ie:

```ts
import { MyType } from 'somewhere';
import { Context } from 'somewhere';
import { MyTypeResolver } from 'somewhere';

export const MyType: MyTypeResolver = {
	myResolvedField(root: MyType, context: Context): string | null {
		return root.myServerField;
	},
};
```

Note that this does a few things:

1. We allow returning either just the value or a promise to the value in the interface that is required to be implemented.
2. In the implementation we never care to return a promise and thus we specified that we won't. It still complies with the interface.

**Note**: I'm not sure if this approach is the best one regarding getting resolvers to work. Eg. it could be nice to simply get them on the class itself - and that might also be feasible.

### Interfaces

Interfaces should flow naturally, you can declare them using the GraphQL Schema DDL. Interfaces also allow splitting fields into `Server` and `GraphQL` fields, using the `@generate` directive. It will create a TypeScript interface type with all the fields specified for the server - which the generated class will extend from.

## Closing notes

This is just a quick write up of my ideas for this. They are far from finished, and I would like some input/ideas on how this could be improved. Clearly if there are similar projects with the same kind of goals - I'd be happy to be pointed in their direction. Error reporting and type safety are two goals I'm not willing to compromise on.
