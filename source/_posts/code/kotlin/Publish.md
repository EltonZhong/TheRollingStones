---
title: Publish kotlin-js package to npm.
cover: imgs/theworst.jpg
tags:
  - Code
categories:
  - code
desc: Publish a node.js package written in kotlin to npm registry.
date: 2018-12-18 22:37:45
mp3: http://link.hhtjim.com/163/26609785.mp3
---

# Publish
As we've talked [before](https://therollingstones.cn/2018/12/18/code/kotlin/KotlinMultiPlatformSetup),  before, we need to take more effort to publish our kotlin.js package.

> Kotlin community is not aware of that they should publish their .js package to npm registry. So It's terrible: when you want to publish your kotlin.js program to npm, you can't, because the packages you depend on is NOT in npm registry.

## What we should know before we publish it.

- Put your .js file into node_modules, and you can `require` it
- When a file is required, it will go to the paths specified in NODE_PATH and node_modules in its current dir, also fathers'. See [`https://www.kancloud.cn/thinkphp/nodejs-mini-book/43662`](https://www.kancloud.cn/thinkphp/nodejs-mini-book/43662) and view details.
- When you publish a module, you can not publish files in its node_modules
- You can not specify a single file dependency in package.json's dependencies, though you can add npm package dir in it.


## So here is the solution:
Given `node` dir is your root dir for your npm package.
First, generate all output files to node/kotlin-dependencies dir, and output your program js file to `node/kotlin/main/index.js`:
```groovy
fromPreset(presets.js, 'js', {
    configure(compilations.main) {
        tasks.getByName(compileKotlinTaskName).kotlinOptions {
            moduleKind = 'commonjs'
            outputFile = "node/kotlin/main/index.js"
        }
    }
}

task node(dependsOn: [jsJar, jsTestClasses]) {
    doLast {
        copy {
            def jsCompilations = kotlin.targets.js.compilations
            from jsCompilations.main.output
            from kotlin.sourceSets.jsMain.resources.srcDirs
            dump(kotlin.sourceSets.jsMain)
            dump(jsCompilations.main)
            jsCompilations.main.runtimeDependencyFiles.each {
                if (it.exists() && !it.isDirectory()) {
                    from zipTree(it.absolutePath).matching { include '*.js' }
                }
            }
            into "node/kotlin-dependencies"
        }
    }
}
```
Second,  configure your package.json:
```js
{
    "main": "kotlin/main/index.js",
    "scripts": {
      "test": "npm run setup && . ./.env && node kotlin/test/index.js",
      "preinstall": "rm -rf ./node_modules && mv -f ./kotlin-dependencies ./node_modules",
      "prepublish": "npm run setup"
      "setup": "*******You should write your own gradle node task here, a example is: cd .. && gradle node && cd node",
   },
   "dependencies": {
     ...
   }
 }
```
- prepublish: the hook before publish. Here we should publish our all kotlin-dependencies and kotlin/main/index.js to npm.
- preinstall: the hook for the package before being installed. So when a user installs your package, he will move all kotlin-dependencies to node_modules, and then install your dependencies written in your package.json.

A better way is to write another .js file to expose your classes which you want to expose and write .d.ts in it. While above is enough for basic use.

Then, you can publish your kotlin.js package.