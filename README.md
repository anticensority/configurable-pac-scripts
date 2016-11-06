# Configurable PAC Scripts (Draft)

## Why

We want PAC scripts to be configurable, e.g. force it to use HTTPS proxies only, force it to use custom proxies,
exclude some sites from proxying or add some, etc.

## Design Principles

### Configurable PAC Scripts should be Interchangeable with Simple PAC Scripts (In Most Cases)

1. Configurable PAC scripts should be backward compatible with software that expects
   plain old PAC scripts. It widens an auditory of the PAC script.
2. Non-configured PAC script MAY still be a valid PAC script even if configs are not touched  
   (so default config values defined inside PAC script are used).
3. PAC script MAY require additional configs to be defined before it may be used. This may be used for legal protection (e.g. user provides his proxy).
   There must be a way to explicitly tell client what is required. In this only case configurable PAC scripts are not backward compatible with old software.
4. PAC configs are part of the PAC script itself, not part of a separate file. But config may contain urls to external resources.
5. Retrieving and changing configs must be easy, as easy as search/replace and parsing JSON, there should
   be no need for lexical parsing of ecmascript. `_start` and `_end` properties are used to mark configs object for easier search/replace.

### Clients should Use Schema Validation as Protection from Malformed Configs

0. Client may use PAC scripts from third parties which may be malicious. Risk of damage should be minimised by data validation.
1. Client may use schemas to validate configs supplied by user or server, this should contribute to client stability against errors.

### Configs should be Versioned

1. You can't know what configs you will need, it varies with use cases and time.
   Clients should be able to reject configs they can't use based on version (not only schema).
   Version also may be used for data migration of configs.
   Migrations bloat the client codebase, so minimum number of migration instructions should be kept on the client,
   migration scripts may be lazy loaded on each update (if allowed).

### Configs should be Modular

7. Separation into components/modules/plugins contributes to:
  * problem decomposition, separation of concerns
  * one standard/code may be used for different use cases (but we have only one -- anticensorship)

### Configs are Dynamic (Not Sure)

8. Don't use classes (e.g. Java) to describe configs object, use hash/dictionary instead with a schema validator.   
   `CONFIGS` are used inside PAC script too, and there I would like to use `CONFIGS.pluginNamedFoo.propBar`, `getConfigsFor('pluginNamedFoo').propBar` seems inconvenient.

## Schemas and Examples

  [JSON Schema standard](http://json-schema.org)
| [Understanding JSON Schema](https://spacetelescope.github.io/understanding-json-schema/)
| [ajv](https://github.com/epoberezkin/ajv)

```js
'use strict';

const configsSchema = {

  title: "PAC Script Configs",

  type: "object",
  properties: {

    _start: {
      constant: "CONFIGS_START",
    },
    _end: {
      constant: "CONFIGS_END",
    }

  },
  required: ["_start", "_end"],
  additionalProperties: true
};

const pluginsSchemas = {};

pluginsSchemas.plugins = {

  title: "PAC Script Plugins",

  definitions: {
    pluginDescription: {
      type: "object",
      properties: {

        version:   { type: "string" },
        schemaUrl: { type: "string", format: "uri" }

      },
      required: ["version"],
      additionalProperties: false
    }
  },

  type: "object",
  properties: {

    plugins: {
      title: "Plugin for supporting other plugins",
      type: "object",
      properties: {

        plugins: {
          $ref: "#/definitions/pluginDescription"
        }

      },
      required: ["plugins"],
      additionalProperties: {
        $ref: "#/definitions/pluginDescription"
      }
    }

  },
  required: ["plugins"],
  additionalProperties: true

};

const hostnameRE   = '([a-z0-9-]+[.])*[a-z0-9-]+'; // e.g.: "local-foobar-host"
const ipv4RE       = '[0-9]{1,3}(.[0-9]{1,3}){3}';
const ipv6RE       = '([0-9A-Fa-f]{0,4}:){2,7}[0-9A-Fa-f]{0,4}';

const portRE       = ':[0-9]{1,5}';
const ipv6portedRE = `\\[${ipv6RE}\\]${portRE}`;

// Without port:
const ipPattern   = `^(${ipv4RE}|${ipv6RE})$`;
const nodePattern = `^(${hostnameRE}|${ipv4RE}|${ipv6RE})$`;
// MUST have port:
const hostPattern = `^((${hostnameRE}|${ipv4RE})${portRE}|${ipv6portedRE})$`;

pluginsSchemas.proxies = {

  title: "PAC Script Proxies",

  type: "object",
  properties: {

    proxies: {
      title: "Plugin for configuring proxies",
      type: "object",
      properties: {

        exceptions: {
          type: "object",
          properties: {

            ifEnabled: { type: "boolean" },
            ifHostProxied: {
              patternProperties: {

                [nodePattern]: { type: "boolean" }

              },
              additionalProperties: false
            }

          },
          required: ["ifHostProxied"],
          additionalProperties: false
        },
        typeToProxies: {
          patternProperties: {

            "^(HTTPS|PROXY)$": {
              type: "array",
              items: {
                type: "string",
                pattern: hostPattern
              }
            }

          },
          additionalProperties: false
        },
        ifHttpsProxyOnly: { type: "boolean" },
        ifHttpsUrlsOnly: { type: "boolean" },
        customProxyString: {
          anyOf: [{
            type: "string"
          }, {
            constant: false
          }]
        }
      },
      required: ["typeToProxies", "exceptions"],
      additionalProperties: false
    }

  },
  required: ["proxies"],
  additionalProperties: true

};

pluginsSchemas.anticensorship = {

  title: "PAC Script for Anticensorship",

  type: "object",
  properties: {

    anticensorship: {
      type: "object",
      properties: {

        ifUncensorByIp: {
          type: "boolean"
        },
        ifUncensorByHost: {
          type: "boolean"
        },
        ipToProxy: {
          patternProperties: {

            [ipPattern]: { type: "string", pattern: hostPattern }

          },
          additionalProperties: false
        }

      },
      required: ["ipToProxy"],
      additionalProperties: false
    }

  },
  required: ["anticensorship"],
  additionalProperties: true

};

// CONFIGS is extracted from PAC script as JSON

const CONFIGS = {"_start":"CONFIGS_START",

  "plugins": {
    "plugins": { "version": "0.0.0.15", "schemaUrl": "https://satan.hell/anticensor.json" },
    "proxies":  { "version": "0.0.0.15" },
    "anticensorship": { "version": "0.0.0.15" }
  },

  "proxies": {
    "exceptions": {
      "ifEnabled": true,
      "ifHostProxied": {
        "youtube.com": false,
        "archive.org": true,
        "bitcoin.org": true
      }
    },
    "typeToProxies": {
      "HTTPS": ["proxy.antizapret.prostovpn.org:3143", "gw2.anticenz.org:443"],
      "PROXY": ["proxy.antizapret.prostovpn.org:3128", "gw2.anticenz.org:8080"]
    },

    "ifHttpsProxyOnly":  false,
    "ifHttpsUrlsOnly":   false,
    "customProxyString": false,
  },

  "anticensorship": {
    "ifUncensorByIp":   true,
    "ifUncensorByHost": true,
    "ipToProxy": {
      "12.33.44.55": "satan.hell:666",
      "2001:0db8:0000:0042:0000:8a2e:0370:7334": "satan.hell:333"
    }
  },

  "_end":"CONFIGS_END"};

var Ajv = require('ajv');
var ajv = Ajv({allErrors: true, v5: true});

if (!ajv.validate(configsSchema, CONFIGS)) {
  console.log(ajv.errors);
  process.exit();
}
console.log('Root is valid.');

for( const schemaName of Object.keys(pluginsSchemas) ) {

  const ifValid = ajv.validate(pluginsSchemas[schemaName], CONFIGS);
  if (!ifValid) {
    console.log(ajv.errors);
  } else
  {
    console.log(schemaName + ' is valid.');
  }

}
```

## Possible Implementation (Not Finished)

```js
// Chrome Extension (Client)

class PacConfigPlugin {

  Fields:

    name
    version
    scheme

};

class PacConfigs {

  Fields:
  
    custom: {...}
    set defauld(newDefauld) {
      this.assertSchemes( this._merge( newDefauld, this.custom ) );
      return newDefauld;
    }
    get defauld(value) {
      return value;
    }

    _schemas: {
      root: rootSchema, // { plugins: ...schema... }
        ???
        plugins: {
          ...
          common: ...PacConfigPlugin...
          anticensorship: ...PacConfigPlugin...
          ...
        }
    }

  Methods:
  
    constructor(defauldConfigs, ...plugins)
      CALLS:
        plugins.forEach( (plugin) => this.usePlugin(plugin) );
        this.configs.defauld = defauldConfigs

    usePlugin(plugin)
      CHANGES: _schemas

    _merge(target, source)
    
    getCustomObject(pathStr)
      more convenient for creating custom props than basic set
      DOES:
        returns modifiable prop of custom without any merging
        returns only values with prototype Object
        if prop is not defined:
          1. checks that defauld has Object on the same path
          2. creates {} on custom and returns it.
    get(pathStr, ifStrict = true)
      DOES: applies custom to defauld, gets prop __strictly__
      RETURNS: merged __copy__
    set(pathStr)
      CALLS:
        sets prop of custom configs
        this.assertScheme()

    assertScheme(configs)
      CALLS:
        configs ? check(configs) : check(this.get())
};

```
