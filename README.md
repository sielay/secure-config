# [**secure-config**](https://github.com/tsmx/secure-config)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![npm (scoped)](https://img.shields.io/npm/v/@tsmx/secure-config)
![node-current (scoped)](https://img.shields.io/node/v/@tsmx/secure-config)
[![Build Status](https://travis-ci.com/tsmx/secure-config.svg?branch=master)](https://travis-ci.org/tsmx/secure-config)
[![Coverage Status](https://coveralls.io/repos/github/tsmx/secure-config/badge.svg?branch=master)](https://coveralls.io/github/tsmx/secure-config?branch=master)

Handling multi-environment configurations with encrypted secrets.

Benefits:
- No need to "hide" your configuration files from code repos etc.
- The only thing to be kept secret is one key per environment.
- No need to use 3rd party secret stores like GCP KMS, Vault or something
- Pure NodeJS solution, no dependencies
- Uses standard environment variable technique to inject the only secret you need

The cipher used is AES-256-CBC.

## Usage

1. Encrypt your secret configuration values, e.g. by using [secure-config-tool](https://www.npmjs.com/package/@tsmx/secure-config-tool). For more details please see [generating encrypted values](#generating-encrypted-entries).
    ```bash
    [tsmx@localhost ]$ secure-config-tool create --secret MySecretDbUser
    ENCRYPTED|50ceed2f97223100fbdf842ecbd4541f|df9ed9002bfc956eb14b1d2f8d960a11
    [tsmx@localhost ]$ secure-config-tool create --secret MySecretDbPass
    ENCRYPTED|8fbf6ded36bcb15bd4734b3dc78f2890|7463b2ea8ed2c8d71272ac2e41761a35
    ```

2. Copy & Paste the encrypted values to your JSON configuration file
    ```json
    {
        "database": {
            "host": "127.0.0.1",
            "user": "ENCRYPTED|50ceed2f97223100fbdf842ecbd4541f|df9ed9002bfc956eb14b1d2f8d960a11",
            "pass": "ENCRYPTED|8fbf6ded36bcb15bd4734b3dc78f2890|7463b2ea8ed2c8d71272ac2e41761a35"
        }
    }
    ```

3. Use your configuration in the code
    ```js
    const conf = require('@tsmx/secure-config');

    function MyFunc() {
        let dbHost = conf.database.host; // = '127.0.0.1'
        let dbUser = conf.database.user; // = 'MySecretDbUser'
        let dbPass = conf.database.pass; // = 'MySecretDbPass'
        //...
    }
    ```
## Injecting the decryption key

The key for decrypting the encrypted values is derived from an environment variable named `CONFIG_ENCRYPTION_KEY`. You can set this variable 
whatever way is most suitable, e.g.
- set/export in the command line or in your bash pofile
  ```
  export CONFIG_ENCRYPTION_KEY=0123456789qwertzuiopasdfghjklyxc
  ```
- using an env block in your VS-Code launch configuration
  ```json
  ...
  "env": {
      "CONFIG_ENCRYPTION_KEY": "0123456789qwertzuiopasdfghjklyxc"
  },
  ...
  ```
- using an env block in your deployment descriptor, e.g. app.yaml for Google App Engine
  ```yaml
  env_variables:
    CONFIG_ENCRYPTION_KEY: "0123456789qwertzuiopasdfghjklyxc"
  ```
- for testing with [Jest](https://jestjs.io/) I recommend to create a test key and set it globally for all tests in the `jest.config.js`, e.g.
  ```javascript
  process.env['CONFIG_ENCRYPTION_KEY'] = '0123456789qwertzuiopasdfghjklyxc';

  module.exports = {
      testEnvironment: 'node'
  };
  ```
- etc.

The key length must be 32 bytes! The value set in `CONFIG_ENCRYPTION_KEY` has to be:
- a string of 32 characters length, or
- a hexadecimal value of 64 characters length (= 32 bytes)

Otherwise an error will be thrown.

Examples of valid key strings:
- 32 byte string: `MySecretConfigurationKey-123$%&/`
- 32 byte hex value: `9af7d400be4705147dc724db25bfd2513aa11d6013d7bf7bdb2bfe050593bd0f`

Different keys for each configuration environment are strongly recommended.

## Generating encrypted entries

### Option 1: secure-config-tool

For better convenience I provided a very basic [secure-config-tool](https://www.npmjs.com/package/@tsmx/secure-config-tool) to easily generate the encrypted entries.

### Option 2: NodeJS crypto functions 

You can simply use `crypto` functions from NodeJS with the following snippet to create the encrypted entries:

```js
const crypto = require('crypto');
const algorithm = 'aes-256-cbc';

function encrypt(value) {
    let iv = crypto.randomBytes(16);
    let key = Buffer.from('YOUR_KEY_HERE');
    let cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(value);
    encrypted = Buffer.concat([encrypted, cipher.final()]);
    return 'ENCRYPTED|' + iv.toString('hex') + '|' + encrypted.toString('hex');
}
```

### Remarks

The generated encrypted entry must always have the form: `ENCRYPTED | IV | DATA`. 

Part | Description
-----|------------
`ENCRYPTED` | The prefix `ENCRYPTED` used to identify configuration values that must be decrypted.
`IV` | The ciphers initialization vector (IV) that was used for encryption. Hexadecimal value.
`DATA` | The AES-256-CBC encrypted value. Hexadecimal value.

## Configuration file name and directory convention

You can have multiple configuration files for different environments or stages. They are distinguished by the environment variable `NODE_ENV`. The basic configuration file name is `config.json` if this variable is not present. If it is present, a configuration file with the name `config-[NODE_ENV].json`
is used. An exception will be thrown if no configuration file is found.

All configuration files must be located in a `conf/` directory of the current running app, meaning a direct subdirectory of the current working directory (`CWD/conf/`).  

### Example structure

- Development stage
  - `NODE_ENV`: not set
  - Configuration file: `conf/config.json`
- Prodcution stage
  - `NODE_ENV`: `production`
  - Configuration file: `conf/config-production.json`
- Test stage, e.g. for Jest
  - `NODE_ENV`: `test`
  - Configuration file: `conf/config-test.json`

```
path-to-your-app/
├── conf/
│   ├── config.json
│   ├── config-production.json
│   └── config-test.json
├── app.js
└── package.json
```

## Test

```
npm install
npm test
```