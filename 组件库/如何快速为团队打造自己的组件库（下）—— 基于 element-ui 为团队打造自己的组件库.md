# 如何快速为团队打造自己的组件库（下）—— 基于 element-ui 为团队打造自己的组件库

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![element-ui](https://gitee.com/liyongning/typora-image-bed/raw/master/202202102106804.png)

## 简介

在了解 [Element 源码架构](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484649&idx=1&sn=8ee67553193fda33e7c637568bb0a86f&chksm=9f69679da81eee8bba046776de07f8848ad9061e6c04ca781bf052fc9bcee70ea34a48c81864&token=103240474&lang=zh_CN#rd) 的基础上，接下来我们基于 element-ui 为团队打造自己的组件库。

## 主题配置

基础组件库在 UI 结构上差异很小，一般只是在主题色上会有较大差异，毕竟每个团队都有了 UI 风格。比如，我们团队的 UI 设计稿其实是基于 Ant Design 来出的，而组件库是基于 Element-UI 来开发，即使是这种情况，对组件本身的改动也很少。所以，基于开源库打造团队自己的组件库时，主题配置就很重要了。

element-ui 的一大特色就是支持自定义主题，它通过在线主题编辑器、Chrome 插件或命令行主题工具这三种方式来定制 element-ui 所有组件的样式。那么 element-ui 是怎么做到这一点的呢？

因为 element-ui 组件样式中的颜色、字体、线条等样式都是通过变量的方式引入的，在 `packages/theme-chalk/src/common/var.scss` 中可以看到这些变量的定义，这就为自定义主题提供了方便，因为我们只需要修改这些变量，就可以实现组件主题的改变。

在线主题编辑器和 Chrome 插件支持实时预览。并且可以下载定制的样式包，然后使用。在线主题编辑器和 Chrome 插件的优点是可视化，简洁明了，但是有个最大的缺点就是，最后下载出来的是一个将所有组件样式打包到一起的样式包，没办法支持按需加载，不推荐使用。这里我们使用命令行主题工具来定制样式。

## [命令行主题工具](https://element.eleme.cn/#/zh-CN/component/custom-theme)

* 初始化项目目录并安装主题生成工具（element-theme）

  ```shell
  mkdir theme && cd theme && npm init -y && npm i element-theme -D
  ```

* 安装白垩主题

  ```shell
  npm i element-theme-chalk -D
  ```

* 初始化变量文件

  ```shell
  node_modules/.bin/et -i
  ```

  命令执行以后可能会得到如下报错信息

  ![image-20220213193714430](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131937601.png)

  原因是 element-theme 包中依赖了低版本的 graceful-fs，低版本 graceful-fs 在高版本的 node.js 中不兼容，最简单的方案是升级 graceful-fs。

  在项目根目录下创建 `npm-shrinkwrap.json` 文件，并添加如下内容：

  ```json
  {
     "dependencies": {
         "graceful-fs": {
             "version": "4.2.2"
         }
     }
  }
  ```

  运行 `npm install` 重新安装依赖即可解决，然后重新执行 `node_modules/.bin/et -i`，执行完以后会在当前目录生成 `element-variables.scss` 文件。

* 修改变量

  直接编辑 `element-variables.scss` 文件，例如修改主题色为红色，将文件中的 `$--color-primary` 的值修改为 `red`，`$--color-primary: red !default;`。

  文件中写了很好的注释，并且样式代码也是按照组件来分割组织的，所以大家可以对照设计团队给到的设计稿来一一修改相关的变量。如果实在觉得看代码比较懵，可以参照在线主题编辑器，两边的变量名是一致的。

  > 题外话：element-ui 还提供了两个资源包，供设计团队使用，所以最理想的是，让设计团队根据 element-ui 的资源包出设计稿，这样两边就可以做到统一，研发团队的工作量也会降低不少。比如我们团队就不是这样，设计团队给到的设计稿是基于 Ant Design 出的，研发组件库时改动的工作量和难度就会相对比较大。所以研发、设计、产品一定要进行很好的沟通。

* 编译主题

  修改完以后，保存文件，然后执行以下命令编译主题，会产生一个 theme 目录。生产出来都是 CSS 样式文件，文件名和组件名一一对应，支持按需引入（指定组件的样式文件）和全量引入（index.css）。

  * 生产未压缩的样式文件

    ```shell
    node_modules/.bin/et --out theme-chalk
    ```

  * 生产经过压缩的样式文件

    ```shell
    node_modules/.bin/et --minimize --out theme-chalk
    ```

  * 帮助命令

    ```shell
    node_modules/.bin/et --help
    ```

  * 启用 `watch` 模式，实时编译主题

    ```shell
    node_modules/.bin/et --watch --out theme-chalk
    ```

* 使用自定义主题

  * 用新生成的主题目录（theme-chalk）替换掉框架中的 `packages/theme-chalk` 目录。重命名老的 theme-chalk 为 theme-chalk.bak，不要删掉，后面需要用

    > 建议将生成主题时用到的 element-variables.scss 文件保存在项目中，因为以后你可能会需要重新生成主题

  * 修改 `/examples/entry.js`、 `/examples/play.js`、`/examples/extension/src/app.js` 中引入的组件库的样式

    ```javascript
    // 用新的样式替换旧的默认样式
    // import 'packages/theme-chalk/src/index.scss
    import 'packages/theme-chalk/index.css
    ```

  * 修改 `/build/bin/iconInit.js` 中引入的图标样式文件

    ```javascript
    // var fontFile = fs.readFileSync(path.resolve(__dirname, '../../packages/theme-chalk/src/icon.scss'), 'utf8');
    var fontFile = fs.readFileSync(path.resolve(__dirname, '../../packages/theme-chalk/icon.css'), 'utf8');
    ```

  * 修改 `/examples/docs/{四种语言}/custom-theme.md` 

    ```
    // @import "~element-ui/packages/theme-chalk/src/index";
    @import "~element-ui/packages/theme-chalk/index";
    ```

    

  * 执行 `make dev` 启动开发环境，[查看效果](http://0.0.0.0:8085/#/zh-CN/component/button)

    ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131929618.png)

  到这一步，主题配置就结束了，你会发现，element-ui 官网的组件样式基本上和设计稿上的一致。但是仔细对比后，会发现有一些组件的样式和设计稿有差异，这时候就需要对这些组件的样式进行深度定制，覆写不一致的样式。

  > 其实这块儿漏掉了 /build/bin/new.js 中涉及的样式目录，这块儿的改动会放到后面

## 样式深度定制

上一步的主题配置，只能解决主题相关的样式，但是有些组件的有些样式不属于主题样式，如果这部分样式刚好又和设计稿不一致的话，那就需要重写这部分样式去覆盖上一步的样式。

> 以下配置还支持为自定义组件添加样式

### 样式目录

* 将 `主题配置` 步骤中备份的 `/packages/theme-chalk.bak` 重命名为 `/packages/theme-lyn`，作为覆写组件和自定义组件的样式目录

* 删掉 `/packages/theme-lyn/src` 目录的所有文件

* 你会写 scss ？

  * 忽略掉下一步，然后后续步骤你只需将对应的 less 操作换成 sass 即可

* 你不会写 scss，扩展其它语法，假设你会写 less

  * 在项目根目录执行以下命令，然后删掉 gulp-sass

    ```shell
    npm i less less-loader gulp-less -D && npm uninstall gulp-sass -D
    ```

    > 如果一会儿启动开发环境以后，报错 “TypeError: this.getOptions is not a function”，则降级 less-loader 版本，比如我的版本是：less@3.7.1、less-loader@7.3.0

  * 在 `/packages/theme-lyn` 目录下执行以下命令，然后删掉 gulp-sass

    ```shell
    npm i gulp-less -D && npm uninstall gulp-sass -D
    ```

  * 将 `/packages/theme-lyn/gulpfile.js` 更改为以下内容

    ```javascript
    'use strict';
    
    /**
     *  将 ./src/*.less 文件编译成 css 文件输出到 ./lib 目录
     *  将 ./src/fonts/中的所有字体文件输出到 ./lib/fonts 中，如果你没有覆写字体样式的需要，则删掉拷贝字体样式部分
     */
    const { series, src, dest } = require('gulp');
    const less = require('gulp-less');
    const autoprefixer = require('gulp-autoprefixer');
    const cssmin = require('gulp-cssmin');
    const path = require('path')
    
    function compile() {
      return src('./src/*.less')
        .pipe(less({
          paths: [ path.join(__dirname, './src') ]
        }))
        .pipe(autoprefixer({
          browsers: ['ie > 9', 'last 2 versions'],
          cascade: false
        }))
        .pipe(cssmin())
        .pipe(dest('./lib'));
    }
    
    function copyfont() {
      return src('./src/fonts/**')
        .pipe(cssmin())
        .pipe(dest('./lib/fonts'));
    }
    
    // 也可以在这里扩展其它功能，比如拷贝静态资源
    
    exports.build = series(compile, copyfont);
    
    ```

  * 在 `build/webpack.demo.js` 中增加解析 `less` 文件的规则

    ```javascript
    {
      test: /\.less$/,
      use: [
        isProd ? MiniCssExtractPlugin.loader : 'style-loader',
        'css-loader',
        'less-loader'
      ]
    }
    ```

    

* 假如你要覆写 button 组件的部分样式

  * 在 `/packages/theme-lyn/src` 目录下新建 `button.less` 文件，编写覆写样式时请遵循如下规则

    * 组件样式的覆写，最好遵循 BEM 风格，目的是提供良好的命名空间隔离，避免样式打包以后发生意料之外的覆盖
    * 只覆写已有的样式，可以在组件上新增类名，但不要删除，目的是兼容线上代码

    ```less
    // 这里我要把主要按钮的字号改大有些，只是为了演示效果
    .el-button--primary {
      font-size: 24px;
    }
    ```

    

* 改造 `build/bin/gen-cssfile.js` 脚本

  ```javascript
  /**
   * 将各个覆写的样式文件在 packages/theme-lyn/src/index.less 文件中自动引入
   */
  
  var fs = require('fs');
  var path = require('path');
  
  // 生成 theme-lyn/src 中的 index.less 文件
  function genIndexLessFile(dir) {
    // 文件列表
    const files = fs.readdirSync(dir);
    /**
     * @import 'x1.less';
     * @import 'x2.less;
     */
    let importStr = "/* Automatically generated by './build/bin/gen-cssfile.js' */\n";
  
    // 需要排除的文件
    const excludeFile = ['assets', 'font', 'index.less', 'base.less', 'variable.less'];
  
    files.forEach(item => {
      if (excludeFile.includes(item) || !/\.less$/.test(item)) return;
  
      // 只处理非 excludeFile 中的 less 文件
      importStr += `@import "./${item}";\n`;
    });
  
    // 在 packages/theme-lyn/src/index.less 文件中写入 @import "xx.less"，即在 index.less 中引入所有的样式文件
    fs.writeFileSync(path.resolve(dir, 'index.less'), importStr);
  }
  
  genIndexLessFile(path.resolve(__dirname, '../../packages/theme-lyn/src/'));
  
  ```

* 在项目根目录下执行以下命令

  ```shell
  npm i shelljs -D
  ```

* 新建 `/build/bin/compose-css-file.js`

  ```javascript
  /**
   * 负责将打包后的两个 css 目录(lib/theme-chalk、lib/theme-lyn)合并
   * lib/theme-chalk 目录下的样式文件是通过主题配置自动生成的
   * lib/theme-lyn 是扩展组件的样式（覆写默认样式和自定义组件的样式）
   * 最后将样式都合并到 lib/theme-chalk 目录下
   */
  const fs = require('fs');
  const fileSave = require('file-save');
  const { resolve: pathResolve } = require('path');
  const shelljs = require('shelljs');
  
  const themeChalkPath = pathResolve(__dirname, '../../lib/theme-chalk');
  const themeStsUIPath = pathResolve(__dirname, '../../lib/theme-lyn');
  
  // 判断样式目录是否存在
  let themeChalk = null;
  let themeStsUI = null;
  try {
    themeChalk = fs.readdirSync(themeChalkPath);
  } catch (err) {
    console.error('/lib/theme-chalk 不存在');
    process.exit(1);
  }
  try {
    themeStsUI = fs.readdirSync(themeStsUIPath);
  } catch (err) {
    console.error('/lib/theme-lyn 不存在');
    process.exit(1);
  }
  
  /**
   * 遍历两个样式目录，合并相同文件，将 theme-lyn 的中样式追加到 theme-chalk 中对应样式文件的末尾
   * 如果 theme-lyn 中的文件在 theme-chalk 中不存在（比如扩展的新组件）,则直接将文件拷贝到 theme-chalk
   */
  const excludeFiles = ['element-variables.css', 'variable.css'];
  for (let i = 0, themeStsUILen = themeStsUI.length; i < themeStsUILen; i++) {
    if (excludeFiles.includes(themeStsUI[i])) continue;
  
    if (themeStsUI[i] === 'fonts') {
      shelljs.cp('-R', pathResolve(themeStsUIPath, 'fonts/*'), pathResolve(themeChalkPath, 'fonts'));
      continue;
    }
  
    if (themeStsUI[i] === 'assets') {
      shelljs.cp('-R', pathResolve(themeStsUIPath, 'assets'), themeChalkPath);
      continue;
    }
  
    if (themeChalk.includes(themeStsUI[i])) {
      // 说明当前样式文件是覆写 element-ui 中的样式
      const oldFileContent = fs.readFileSync(pathResolve(themeChalkPath, themeStsUI[i]), { encoding: 'utf-8' });
      fileSave(pathResolve(themeChalkPath, themeStsUI[i])).write(oldFileContent).write(fs.readFileSync(pathResolve(themeStsUIPath, themeStsUI[i])), 'utf-8').end();
    } else {
      // 说明当前样式文件是扩展的新组件的样式文件
      // fs.writeFileSync(pathResolve(themeChalkPath, themeStsUI[i]), fs.readFileSync(pathResolve(themeStsUIPath, themeStsUI[i])));
      shelljs.cp(pathResolve(themeStsUIPath, themeStsUI[i]), themeChalkPath);
    }
  }
  
  // 删除 lib/theme-lyn
  shelljs.rm('-rf', themeStsUIPath);
  
  ```

* 改造 package.json 中的 scripts

  ```json
  {
    "gen-cssfile:comment": "在 /packages/theme-lyn/src/index.less 中自动引入各个组件的覆写样式文件",
    "gen-cssfile": "node build/bin/gen-cssfile",
    "build:theme:comment": "构建主题样式：在 index.less 中自动引入各个组件的覆写样式文件 && 通过 gulp 将 less 文件编译成 css 并输出到 lib 目录 && 拷贝基础样式 theme-chalk 到 lib/theme-chalk && 拷贝 编译后的 theme-lyn/lib/* 目录到 lib/theme-lyn && 合并 theme-chalk 和 theme-lyn",
    "build:theme": "npm run gen-cssfile && gulp build --gulpfile packages/theme-lyn/gulpfile.js && cp-cli packages/theme-lyn/lib lib/theme-lyn && cp-cli packages/theme-chalk lib/theme-chalk && node build/bin/compose-css-file.js",
  }
  ```

* 执行以下命令

  ```shell
  npm run gen-cssfile
  ```

* 改造 `/examples/entry.js` 和 `/examples/play.js`

  ```javascript
  // 用新的样式替换旧的默认样式
  // import 'packages/theme-chalk/src/index.scss
  import 'packages/theme-chalk/index.css	// 在这行下面引入自定义样式
  // 引入自定义样式
  import 'packages/theme-lyn/src/index.less'
  ```

* [访问官网](http://0.0.0.0:8085/#/zh-CN/component/button)，查看 button 组件的覆写样式是否生效

  ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131930853.png)

## 自定义组件

组件库在后续的开发和迭代中，需要两种自定义组件的方式：

* 增加新的 element-ui 组件

  > element-ui 官网可能在某个时间点增加一个你需要的基础组件，这时你需要将其集成进来

* 增加业务组件

  > 基础组件就绪以后，团队就会开始推动业务组件的建设，这时候就会向组件库中增加新的组件

### 新的 element-ui 组件

element-ui 提供了增加新组件的脚本，执行 `make new <component-name> [中文名]` 即可生成新组件所需的所有文件以及配置，比如：`make new button 按钮`，有了该脚本可以让你专注于组件的编写，不需要管任何配置。

#### `/build/bin/new.js`

但是由于我们调整了框架主题库的结构，所以脚本文件也需要做相应的调整。需要将 `/build/bin/new.js` 文件中处理样式的代码删掉，样式文件不再需要脚本自动生成，而是通过重新生成主题的方式实现。

```javascript
// /build/bin/new.js 删掉以下代码
{
    filename: path.join('../../packages/theme-chalk/src', `${componentname}.scss`),
    content: `@import "mixins/mixins";
@import "common/var";

@include b(${componentname}) {
}`
},
  
// 添加到 index.scss
const sassPath = path.join(__dirname, '../../packages/theme-chalk/src/index.scss');
const sassImportText = `${fs.readFileSync(sassPath)}@import "./${componentname}.scss";`;
fileSave(sassPath)
  .write(sassImportText, 'utf8')
  .end('\n');
```

#### `Makefile`

改造 `Makefile` 文件，在 `new` 配置后面增加 `&& npm run build:file` 命令，重新生成组件库入口文件，不然不会引入新增加的组件。

```makefile
new:
    node build/bin/new.js $(filter-out $@,$(MAKECMDGOALS)) && npm run build:file
```

#### 增加新组件

完成上述改动以后，只需两步即可完成新 element-ui 组件的创建：

* 执行 `make new <component-name> [组件中文名]` 命令新建新的 element-ui 组件

  > 这一步会生成众多文件，你只需要从新的 element-ui 源码中将该组件对应的代码复制过来填充到对应的文件即可

* 重新生成主题，然后覆盖现在的 `/packages/theme-chalk`

### 业务组件

新增的业务组件就不要以 el 开头了，避免和 element 组件重名或造成误会。需要模拟 `/build/bin/new.js` 脚本写一个新建业务组件的脚本 `/build/bin/new-lyn-ui.js`，大家可以基于该脚本去扩展。

#### `/build/bin/new-lyn-ui.js`

```javascript
'use strict';

/**
 * 新建组件脚本，以 lyn-city 组件为例
 * 1、在 packages 目录下新建组件目录，并完成目录结构的基本创建
 * 2、创建组件文档
 * 3、组件单元测试文件
 * 4、组件样式文件
 * 5、组件类型声明文件
 * 6、并将上述新建的相关资源自动添加的相应的文件，比如组件组件注册到 components.json 文件、样式文件在 index.less 中自动引入等
 * 总之你只需要专注于编写你的组件代码即可，其它一概不用管
 */

console.log();
process.on('exit', () => {
  console.log();
});

if (!process.argv[2]) {
  console.error('[组件名]必填 - Please enter new component name');
  process.exit(1);
}

const path = require('path');
const fs = require('fs');
const fileSave = require('file-save');
const uppercamelcase = require('uppercamelcase');
// 组件名称 city
const componentname = process.argv[2];
// 组件中文名 城市列表
const chineseName = process.argv[3] || componentname;
// 组件大驼峰命名 City
const ComponentName = uppercamelcase(componentname);
// 组件路径：/packages/city
const PackagePath = path.resolve(__dirname, '../../packages', componentname);
const Files = [
  // packages/city/index.js 的内容
  {
    filename: 'index.js',
    content: `import ${ComponentName} from './src/main';

/* istanbul ignore next */
${ComponentName}.install = function(Vue) {
  Vue.component(${ComponentName}.name, ${ComponentName});
};

export default ${ComponentName};`
  },
  // packages/city/src/main.vue 组件定义
  {
    filename: 'src/main.vue',
    content: `<template>
  <div class="lyn-${componentname}"></div>
</template>

<script>
export default {
  name: 'Lyn${ComponentName}'
};
</script>`
  },
  // 组件中文文档
  {
    filename: path.join('../../examples/docs/zh-CN', `${componentname}.md`),
    content: `## ${ComponentName} ${chineseName}`
  },
  // 组件单元测试文件
  {
    filename: path.join('../../test/unit/specs', `${componentname}.spec.js`),
    content: `import { createTest, destroyVM } from '../util';
import ${ComponentName} from 'packages/${componentname}';

describe('${ComponentName}', () => {
  let vm;
  afterEach(() => {
    destroyVM(vm);
  });

  it('create', () => {
    vm = createTest(${ComponentName}, true);
    expect(vm.$el).to.exist;
  });
});
`
  },
  // 组件样式文件
  {
    filename: path.join(
      '../../packages/theme-lyn/src',
      `${componentname}.less`
    ),
    content: `@import "./base.less";\n\n.lyn-${componentname} {
}`
  },
  // 组件类型声明文件
  {
    filename: path.join('../../types', `${componentname}.d.ts`),
    content: `import { LynUIComponent } from './component'

/** ${ComponentName} Component */
export declare class Lyn${ComponentName} extends LynUIComponent {
}`
  }
];

// 将新组件添加到 components.json
const componentsFile = require('../../components.json');
if (componentsFile[componentname]) {
  console.error(`${componentname} 已存在.`);
  process.exit(1);
}
componentsFile[componentname] = `./packages/${componentname}/index.js`;
fileSave(path.join(__dirname, '../../components.json'))
  .write(JSON.stringify(componentsFile, null, '  '), 'utf8')
  .end('\n');

// 在 index.less 中引入新组件的样式文件
const lessPath = path.join(
  __dirname,
  '../../packages/theme-lyn/src/index.less'
);
const lessImportText = `${fs.readFileSync(
  lessPath
)}@import "./${componentname}.less";`;
fileSave(lessPath).write(lessImportText, 'utf8').end('\n');

// 添加到 element-ui.d.ts
const elementTsPath = path.join(__dirname, '../../types/element-ui.d.ts');

let elementTsText = `${fs.readFileSync(elementTsPath)}
/** ${ComponentName} Component */
export class ${ComponentName} extends Lyn${ComponentName} {}`;

const index = elementTsText.indexOf('export') - 1;
const importString = `import { Lyn${ComponentName} } from './${componentname}'`;

elementTsText =
  elementTsText.slice(0, index) +
  importString +
  '\n' +
  elementTsText.slice(index);

fileSave(elementTsPath).write(elementTsText, 'utf8').end('\n');

// 新建刚才声明的所有文件
Files.forEach(file => {
  fileSave(path.join(PackagePath, file.filename))
    .write(file.content, 'utf8')
    .end('\n');
});

// 将新组建添加到 nav.config.json
const navConfigFile = require('../../examples/nav.config.json');

Object.keys(navConfigFile).forEach(lang => {
  const groups = navConfigFile[lang].find(item => Array.isArray(item.groups))
    .groups;
  groups[groups.length - 1].list.push({
    path: `/${componentname}`,
    title:
      lang === 'zh-CN' && componentname !== chineseName
        ? `${ComponentName} ${chineseName}`
        : ComponentName
  });
});

fileSave(path.join(__dirname, '../../examples/nav.config.json'))
  .write(JSON.stringify(navConfigFile, null, '  '), 'utf8')
  .end('\n');

console.log('DONE!');

```

#### Makefile

在 `Makefile` 中增加如下配置：

```makefile
new-lyn-ui:
    node build/bin/new-lyn-ui.js $(filter-out $@,$(MAKECMDGOALS)) && npm run build:file
	
help:
    @echo "   \033[35mmake new-lyn-ui <component-name> [中文名]\033[0m\t---  创建新的 LynUI 组件 package. 例如 'make new-lyn-ui city 城市选择'"
```

## icon 图标

element-ui 虽然提供了大量的 icon，但往往不能满足团队的业务需求，所有就需要往组件库中增加业务 icon，这里以 Iconfont 为例。不建议直接使用设计给的图片或者 svg，太占资源了。

* [打开 iconfont](https://www.iconfont.cn/)

* 登陆 -> 资源管理 -> 我的项目 -> 新建项目

  ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131931952.png)

  > 注意，这里为 icon 设置前缀时不要使用 el-icon-，避免和 element-ui 中的 icon 重复。这个项目就作为团队项目使用了，以后团队所有的业务图标都上传到该项目，所以最好注册一个团队账号。

* 新建成功后，点击 `上传图标至项目` ，选择 `上传图标` ，上传设计给的 svg（必须是 svg），根据需要选择 `保留颜色或不保留并提交`

* 上传完毕，编辑、检查没问题后，点击 `下载至本地`

* 复制其中的 `iconfont.ttf` 和 `iconfont.woff` 到 `/packages/theme-lyn/src/fonts` 目录下

* 新建 `/packages/theme-lyn/src/icon.less` 文件，并添加如下内容

  ```less
  @font-face {
    font-family: 'iconfont';
    src: url('./fonts/iconfont.woff') format('woff'), url('./fonts/iconfont.ttf') format('truetype');
    font-weight: normal;
    font-display: auto;
    font-style: normal;
  }
  
  [class^="lyn-icon-"], [class*=" lyn-icon-"] {
    font-family: 'iconfont' !important;
    font-style: normal;
    font-weight: normal;
    font-variant: normal;
    text-transform: none;
    line-height: 1;
    vertical-align: baseline;
    display: inline-block;
  
    /* Better Font Rendering =========== */
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale
  }
  
  /**
   * 示例：
   * .lyn-icon-iconName:before {
   *   content: "\unicode 16进制码" 
   * }
   * .lyn-icon-add:before {
   *   content: "\e606"
   * }
   */
  ```

* 执行 `npm run gen-cssfile`

* 更新 `/build/bin/iconInit.js` 文件为以下内容

  ```javascript
  'use strict';
  
  var postcss = require('postcss');
  var fs = require('fs');
  var path = require('path');
  
  /**
   * 从指定的 icon 样式文件（entry）中按照给定正则表达式（regExp）解析出 icon 名称，然后输出到指定位置（output）
   * @param {*} entry 被解析的文件相对于当前文件的路径，比如：../../packages/theme-chalk/icon.css
   * @param {*} regExp 被解析的正则表达式，比如：/\.el-icon-([^:]+):before/
   * @param {*} output 解析后的资源输出到相对于当前文件的指定位置，比如：../../examples/icon.json
   */
  function parseIconName(entry, regExp, output) {
    // 读取样式文件
    var fontFile = fs.readFileSync(path.resolve(__dirname, entry), 'utf8');
    // 将样式内容解析为样式节点
    var nodes = postcss.parse(fontFile).nodes;
    var classList = [];
  
    // 遍历样式节点
    nodes.forEach((node) => {
      // 从样式选择器中根据给定匹配规则匹配出 icon 名称
      var selector = node.selector || '';
      var reg = new RegExp(regExp);
      var arr = selector.match(reg);
  
      // 将匹配到的 icon 名称放入 classList
      if (arr && arr[1]) {
        classList.push(arr[1]);
      }
    });
  
    classList.reverse(); // 希望按 css 文件顺序倒序排列
  
    // 将 icon 名称数组输出到指定 json 文件中
    fs.writeFile(path.resolve(__dirname, output), JSON.stringify(classList), () => { });
  }
  
  // 根据 icon.css 文件生成所有的 icon 图标名
  parseIconName('../../packages/theme-chalk/icon.css', /\.el-icon-([^:]+):before/, '../../examples/icon.json')
  
  // 根据 icon.less 文件生成所有的 sts icon 图标名
  parseIconName('../../packages/theme-lyn/src/icon.less', /\.lyn-icon-([^:]+):before/, '../../examples/lyn-icon.json')
  
  ```

* 执行 `npm run build:file`，会看到在 `/examples` 目录下生成了一个 `lyn-icon.json` 文件

* 在 `/examples/entry.js` 中增加如下内容

  ```javascript
  import lynIcon from './lyn-icon.json';
  Vue.prototype.$lynIcon = lynIcon; // StsIcon 列表页用
  ```

* 在 `/examples/nav.config.json` 中业务配置部分增加 `lyn-icon` 路由配置

  ```json
  {
    "groupName": "LynUI",
    "list": [
      {
        "path": "/lyn-icon",
        "title": "icon 图标"
      }
    ]
  }
  ```

* 增加文档 `/examples/docs/{语言}/lyn-icon.md`，添加如下内容

  ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131933186.png)

  

* [查看官网](http://0.0.0.0:8085/#/zh-CN/component/lyn-icon) 看图标是否生效

  ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131934292.png)

* 后续如需扩展新的 icon

  * 在前面新建的 iconfont 项目中上传新的图标，然后点击 `下载至本地`，将其中的 `iconfont.ttf` 和 `iconfont.woff` 复制 `/packages/theme-lyn/src/fonts` 目录下即可（替换已有的文件）

  * 在 `/packages/theme-lyn/src/icon.less` 中设置新的 icon 样式声明

  * 执行 `npm run build:file`

  * [查看官网](http://0.0.0.0:8085/#/zh-CN/component/lyn-icon) 看图标添加是否成功

    ![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202131935092.png)

## 升级 Vue 版本

element-ui 本身依赖的是 vue@^2.5.x，该版本的 vue 不支持最新的 `v-slot` 插槽语法（v-slot 是在 2.6.0 中新增的），组件的 markdown 文档中使用 `v-slot` 语法不生效且会报错，所以需要升级 vue 版本。涉及三个包：vue@^2.6.12、@vue/component-compiler-utils@^3.2.0、vue-template-compiler@^2.6.12。执行以下命令即可完成更新：

* 删除旧包

  ```shell
  npm uninstall vue @vue/component-compiler-utils vue-template-compiler -D
  ```

* 安装新包

  ```shell
  npm install vue@^2.6.12 @vue/component-compiler-utils@^3.2.0 vue-template-compiler@^2.6.12 -D
  ```

* 更新 package.json 中的 peerDependencies

  ```json
  {
    "peerDependencies": {
      "vue": "^2.6.12"
    }
  }
  ```

## 扩展

到这里，组件库的架构调整其实已经完成了，接下来只需组织团队成员对照设计稿进行组件开发就可以了。但是对于一些有洁癖的开发者来说，其实还差点。

比如：

* 团队的组件库不想叫 element-ui，有自己的名称，甚至整个组件库的代码都不想出现 element 字样

* element-ui 的某些功能团队不需要，比如：官网项目（examples）中的主题、资源模块、chrome 插件（extension）、国际化相关（只保留中文即可）

* 静态资源，element-ui 将所有的静态资源都上传到自己的 CDN 上了，我们去访问其实优点慢，可以将相关资源挪到团队自己的 CDN 上

* 工程代码质量问题，element-ui 本身提供了 eslint，做了一点代码质量的控制，但是做的不够，比如格式限制、自动格式化等，可以参考 [搭建自己的 typescript 项目 + 开发自己的脚手架工具 ts-cli](https://juejin.cn/post/6901552013717438472#heading-6) 中的 **代码质量** 部分去配置

* 替换官网 logo、渲染信息等

* element-ui 样式库的优化，其实 element-ui 的样式存在重复加载的问题

  > 虽然它通过 webpack 打包已经解决了一部分问题，但是某些情况还是会出现重复加载，比如 table 组件中使用 checkbox 组件，就会加载两次 checkbox 组件的样式代码。有精力的同学可以去研究研究

* 你的业务只需要 element-ui 的部分基础组件，把不需要的删掉，可以降低组件库的体积，提升加载速度

* ...

这些工作有一些是对官网项目（examples）的裁剪，有一些是项目整体优化，还有一些是洁癖，不过，相信凡是进行到这一步的同学，都已经为团队构建出了自己的组件库，解决以上列出的那些问题完全不再话下，这里就不一一列出方法了。

## 链接

* [Element 源码架构](https://www.processon.com/view/link/60372ab3f346fb2a7e38c0e8#map) 思维导图版
* [组件库专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2259813235891863559#wechat_redirect)
  * 如何快速为团队打造自己的组件库（上）—— Element 源码架构
  * Element 源码架构 视频版，关注微信公众号，回复: "Element 源码架构视频版" 获取



感谢各位的：**点赞**、**收藏**和**评论**，我们下期见。

---

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。