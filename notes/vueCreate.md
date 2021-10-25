cli的入口文件在`@vue/cli/bin/vue.js`

`vue-cli`引入了 [Commander.js](https://github.com/tj/commander.js/blob/master/Readme_zh-CN.md) 进行命令定义，以下：
```js
const program = require('commander')
program
  .version(`@vue/cli ${require('../package').version}`)
  .usage('<command> [options]')
```

看`vue create`命令的定义：
```js
program
  .command('create <app-name>')
  .description('create a new project powered by vue-cli-service')
  // 省略
  .action((name, options) => {
    // 省略
      require('../lib/create')(name, options)
  })
```

当运行`vue create`命令时，相当于`node vue.js create [其他参数]`, 然后执行的是`../lib/create`导出的函数

在`../lib/create`中，会先对`项目名称`、`项目目标位置`进行校验，最后执行：

```js
// @vue/cli/lib/create.js
// 省略：对项目名称、目标位置校验
const creator = new Creator(name, targetDir, getPromptModules())
await creator.create(options)
```
传递参数，初始化`Creator`并调用`create`方法

上面的传参`getPromptModules()`是个数组，数组项是`../promptModules`下各个文件导出的函数：

```js
exports.getPromptModules = () => {
  return [
    'vueVersion',
    'babel',
    'typescript',
    'pwa',
    'router',
    'vuex',
    'cssPreprocessors',
    'linter',
    'unit',
    'e2e'
  ].map(file => require(`../promptModules/${file}`))
}
```

`../promptModules/`下各个文件的导出都是一个函数，如`babel`部分：

```js
// @vue/cli/lib/promptModules/babel.js
module.exports = cli => {
  cli.injectFeature({
    name: 'Babel',
    value: 'babel',
    short: 'Babel',
    description: 'Transpile modern JavaScript to older versions (for compatibility)',
    link: 'https://babeljs.io/',
    checked: true
  })

  cli.onPromptComplete((answers, options) => {
    if (answers.features.includes('ts')) {
      if (!answers.useTsWithBabel) {
        return
      }
    } else if (!answers.features.includes('babel')) {
      return
    }
    options.plugins['@vue/cli-plugin-babel'] = {}
  })
}
```

之后会在`Creator`类的构造函数中依次调用：

```js
// @vue/cli/lib/Creator.js
constructor (name, context, promptModules) {
    super()
    // 省略
    const promptAPI = new PromptModuleAPI(this)
    promptModules.forEach(m => m(promptAPI))
}
```

而注入的`PromptModuleAPI`参数的内容如下：
```js
// @vue/cli/lib/PromptModuleAPI
module.exports = class PromptModuleAPI {
  constructor (creator) {
    this.creator = creator
  }

  injectFeature (feature) {
    this.creator.featurePrompt.choices.push(feature)
  }

  injectPrompt (prompt) {
    this.creator.injectedPrompts.push(prompt)
  }

  injectOptionForPrompt (name, option) {
    this.creator.injectedPrompts.find(f => {
      return f.name === name
    }).choices.push(option)
  }

  onPromptComplete (cb) {
    this.creator.promptCompleteCbs.push(cb)
  }
}
```

可以看出，上面 `@vue/cli/lib/promptModules/babel.js` 中的参数`cli`其实是 `PromptModuleAPI` 的实例。`PromptModuleAPI`这个类里面定义了`prompt`选项相关的一些方法，如：
1. `injectFeature`的作用是将功能特性选项添加到 `Creator` 实例的`featurePrompt`中
2. `onPromptComplete` 则是注册了询问完之后的回调函数

继续看向`lib/Creator.js`中`Create`这个类的逻辑

#### 一、工具方法和工具库的引入：

```js
const path = require('path')
// debug工具，可以记录程序运行时间
const debug = require('debug')
// 交互式询问
const inquirer = require('inquirer')
const EventEmitter = require('events')
const Generator = require('./Generator')
const cloneDeep = require('lodash.clonedeep')
//  排序：将指定的健值对排在前面
const sortObject = require('./util/sortObject')
// 获取CLI plugin的最新版本
const getVersions = require('./util/getVersions')
// 包管理器：npm、pnpm、yarn
const PackageManager = require('./util/ProjectPackageManager')
// 清空命令窗口 并输出指定内容
const { clearConsole } = require('./util/clearConsole')
// 针对询问时交互prompt定义的一些方法
const PromptModuleAPI = require('./PromptModuleAPI')
// 生成文件，如：package.json、.npmrc、README.md等
const writeFileTree = require('./util/writeFileTree')
const { formatFeatures } = require('./util/features')
// 读取本地preset.json信息
const loadLocalPreset = require('./util/loadLocalPreset')
// 下载远程preset信息至系统临时目录 再读取
const loadRemotePreset = require('./util/loadRemotePreset')
// 生成 readme
const generateReadme = require('./util/generateReadme')
// resolvePkg: 使用【read-pkg】读取package.json文件
// isOfficialPlugin: 是否是官方插件
const { resolvePkg, isOfficialPlugin } = require('@vue/cli-shared-utils')

// 读取、保存、更新、校验 .vuerc文件
const {
  defaults,
  saveOptions,
  loadOptions,
  savePreset,
  validatePreset,
  rcPath
} = require('./options')

const {
  chalk,
  execa,

  log,
  warn,
  error,

  hasGit,
  hasProjectGit,
  hasYarn,
  hasPnpm3OrLater,
  hasPnpmVersionOrLater,

  exit,
  loadModule // 加载文件
} = require('@vue/cli-shared-utils')

// 是否是手动选择模式
const isManualMode = answers => answers.preset === '__manual__'
```

上面的`require(./options)`部分，主要是用来读取、保存、更新、校验 `.vuerc`，这个文件会保存在项目的根目录下（如果有），文件的配置如：

```json
{
  "useTaobaoRegistry": true,
  "latestVersion": "4.2.2",
  "lastChecked": 1634718813042,
  "packageManager": "yarn",
  "presets": {
    "vue-cli-demo": {
      "useConfigFiles": true,
      "plugins": {
        "@vue/cli-plugin-babel": {},
        "@vue/cli-plugin-router": {
          "historyMode": false
        },
        "@vue/cli-plugin-eslint": {
          "config": "base",
          "lintOn": [
            "save"
          ]
        }
      },
      "cssPreprocessor": "less"
    }
  }
}
```

#### 二、`Constructor`：

```js
constructor (name, context, promptModules) {
    super()

    this.name = name // 项目名称
    this.context = process.env.VUE_CLI_CONTEXT = context // 项目目录地址
    const { presetPrompt, featurePrompt } = this.resolveIntroPrompts()

    this.presetPrompt = presetPrompt
    this.featurePrompt = featurePrompt
    this.outroPrompts = this.resolveOutroPrompts()
    this.injectedPrompts = []
    this.promptCompleteCbs = []
    this.afterInvokeCbs = []
    this.afterAnyInvokeCbs = []

    this.run = this.run.bind(this)

    const promptAPI = new PromptModuleAPI(this)
    promptModules.forEach(m => m(promptAPI))
  }
```

`resolveIntroPrompts()`方法：

```js
  getPresets () {
      // 读取.vuerc内容
      const savedOptions = loadOptions()
      // 将默认的presets配置与读取到的.vuerc里的presets配置合并
      // 默认的presets包括：default、__default_vue_3__
      // 假设.vuerc里有之前保存的：vue-cli-demo
      // 那么最终`getPresets`返回的presets这个对象包含有3个key: default、__default_vue_3__、vue-cli-demo
      return Object.assign({}, savedOptions.presets, defaults.presets)
  }         

  resolveIntroPrompts () {
    const presets = this.getPresets()
    // 处理预设选项
    const presetChoices = Object.entries(presets).map(([name, preset]) => {
      let displayName = name
      if (name === 'default') {
        displayName = 'Default'
      } else if (name === '__default_vue_3__') {
        displayName = 'Default (Vue 3)'
      }

      return {
        name: `${displayName} (${formatFeatures(preset)})`,
        value: name
      }
    })
    // 预设选项，选择项包括： 默认的2个 + .vuerc里的（如果有）+ 手动选择
    const presetPrompt = {
      name: 'preset',
      type: 'list',
      message: `Please pick a preset:`,
      choices: [
        ...presetChoices,
        {
          name: 'Manually select features',
          value: '__manual__'
        }
      ]
    }
    const featurePrompt = {
      name: 'features',
      when: isManualMode, // 只有在用户选择了【手动选择】才有这个询问
      type: 'checkbox',
      message: 'Check the features needed for your project:',
      choices: [],
      pageSize: 10
    }
    return {
      presetPrompt,
      featurePrompt
    }
  }
```

预设提示选项`presetPrompt`是合并了：默认的`Default`和`Default (Vue 3)`、保存在`.vuerc`的`presets`中的选项（如果有）、手动选择`Manually select features`。功能提示`featurePrompt`只在用户选择了`Manually select features`才会出现，此时`choices`是`[]`

再看`resolveOutroPrompts()`这个方法，返回的提示有：

1. 将`babel/eslint`等配置放在`package.json` 还是 分别抽离成独立的文件？
2. 是否将预设配置保存在项目中（`.vuerc`）
3. 输入将要保存的`presets`的名称
4. 选择包管理工具：`yarn`、`npm`、`pnpm`


初始化最后一步：

```js
const promptAPI = new PromptModuleAPI(this)
promptModules.forEach(m => m(promptAPI))
```

如上文提到，这里是将`功能`、`提示`、`回调`分别添加到`Create`实例上相应的`featurePrompt.choices`，`injectedPrompts`，`promptCompleteCbs`中。

此时，`featurePrompt`中的`choices`已经不为空了。



#### 三、`create`函数：

```js
async create (cliOptions = {}, preset = null) {
  if (!preset) {
      // cliOptions是命令行参数对象。
      if (cliOptions.preset) {
        // vue create foo --preset bar
        // bar有2种来源：1. 远程 Preset 2. 本地（包含 preset.json 的文件夹、当前工作目录下的 json 文件）
        preset = await this.resolvePreset(cliOptions.preset, cliOptions.clone)
      } else if (cliOptions.default) {
        // vue create foo --default
        // 默认预设
        preset = defaults.presets.default
      } else if (cliOptions.inlinePreset) {
        // vue create foo --inlinePreset {...}
        // 内联预设：直接传json对象
        try {
          preset = JSON.parse(cliOptions.inlinePreset)
        } catch (e) {
          error(`CLI inline preset is not valid JSON: ${cliOptions.inlinePreset}`)
          exit(1)
        }
      } else {
        preset = await this.promptAndResolvePreset()
      }
    }
    // 其他
}
```

整体逻辑是先取命令行参数传递的preset，如果没有，则采用提示交互`手动选择`，执行`promptAndResolvePreset()`：

```js
async promptAndResolvePreset (answers = null) {
    // prompt
    if (!answers) {
      // 清空命令窗口
      await clearConsole(true)
      /* 
      各种询问：
      1. presetPrompt：预设选项
      2. featurePrompt：功能选项（如：vue版本，是否集成vuex、vueRouter、单测等）
      3. injectedPrompts：针对功能的询问（如：选择了集成单测，会提示询问选择哪一种：Jest或Mocha + Chai）
      4. outroPrompts：参考上文resolveOutroPrompts()的返回
       */
      answers = await inquirer.prompt(this.resolveFinalPrompts())
    }
    debug('vue-cli:answers')(answers)

    if (answers.packageManager) {
    // 将选择的包管理选项保存到`.vuerc`
      saveOptions({
        packageManager: answers.packageManager
      })
    }

    let preset
    if (answers.preset && answers.preset !== '__manual__') {
    // 如果不是手动选择模式，preset则根据answers.preset，从【.vuerc的presets】或【默认配置】中读取
      preset = await this.resolvePreset(answers.preset)
    } else {
      // manual
      preset = {
        useConfigFiles: answers.useConfigFiles === 'files',
        plugins: {}
      }
      answers.features = answers.features || []
      // run cb registered by prompt modules to finalize the preset
      // 询问完毕后 执行回调 向preset这个对象注入配置
      this.promptCompleteCbs.forEach(cb => cb(answers, preset))
    }

    // validate
    validatePreset(preset)

    // save preset：将最后得到的preset配置保存在.vuerc中，以便下次使用
    if (answers.save && answers.saveName && savePreset(answers.saveName, preset)) {
      log()
      log(`🎉  Preset ${chalk.yellow(answers.saveName)} saved in ${chalk.yellow(rcPath)}`)
    }

    debug('vue-cli:preset')(preset)
    return preset
  }
```

之后的主要步骤：

1. 向`preset.plugins`中注入`@vue/cli-service`
2. 确定包管理器：yarn/npm/pnpm
3. 生成`package.json`
4. 将项目目录初始化为git仓库
5. 执行`install`，安装CLI plugins（如：@vue/cli-plugin-babel、@vue/cli-plugin-eslint、@vue/cli-plugin-router等）
6. 执行生成器`Generator`
7. 再次执行`install`，安装生产依赖（由生成器generators注入，如：eslint、vue、vue-router、vuex等）
8. 完成

第6步`Generator`非常关键，是在确定`options`选择后，分别执行各个插件里的`generator`逻辑，对文件进行整合，写入目录，最后才形成一个完整项目。


```js
    // run generator
    const plugins = await this.resolvePlugins(preset.plugins, pkg)
    const generator = new Generator(context, {
      pkg,
      plugins,
      afterInvokeCbs,
      afterAnyInvokeCbs
    })
    await generator.generate({
      extractConfigFiles: preset.useConfigFiles
    })
```

`Generator`类及`generator()`接受的参数有：
1. pkg： `package.json`,
2. plugins： 
  ```js
  // { id: options } => [{ id, apply, options }]
  async resolvePlugins (rawPlugins, pkg) {
    // ensure cli-service is invoked first
    rawPlugins = sortObject(rawPlugins, ['@vue/cli-service'], true)
    const plugins = []
    for (const id of Object.keys(rawPlugins)) {
      // 使用Module.createRequire创建的require()方法加载模块
      const apply = loadModule(`${id}/generator`, this.context) || (() => {})
      let options = rawPlugins[id] || {}

      // 处理plugin里的prompts
      if (options.prompts) {
        let pluginPrompts = loadModule(`${id}/prompts`, this.context)

        if (pluginPrompts) {
          const prompt = inquirer.createPromptModule()

          if (typeof pluginPrompts === 'function') {
            pluginPrompts = pluginPrompts(pkg, prompt)
          }
          if (typeof pluginPrompts.getPrompts === 'function') {
            pluginPrompts = pluginPrompts.getPrompts(pkg, prompt)
          }

          log()
          log(`${chalk.cyan(options._isPreset ? `Preset options:` : id)}`)
          options = await prompt(pluginPrompts)
        }
      }

      plugins.push({ id, apply, options })
    }
    return plugins
  }
  ```

  对`preset.plugins`处理后，得到`Generator`需要的`plugins`参数，如：

  ```js
  {
    "@vue/cli-plugin-babel": {},
    "@vue/cli-plugin-router": {
      "historyMode": false
    },
    "@vue/cli-plugin-eslint": {
      "config": "base",
      "lintOn": [
        "save"
      ]
    }
  }
  ```

  `plugins`：

  ```js
  [
    {
      id: "@vue/cli-plugin-babel",
      apply: "@vue/cli-plugin-babel里的generator.js或generator/index.js导出的函数",
      options: {}
    },
    ...
  ]
  ```

  3. afterInvokeCb、afterAnyInvokeCbs： `Generator`实例收集的`plugins`里的回调，此时均为`[]`
  4. extractConfigFiles：是否将配置抽离成独立的文件，如：`vue.config.js`、`babel.config.js`、`.eslintrc`等


`generate()`方法：

```js
async generate ({
    extractConfigFiles = false,
    checkExisting = false
  } = {}) {
    await this.initPlugins()

    // save the file system before applying plugin for comparison
    const initialFiles = Object.assign({}, this.files)
    // extract configs from package.json into dedicated files.
    this.extractConfigFiles(extractConfigFiles, checkExisting)
    // wait for file resolve
    await this.resolveFiles()
    // set package.json
    this.sortPkg()
    this.files['package.json'] = JSON.stringify(this.pkg, null, 2) + '\n'
    // write/update file tree to disk
    await writeFileTree(this.context, this.files, initialFiles, this.filesModifyRecord)
  }
```

分为以下几步：

1. 初始化所有插件（包括官方插件、第三方插件），执行插件内的`generator`：

```js
async initPlugins () {
    const { rootOptions, invoking } = this
    const pluginIds = this.plugins.map(p => p.id)

    // avoid modifying the passed afterInvokes, because we want to ignore them from other plugins
    const passedAfterInvokeCbs = this.afterInvokeCbs
    this.afterInvokeCbs = []
    // apply hooks from all plugins to collect 'afterAnyHooks'
    // 所有插件
    for (const plugin of this.allPlugins) {
      const { id, apply } = plugin
      const api = new GeneratorAPI(id, this, {}, rootOptions)

      if (apply.hooks) {
        await apply.hooks(api, {}, rootOptions, pluginIds)
      }
    }

    // We are doing save/load to make the hook order deterministic
    // save "any" hooks
    const afterAnyInvokeCbsFromPlugins = this.afterAnyInvokeCbs

    // reset hooks
    this.afterInvokeCbs = passedAfterInvokeCbs
    this.afterAnyInvokeCbs = []
    this.postProcessFilesCbs = []

    // apply generators from plugins
    for (const plugin of this.plugins) {
      const { id, apply, options } = plugin
      // GeneratorAPI类包含很多方法，如：
      // 1. extendPackage： 扩展package.json
      // 2. render：Render template files into the virtual files tree object.
      // 3. injectImports： Add import statements to a file.
      // ......
      const api = new GeneratorAPI(id, this, options, rootOptions)
      // 上文提到，apply是插件里的generator.js或generator/index.js导出的函数
      // 这里将GeneratorAPI类的实例api传入并执行
      await apply(api, options, rootOptions, invoking)

      if (apply.hooks) {
        // while we execute the entire `hooks` function,
        // only the `afterInvoke` hook is respected
        // because `afterAnyHooks` is already determined by the `allPlugins` loop above
        await apply.hooks(api, options, rootOptions, pluginIds)
      }
    }
    // restore "any" hooks
    this.afterAnyInvokeCbs = afterAnyInvokeCbsFromPlugins
  }
```

2. 将package.json里的配置抽离成独立文件（根据配置）

```js
extractConfigFiles (extractAll, checkExisting) {
    const configTransforms = Object.assign({},
      defaultConfigTransforms,
      this.configTransforms,
      reservedConfigTransforms
    )
    const extract = key => {
      if (
        configTransforms[key] &&
        this.pkg[key] &&
        // do not extract if the field exists in original package.json
        !this.originalPkg[key]
      ) {
        const value = this.pkg[key]
        const configTransform = configTransforms[key]
        const res = configTransform.transform(
          value,
          checkExisting,
          this.files,
          this.context
        )
        const { content, filename } = res
        this.files[filename] = ensureEOL(content)
        delete this.pkg[key]
      }
    }
    if (extractAll) {
      for (const key in this.pkg) {
        extract(key)
      }
    } else {
      if (!process.env.VUE_CLI_TEST) {
        // by default, always extract vue.config.js
        extract('vue')
      }
      // always extract babel.config.js as this is the only way to apply
      // project-wide configuration even to dependencies.
      // TODO: this can be removed when Babel supports root: true in package.json
      extract('babel')
    }
  }
```

3. 执行插件初始化时收集的各种文件的处理数组，然后将返回的内容收集到files字段。

```js
async resolveFiles () {
    const files = this.files
    for (const middleware of this.fileMiddlewares) {
      await middleware(files, ejs.render)
    }

    // normalize file paths on windows
    // all paths are converted to use / instead of \
    normalizeFilePaths(files)

    // handle imports and root option injections
    Object.keys(files).forEach(file => {
      let imports = this.imports[file]
      imports = imports instanceof Set ? Array.from(imports) : imports
      if (imports && imports.length > 0) {
        files[file] = runTransformation(
          { path: file, source: files[file] },
          require('./util/codemods/injectImports'),
          { imports }
        )
      }

      let injections = this.rootOptions[file]
      injections = injections instanceof Set ? Array.from(injections) : injections
      if (injections && injections.length > 0) {
        files[file] = runTransformation(
          { path: file, source: files[file] },
          require('./util/codemods/injectOptions'),
          { injections }
        )
      }
    })

    for (const postProcess of this.postProcessFilesCbs) {
      await postProcess(files)
    }
    debug('vue:cli-files')(this.files)
  }
```

4. 写入文件




















