# fh-config

## Description

A utility module to parse configurations files for FH node components.

## Usage

To use, simply require the module and initialise with the path of the configuration file:

```javascript
var config = require('fh-config');

var confFilePath = '/etc/feedhenry/fh-supercore/conf.json';
config.init(confFilePath, function(err, fhconfig){
  if(err){
    console.error('Failed to load configuration file');
  } else {
    console.log('Configuration file loaded');

    //fh-config is now initialised. All the other calls to require('fh-config') will return the same singleton instance.
  }
});
```

Once the config is initialsed, it can be easily retrieved:

```javascript
var config = require('fh-config');

var value = config.value('settings.name');
...
```

## Build

```javascript
grunt
```

## Template support

For docker based images we use environment variables as part of configuration.
fh-config supports placeholders in configuration that may be expanded by following handlers:

  - `env`: allows to reference to environment variable by value.

    	"test": "{{env.TEST_MISSING_ENV_VALUE}}"

  - `exec`: allows to execute command with separator (supports default value)

    	"test": "{{exec('dig mongodb', 'separator' , 'defaultValue')}}"

  - `fallback`: allows to set default value if value is missing

     	"test": "{{fallback(env.ENV_VALUE,'defaultValue')}}"

## APIs

### config.init(confSource, [validator], callback);

Initialise the config.

* confSource: the path to the conf file.
* validator: Optional function/keys array to validate the conf file. See config.validate.
* callback: Called when the conf file is loaded.

### config.validate(keysOrFunction);

Validate the configuration object. Throw errors if the validation is failed.

* keysOrFunction: You can either specify an array of keys or a function.

  * If it is an array of keys, the default implementation will check if all keys are presented in the configuration object, and if any key is missing, an error will be thrown. "." can be used to specify nested objects. The key can also be the name of a FH component. If this is the case, all the options required to use that component will be validated. Currently supported components including:
    * fhditch - require fhditch.host, fhditch.port, fhditch.protocol in the conf object
    * fhdfc - require fhdfc.dynofarm, fhdfc.username, fhdfc.\_password in the conf object
    * mongo - require mongo.enabled, mongo.name, mongo.host, mongo.port, mongo.auth.enanbled, mongo.auth.user, mongo.auth.pass in the conf object

  * If a function is passed, the function will be used for the validation and it will be passed the configuration object.

```javascript
var config = require('fh-config');

//require all the configurations for fhditch, and an object called 'settings' with the following keys: host, port, protocol
config.validate(['fhditch', 'settings.host', 'settings.port', 'settings.protocol']);

//Or you can define your own validation function:
config.validate(function(conf){
  assert.ok(null != conf.settings, 'Settings object is missing');
});
```

### config.value(key)

Get the conf value for the key.

* key: The path of the value. "." can be used to specify nested objects.

```javascript
var config = require('fh-config');

var settingsHost = config.value('settings.host');
```

#### config.int(key)

Similar to config.value(), but parse the value to int if possible.

* key: The path of the value. "." can be used to specify nested objects.

```javascript
var config = require('fh-config');
var settingsPort = config.int('settings.port');
```

### config.bool(key)

Similar to config.value(), but parse the value to boolean value if possible.

* key: The path of the value. "." can be used to specify nested objects.

```javascript
var config = require('fh-config');
var enabled = config.bool('settings.enabled');
```

### config.mongoConnectionString()

If 'mongo' configurations are specified in the conf object and is enabled, this will return the [standard MongoClient connection string](http://docs.mongodb.org/manual/reference/connection-string/).

The mongo configurations should be specified like this in the conf object:

```javascript
  {
    ...,
    mongo:{
      enabled: //required.true of false,
      name: //required. the default db name to connect,
      host: //required. the mongo db host. For replicaset, use ',' to separate them,
      port: //required. the mongdb port. If different ports are used for replicaset, use ',' to separate them,
      replicaset_name: //Optional. The name of the replicaset.
      auth:{
        enabled: //required. Of user auth is enabled,
        user: //required. the username,
        pass: //required. the password
      }
    },
    ...
  }
```

```javascript
var MongoClient = require('mongodb').MongoClient;
var fhconfig = require('fh-config');
MongoClient.connect(fhconfig.mongoConnectionString(), function(err,db){
  if(err){
    console.error(err);
  } else {
    console.log('mongo db connected');
  }
});
```

### config.mongooseConnectionString()

If 'mongo' configurations are specified in the conf object and is enabled, this will return the [Mongoose connection string](http://mongoosejs.com/docs/api.html#index_Mongoose-connect).

```javascript
var mongoose = require('mongoose');
var fhconfig = require('fh-config');
connection = mongoose.createConnection(fhconfig.mongooseConnectionString());
connection.once('open', function(){
  //connection established
});
```

### config.RELOAD_CONFIG_SIGNAL;
### config.reload(workers, callback);

Can reload the configuration if RELOAD_CONFIG_SIGNAL signal is received. Emit "reloaded" event.

* workers: An array of works if cluster mode is enabled.
* callback: Called when the conf file is updaed.

```javascript
//init config
....

var workers = [];
//enable reload
process.on(config.RELOAD_CONFIG_SIGNAL, function(){
  config.reload(workers, function(err){
    if(err){

    } else {

    }
  });
});
```

### config.getLogger()

If the conf object contains options for logger, this API can be used to retrieve the logger object. Currently it support winston and bunyan loggers.

The configuration object should contain a key called 'logger'.

* For winston logger, the 'logger' configuration should be like this:

    ```javascript
    'logger':{
      'type':'winston',
      'transports':[{
        type:'winston.transports.Console',
        level: 'silly'
      }, {
        type: 'winston.transports.File',
        filename:'/tmp/fh_config_test.log',
        level: 'debug'
      }]
    }
    ```
  To keep backward compatibility, if the 'logger' is an array, it will be treated as winston logger transports.

* For bunyan logger, the 'logger' configuration should be like this:

    ```javascript
      'logger':{
        'type':'bunyan',
        'name':'fhconfigtest',
        'streams':[{
          'type':'stream',
          'stream':'process.stdout',
          'src': true
        }]
      }
    ```

If no 'logger' configuration specified, a default logger will be provided using bunyan.


```javascript
var config = require('fh-config');
var logger = config.getLogger();
```

### config.setLogger(logger)

Mainly for tests, override the logger.

### config.setRawConfig(confObj);

Mainly used for tests. Set the raw conf json object. If this method is used, you do not need to call init method anymore.

* confObj: A JSON object.

```
var config = require('fh-config');
config.setRawConfig({
  'settings':{
    ...
  }
});
```

##New functionality for openshift 3 enterprise

### config.init(confSource, required, cb)

#### The existing init function is still used but has additional features based on the config.json file used.

* confSource : name of json config file to load
* required   : array of required config parameters in the form :-
  - [ 'parameter1',
  - 'parameter1.subA',
  - 'parameter1.subB' ]

OR (using a function called "configvalidation")

### config.init(confSource,somemodule.configvalidation, cb)  

A typical json config file could then be :-

```
{
  "parameter1" : {
    "subA" : "value",
    "subB" : "value"
}
```

To enable the new functionality (i.e assign environment variable values to the config parameter and do a dns lookup)
use the following convention (value embedded in "$()" to ensure its unique, where the regex pattern is /[\\sA-Za-z0-0-+\_.]\*/)

```
{
  "parameter1" : {
    "subA" : "$(ENV_VARIABLE_NAME)", // this will assign the env variable value ENV_VARIABLE_NAME to subA
    "subB" : "$(exec.dig mongodb-service +search +short)" // this will assign the dns lookup array list to subB
}
```
##License

fh-config is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/).
