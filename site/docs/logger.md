# 日志

## 简介

Midway 为不同场景提供了一套统一的日志接入方式。通过 `@midwayjs/logger` 包导出的方法，可以方便的接入不同场景的日志系统。

Midway 的日志系统基于社区的 [winston](https://github.com/winstonjs/winston)，是现在社区非常受欢迎的日志库。

实现的功能：

- 日志分级
- 按大小和时间自动切割
- 自定义输出格式
- 统一错误日志



## 日志路径和文件

Midway 会在日志根目录创建一些默认的文件。


- `midway-core.log` 框架、组件打印信息的日志，对应 `coreLogger` 。
- `midway-app.log` 应用打印信息的日志，对应 `appLogger`
- `common-error.log` 所有错误的日志（所有 Midway 创建出来的日志，都会将错误重复打印一份到该文件中）



## 默认日志对象

Midway 默认在框架提供了三种不同的日志，对应三种不同的行为。

| 日志                                | 释义                 | 描述                                                         | 常见使用                                                     |
| ----------------------------------- | -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| coreLogger                          | 框架，组件层面的日志 | 默认会输出控制台日志和文本日志 `midway-core.log` ，并且默认会将错误日志发送到 `common-error.log` 。 | 框架和组件的错误，一般会打印到其中。                         |
| appLogger                           | 业务层面的日志       | 默认会输出控制台日志和文本日志 `midway-app.log` ，并且默认会将错误日志发送到 `common-error.log` 。 | 业务使用的日志，一般业务日志会打印到其中。                   |
| 上下文日志（复用 appLogger 的配置） | 请求链路的日志       | 默认使用 `appLogger` 进行输出，除了会将错误日志发送到 `common-error.log` 之外，还增加了上下文信息。 | 修改日志输出的标记（Label），不同的框架有不同的请求标记，比如 HTTP 下就会输出路由信息。 |



## 使用日志

Midway 的常用日志使用方法。

### 上下文日志

```typescript
import { Get, Inject, Controller, Provide } from '@midwayjs/decorator';
import { ILogger } from '@midwayjs/logger';

@Controller()
export class HelloController {

  @Inject()
  logger: ILogger;

  @Inject()
  ctx;

  @Get("/")
  async hello(){
    this.logger.info("hello world");
    this.logger.debug('debug info');
    this.logger.warn('WARNNING!!!!');

    // 错误日志记录，直接会将错误日志完整堆栈信息记录下来，并且输出到 errorLog 中
    // 为了保证异常可追踪，必须保证所有抛出的异常都是 Error 类型，因为只有 Error 类型才会带上堆栈信息，定位到问题。
    this.logger.error(new Error('custom error'));
    // ...

    // this.logger === ctx.logger
  }
}
```
访问后，我们能在两个地方看到日志输出：


- 控制台看到输出。
- 日志目录的 midway-app.log 文件中


输出结果：
```text
2021-07-22 14:50:59,388 INFO 7739 [-/::ffff:127.0.0.1/-/0ms GET /api/get_user] hello world
```



### 应用日志（App Logger）

如果我们想做一些应用级别的日志记录，如记录启动阶段的一些数据信息，可以通过 App Logger 来完成。

```typescript
import { Configuration, Logger } from '@midwayjs/decorator';
import { ILogger } from '@midwayjs/logger';

@Configuration()
export class ContainerConfiguration implements ILifeCycle {

  @Logger()
  logger: ILogger;

  async onReady(container: IMidwayContainer): Promise<void> {
    this.logger.debug('debug info');
    this.logger.info('启动耗时 %d ms', Date.now() - start);
    this.logger.warn('warning!');

    this.logger.error(someErrorObj);
  }

}
```

注意，这里使用的是 `@Logger()` 装饰器。



### CoreLogger

在组件或者框架层面的研发中，我们会使用 coreLogger 来记录日志。

```typescript

@Configuration()
export class ContainerConfiguration implements ILifeCycle {

  @Logger()
  logger: ILogger;

  async onReady(container: IMidwayContainer): Promise<void> {
    this.logger.debug('debug info');
    this.logger.info('启动耗时 %d ms', Date.now() - start);
    this.logger.warn('warning!');

    this.logger.error(someErrorObj);
  }

}
```






## 输出方法和格式


Midway 的日志对象继承与 winston 的日志对象，一般情况下，只提供 `error()` ， `warn()` ， `info()` , `debug` 四种方法。


示例如下。
```typescript
logger.debug('debug info');
logger.info('启动耗时 %d ms', Date.now() - start);
logger.warn('warning!');
logger.error(new Error('my error'));
```


### 默认的输出行为


在大部分的普通类型下，日志库都能工作的很好。


比如：
```typescript
logger.info('hello world');																					// 输出字符串
logger.info(123);																										// 输出数字
logger.info(['b', 'c']);																						// 输出数组
logger.info(new Set([2, 3, 4]));																		// 输出 Set
logger.info(new Map([['key1', 'value1'], ['key2', 'value2']]));			// 输出 Map
```
> Midway 针对 winston 无法输出的 `Array` ， `Set` ， `Map` 类型，做了特殊定制，使其也能够正常的输出。


不过需要注意的是，日志对象在一般情况下，只能传入一个参数，它的第二个参数有其他作用。
```typescript
logger.info('plain error message', 321);			// 会忽略 321
```


### 错误输出


针对错误对象，Midway 也对 winston 做了定制，使其能够方便的和普通文本结合到一起输出。
```typescript
// 输出错误对象
logger.error(new Error('error instance'));

// 输出自定义的错误对象
const error = new Error('named error instance');
error.name = 'NamedError';
logger.error(error);

// 文本在前，加上 error 实例
logger.info('text before error', new Error('error instance after text'));
```
:::caution
注意，错误对象只能放在最后，且有且只有一个，其后面的所有参数都会被忽略。
:::




### 格式化内容
基于 `util.format` 的格式化方式。
```typescript
logger.info('%s %d', 'aaa', 222);
```
常用的有


- `%s` 字符串占位
- `%d` 数字占位
- `%j` json 占位

更多的占位和详细信息，请参考 node.js 的 [util.format](https://nodejs.org/dist/latest-v14.x/docs/api/util.html#util_util_format_format_args) 方法。



### 输出自定义对象或者复杂类型


基于性能考虑，Midway（winston）大部分时间只会输出基本类型，所以当输出的参数为高级对象时，**需要用户手动转换为需要打印的字符串**。


如下示例，将不会得到希望的结果。
```typescript
const obj = {a: 1};
logger.info(obj);					// 默认情况下，输出 [object Object]
```
需要手动输出希望打印的内容。
```typescript
const obj = {a: 1};
logger.info(JSON.stringify(obj));				// 可以输出格式化文本
logger.info(obj.a);												// 直接输出属性值
logger.info('%j', a);										// 直接占位符输出整个 json
```



### 纯输出内容


特殊场景下，我们需要单纯的输出内容，不希望输出时间戳，label 等和格式相关的信息。这种需求我们可以使用 `write` 方法。

`write` 方法是个非常底层的方法，并且不管什么级别的日志，它都会写入到文件中。


虽然 `write` 方法在每个 logger 上都有，但是我们只在 `IMidwayLogger` 定义中提供它，我们希望你能明确的知道自己希望调用它。
```typescript
(logger as IMidwayLogger).write('hello world');		// 文件中只会有 hello world
```



## 日志类型定义


默认的情况，用户应该使用最简单的 `ILogger` 定义。
```typescript
import { Provide, Logger } from '@midwayjs/decorator';
import { ILogger } from '@midwayjs/logger';

@Provide()
export class UserService {

  @Inject()
  logger: ILogger;						// 获取上下文日志

  async getUser() {
  	this.logger.info('hello user');
  }

}
```


`ILogger` 定义只提供最简单的 `debug` ， `info` ， `warn` 以及 `error` 方法。


在某些场景下，我们需要更为复杂的定义，比如修改日志属性或者动态调节，这个时候需要使用更为复杂的 `IMidwayLogger` 定义。


```typescript
import { Provide, Logger } from '@midwayjs/decorator';
import { IMidwayLogger } from '@midwayjs/logger';

@Provide()
export class UserService {

  @Inject()
  logger: IMidwayLogger;						// 获取上下文日志

  async getUser() {
    this.logger.disableConsole();		// 禁止控制台输出
  	this.logger.info('hello user');	// 这句话在控制台看不到
    this.logger.enableConsole();		// 开启控制台输出
    this.logger.info('hello user');	// 这句话在控制台可以看到
  }

}
```
`IMidwayLogger`  的定义可以参考 interface 中的描述，或者查看 [代码](https://github.com/midwayjs/logger/blob/main/src/interface.ts)。



## 日志配置

我们可以在配置文件中配置日志的各种行为。

Midway 中的的日志配置包含 **全局配置** 和 **单个日志配置** 两个部分，两者配置会合并和覆盖。

```typescript
// src/config/config.default.ts
import { MidwayConfig } from '@midwayjs/core';

export default {
  midwayLogger: {
    defaulit: {
      // ...
    },
    clients: {
      coreLogger: {
        // ...
      },
      appLogger: {
        // ...
      }
    }
  },
} as MidwayConfig;
```

如上所述，`clients` 配置段中的每个对象都是一个独立的日志配置项，其配置会和 `default` 段落合并后创建 logger 实例。



## 日志等级


winston 的日志等级分为下面几类，日志等级依次降低（数字越大，等级越低）：
```typescript
const levels = {
  none: 0,
  error: 1,
  trace: 2,
  warn: 3,
  info: 4,
  verbose: 5,
  debug: 6,
  silly: 7,
  all: 8,
}
```
在 Midway 中，为了简化，一般情况下，我们只会使用 `error` ， `warn` ， `info` ， `debug` 这四种等级。


日志等级表示当前可输出日志的最低等级。比如当你的日志 level 设置为 `warn`  时，仅 `warn` 以及更高的 `error` 等级的日志能被输出。



### 框架的默认等级


在 Midway 中，有着自己的默认日志等级。


- 在开发环境下（local，test，unittest），日志等级统一为 `info` 。
- 在服务器环境（除开发环境外），为减少日志数量，日志等级统一为 `warn` 。



### 调整日志等级

一般情况下，我们不建议调整全局默认的日志等级，而是调整特定的 logger 的日志等级，比如：

调整 `coreLogger` 或者 `appLogger` 。

```typescript
// src/config/config.default.ts
import { MidwayConfig } from '@midwayjs/core';

export default {
  midwayLogger: {
    clients: {
      coreLogger: {
        level: 'warn',
        consoleLevel: 'warn'
        // ...
      },
      appLogger: {
        level: 'warn',
        consoleLevel: 'warn'
        // ...
      }
    }
  },
} as MidwayConfig;
```

特殊场景，也可以临时调整全局的日志等级。

```typescript
// src/config/config.default.ts
import { MidwayConfig } from '@midwayjs/core';

export default {
  midwayLogger: {
    default: {
      level: 'info',
      consoleLevel: 'warn'
    },
    // ...
  },
} as MidwayConfig;
```



## 日志根目录

默认情况下，Midway 会在本地开发和服务器部署时输出日志到 **日志根目录**。


- 本地的日志根目录为 `${app.appDir}/logs/项目名` 目录下
- 服务器的日志根目录为用户目录 `${process.env.HOME}/logs/项目名` （Linux/Mac）以及 `${process.env.USERPROFILE}/logs/项目名` （Windows）下，例如 `/home/admin/logs/example-app`。

我们可以配置日志所在的根目录。

```typescript
// src/config/config.default.ts
import { MidwayConfig } from '@midwayjs/core';

export default {
  midwayLogger: {
    default: {
      dir: '/home/admin/logs',
    },
    // ...
  },
} as MidwayConfig;
```



## 日志切割（轮转）


默认行为下，同一个日志对象 **会生成两个文件**。

以 `midway-core.log` 为例，应用启动时会生成一个带当日时间戳 `midway-core.YYYY-MM-DD` 格式的文件，以及一个不带时间戳的 `midway-core.log` 的软链文件。

> windows 下不会生成软链


为方便配置日志采集和查看，该软链文件永远指向最新的日志文件。


当凌晨 `00:00` 时，会生成一个以当天日志结尾 `midway-core.log.YYYY-MM-DD` 的形式的新文件。

同时，当单个日志文件超过 200M 时，也会自动切割，产生新的日志文件。

可以通过配置调整按大小的切割行为。

```typescript
export default {
  midwayLogger: {
    default: {
      maxSize: '100m',
    },
    // ...
  },
} as MidwayConfig;
```



## 日志清理

默认情况下，日志会存在 31 天。

可以通过配置调整该行为，比如改为保存 3 天。

```typescript
export default {
  midwayLogger: {
    default: {
      maxFiles: '3d',
    },
    // ...
  },
} as MidwayConfig;
```



## 日志输出格式


显示格式指的是日志输出时单行文本的字符串结构。Miidway 对 Winston 的日志做了定制，提供了一些默认对象。

显示格式是一个返回字符串结构的方法，参数为 Winston 的 [info 对象](https://github.com/winstonjs/logform#info-objects)。

一般情况下，我们只会自定义单个日志的输出格式，比如 `appLogger`。

```typescript
export default {
  midwayLogger: {
    clients: {
      appLogger: {
        format: info => {
          return `${info.timestamp} ${info.LEVEL} ${info.pid} ${info.labelText}${info.message}`;
        }
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```

info 对象的默认属性如下：

| **属性名**  | **描述**                                         | **示例**                                                     |
| ----------- | ------------------------------------------------ | ------------------------------------------------------------ |
| timestamp   | 时间戳，默认为 `'YYYY-MM-DD HH:mm:ss,SSS` 格式。 | 2020-12-30 07:50:10,453                                      |
| level       | 小写的日志等级                                   | info                                                         |
| LEVEL       | 大写的日志等级                                   | INFO                                                         |
| pid         | 当前进程 pid                                     | 3847                                                         |
| labelText   | 标签的聚合文本                                   | [abcde]                                                      |
| message     | 普通消息 + 错误消息 + 错误堆栈的组合             | 1、普通文本，如 `123456` ， `hello world`<br />2、错误文本（错误名+堆栈）Error: another test error at Object.anonymous (/home/runner/work/midway/midway/packages/logger/test/index.test.ts:224:18)<br />3、普通文本+错误文本 hello world Error: another test error at Object.anonymous (/home/runner/work/midway/midway/packages/logger/test/index.test.ts:224:18) |
| stack       | 错误堆栈                                         |                                                              |
| originError | 原始错误对象                                     | 错误实例本身                                                 |
| originArgs  | 原始的用户入参                                   | [ 'a', 'b', 'c' ]                                            |




## 修改上下文日志输出格式

上下文日志是基于 appLogger 来打日志的，**会复用 appLogger 的所有信息**。我们可以单独对其进行输出格式的配置。

我们依旧以修改 `appLogger` 的上下文日志为例。

```typescript
export default {
  midwayLogger: {
    clients: {
      appLogger: {
        contextFormat: info => {
          const ctx = info.ctx;
          return `${info.timestamp} ${info.LEVEL} ${info.pid} [${Date.now() - ctx.startTime}ms ${ctx.method}] ${info.message}`;
        }
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```

则你在使用 `ctx.logger` 输出时，会默认变成你 format 的样子。

```typescript
ctx.logger.info('hello world');
// 2021-01-28 11:10:19,334 INFO 9223 [2ms POST] hello world
```

:::tip
注意，由于 app logger 是所有框架通用的，这一修改会影响所有类型的 context logger。
:::

未避免影响过大，你也可以单独修改某一框架的上下文日志格式，比如 [修改 koa 的上下文日志格式](./extensions/koa)。


## 自定义日志


如果用户不满足于默认的日志对象，也可以自行创建。



### 增加自定义日志

可以如下配置：

```typescript
export default {
  midwayLogger: {
    clients: {
      abcLogger: {
        fileLogName: 'abc.log'
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```

自定义的日志可以通过 `@Logger('abcLogger')` 获取。

更多的日志选项可以参考 interface 中 [LoggerOptions 描述](https://github.com/midwayjs/logger/blob/main/src/interface.ts)。



### 日志默认 Transport

每个日志包含几个默认的 Transport。

| 名称              | 默认行为 | 描述                           |
| ----------------- | -------- | ------------------------------ |
| Console Transport | 开启     | 用于输出到控制台               |
| File Transport    | 开启     | 用于输出到文本文件             |
| Error Transport   | 开启     | 用于将错误输出到特定的错误日志 |
| JSON Transport    | 关闭     | 用于输出 JSON 格式的文本       |

可以通过配置进行修改。

**示例：只开启控制台输出**

```typescript
export default {
  midwayLogger: {
    clients: {
      abcLogger: {
        enableFile: false,
        enableError: false,
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```

**示例：关闭控制台输出**

```typescript
export default {
  midwayLogger: {
    clients: {
      abcLogger: {
        enableConsole: false,
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```

**示例：开启文本和 JSON 同步输出，关闭错误输出**

```typescript
export default {
  midwayLogger: {
    clients: {
      abcLogger: {
        enableConsole: false,
        enableFile: true,
        enableError: false,
        enableJSON: true,
        // ...
      }
    }
    // ...
  },
} as MidwayConfig;
```



### 自定义 Transport

框架提供了扩展 Transport 的功能，比如，你可以写一个 Transport 来做日志的中转，上传到别的日志库等能力。

比如下面的示例，我们就将日志中转到另一个本地文件中。

```typescript
import { EmptyTransport } from '@midwayjs/logger';

class CustomTransport extends EmptyTransport {
  log(info, callback) {
    const levelLowerCase = info.level;
    if (levelLowerCase === 'error' || levelLowerCase === 'warn') {
      writeFileSync(join(logsDir, 'test.log'), info.message);
    }
    callback();
  }
}
```

我们可以初始化，加到 logger 中，也可以单独对 Transport 设置 level。

```typescript
const customTransport = new CustomTransport({
  level: 'warn',
});

logger.add(customTransport);
```

这样，原有的 logger 打印日志时，会自动执行该 Transport。

所有的 Transport 是附加在原有的 logger 实例之上（非 context logger），如需 ctx 数据，可以从 info 获取，注意判空。


```typescript
class CustomTransport extends EmptyTransport {
  log(info, callback) {
    if (info.ctx) {
      // ...
    } else {
      // ...
    }
    callback();
  }
}
```


我们也可以使用依赖注入的方式来定义 Transport。

```typescript
import { EmptyTransport } from '@midwayjs/logger';
import { Provide, Scope, ScopeEnum } from '@midwayjs/decorator';
import { MidwayLoggerService } from '@midwayjs/core';

@Provide()
@Scope(ScopeEnum)
export class CustomTransport extends EmptyTransport {
  log(info, callback) {
    // ...
    callback();
  }
}

// src/configuration.ts
@Configuration(/*...*/)
export class AutoConfiguration {

  @Inject()
  loggerService: MidwayLoggerService;

  @Inject()
  customTransport: CustomTransport;

  async onReady() {
    const appLogger = this.loggerService.getLogger('customLogger');
    appLogger.add(this.customTransport);
  }
}
```