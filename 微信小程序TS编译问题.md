## 问题描述
注意：我在新建项目的时候用的就是TS的模板，所以相关的依赖都是全的，如果你的不是，那这里并不能解决你的疑惑。

在微信开发者工具(**版本：1.0**)开发小程序的时候遇到修改了代码，但是视图没刷新。我的项目放在了git上，然后在不同的设备开发。在项目创建的设备上，编译是正常的，TS（typescript）能正常的编译成JS。但是我换了一台设备，clone项目，然后使用微信开发者工具的导入功能来导入工程。开发的时候就发现了一些诡异的问题，我修改了一些TS文件，但是视图并没有跟着改，我起初以为是缓存没刷新的问题，尝试过清编译缓存，清理所有缓存，再编译。但是还是没起作用。后来干脆重启微信开发者工具，但是也没有用。
在我快爆炸的前夕，看了一眼js文件，才发现是TS并没有编译成JS。

## 我尝试的解决方案
我也翻了很多帖子，说是要加一堆的配置文件。我挨个检查之后发现这些配置文件我都有，配置也正确（后面会附上这些配置文件及说明），一直走到最后一步，发现是自定义编译没有打开。TS（typescript）的是需要走自定义编译流程，才能编译成JS。具体的操作是：微信开发者工具右上角的“详情”->“本地设置”->“项目设置”->往下滑，直到"启用自定义处理命令"->打钩->并在编译/预览/上传前预处理，这个框填入npm run tsc。
然后清理编译缓存，再点一下编译。我这里出现了个报错，但是我忘记截下来了，大概的意思是我的某个node_modules没有安装，后来看了一下是忘记执行 npm i 了。

## 附件
我在新建项目的时候用的就是TS的模板，所以相关的依赖都是全的，如果你的不是，那这里并不能解决你的疑惑。

* package.json

```json
    {
  "name": "miniprogram-ts-quickstart",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "compile": "./node_modules/typescript/bin/tsc",
    "tsc": "node ./node_modules/typescript/lib/tsc.js"
  },
  "keywords": [],
  "author": "",
  "license": "",
  "dependencies": {},
  "devDependencies": {
    "miniprogram-api-typings": "^2.8.3-1",
    "typescript": "^3.3.3333"
  }
}

```

* tsconfig.json

```json
{
  "compilerOptions": {
    "strictNullChecks": true,
    "noImplicitAny": true,
    "module": "CommonJS",
    "target": "ES5",
    "allowJs": false,
    "experimentalDecorators": true,
    "noImplicitThis": true,
    "noImplicitReturns": true,
    "alwaysStrict": true,
    "inlineSourceMap": true,
    "inlineSources": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "strict": true,
    "removeComments": true,
    "pretty": true,
    "strictPropertyInitialization": true,
    "lib": ["es5"],
    "typeRoots": [
      "./typings"
    ]
  },
  "include": [
    "./**/*.ts"
  ],
  "exclude": [
    "node_modules"
  ]
}
```

* project.config.json(appid已经隐藏)
```json
    {
  "description": "项目配置文件",
  "packOptions": {
    "ignore": []
  },
  "miniprogramRoot": "miniprogram/",
  "compileType": "miniprogram",
  "libVersion": "2.8.2",
  "projectname": "wechat-hello",
  "scripts": {
    "beforeCompile": "npm run tsc",
    "beforePreview": "npm run tsc",
    "beforeUpload": "npm run tsc"
  },
  "setting": {
    "urlCheck": true,
    "es6": true,
    "enhance": false,
    "postcss": true,
    "preloadBackgroundData": false,
    "minified": true,
    "newFeature": false,
    "coverView": true,
    "nodeModules": false,
    "autoAudits": false,
    "showShadowRootInWxmlPanel": true,
    "scopeDataCheck": false,
    "uglifyFileName": false,
    "checkInvalidKey": true,
    "checkSiteMap": true,
    "uploadWithSourceMap": true,
    "compileHotReLoad": false,
    "useMultiFrameRuntime": true,
    "useApiHook": true,
    "useApiHostProcess": false,
    "babelSetting": {
      "ignore": [],
      "disablePlugins": [],
      "outputPath": ""
    },
    "enableEngineNative": false,
    "bundle": false,
    "useIsolateContext": true,
    "useCompilerModule": true,
    "userConfirmedUseCompilerModuleSwitch": false,
    "userConfirmedBundleSwitch": false,
    "packNpmManually": false,
    "packNpmRelationList": [],
    "minifyWXSS": true
  },
  "simulatorType": "wechat",
  "simulatorPluginLibVersion": {},
  "appid": "******",
  "condition": {}
}
```