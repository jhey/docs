# Using AssemblyScript

CasperLabs maintains the package [@casperlabs/contract](https://www.npmjs.com/package/@casperlabs/contract) to allow a dapp developer to create smart contracts using [AssemblyScript](https://www.npmjs.com/package/assemblyscript). The package source is hosted in the [main CasperLabs repository](https://github.com/CasperLabs/CasperLabs/tree/dev/execution-engine/contract-as).

## Installation
For each smart contract you create, make a project directory and initialize it.

```sh
mkdir project
cd project
npm init
```

npm init will prompt you for various details about your project;
answer as you see fit but you may safely default everything except `name` which should follow the convention of
`your-contract-name`.

Then install assembly script and this package in the project directory.

```sh
npm install --save-dev assemblyscript@0.9.1
npm install --save @casperlabs/contract
```

## Usage
Add script entries for assembly script to your project's `package.json`; note that your contract name is used
for the name of the wasm file.

```json
{
  "name": "your-contract-name",
  ...
  "scripts": {
    "asbuild:optimized": "asc assembly/index.ts -b dist/your-contract-name.wasm --validate --optimize --use abort=",
    "asbuild": "npm run asbuild:optimized",
    ...
  },
  ...
}
```

In your project root, create an `index.js` file with the following contents:

```js
const fs = require("fs");

const compiled = new WebAssembly.Module(fs.readFileSync(__dirname + "/dist/your-contract-name.wasm"));

const imports = {
    env: {
        abort(_msg, _file, line, column) {
            console.error("abort called at index.ts:" + line + ":" + column);
        }
    }
};

Object.defineProperty(module, "exports", {
    get: () => new WebAssembly.Instance(compiled, imports).exports
});
```

Create an `assembly/tsconfig.json` file in the following way:
```json
{
  "extends": "../node_modules/assemblyscript/std/assembly.json",
  "include": [
    "./**/*.ts"
  ]
}
```

### Sample smart contract
Create a `assembly/index.ts` file. This is where the code for your contract will go.

You can use the following sample snippet which demonstrates a very simple smart contract that immediately returns an error, which will write a message to a block if executed on the CasperLabs platform.

```typescript
//@ts-nocheck
import {Error, ErrorCode} from "@casperlabs/contract/error";

// simplest possible feedback loop
export function call(): void {
    Error.fromErrorCode(ErrorCode.None).revert(); // ErrorCode: 1
}
```
If you prefer a more complicated first contract, you can look at example contracts on the [CasperLabs](https://github.com/CasperLabs/CasperLabs/tree/master/execution-engine/contracts-as/examples) github repository for inspiration.

### Compile to wasm
To compile your contract to wasm, use npm to run the asbuild script from your project root.
```
npm run asbuild
```
If the build is successful, you should see a `dist` folder in your root folder and in it
should be `your-contract-name.wasm`