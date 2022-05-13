We use **yarn** as our node package manager. To install yarn on your machine you can follow the instructions at those links :
- [The official yarn documentation](https://classic.yarnpkg.com/lang/en/docs/install/#debian-stable)
- [Digital Ocean tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-the-yarn-package-manager-for-node-js)
- [For Ubuntu users](https://linuxize.com/post/how-to-install-yarn-on-ubuntu-20-04/)
In your project directory, add **typescript** and the types for node.js :
```bash
yarn add -D @types/node typescript
```

> **-D** means we are adding those packages as dev dependencies, to learn more about it check the following [link](https://stackoverflow.com/questions/18875674/whats-the-difference-between-dependencies-devdependencies-and-peerdependencies).

typescript requires a **tsconfig.json** file. Ben created a package to generate one, just run the following command :
```bash
npx tsconfig.json
```

We'll then add some scripts that we are going to use in the rest of the tutorial. In **package.json**, add the following code :
```json
{
	"scripts": {
		"watch": "tsc -w",
		"start": "node dist/index.js",
		"dev": "nodemon dist/index.js"
	}
}
```

> **watch: tsc -w** compiles the typescript files in **src/** to javascript files located in **dist/** on the fly.

You may have notice that the **dev** script uses **nodemon**, add it :
```bash
yarn add -D nodemon
```

> **nodemon** allows us to restart our server automaticaly on changes.

