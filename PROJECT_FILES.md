# Nest 项目文件说明

这份文档按当前项目实际文件说明每个文件的作用。你是前端开发的话，可以先把 Nest 项目理解成“后端版的组件化应用”：`main.ts` 负责启动应用，`module` 负责组织功能，`controller` 负责接收请求，`service` 负责处理业务逻辑。

## 一、核心运行流程

当前项目访问 `GET /` 时，请求链路是：

```txt
浏览器 / HTTP 客户端
  -> src/main.ts 启动的 Nest 应用
  -> AppModule 注册的 AppController
  -> AppController.getHello()
  -> AppService.getHello()
  -> 返回 "Hello World!"
```

对应代码关系：

```txt
main.ts
  创建 AppModule

app.module.ts
  注册 AppController 和 AppService

app.controller.ts
  定义接口路由 GET /

app.service.ts
  提供真正返回数据的方法
```

## 二、目录说明

### `src/`

项目的主要源码目录，业务代码基本都写在这里。类似前端项目里的 `src` 目录。

### `test/`

端到端测试目录，用来测试整个 HTTP 接口是否按预期工作。它不是只测某个函数，而是启动一个接近真实的 Nest 应用后发请求测试。

### `node_modules/`

安装依赖后生成的目录，里面是 npm 下载的第三方包。这个目录不需要手动改，也不会提交到 Git。

### `.git/`

Git 仓库内部目录，保存提交历史、分支等版本控制数据。平时不要手动修改。

## 三、源码文件

### `src/main.ts`

应用入口文件，相当于前端项目里的 `main.ts` / `main.jsx`。

主要职责：

- 使用 `NestFactory.create(AppModule)` 创建 Nest 应用。
- 把根模块 `AppModule` 作为整个应用的起点。
- 使用 `app.listen(process.env.PORT ?? 3000)` 启动 HTTP 服务。
- 如果环境变量 `PORT` 存在，就使用它；否则默认监听 `3000` 端口。

运行 `npm run start:dev` 后，实际就是从这里启动服务。

### `src/app.module.ts`

根模块文件，是当前应用的组织入口。

Nest 使用 `@Module()` 装饰器声明模块：

- `imports`: 引入其他模块。当前为空，表示还没有拆分其他业务模块。
- `controllers`: 注册控制器。当前注册了 `AppController`。
- `providers`: 注册可注入的服务。当前注册了 `AppService`。

你可以把它理解成前端里的“应用装配层”：哪些页面、哪些服务、哪些功能要被应用使用，需要在这里或对应模块里声明。

### `src/app.controller.ts`

控制器文件，负责接收 HTTP 请求并返回响应。

当前代码里：

- `@Controller()` 表示这是一个控制器。
- `@Get()` 表示定义一个 `GET /` 接口。
- `constructor(private readonly appService: AppService)` 表示通过依赖注入拿到 `AppService`。
- `getHello()` 处理请求，并调用 `this.appService.getHello()` 获取返回值。

类比前端：

- Controller 像“页面事件入口”或“接口路由处理器”。
- 它不应该放太多复杂业务逻辑，复杂逻辑通常交给 Service。

### `src/app.service.ts`

服务文件，负责放业务逻辑。

当前代码里：

- `@Injectable()` 表示这个类可以被 Nest 的依赖注入系统管理。
- `getHello()` 返回字符串 `'Hello World!'`。

随着项目变大，Service 通常会负责：

- 查询数据库。
- 调用第三方接口。
- 处理业务规则。
- 组装返回给前端的数据。

### `src/app.controller.spec.ts`

单元测试文件，用来测试 `AppController` 的行为。

当前测试做了这些事：

- 使用 `Test.createTestingModule()` 创建一个测试模块。
- 注册 `AppController` 和 `AppService`。
- 获取 `AppController` 实例。
- 断言 `appController.getHello()` 返回 `'Hello World!'`。

它测试的是类方法本身，不是真正通过 HTTP 发请求。

## 四、端到端测试文件

### `test/app.e2e-spec.ts`

端到端测试文件，测试真实 HTTP 接口效果。

当前测试做了这些事：

- 创建包含 `AppModule` 的测试应用。
- 初始化 Nest 应用。
- 使用 `supertest` 发送 `GET /` 请求。
- 断言响应状态码是 `200`。
- 断言响应内容是 `'Hello World!'`。
- 测试结束后关闭应用。

它比 `src/app.controller.spec.ts` 更接近真实用户访问接口的过程。

### `test/jest-e2e.json`

端到端测试的 Jest 配置文件。

主要作用：

- 指定测试环境是 Node.js。
- 指定匹配 `.e2e-spec.ts` 结尾的测试文件。
- 使用 `ts-jest` 让 Jest 可以运行 TypeScript 测试代码。

运行下面命令时会用到它：

```bash
npm run test:e2e
```

## 五、项目配置文件

### `package.json`

npm 项目配置文件，前端项目也有同类文件。

主要包含：

- `scripts`: 常用命令。
- `dependencies`: 生产运行依赖。
- `devDependencies`: 开发、构建、测试依赖。
- `jest`: 单元测试配置。

常用脚本：

```bash
npm run start       # 启动项目
npm run start:dev   # 开发模式启动，文件变化会自动重启
npm run build       # 编译项目到 dist
npm run test        # 运行单元测试
npm run test:e2e    # 运行端到端测试
npm run lint        # 运行 ESLint 并自动修复
npm run format      # 使用 Prettier 格式化代码
```

### `package-lock.json`

npm 依赖锁定文件。

作用是记录当前安装的每个依赖的精确版本，保证不同电脑、不同时间安装出来的依赖尽量一致。这个文件通常应该提交到 Git。

### `tsconfig.json`

TypeScript 编译配置文件。

当前项目里比较重要的配置：

- `target: "ES2023"`: 编译目标是 ES2023。
- `module: "nodenext"`: 使用 Node.js 新版模块解析方式。
- `outDir: "./dist"`: 编译后的文件输出到 `dist`。
- `experimentalDecorators: true`: 允许使用装饰器，比如 `@Controller()`、`@Get()`。
- `emitDecoratorMetadata: true`: 生成装饰器元数据，Nest 依赖注入会用到。
- `strictNullChecks: true`: 开启严格空值检查。

Nest 大量使用装饰器，所以 `experimentalDecorators` 和 `emitDecoratorMetadata` 很关键。

### `tsconfig.build.json`

构建专用的 TypeScript 配置。

它继承 `tsconfig.json`，但排除了：

- `node_modules`
- `test`
- `dist`
- `**/*spec.ts`

也就是说，执行 `npm run build` 时，只编译真正的应用代码，不编译测试文件。

### `nest-cli.json`

Nest CLI 配置文件。

主要作用：

- 指定源码根目录是 `src`。
- 指定使用 `@nestjs/schematics` 生成代码。
- `deleteOutDir: true` 表示构建前清空旧的输出目录。

以后如果使用 Nest CLI 生成模块、控制器、服务，例如：

```bash
nest g module users
nest g controller users
nest g service users
```

CLI 会参考这个配置。

### `eslint.config.mjs`

ESLint 配置文件，用来检查代码质量和风格问题。

当前配置做了这些事：

- 启用 JavaScript 推荐规则。
- 启用 TypeScript ESLint 推荐规则。
- 启用 Prettier 集成。
- 设置 Node.js 和 Jest 全局变量。
- 关闭 `@typescript-eslint/no-explicit-any`。
- 对未处理的 Promise、非安全参数等问题给出警告。
- 设置 Prettier 的换行符为自动识别。

运行下面命令会使用它：

```bash
npm run lint
```

### `.prettierrc`

Prettier 格式化配置。

当前规则：

- `singleQuote: true`: 使用单引号。
- `trailingComma: "all"`: 多行对象、数组、参数等保留尾随逗号。

运行下面命令会使用它：

```bash
npm run format
```

### `.gitignore`

Git 忽略配置文件。

里面声明了哪些文件或目录不需要提交，例如：

- `node_modules`
- `dist`
- `coverage`
- 日志文件
- 临时文件
- `.env` 环境变量文件
- 编辑器生成的本地配置

这可以避免把依赖、编译产物、敏感配置提交到仓库。

### `README.md`

项目说明文档。

当前文件是 Nest CLI 生成的默认 README，包含：

- 项目介绍。
- 安装依赖命令。
- 启动项目命令。
- 测试命令。
- Nest 官方资源链接。

后续项目正式开发时，可以把它改成你自己项目的接口说明、启动方式、环境变量说明等。

## 六、几个 Nest 核心概念

### Module

模块，用来组织代码。一个项目通常会按业务拆成多个模块，例如：

```txt
users module
orders module
auth module
```

### Controller

控制器，用来定义接口路由。它接收请求、读取参数、调用服务、返回结果。

### Service

服务，用来放业务逻辑。Controller 通常不直接处理复杂逻辑，而是调用 Service。

### Provider

可被 Nest 依赖注入系统管理的对象。Service 是最常见的 Provider。

### Decorator

装饰器，例如：

```ts
@Controller()
@Get()
@Injectable()
@Module()
```

Nest 用装饰器给类和方法增加框架需要的元信息。

## 七、从前端角度理解当前项目

可以这样类比：

```txt
前端项目 main.ts
  -> 挂载 Vue / React 应用

Nest 项目 main.ts
  -> 启动后端 HTTP 服务
```

```txt
前端路由
  -> 根据 URL 渲染页面

Nest Controller
  -> 根据 URL 执行接口处理函数
```

```txt
前端 service/api 文件
  -> 封装请求或业务逻辑

Nest Service
  -> 封装后端业务逻辑、数据库操作、第三方接口调用
```

## 八、建议的学习顺序

1. 先看 `src/main.ts`，知道项目怎么启动。
2. 再看 `src/app.module.ts`，知道 Nest 如何注册 Controller 和 Service。
3. 再看 `src/app.controller.ts`，知道接口路由怎么写。
4. 再看 `src/app.service.ts`，知道业务逻辑放在哪里。
5. 最后看测试和配置文件，理解项目工程化部分。
