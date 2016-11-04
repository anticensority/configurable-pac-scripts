# Configurable PAC Scripts

## Why

We want PAC scripts to be configurable, e.g. force it to use HTTPS proxies only, force it to use custom proxies,
exclude some sites from proxying or add some, etc.

## Design Principles

### Configurable PAC Scripts should be Interchangeable with Simple PAC Scripts (In Most Cases)

0. Configurable PAC scripts should be backward compatible with software that expects
   palin old PAC scripts. It widens an audiotry of the PAC script.
1. Non-configured PAC script MAY still be a valid PAC script even if configs are not touched  
   (so default config values defined inside PAC script are used).
2. PAC script MAY require additional configs to be defined before it may be used. This may be used for legal protection (e.g. user provides his proxy).
   There must be a way to explicitly tell client what is required. In this only case configurable PAC scripts are not backward compatible with old software.
3. PAC configs are part of the PAC script itself, not part of a separate file. But config may contain urls to external resources.
4. Retrieving and changing configs must be easy, as easy as search/replace and parsing JSON, there should
   be no need for lexical parsing of ecmascript.

### Clients should Use Schema Validation as Protection from Malformed Configs

0. Client may use PAC scripts from third parties which may be malicious. Risk of damage should be minimised by data validation.
1. Cleint may use schemas to validate configs supplied by user or server, this should contribute to client stability against errors.

### Configs should be Versioned

1. You can't know what configs you will need, it varies with use cases and time.
   Clients should be able to reject configs they can't use based on version (not only schema).
   Version also may be used for data migration of configs.
   Migrations bloat the client codebase, so minimum number of migration instructions should be kept on the cleint,
   migration scripts may be lazy loaded on each update (if allowed).

### Configs should be Modular

7. Separation into components/modules/plugins contributes to:
  * problem decomposition, separation of concerns
  * one standard/code may be used for different use cases (but we have only one -- anticensorship)

### Configs are Dynamic

8. Don't use classes (e.g. Java) to describe configs object, use hash/dictionary instead with a schema validator.
   I would like to use CONFIGS.pluginNamedFoo.propBar, CONFIGS.plugins[2] is not usable at all.

## Example

```js
// File: proxy-0.0.0.15.pac
var CONFIGS = {"_start":"CONFIGS_START",

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
    "ipToProxy": __IP2PROXY__
  },

  "_end":"CONFIGS_END"};

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

## JSON Schemas of Configs

### Common Code
```js
const hostPattern = '^([a-z-]+[.])+[a-z-]+(:[0-9]{1,65535})?$';
```

### Plugin for Supporting Plugins (Root)

Plugin support is itself implemented via plugin.

```js
{
  title: "PAC Script Configs",

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

    _start: {
      constant: "CONFIGS_START",
    },
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
    },
    _end: {
      constant: "CONFIGS_END",
    }

  },
  required: ["_start", "_end", "plugins"],
  additionalProperties: true
});
```

### Plugin for Configuring Proxies

```js
{
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

                "^([a-z-]+[.])+[a-z-]+(:[0-9]{1,65535})?$": { type: "boolean" }

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
  required: ["proxies"]
});
```

### Plugin for Configuring Anticensorship Behavior

```js
{
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

            "^regex_for_ipv4_ipv6$": { type: "string", pattern: hostPattern }

          },
          additionalProperties: false
        }

      },
      required: ["ipToProxy"],
      additionalProperties: false
    }

  },
  required: ["anticensorship"],
  additionalProperties: false
}
```
