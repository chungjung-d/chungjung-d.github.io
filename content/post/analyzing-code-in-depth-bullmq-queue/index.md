---
title: "[BullMQ] Queue 뜯어보기"
description: BullMQ Queue 코드 뜯어보기
slug: bullmq-queue-analyzing-code-in-depth
date: 2024-10-03 00:00:00+0000
image: cover.png
categories:
    - BullMQ
tags:
    - Analyzing code in depth
weight: 1
---

## Queue란?

BullMQ 에서 Queue는 **A Queue is nothing more than a list of jobs waiting to be processes** 라고 공식문서에서 설명한다. 그러나 완전히 그런 대기열 역할만 하는 것은 아니고, 대기열에 있는 현재 Job들의 상태나, 특정 Job이 등록되었을때 Job의 상태변화에 따라서 event를 제공하고 Scheduler를 이용해서 동일한 Job을 생성하는 등의 여러가지 역할을 수행하기는 한다.

이번 글에서는 공식문서에서는 나와있지 않은 BullMQ의 Queue Class가 내부적으로 어떻게 구현되었는지 간단하게 코드를 뜯어보면서 알아보려고 한다. 
## 상속관계 살펴보기

가정 먼저 해야할 일은 Queue Class의 상속관계를 살펴보는 것이다.

```ts
export class Queue<
  DataType = any,
  ResultType = any,
  NameType extends string = string,
> extends QueueGetters<DataType, ResultType, NameType> 

export class QueueGetters<
  DataType,
  ResultType,
  NameType extends string,
> extends QueueBase 

export class QueueBase extends EventEmitter implements MinimalQueue
```

궁극적으로 Queue Class는 QueueBase에 의존한다는 것을 확인할 수 있다. 그리고 QueueBase Class가 상속하는 `EventEmitter` , `MinimalQueue` 는 다음과 같은 선언부를 가진다.

```ts
class EventEmitter extends NodeJS.EventEmitter

export type MinimalQueue = Pick<
  QueueBase, ... >
```

`MinimalQueue` 는 타입을 조금 더 명확하게 사용하기 위해서 사용한 것이기 때문에 사실상 무시해도 좋다. 그렇다면 중요한건 **NodeJS의 EventEmitter를 상속받는 사실만 기억해두면 좋을 것 같다.**

## QueueBase

지금까지의 상속관계를 대충 파악한 것으로 봤을 때 QueueBase가 Queue를 이루는 기본적인 역할을 한다는 사실을 어렴풋이 알 수 있다. 그러면 한번 QueueBase를 살펴보도록 하자.

**먼저 Constructor를 살펴보자**

```ts
  constructor(
    public readonly name: string,
    public opts: QueueBaseOptions = { connection: {} },
    Connection: typeof RedisConnection = RedisConnection,
  ) {
  
    // --- (1)
    super();

    this.opts = {
      prefix: 'bull',
      ...opts,
    };

    if (!name) {
      throw new Error('Queue name must be provided');
    }

    if (name.includes(':')) {
      throw new Error('Queue name cannot contain :');
    }

    this.connection = new Connection(        
      opts.connection,
      isRedisInstance(opts.connection),
      opts.blockingConnection,
      opts.skipVersionCheck,
    );

		// --- (2)  
    this.connection.on('error', (error: Error) => this.emit('error', error));
    this.connection.on('close', () => {
      if (!this.closing) {
        this.emit('ioredis:close');
      }
    });

		// --- (3)
    const queueKeys = new QueueKeys(opts.prefix);
    this.qualifiedName = queueKeys.getQueueQualifiedName(name);
    this.keys = queueKeys.getKeys(name);
    this.toKey = (type: string) => queueKeys.toKey(name, type);
    this.setScripts();
  }
```

(1)을 보면 필수 값으로 name 그리고 redis Connection 정보를 받아온다는 사실을 알 수 있다. 공식문서에서 Redis Connection은 ioredis 라이브러리에 의존하고 있다고 밝힌만큼, **QueueBase는 ioredis로 생성한 connection정보를 객체의 property로 저장한다는 점을 알 수 있다.**

(2)는 아래에서 설명하고, 바로 (3)으로 넘어가보자. `QueueKeys` 와 `qualifiedName` 는 Queue의 상태에 대한 key들을 담고 있는 속성이다. 나머지도 이런식으로 기본적인 설정을 담당한다고 생각하면 된다.

**그 다음으로 봐야할 점은 emit 메서드이다.**

```ts
emit(event: string | symbol, ...args: any[]): boolean {
  try {
    return super.emit(event, ...args);  // --- (1) 일단 Eventemitter Emit 이용
  } catch (err) {
    try {
      return super.emit('error', err);  // --- (2) 실패시 에러를 Eventemitter Emit 
    } catch (err) {
      // --- (3) 그래도 안되면 포기하고 그냥 콘솔에 에러 찍어버림
      console.error(err);
      return false;
    }
  }
}
```

하는 역할은 매우 간단한데, emit을 호출하면 super.emit()을 호출한다. super객체는 NodeJS.EventEmitter기 때문에 nodeJS의 EventEmitter API를 이용한다고 보면 된다.

다시 QueueBase의 Constructor의 (2)로 돌아가면 이 emit 메서드가 Redis Connection을 한번 감싸는 용도로 사용되는 것을 확일할 수 있다.

그 외에도 `close()`, `disconnect()`, `checkConnectionError<T>()` 등의 메서드들도 있지만 생략하겠다.

결과적으로 QueueBase객체는 “**Redis 연결 관리, 상태를 조회할수 있는 이벤트를 관리 그리고 기본 설정값을 저장하는 객체**” 로 생각하면 될 듯 하다.

## QueueGetters

QueueBase가 기본적인 Redis Connection 및 그 외적으로 기본적으로 들어가야 하는 값들을 설정하는 객체였다면, Queue Getters는 본격적으로 Queue와 함께 Bullmq를 이루는 핵심요소인 Worker, Job, Listener 정보를 가져오는 역할을 담당한다고 볼 수 있다.

여기서 정보를 가져온다고 말한 이유는 BullMQ는 Queue 인스턴스 내부에 Worker나 Job의 정보를 저장하지 않기 때문이다. 다음은 QueueGetters에 있는 메서드들이다.

**Job Get Methods**
```ts
getJob(
  jobId: string,
): Promise<Job<DataType, ResultType, NameType> | undefined> {
  return this.Job.fromId(this, jobId) as Promise<
    Job<DataType, ResultType, NameType>
  >;
}
  
 
static async fromId<T = any, R = any, N extends string = string>(
  queue: MinimalQueue,
  jobId: string,
): Promise<Job<T, R, N> | undefined> {
  // jobId can be undefined if moveJob returns undefined
  if (jobId) {
    const client = await queue.client;
    const jobData = await client.hgetall(queue.toKey(jobId));
    return isEmpty(jobData)
      ? undefined
      : this.fromJSON<T, R, N>(
          queue,
          (<unknown>jobData) as JobJsonRaw,
          jobId,
        );
 }
```



**Worker Get Methods**
```ts
  getWorkers(): Promise<
    {
      [index: string]: string;
    }[]
  > {
    const unnamedWorkerClientName = `${this.clientName()}`;
    const namedWorkerClientName = `${this.clientName()}:w:`;

    const matcher = (name: string) =>
      name &&
      (name === unnamedWorkerClientName ||
        name.startsWith(namedWorkerClientName));

    return this.baseGetClients(matcher);
  }
  
private async baseGetClients(matcher: (name: string) => boolean): Promise<
    {
      [index: string]: string;
    }[]
  > {
    const client = await this.client;
    try {
      const clients = (await client.client('LIST')) as string;
      const list = this.parseClientList(clients, matcher);
      return list;
    } catch (err) {
      if (!clientCommandMessageReg.test((<Error>err).message)) {
        throw err;
      }

      return [{ name: 'GCP does not support client list' }];
    }
  }
```

보면 알 수 있듯이, Job같은 경우 Job의 static method에서 Queue의 client를 이용해서 각 Queue에 속한  Jobs를 Redis에서 불러오고, Workers 같은 경우는 baseGetCleints method에서 worker정보를 redis에서 조회해서 불러오는 것으로 확인할 수 있다.

한 가지 알아두면 좋은점은 Worker를 Job처럼 관리하지 않는 이유는 Worker자체는 Queue처럼 Redis client를 직접 가지고 있는 객체라서 그렇다. 굳이 Worker에 redis client가 필요한 이유라면 그 둘은 독립적으로 작동하기 때문이라고 설명할 수 있을 것 같다. **간단히 비유하면 Redis Pub/Sub에서 Queue가 Pub역할을 맡고, Wokrer가 Sub역할을 맡고 있다고 생각하면 된다.**

그 외에도 `getMetrics()` , `getQueueEvents()` 같이 Metric 정보다 QueueEvent 객체에 대한 정보를 불러올수 있는 method들이 이 객체에 존재한다. (getQueueEvents는 depricated 되었다.)

## Queue

이제 직접적인 Queue 구현체를 살펴보려고 한다. 우선 Queue 구현체를 살펴보기 전에 앞서서 먼저 같이 봐야할 객체들이 더 있다.

### QueueListener

```ts
export interface QueueListener<DataType, ResultType, NameType extends string>
  extends IoredisListener {
  cleaned: (jobs: string[], type: string) => void;
  error: (err: Error) => void;
  paused: () => void;
  progress: (
    job: Job<DataType, ResultType, NameType>,
    progress: number | object,
  ) => void;
  removed: (job: Job<DataType, ResultType, NameType>) => void;
  resumed: () => void;
  waiting: (job: Job<DataType, ResultType, NameType>) => void;
}
```

QueueListener interface는 Queue에게 허용된 이벤트 들의 목록을 정의한다. 예를 들어 Queue의 이벤트 관련 메서드 중 emit을 보자

```ts
emit<U extends keyof QueueListener<DataType, ResultType, NameType>>(
  event: U,
  ...args: Parameters<QueueListener<DataType, ResultType, NameType>[U]>
): boolean {
  return super.emit(event, ...args);
}
```

여기서 보면 제네릭으로 U를 받는데 U는 `QueueListener`의 key 중 하나이고, 그 key를 이용해서 args로 들어오는 함수의 타입을 검증하는 용도로 사용된다. 예시를 보여주자면 다음과 같다.

```ts
import { Queue } from 'bullmq';

const myQueue = new Queue('Paint');

// waiting은 QueueListener의 key중 하나. 
// 따라서 2번째 인자는 waiting: (job: Job<DataType, ResultType, NameType>) => void;
// 함수의 시그니처를 따르게 된다. 
myQueue.on('waiting', (job: Job) => {});
```

### Queue

자 그러면 이제 진짜 Queue Class에 대해서 살펴보도록 하자. **Queue 클래스에서는 대부분 직접 Redis에 Write 명령을 내리는 기능과, Queue내에서 실행된 다양한 작업에 의해 트리거 된 이벤트를 직접적으로 제공하는 역할을 하고 있다.**

**Constructor**

```ts
constructor(
  name: string,
  opts?: QueueOptions,
  Connection?: typeof RedisConnection,
) {
  super(
    name,
    {
      blockingConnection: false,   // --- (1)
      ...opts,
    },
    Connection,
  );

  this.jobsOpts = opts?.defaultJobOptions ?? {};

  this.waitUntilReady()   // --- (2)
    .then(client => {
      if (!this.closing && !opts?.skipMetasUpdate) {
        return client.hmset(this.keys.meta, this.metaValues);
      }
    })
    .catch(err => {});
}
```

**(1) `blockingConnection`**

blocking Connection은 무조건 false로 되어 있다. 공식문서에서 설명되어 있듯이, Queue나 Worker같은 경우 기본적으로 독립적인 Connection을 가지는 것이 기본 옵션이나, Connection을 재활용 할 수 있다고 나와 있다. 이 기능을 위해서라도 blocking connection을 비활성화 시켜야 하기 때문에 false로 설정해 둔 듯 하다. Redis의 Blocking Connection에 대해서는 나중에 더 자세히 설명하도록 하고 이 글에서는 넘어가도록 하겠다.

**(2) `waitUntilReady`**

waitUntilReady함수는 `Promise<RedisClinet>` 타입을 반환한다. 즉 Redis Client를 이용해서 redis에 queue에 필요한 메타데이터를 저장해서 초기화하는 역할을 한다.

**Event methods**

```ts
emit<U extends keyof QueueListener<DataType, ResultType, NameType>>(
  event: U,
  ...args: Parameters<QueueListener<DataType, ResultType, NameType>[U]>
): boolean {
  return super.emit(event, ...args);
}

off<U extends keyof QueueListener<DataType, ResultType, NameType>>(
  eventName: U,
  listener: QueueListener<DataType, ResultType, NameType>[U],
): this {
  super.off(eventName, listener);
  return this;

on<U extends keyof QueueListener<DataType, ResultType, NameType>>(
  event: U,
  listener: QueueListener<DataType, ResultType, NameType>[U],
): this {
  super.on(event, listener);
  return this;
}

once<U extends keyof QueueListener<DataType, ResultType, NameType>>(
  event: U,
  listener: QueueListener<DataType, ResultType, NameType>[U],
): this {
  super.once(event, listener);
  return this;
}
```

각 메서드들은 NodeJS.EventEmitter의 메서드들을 오버라이딩 한 함수들이다. 위에서 말했던 것 처럼 U 타입을 제네릭 인자로 받고, 리스너 함수를 검증해서 사용하는데 쓴다.

그 외에 대부분은 Redis에 저장된 스크립트를 실행시키는 코드들이 대부분이다.

```ts
// Queue Class의 메서드 

async pause(): Promise<void> {
  await this.scripts.pause(true);
  this.emit('paused');
}
  

// Script Class의 메서드 
async pause(pause: boolean): Promise<void> {
  const client = await this.queue.client;

  const args = this.pauseArgs(pause);
  return this.execCommand(client, 'pause', args);
}
```

Queue에서 redis에 관련된 작업을 하는것은 크게 3개의 단계로 나뉜다.

1. script를 실행하기 전에 필요한 작업 (Job 객체를 생성 및 등록 같은 작업)
2. script 객체에서 해당된 메서드를 호출 및 실행
3. 이벤트 발생 등 후처리 작업

이 중에서 2번을 좀 더 자세히 보면 script class에 정의된 메서드들은 전부 `execCommand()` 메서드를 이용해서 전부 redis에 명령을 내린다. 그렇다면 `execCommand()` 에 대해서 좀 더 자세히 살펴보자

```ts
  public execCommand(
    client: RedisClient | ChainableCommander,
    commandName: string,
    args: any[],
  ) {
    const commandNameWithVersion = `${commandName}:${this.version}`;
    return (<any>client)[commandNameWithVersion](args);
  }
```

실제로 이 함수는 아래와 같은 함수가 되어 실행된다.

```ts
const result = client['set:v1'](args);
```

그렇다면 이 `'set:v1'` 같은 명령어는 어디서 오는가에 대해서 의문을 가질 수 있다. 소스 코드의 `src/commands` 경로에 보면 Lua script들이 모여있는 것을 확인할 수 있다. 이 Lua Script들이 redis에 등록되고, 해당 redis client는 redis에 등록되어있던 Lua script를 호출하는 방식으로 실행된다.

## Summary

정리하자면, QueueBase는 Connection 정보를 설정하고 제공한다.

QueueGetters는 각종 Queue와 연관이 있는 Worker, Jobs, Metrics에 대한 정보 조회를 제공하는 역할을 담당한다.

Queue는 그 외에 Queue 또는 Queue Job의 상태 변화에 따른 정보를 Event를 통해 제공하고, redis client를 통해서 직접 redis에 write하는 명령을 담당한다.