# Configurable PAC Scripts (Deprecated)

This document is deprecated, we don't maintain it any more.

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
   be no need for lexical parsing of ecmascript.  
   For easier search/replace we will use some marks for start and end of the object description.  
   Object properteis can't be used as marks because JS doesn't guarantee strict order of object properties, so we will use comments:
   
   ```js
var CONFIGS = /*CONFIGS_START*/{
... json description ...
}/*CONFIGS_END*/;
   ```
   Some JS environments don't preserve comments (gscript, e.g.) -- take care.

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
  * separate versioning of components may be used to organise migration code in a modular way
  * one standard/code may be used for different use cases (but we have only one -- anticensorship)
  * different groups of developers may evolve their own component (and version it separately)

### Configs are Dynamic (Not Sure)

8. Don't use classes (e.g. Java) to describe configs object, use hash/dictionary instead with a schema validator.   
   `CONFIGS` are used inside PAC script too, and there I would like to use `CONFIGS.pluginNamedFoo.propBar`, `getConfigsFor('pluginNamedFoo').propBar` seems inconvenient.

## Schemas and Examples

  [JSON Schema standard](http://json-schema.org)
| [Understanding JSON Schema](https://spacetelescope.github.io/understanding-json-schema/)
| [ajv](https://github.com/epoberezkin/ajv)

### Terminology

* _hostname_ -- `/([a-z0-9-]+[.])*[a-z0-9-]+/` (no protocol or port).
* _node_ -- an IPv4, IPv6 or a network hostname (defined in `getaddrinfo(3)`) (no protocol or port).
* _host_ -- _node_ with a mandatory port (no protocol).

Definitions of _host_ and _hostname_ are looked up form the properties of `<a>` DOM.  
IDN must be punycoded.

### Code

```js
'use strict';

// ##### CONFIGS #####
// CONFIGS are extracted from PAC script as JSON
// Inside PAC it looks like:

const CONFIGS = /*CONFIGS_START*/{

  "proxies": {
    "version": 0.1,
    "exceptions": {
      "ifByHostname": true,
      "ifHostnameProxied": {
        "youtube.com": false,
        "archive.org": true,
        "bitcoin.org": true
      },
      "ifByIp": true,
      "ifIpProxied": {
        "22.33.44.55": false
      }
    },
    "typeToProxyHosts": {
      "HTTPS": ["proxy.antizapret.prostovpn.org:3143", "gw2.anticenz.org:443"],
      "PROXY": ["proxy.antizapret.prostovpn.org:3128", "gw2.anticenz.org:8080"]
    },
    "ipToProxyNode": {
      "12.33.44.55": "satan.hell",
      "2001:0db8:0000:0042:0000:8a2e:0370:7334": "satan.hell"
    },

    "ifHttpsProxyOnly":  false,
    "ifHttpsUrlsOnly":   false,
    "customProxyString": false,
  },

  "anticensorship": {
    "version": 0.1,
    "ifUncensorByIp":   true,
    "ifUncensorByHost": true
  }

}/*CONFIGS_END*/;

// ##### SCHEMAS #####

const configsRootSchema = {

  title: "PAC Script Configs",

  type: "object",
  additionalProperties: {
    type: "object",
    properties: {

      version: { type: "number", multipleOf: 0.01 }

    },
    required: ["version"]
  }

};

const pluginsSchemas = {};

const hostnameRE   = '([a-z0-9-]+[.])*[a-z0-9-]+'; // e.g.: "local-foobar-host"
const ipv4RE       = '[0-9]{1,3}(.[0-9]{1,3}){3}';
const ipv6RE       = '([0-9A-Fa-f]{0,4}:){2,7}[0-9A-Fa-f]{0,4}';

const portRE       = ':[0-9]{1,5}';
const ipv6portedRE = `\\[${ipv6RE}\\]${portRE}`;

// Without port:
const hostnamePattern = `^${hostnameRE}$`;
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

        version: { constant: 0.01 },
        exceptions: {
          type: "object",
          properties: {

            ifByHostname: { type: "boolean" },
            ifHostnameProxied: {
              patternProperties: {

                [hostnamePattern]: { type: "boolean" }

              },
              additionalProperties: false
            },
            ifByIp: { type: "boolean" },
            ifIpProxied: {
              patternProperties: {

                [ipPattern]: { type: "boolean" }

              },
              additionalProperties: false
            }

          },
          required: ["ifHostnameProxied"],
          additionalProperties: false
        },
        typeToProxyHosts: {
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
        ipToProxyNode: {
          patternProperties: {

            [ipPattern]: { type: "string", pattern: nodePattern }

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
      required: ["version", "exceptions", "typeToProxyHosts", "ipToProxy"],
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

        version: { constant: 0.01 },
        ifUncensorByIp: {
          type: "boolean"
        },
        ifUncensorByHost: {
          type: "boolean"
        },

      },
      required: ["version", "ifUncensorByIp", "ifUncensorByHost"],
      additionalProperties: false
    }

  },
  required: ["anticensorship"],
  additionalProperties: true

};

// ##### TESTS OF SCHEMAS #####

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

// ##### CLIENT IMPLEMENTATION #####

/*

Client is a Chrome Extension.

Configs inside a PAC script may be updated.
For updates incompatible with a client it's better to change PAC script url
and use new url in the new version of a client.

User may tweak configs and his changes should be preserved on updates.
There are two ways of applying user configs:

1. Merge user configs with updatable defaults.
2. Use user configs and ignore changes in defaults (frozen configs).

If user deletes element from an array or a property in configs then
this kind of change would be difficult to keep and merge with new defaults
in terms of code to be written and maintained.

TODO: think how to do it the most reasonable way.

*/

class PacConfigsPlugin {

  constructor(name, version, schema) {

    this.name = name;
    this.version = version;
    this.schema = schema

  }

};

class PacConfigs {

  constructor(defauld, ...plugins) {

    this._rootSchema = configsRootSchema;
    
    this._plugins = new Map();
    plugins.forEach( (plugin) => this._plugins.set(plugin.name, plugin) );
    
    this.defauld  = defauld;
    this.custom   = {};
    this.ifDirty = true;
    assertSchemas( this.defauld );

  }

  assertSchemas(configs = this.getMerged()) {

    assert( ajv.validate(this._rootSchema, configs) );

    for( const plugin of this._plugins.values() ) {
      assert( ajv.validate(plugin.schema, configs) );
    }

    this.ifDirty = false;

  }  
  
  /* 
   * `path` example: 'proxies.exceptions.ifHostnameProxied'
   * If 'exceptions' doesn't exist, it is initialized with `{}`.
   * If 'ifHostnameProxied' doesn't exist, it is not created.
   * Reference to property of custom is returned.
  **/
  getCustomByRef(path, ifPathMustExist = false) {

    path = path.split('.');
    let custom = this.custom;
    const checkIfPropExists = (obj, prop) => {

      const ifOwn = obj.hasOwnProperty(prop);
      if ( !ifOwn && ifPathMustExist ) {
        throw new Error('Can\'t get prop in custom configs: ' + prop + ' in ' + _path);
      }
      return ifOwn;

    };
    let prop = path.shift();
    while( path.length > 1 ) {
      if ( !checkIfPropExists(custom, prop) ) {
        custom[ prop ] = {}; // Not for leaves.
      }
      custom = custom[ prop ];
      prop = path.shift();
    }

    checkIfPropExists(custom, prop);

    this.ifDirty = true;
    return custom[ prop ];

  },

  setCustom(path, value) {

    path = path.split('.');
    const prop = path.pop();
    let custom = this.custom;
    if ( path.length > 0 ) {
      custom = this.getCustomByRef( path.join('.') );
    }
    custom[ prop ] = value;
    this.ifDirty = true;

  },

  _deepMerge(target, source, ifTargetNotOwn, ifSourceNotOwn) {

    const clone = (value) => JSON.parse( JSON.stringify( { foo: value } ) ).foo; // Obj MUSTN'T have Date property.

    if ( ifTargetNotOwn && ifSourceNotOwn ) {
      throw new Error('At least one value must be flagged as own property.');
    }
    if ( ifTargetNotOwn ) {
      return clone(source);
    }
    if ( ifSourceNotOwn ) {
      return clone(target);
    }
    // Types must match.
    if ( !(target && source ? target.constructor !== source.constructor : typeof(target) !== typeof(source) ) ) {
      throw new Error(
        'You can\'t change type of default configs: default is ' + target + ', custom is ' + source
      );
    }
    const ifTargetPlain = !(tvalue && tvalue.constructor === Object);
    const ifSourcePlain = !(svalue && svalue.constructor === Object);
    const ifBothPlain = ifTargetPlain && ifSourcePlain;
    if ( ifBothPlain ) {
      return clone(source);
    }
    // Both objects.
    const merged = {};
    // Get all props of both.
    const props = new Set(Object.keys(target));
    Object.keys(source).forEach( (p) => props.add(p) );

    for( const prop of props ) {
      merged[ prop ] = this._deepMerge(target[ prop ], source[ prop ], target.hasOwnProperty(prop), source.hasOwnProperty(prop) );
    }
    return merged;
  }

  getMerged(path) {

    let defauld = this.default;
    let custom  = this.custom;
    path = path.split('.');

    const ifHasNoSuchProperty = {
      defauld: false,
      custom: false
    };
    while( path.length ) {
      let prop  = path.shift();
      if (!defauld && !custom) {
        throw new Error('Can\'t get ' + prop + ' of undefined.');
      }
      ifHasNoSuchProperty.defauld = !( defauld && defauld.hasOwnProperty(prop) );
      ifHasNoSuchProperty.custom  = !( custom  && custom.hasOwnProperty(prop) );
      defauld = defauld && defauld[ prop ];
      custom  = custom  && custom[ prop ];
    }
    return this._deepMerge(defauld, custom, ifHasNoSuchProperty.defauld, ifHasNoSuchProperty.custom );

  }

};
```
