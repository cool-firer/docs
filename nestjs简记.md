从Java的Annotation，到Python的Decorator，再到现在的nestjs。

终究还是逃不过Decorator的魔爪。

简记一下，不深究nestjs内部过程了。



# Module装饰器

从main.ts开始

```typescript
import { NestFactory } from '@nestjs/core';

async function bootstrap() {
  const { AppModule } = await import('./app.module');
  const app = await NestFactory.create(AppModule, {
    logger: new NestLogger(),
    cors: true,
  });
  await app.listen(process.env.PORT ?? 3031);
}

void bootstrap();

```



app.module.ts：是个类装饰器。本质上是在 AppModule类上添加元信息

```typescript
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';
import { XxxController } from './controllers/xxx.controller';
import { XxxService } from './services/xxx.service';

// 类装饰器, 实际是作用于类的构造方法
// 可以推断出, Module({})是个func, 返回一个func
@Module({
  /**
   * forRoot() 返回 {
      global: true,
      module: ScheduleModule,
      providers: [ScheduleExplorer, SchedulerRegistry],
      exports: [SchedulerRegistry],
    };

    在App上定义 { 'imports' -> xx,
                 'controllers' -> xx,
                 ...
               }
   */

  imports: [ScheduleModule.forRoot()],
  controllers: [
    XxxController,
  ],
  providers: [
    XxxService,
  ],
})
export class AppModule {}
```



# Controller装饰器

也是个类装饰器，在类上添加元信息。

```typescript
/**
 * Controller装饰器: 只是在XxxController添加元信息. (跟Module装饰一样)
 * OrderController: { 有效的就只有两个
 *  '__controller__': true,
 *  'path': 'orders',
 * }
 */
@Controller('orders')
export class XxxController {
  constructor(
    private readonly xxxService: XxxService
  ) {}
}
```



# Controller get/post方法装饰器 + 参数装饰器

```typescript
@Controller('orders')
export class OrderController {
  constructor(
    private readonly xxxService: XxxService
  ) {}

  /**
   * 方法装饰器
   * 只是在 requestOrdersWithChainId 方法上添加元 { path: 'dictionary/:id', method: 'get' }
   */
  @Get('dictionary/:id')
  async requestOrdersWithChainId(
    /**
     * 参数装饰器 
     * 可以拿到类的原型对象(通过原型对象上的constructor可以拿到类本身), 方法名, 参数的索引
     * 1. 从OrderController类上的 requestOrdersWithChainId 方法取 __routeArguments__ 元.
     * 2. 在类上的方法上设置 __routeArguments__元 -> {
     *        'PARAM:0': { index:0, data:'id', pipes:[] },
     *        'QUERY:1': { index:1, data:'apiKey', pipes:[] },
     * }
     */
    @Param('id') id: string,
    @Query('apikey') apikey?: string
  ): Promise<OrdersResult> {}
}
```



# 总结

@Module、@Controllers、@Get、@Post、@param

都是在类上，类的方法上添加元。

估计后面就是从这些类上取元，再组织起来。