# vue3-ts-vite-temp

👋 这是一个基于 [Vue3](https://vuejs.org/)、[TypeScript](https://www.typescriptlang.org/)、[Vite](https://vitejs.dev/) 的模板，对一些常用插件和库进行了集成。

## 规范化插件

-  [prettier](https://prettier.io/)：集成 prettier 格式化代码。

- [eslint](https://eslint.org/docs/latest/)：集成 eslint 规范 Vue、TypeScript 、JavaScript 代码。

- [stylelint](https://stylelint.bootcss.com/)：集成 stylelint 规范 CSS 代码。

- [commitizen](http://commitizen.github.io/cz-cli/) 和 [commitlint](https://commitlint.js.org/#/)：集成 commitizen 和 commitlint 规范 Git 提交消息的格式。

- [lint-staged](https://github.com/okonet/lint-staged)：集成 lint-staged 对暂存区的代码进行检查。

- [husky](https://typicode.github.io/husky/#/)：集成 husky 添加 [git hook](https://git-scm.com/docs/githooks)，对提交进行增强（运行 test、lint 等）。

### 集成 prettier

1. 安装 prettier 模块：
```bash
$ pnpm install --save-dev prettier 
```

2. 创建 prettier 配置文件 *prettier.config.cjs* ：
```js
module.exports = {
  // 语句末尾添加分号
  semi: true,
  // 每个缩进的空格数
  tabWidth: 2,
  // 是否使用制表符代替空格缩进
  useTabs: false,
  // 单行显示的字符个数
  printWidth: 80,
  // 是否使用单引号而不是双引号
  singleQuote: true,
  // jsx 是否使用单引号而不是双引号
  jsxSingleQuote: false,
  // 多行数据结构添加尾随逗号，如数组，对象，
  trailingComma: "es5",
  // 是否在大括号内的首尾添加空格，'{ foo: bar }'
  bracketSpacing: true,
  // 换行符，‘auto’ 表示保持现状，window 为 crlf，linux/mac 为 lf
  // 'auto' 当团队成员使用不同设备开发时，会因此出现换行符跨平台开发问题
  endOfLine: "auto",
  // 是否将多行元素(HTML、JSX、Vue、Angular)的‘>’单独放在下一行，而不是放在行尾
  bracketSameLine: false,
  // 总是为单个的箭头函数参数添加括号，'(x) => x'
  arrowParens: "always",
  // vue 文件中的 `<script>` 和 `<style>` 标签的内容是否缩进
  vueIndentScriptAndStyle: true,
  // 是否强制每个元素或组件的属性单独一行
  singleAttributePerLine: false,
  // 设置元素或组件的空格敏感性为严格 
  // https://prettier.io/docs/en/options.html#html-whitespace-sensitivity
  htmlWhitespaceSensitivity: "strict",
};
```

3. 创建忽略文件 *.prettierignore*，可以在这里配置我们不想让 prettier 格式化的目录和文件：
```.ignore
/dist/*
.local
/node_modules/**
  
**/*.svg
**/*.sh

/public/*
```

### 集成 eslint

1. 安装 eslint 相关模块：
```bash
# eslint 整合 vue 需要的模块
$ pnpm install --save-dev eslint eslint-plugin-vue
# eslint 整合 ts 需要的模块
$ pnpm install --save-dev @typescript-eslint/parser @typescript-eslint/eslint-plugin 
# 解决 eslint 和 prettier 规则冲突需要的模块 
$ pnpm install --save-dev eslint-config-prettier eslint-plugin-prettier
# 支持 eslint 配置类型提示
$ pnpm install --save-dev eslint-define-config 
```
