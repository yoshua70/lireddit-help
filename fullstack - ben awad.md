## Project setup

```bash
mkdir lireddit-server
cd lireddit-server
npm init -y
```

## Node and Typescript setup

```bash
yarn add -D @types/node typescript
```

> **-D** to add dependencies as a dev dependency

Config file for typescript : 
```bash
npx tsconfig.json
```

Add new scripts to package.json
```json
{
	"scripts": {
		"watch": "tsc -w",
		"start": "node dist/index.js",
		"dev": "nodemon dist/index.js"
	}
}
```

_watch: tsc -w_ : compile the typescript file to javascript
Compile the typescript files in real time and serving the compiled javascript files with _node_.

In order to watch changes to the project files and restart the server automatically, install _nodemon_ :
```bash
yarn add -D nodemon
```

## MikroORM setup

Install the module and the database driver :
```bash
yarn add @mikro-orm/core @mikro-orm/postgresql
```

Enable support for _decorators_ as well as _esModuleInterop_ in _tsconfig.json_ :
```json
"experimentalDecorators": true,
"emitDecoratorMetadata": true,
"esModuleInterop": true,
```

In order to use migrations, install the following packages :
```bash
yarn add @mikro-orm/migrations
```

And to use work with the CLI, first install the following package :
```bash
yarn add @mikro-orm/cli
```

In the _index.ts_, import _MikroORM_ to create a mikroORM instance :
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

Create the _constants.ts_ file :
```ts
export const __prod__ = process.env.NODE_ENV !=== 'production'
```

_entities_ correspond to all of the database tables.
```bash
mkdir src/entities/Post.ts
```

The _Post.ts_ will be the first database table :
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

The _@Propery()_ decorator tells MikroORM that the field is a database column and not just a regular class field.

Create a Post :
```ts
const post = orm.em.create(Post, {title: "first post"});
await orm.em.persistAndFlush(post);
```

### Set up mikroORM CLI
Add the following config options to the package.json file :
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

### Set up mikroORM migrations
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

## Server setup
Install the following packages :
```bash
yarn add express apollo-server-express graphql type-graphql
```

Install the types for express :
```bash
yarn add -D @types/express
```

Create a new express server in the ==index.ts== file :
```ts
import expres from "express";

const main = async () => {
	const app = express();

	app.listen(4000, () => {
		console.log("server listening at localhost:4000");
	});
}
```

Create a graphql schema :
```ts
import {buildSchema} from 'type-graphql'
```

Then, add a graphql endpoint :
```ts
import {ApolloServer} from "apollo-server-express";

// inside of the main fonction
const apolloServer = new ApolloServer({
	schema: await buildSchema({
		resolvers: [],
		validate: false
	})
})
```

The schema requires resolvers :
```bash
mkdir src/resolvers/
touch hello.ts
```

Inside ==hello.ts==, create a class :
```ts
import {Resolver, Query} from "type-graphql";

@Resolver()
export class HelloResolver {
	@Query(() => String)
	hello() {
		return "hello world";
	}
}
```

Import the previously created resolver inside the schema :
```ts
import {HelloResolver} from "./resolvers/hello"

// inside the apolloServer instance code
schema: await buildSchema({
		resolvers: [HelloResovler],
		validate: false
	})
```

Pass the express as a middleware to the apollo server instance :
```ts
apolloServer.applyMiddleware({app});
```

Go to ==localhost:4000/graphql== to access the graphql playground.

## MikroORM and TypeGraphQL CRUD
Create a new post resolver in  ==src/resolvers/post.ts== :
```ts
@Resolver()
export class PostResolver {
	@Query(() => [Post]) // return a list of Post
	posts() {
		return "bye";
	}
}
```

Add the post resolver to the schema :
```ts
resovlers: [HelloResolver, PostResolver],
```

Set the ==context== property of the apolloServer :
```ts
const apolloServer = new ApolloServer({
	context: () => ({ em: orm.em.fork() })
})
```

The context property is set to function which returns the ==orm.em== object created early. It allows access to method for accessing and modifying the data stored in the database.

Create a new file ==src/types.ts== for any custom types. Create a new type ==MyContext== :
```ts
export type MyContext = {
	em: EntityManager<IDatabaseDriver<Connection>>
}
```

Then add the ==@Field()== decorator to each property of the Posts class/entity :
```ts
@Field(() => property_type)
@Propery({type: property_type})
// define the property
```
Adding the field decorator exposes the property to graphql. If the decorator isn't set for a property, that property won't be accessible via graphql API.

Also add the ==@ObjectType()== decorator for the class Post :
```ts
@ObjectType()
@Entity()
export class Post {
	// ...
}
```

In the post resolver, add ==em== as a parameter to the resolver function :
```ts
posts(@Ctx() {em}: MyContext) {
	// return ...
}
```

### Read
Update the resolver return :
```ts
return em.find(Post, {});
```

The code for the post resolver should look like this at this points :
```ts
import {Resolver, Query, Ctx} from "type-graphql";
import {Post} from "../entities/Post";
import {MyContext} from "../types";
@Resolver()
export class PostResolver {
	@Query(() => [Post]) // return a list of Post
	posts(@Ctx {em}: MyContext): Promise<Post[]> {
		return em.find(Post, {});
	}
}
```

Let's find a post by its id. First, create a new query fo the PostResolver and add id as an argument :
```ts
@Query(() => Post, {nullable: true})
post(
	@Arg("id", () => Int) id: Number,
	@Ctx() {em}: MyContext
): Promise<Post | null> {
	return em.FindOne(Post, {id});
}

```

### Create
For updating or inserting data, use the ==@Mutation== decorator :
```ts
@Mutation(() => Post)
async createPost(
	@Arg("title") title: string,
	@Ctx() {em}: MyContext
): Promise<Post> {
	const post = em.create(Post, {title});
	await em.persistAndFlush(post);
	return post;
}
```

### Update
First fetch the post to be updated, then modify the object received from the database and finally persist and flush the object :
```ts
@Mutation(() => Post, {nullable: true})
async createPost(
	@Arg("id") id: number,
	@Arg("title") title: string,
	@Ctx() {em}: MyContext
): Promise<Post | null> {
	const post = await em.findOne(Post, {id});
	if (!post) {
		return null;
	}
	if (typeof title !== "undefined") {
		post.title = title;
		await em.persistAndFlush(post);
	}
	
	return post;
}
```

If some fields are optionnal, they can be made nullable by doing as follow :
```ts
@Arg("title", () => String, {nullable: true}) title string,
```

### Delete
```ts
@Mutation(() => Boolean)
async deletePost(
	@Arg("id") id: number,
	@Ctx() {em}: MyContext
): Promise<Post> {
	await em.nativeDelete(Post, {id});
	
	return true;
}
```

It is possible to try/catch the nativeDelete operation, in case an error occured, a value of ==false== may be returned in the catch section.

## Authentication
### Register Resolver
Create a new user entity inside ==src/entities/User.ts== :
```ts
@ObjectType()
@Entity()
export class User {
	// keep the id and the timestamps fields
	@Field(() => String)
	@Property({type: "text", unique: true})
	username!: string;

	@Propery({type: "text"})
	password!: string;
}
```
Notice that there's no ==@Field()== decorator on the password property, thus the password property and is not exposed and not selectable in the API.

_Optional : create a yarn script for migration_
```json
{
	"scripts": {
		"create:migration": "mikro-orm migration:create"
	}
}
```

Create a resolver for registration inside ==src/resolvers/user.ts==, it will use an ==@InputType()== to store the user data :
```ts
@InputType()
class UsernamePasswordInput {
	@Field()
	username: string;

	@Field()
	password: string;
}

@Resolver()
export class UserResolver {
	@Mutation(() => User)
	async register(
		@Arg("data") data: UsernamePasswordInput,
		@Ctx() {em}: MyContext
	) {
		const hashedPassword = await argon2.hash(data.password);
		const user = em.create(User, {
			username: data.username,
			password: data.password
		});
		await em.persistAndFlush(user);
		return user;
	}
}
```

The type of the argument for the resolver can be specified or inferred by typescript, in case it must be specified, do as follow :
```ts
@Arg("data", () => UsernamePasswordInput) data: UsernamePasswordInput,
```

The user password is not stored in plain text, it is hashed first. The hash algorithm used here is _argon2_, there's a node package for it :
```bash
yarn add argon2
```

After that, add the UserResolver to the schema inside of ==src/index.ts== :
```ts
schema: await buildSchema({
	resolvers: [UserResolver]
})
```

### Login resolver
Let's handle the login :
```ts
@Mutation(() => UserResponse)
async login(
	@Arg("data") data: UsernamePasswordInput,
	@Ctx() { em }: MyContext
): Promise<UserResponse> {
	const user = await em.findOne(User, { username: data.username });

	if (!user) {
		return {
			errors: [
				{
					field: "username",
					message: "that username doesn't exist",
				},
			],
		};
	}

	const valid = await argon2.verify(user.password, data.password);

	if (!valid) {
		return {
			errors: [
				{
					field: "password",	
					message: "invalid logins",
				},
			],
		};
	}

	return {
		user,
	};
}
```

In order to handle the error, create a new type called FieldError :
```ts
@ObjectType()
class FieldError {
	@Field()
	field: string;

	@Field()
	message: string;
}
```

The return type of the login mutation is the following :
```ts
@ObjectType()
class UserResponse {
	@Field(() => [FieldError], {nullable: true})
	errors?: FieldError[];

	@Field(() => User, {nullable: true})
	user?: User;
}
```
### Session
To store the user's data on the server, let's use express-session. Many databases can be used with express-session for example reddis. For that a middleware is required.
```bash
yarn add reddis connect-reddis express-session
```

Install the types for installed packages :
```bash
yarn add -D @types/redis @types/express-session @types/connect-redis
```

If get an error or assignable types, try rolling back the following packages in package.json :
```json
"@types/express-session": "1.17.1"
"@types/connect-redis": "^0.0.14"
```

And execute :
```bash
yarn install
```

Install redis :
```bash
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt-get update
sudo apt-get install redis
```

Verify the installation :
```bash
redis-cli ping
```

It should respond with _pong_.

In the ==src/index.ts== add the following code :
```ts
import redis from "redis";
import session from "express-session";
import connectRedis from "connect-redis";

const main = async () => {

	// After creating a new app instance of express
	
	const RedisStore = connectRedis(session);
	const redisClient = redis.createClient();

	app.use(
		session({
			name: 'qid', // name of the cookie
			store: new RedisStore({client: redisClient}),
			// think about making the secret an env variable
			secret: "keyboard cat",
			resave: false,
			saveUninitialized: false,
		}),
	)

	// Before applying middleware to the apollo server
}
```

There's also some options that can be specified in the ==RedisStore== :
```ts
store: new RedisStore({
	client: redisClient,
	disableTouch: true,
})
```

The ==disbaleTouch== option is about the resetting the TTL when the user interacts with the session. It helps keep the users session alive if changes are infrequent.

Add another option for creating the cookie to the session :
```ts
session({
	cookie: {
		maxAge: 1000 * 60 * 60 * 24 * 365 * 10, // 10 years
		httpOnly: true, // cookie not accessible on the client
		sameSite: 'lax', // csrf
		secure: __prod__ // cookie only works in https
	},
})
```

Then, pass the session to the context object in order to access them in the resolvers :
```ts
context: ({req, res}): MyContext => ({em: orm.em.fork(), req, res})
```

In the user resolver, on login, access the request object from the context :
```ts
@Ctx() {em, req}: MyContext
```

It possesses a session variable inside of which we can store anything to use later :
```ts
req.session.userId = user.id;
```

You may run into an issue where typescript tells you that the attribut _userId_ doesn't exist on the session object. To solve this follow these instructions :

In ==tsconfig.json== add the following options :
```json
{
	"compilerOptions": {
		"typeRoots": {
			"./src/types",
			"./node_modules/@types"
		}
	}
}
```

Then create a new file ==src/types/index.ts== :
```ts
import session from "express-session";

export = session;

declare module "express-session" {
	interface SessionData {
		userId: number;
	}
}
```

Rename the file ==src/types.ts== to ==src/types/MyContext.ts==.

In ApolloStudio set ==request.credentials== to ==include==.

Create a new query in the user resolver :
```ts
@Query(() => User, {nullable: true})
async me(
	@Ctx() {req, em}: MyContext
) {
	// you are not logged in
	if (!req.session.userId) {
		return null
	}

	const user = await em.findOne(User, {id: req.session.userId});
	return user;
}
```

This query returns the current logged in user informations.

To autologin after registering :
```ts
@Ctx() {em, req}: MyContex

// Before returning the user
// Set the cookie on the user
// Keep the user logged in
req.session.userId = user.id;
```