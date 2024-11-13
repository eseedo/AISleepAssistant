# 基础项目：OpenAI 实时语音项目模板

### 参考客户端：实时API（测试版）
这个库包含了一个参考客户端（也称为示例库），用于连接到OpenAI的实时API。由于该库处于测试版阶段，因此它不应被视为最终实现。你可以使用它轻松地构建对话应用程序原型。

要快速使用API，最简单的方法是通过“实时控制台”进行，它使用参考客户端来提供功能齐全的API检查器，并带有语音可视化等示例功能。

### 快速入门
该库可用于服务器端（Node.js）和浏览器端（例如React、Vue）中，支持JavaScript和TypeScript。因为它处于测试阶段，所以需要直接从GitHub安装：

```bash
$ npm i openai/openai-realtime-api-beta --save
```

以下是如何导入并使用`RealtimeClient`的示例：

```javascript
import { RealtimeClient } from '@openai/realtime-api-beta';

const client = new RealtimeClient({ apiKey: process.env.OPENAI_API_KEY });

// 在连接之前可以设置多个参数
client.updateSession({ instructions: 'You are a great, upbeat friend.' });
client.updateSession({ voice: 'alloy' });
client.updateSession({
  turn_detection: { type: 'none' }, // 或 'server_vad'
  input_audio_transcription: { model: 'whisper-1' },
});

// 设置事件处理
client.on('conversation.updated', (event) => {
  const { item, delta } = event;
  const items = client.conversation.getItems();
  /**
   * item 是当前正在更新的对话项
   * delta 可以为空或包含更新信息
   * 可以随时获取完整的对话项列表
   */
});

// 连接到实时API
await client.connect();

// 发送一个对话项并触发生成
client.sendUserMessageContent([{ type: 'input_text', text: `How are you?` }]);
```

### 浏览器（前端）快速入门
你可以直接在浏览器中使用该客户端，例如在React或Vue应用中。但我们不推荐这样做，因为直接在浏览器中连接到OpenAI会使API密钥面临风险。要在浏览器环境中实例化客户端，可以使用以下代码：

```javascript
import { RealtimeClient } from '@openai/realtime-api-beta';

const client = new RealtimeClient({
  apiKey: process.env.OPENAI_API_KEY,
  dangerouslyAllowAPIKeyInBrowser: true,
});
```

如果你运行自己的中继服务器（例如使用实时控制台），可以改为连接到中继服务器URL：

```javascript
const client = new RealtimeClient({ url: RELAY_SERVER_URL });
```

#### 解释
这个参考客户端库提供了一个简化的接口来连接OpenAI的实时API，并处理音频和文本输入。前端使用时，建议通过中继服务器连接而不是直接暴露API密钥，以确保安全。服务器端使用方式则可以直接使用API密钥。

### 内容目录
- 项目结构
- 使用参考客户端
- 发送消息
- 发送流式音频
- 添加和使用工具
- 手动使用工具
- 中断模型
- 客户端事件
- 参考客户端实用事件
- 服务器事件
- 运行测试
- 致谢和联系

### 项目结构
在该库中，有三种基本模块可用于与实时API接口交互。建议从`RealtimeClient`入手，但更高级的用户可能更喜欢使用底层模块。

#### 1. RealtimeClient
- 是与实时API交互的主要抽象模块。
- 通过简化的控制流支持快速应用开发。
- 提供自定义事件：`conversation.updated`、`conversation.item.appended`、`conversation.item.completed`、`conversation.interrupted` 和 `realtime.event`。
- 这些事件发送对话项的增量更新和对话历史。

#### 2. RealtimeAPI
- 作为 `client.realtime` 存在于客户端实例上。
- 是WebSocket的轻量封装。
- 用于连接API、身份验证和发送对话项。
- 不进行项验证，需要直接依赖API规范。
- 分别以 `server.{event_name}` 和 `client.{event_name}` 形式调度事件。

#### 3. RealtimeConversation
- 作为 `client.conversation` 存在于客户端实例上。
- 在客户端缓存当前对话。
- 具有事件验证功能，以确保缓存事件的有效性。

### 解释
在这个结构中，`RealtimeClient`是最适合快速上手的模块，因为它封装了与API的主要交互流程并提供了丰富的事件支持。`RealtimeAPI`更接近底层实现，直接控制WebSocket连接、身份验证等操作。`RealtimeConversation`则帮助管理和缓存当前会话内容，确保客户端始终能够访问最新的对话数据。

这个分层结构使得用户可以根据需求选择适合的接口级别。`RealtimeClient`适合大多数用户使用，而`RealtimeAPI`和`RealtimeConversation`为那些需要更多自定义和控制的用户提供了额外的灵活性。

### 使用参考客户端
这个客户端附带了一些基本的实用工具，使您可以快速构建实时应用程序。

#### 发送消息
可以很简单地从用户向服务器发送消息：

```javascript
client.sendUserMessageContent([{ type: 'input_text', text: `How are you?` }]);
// 或发送空音频
client.sendUserMessageContent([
  { type: 'input_audio', audio: new Int16Array(0) },
]);
```

#### 发送流式音频
要发送流式音频，可以使用 `.appendInputAudio()` 方法。如果处于 `turn_detection: 'disabled'` 模式，则需要使用 `.createResponse()` 来告诉模型生成响应：

```javascript
// 发送用户音频数据，格式必须是 Int16Array 或 ArrayBuffer
// 默认音频格式为 pcm16，采样率为 24,000 Hz
for (let i = 0; i < 10; i++) {
  const data = new Int16Array(2400);
  for (let n = 0; n < 2400; n++) {
    const value = Math.floor((Math.random() * 2 - 1) * 0x8000);
    data[n] = value;
  }
  client.appendInputAudio(data);
}
// 提交待处理音频并请求模型生成响应
client.createResponse();
```

#### 添加和使用工具
使用工具非常方便，只需调用 `.addTool()` 方法并设置一个回调函数作为第二个参数。回调函数会在工具的参数完成后自动执行，并将结果发送回模型。

```javascript
client.addTool(
  {
    name: 'get_weather',
    description:
      '获取指定经纬度坐标的天气信息。请指定地点标签。',
    parameters: {
      type: 'object',
      properties: {
        lat: {
          type: 'number',
          description: '纬度',
        },
        lng: {
          type: 'number',
          description: '经度',
        },
        location: {
          type: 'string',
          description: '地点名称',
        },
      },
      required: ['lat', 'lng', 'location'],
    },
  },
  async ({ lat, lng, location }) => {
    const result = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lng}&current=temperature_2m,wind_speed_10m`,
    );
    const json = await result.json();
    return json;
  },
);
```

#### 手动使用工具
`.addTool()` 方法会自动运行工具并在完成时触发响应。但有时您可能不希望这样，可能希望手动使用工具生成某些结构用于其他目的。在这种情况下，可以通过 `updateSession` 方法来设置工具，并指定 `type: 'function'`。

注意：通过 `.addTool()` 添加的工具不会被 `updateSession` 手动更新覆盖，但每次调用 `updateSession()` 都会覆盖之前的更改。通过 `.addTool()` 添加的工具会被保留并附加到这里手动设置的内容中。

```javascript
client.updateSession({
  tools: [
    {
      type: 'function',
      name: 'get_weather',
      description:
        '获取指定经纬度坐标的天气信息。请指定地点标签。',
      parameters: {
        type: 'object',
        properties: {
          lat: {
            type: 'number',
            description: '纬度',
          },
          lng: {
            type: 'number',
            description: '经度',
          },
          location: {
            type: 'string',
            description: '地点名称',
          },
        },
        required: ['lat', 'lng', 'location'],
      },
    },
  ],
});
```

要处理函数调用，可以监听 `conversation.updated` 和 `conversation.item.completed` 事件：

```javascript
client.on('conversation.updated', ({ item, delta }) => {
  if (item.type === 'function_call') {
    // 处理函数调用
    if (delta.arguments) {
      // 填充参数
    }
  }
});

client.on('conversation.item.completed', ({ item }) => {
  if (item.type === 'function_call') {
    // 函数调用完成后执行自定义代码
  }
});
```

#### 解释
`addTool`方法允许你添加交互式工具，`updateSession`则可以在不触发工具自动运行的情况下手动配置工具，这样可以更灵活地处理不同场景中的功能调用。通过事件监听可以动态控制函数调用的执行和完成，并处理所需的自定义逻辑。这种设计使得你在需要更高的控制时拥有更多的灵活性。

### 中断模型
在需要手动中断模型（尤其在 `turn_detection: 'disabled'` 模式下）时，可以使用以下方法：

```javascript
// id 是当前生成项的标识符
// sampleCount 是监听器接收的音频样本数量
client.cancelResponse(id, sampleCount);
```

这个方法会立即停止模型的生成，同时删除 `sampleCount` 之后的所有音频，并清除文本响应。通过这种方式，可以中断模型并防止它“记住”超出用户状态的内容。

### 客户端事件
如果你需要更高的手动控制，并希望根据实时客户端事件API参考发送自定义客户端事件，可以使用 `client.realtime.send()` 方法：

```javascript
// 手动发送函数调用输出
client.realtime.send('conversation.item.create', {
  item: {
    type: 'function_call_output',
    call_id: 'my-call-id',
    output: '{function_succeeded:true}',
  },
});
client.realtime.send('response.create');
```

### 参考客户端实用事件
`RealtimeClient` 将服务器事件简化为五个主要事件，以帮助控制应用程序流程。这些事件并非API规范的一部分，而是封装逻辑以简化开发。

```javascript
// 处理连接失败等错误
client.on('error', (event) => {
  // 处理错误事件
});

// VAD模式下，用户开始说话时触发
// 可用于在必要时停止先前响应的音频播放
client.on('conversation.interrupted', () => {
  // 处理中断事件
});

// 包含所有对话更新，delta可能包含额外信息
client.on('conversation.updated', ({ item, delta }) => {
  const items = client.conversation.getItems();
  switch (item.type) {
    case 'message':
      // 系统、用户或助手消息
      break;
    case 'function_call':
      // 模型的函数调用
      break;
    case 'function_call_output':
      // 用户或应用程序的响应
      break;
  }
  if (delta) {
    // 事件中的变化可能包含音频、转录或参数
  }
});

// 对话项添加后触发
client.on('conversation.item.appended', ({ item }) => {
  // 项目状态为 'in_progress' 或 'completed'
});

// 对话项完成后触发，总是在conversation.item.appended之后
client.on('conversation.item.completed', ({ item }) => {
  // 项目状态总为 'completed'
});
```

### 服务器事件
如果你希望更精细地控制应用开发，可以使用 `realtime.event` 事件，仅响应服务器事件。有关这些事件的详细文档，请参考实时服务器事件API参考：

```javascript
// 所有事件，可用于日志、调试或手动事件处理
client.on('realtime.event', ({ time, source, event }) => {
  if (source === 'server') {
    doSomething(event); // 处理服务器事件
  }
});
```

### 运行测试
在运行测试之前，请确保 `.env` 文件中设置了 `OPENAI_API_KEY`。之后，运行测试套件非常简单：

```bash
$ npm test
```

若要运行包含调试日志的测试（记录发送到WebSocket的事件），使用：

```bash
$ npm test -- --debug
```

#### 总结
`RealtimeClient` 提供了灵活的事件控制和丰富的API接口，允许开发者在实时应用中对模型进行动态控制。通过事件处理，可以在不同阶段截取和响应事件数据，进行调试和日志记录。希望这些内容能帮助你在项目中顺利实现对实时API的控制！