**MikroORM** is 
> TypeScript ORM for Node.js based on Data Mapper, Unit of Work and Identity Map patterns.

Ben later switch to **TypeORM**. You may want to use a more recent ORM like **Prisma** but the tutorial doesn't cover that use case.

The database used is **Postgresql**.

Add the packages for mikro-orm and the database driver :
```bash
yarn add @mikro-orm/core @mikro-orm/postgresl
```

Enable support for decorators and other options in **tsconfig.json** :
```bash
"experimentalDecorators": true,
"emitDecoratorMetadata": true,
"esModuleInterop": true,
```

Add the migrations package for mikro-orm :
```bash
yarn add @mikro-orm/migrations
```

And the mikro-orm's cli :
```bash
yarn add @mikro-orm/cli
```

Create a new mikro-orm instance in **src/index.ts** :
```ts
import {MikroORM} from "@mikro-orm/core";
import type {PostgreSqlDriver} from "mikro-orm/postgresql";
import {__prod__} from "./constants";
import {Post} from "./entities/Post"

const main = async () => {
	const orm = await MikroORM.init<PostgreSqlDriver>({
		entities: [Post],
		dbName: "lireddit",
		type: "postgresql",
		user: "",
		password: "",
		debug: !__prod__
	});
};

main();
```

Create the **constants.ts** file :
```ts
export const __prod__ = process.env.NODE_ENV !=== 'production'
```

Create the Post entity in **src/entities/Post.ts** :
```ts
@Entity()
export class Post {
	@PrimaryKey()
	id!: number;

	@Property()
	createdAt = new Date()

	@Property({ onUpdate: () => new Date() })
	updatedAt = new Date()

	@Property()
	title!: string
}
```

> The _@Propery()_ decorator tells MikroORM that the field is a database column and not just a regular class field.

To create a new post :
```ts
const post = orm.em.create(Post, {title: "first post"});
await orm.em.persistAndFlush(post);
```

## Set up mikro-orm CLI

Add the following options to **package.json** :
```json
"dependencies": { ... },  
"mikro-orm": {  
	"useTsNode": true,  
	"configPaths": [  
		"./src/mikro-orm.config.ts",  
		"./dist/mikro-orm.config.js"  
	]  
}
```

Create the mikro-orm.config.ts config file :
```ts
import {Options} from "@mikro-orm/core";

const config: Options {  
	entities: [Author, Book, BookTag], // no need for `entitiesTs` this way  
	dbName: 'my-db-name',  
	type: 'mongo', // one of `mongo` | `mysql` | `mariadb` | `postgresql` | `sqlite`  
	debug: !__prod__
};

export default config;
```

And then, init mikroORM with the previous config :
```ts
import microConfig from "./mikro-orm.config.ts";

const orm = await MikroORM.init(microConfig);
```

## Set up mikroORM migrations
From the mikroORM documentation :
> By default, each migration will be all executed inside a transaction, and all of them will be wrapped in one master transaction, so if one of them fails, everything will be rolled back.

Add the migrations options to the _mikro-orm.config.ts_ file :
```ts
import path from "path";

await MikroORM.init({  
	// default values:  
	migrations: {  
	tableName: 'mikro_orm_migrations', // name of database table with log of executed transactions  
	path: path.join(__dirname, "./migrations"), // path to the folder with migrations 
	glob: '!(*.d).{js,ts}', // how to match migration files (all .js and .ts files, but not .d.ts) 
	},  
})
```

Run the migration command :
```bash
npx mikro-orm migration:create
```

> It may be required to create the database before running the initial migration.

Install _ts-node_ to have the migrations folder and files inside the _src_ folder. Otherwise, the migrations will only show up in the _dist_ folder which contains the compiled js.
```bash
yarn add -D ts-node
```

To automatically run the migrations, add the following to mikro-orm init :
```ts
await orm.getMigrator().up();
```

In the config passed to the mikro-orm init, set the allowGlobalContext to true:
```ts
{
	allowGlobalContext: true
}
```
