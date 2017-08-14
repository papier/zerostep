# zerostep
ZeroStep is a small library to organize, wire and manage the
lifecycle of modules.

## Features
- Initialize and destroy modules
- Import and export services from modules
- Check imports/exports at registration time
- Take care of process signals for SIGTERM, SIGINT and SIGUSR2 if whished
- Plays nice with nodemon
- Zero dependencies
- Promise based
- Gracefully handle error conditions in module initialization/destruction

## Examples

### Shortcuts
    'use strict'
    
    const ZeroStep = require('../index')
    
    const zs = new ZeroStep()
    
    zs
      .register((() => {
        let secret = 'S3C43T'
        let obj = {
          destroy: () => console.log(`Destroying obj with secret:= ${secret}`),
        }
    
        return {
          name: 'Secret and initValue usage in destroy',
          init: () => obj,
          destroy: (ctx, obj) => obj.destroy(),
        }
      })())
      .register({
        name: 'Module which was added by chaining register calls!',
        init: () => console.log('Hello from chained module!'),
      })
    
    
    zs.init().then(() => zs.destroy())
    
Will print:

    ZeroStep:- Initializing module Secret and initValue usage in destroy() -> []
    ZeroStep:- Initializing module Module which was added by chaining register calls!() -> []
    Hello from chained module!
    ZeroStep:- Destroying module Module which was added by chaining register calls!
    ZeroStep:- Destroying module Secret and initValue usage in destroy
    Destroying obj with secret:= S3C43T
    ZeroStep:- Destroyed all modules

### Hello, world
    const ZeroStep = require('zerostep')

    const zs = new ZeroStep()

    zs.register({
      name: 'hello-world',
      init: () => console.log('hello, world'),
      destroy: () => console.log('goodbye, world')
    })

    zs.init().then(() => zs.destroy())

Will print:

    ZeroStep:- Initializing module hello-world() -> []
    hello, world
    ZeroStep:- Destroying module hello-world
    goodbye, world

## Two modules

    const ZeroStep = require('zerostep')

    const zs = new ZeroStep()

    zs.register({
      name: 'one',
      export: 'symbolFromOne',
      init: () => {
        console.log('hello from 1')
        return 'Message from one'
      },
      destroy: () => console.log('goodbye from 1')
    })

    zs.register({
      name: 'two',
      imports: ['symbolFromOne'],
      init: (ctx) => {
        console.log('hello from 2')
        console.log(`got ${ctx.symbolFromOne}`)
      },
      destroy: () => console.log('goodbye from 2')
    })

    zs.init().then(() => zs.destroy())

Will print:

    ZeroStep:- Initializing module one() -> [symbolFromOne]
    hello from 1
    ZeroStep:- Initializing module two(symbolFromOne) -> []
    hello from 2
    got Message from one
    ZeroStep:- Destroying module two
    goodbye from 2
    ZeroStep:- Destroying module one
    goodbye from 1

*Note* Module two gets initialized after module one but destroyed before module one.
This is almost always what you want!

## Documentation

### ZeroStep constructor
Create a new instance.

### ZeroStep.prototype.register(module) -> ZeroStep(this)
- module must have a *name* attribute and the methods *init* and *destroy*
  - *name*: string with the name of the module
  - *init*: is called to initialize the module with its context object and must return a non null/undefined value if optional *export* attribute is set
  - *destroy*: is called to destroy the module with its context object as first and the return value of its init method as second argument
- module has optional attributes
  - *export*: string which names the value returned by init
  - *imports*: an array of string names which named exports of other modules should be importet.
    _IMPORTANT: every name must have been registered in an *export* attribute before_

*Note* The context object for init/destroy is created once for the init method and provided to the destroy method
*Note* You can get a handle to destroy an object by just returning it from init - it will be provided to destroy w/o a need to export it

### ZeroStep.prototype.init() -> Promise
Initialize all registered modules in the order of their registration.

See ZeroStep.prototype.initAsApplicationCore() if you want ZeroStep to take care of SIGINT, SIGTERM and SIGUSR2.

### ZeroStep.protoype.destroy() -> Promise
Destroy all registered modules in the opposite order in which they where registered.

### ZeroStep.protoype.initAsApplicationCore() -> Promise
Registers a handler for disconnect, uncaughtException, unhandledRejection, error, SIGINT, SIGTERM and SIGUSR2 which will call ZeroStep.prototype.destroy() and calls ZeroStep.prototype.init().
