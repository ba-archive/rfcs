# 剧情播放器架构

## 技术栈

- 状态管理：Pinia + Vue
- Spine 渲染：pixi v6, pixi-spine

## DOM 结构

最外层为 Vue Wrapper, 用于管理状态。

播放器分为五层。从上到下分别是：对话框（对话 LOG 和梗概），控制按钮（组），地点指示器，字幕，canvas（显示人物和背景）。

涉及文字渲染的元素统一交给 Vue 进行处理。

![](https://hackmd.io/_uploads/Hy6oHW4Bs.jpg)


### 对话框

使用 Vue 组件实现。

### 控制按钮（组）

使用 Vue 组件实现。

### 地点指示器

### 字幕

使用 Vue 组件实现。

### canvas

使用 pixi 和 pixi-spine 实现。具体层级为：

- 特效层（背景特效，如雨、雾、烟等；转场特效）
- 人物层（人物）
- 背景层（背景）

## 数据流动

### 1. 从后端取到原始剧情数据

> 原始数据应当是一个 Array，包含剧情标题和内容。

> Array 的例子如下。

<details>
<summary>点击展开</summary>

```json=
[
  {
    "GroupId": 1000502,
    "SelectionGroup": 0,
    "BGMId": 3,
    "Sound": "",
    "Transition": 0,
    "BGName": 719512360,
    "BGEffect": 0,
    "PopupFileName": "",
    "ScriptKr": "#Title;잠자는 고래",
    "TextJp": "眠る鯨",
    "TextCN": "沉睡的鲸鱼",
    "TextTw": "沉睡的鯨魚",
    "TextEn": "The Sleeping Whale",
    "VoiceJp": ""
  },
  {
    "GroupId": 1000502,
    "SelectionGroup": 0,
    "BGMId": 0,
    "Sound": "",
    "Transition": 0,
    "BGName": 0,
    "BGEffect": 0,
    "PopupFileName": "",
    "ScriptKr": "#Place;대책위원회 동아리실",
    "TextJp": "対策委員会・教室",
    "TextCN": "对策委员会・教室",
    "TextTw": "對策委員會社團室",
    "TextEn": "Foreclosure Task Force Room",
    "VoiceJp": ""
  },
  {
    "GroupId": 1000502,
    "SelectionGroup": 0,
    "BGMId": 0,
    "Sound": "",
    "Transition": 0,
    "BGName": 0,
    "BGEffect": 0,
    "PopupFileName": "",
    "ScriptKr": "3;호시노;03;그러니까 고래는 잘 때도 숨을 참고 있다는 얘기잖아. 굉장하지 않아?\n#3;em;[반응]\n#3;a",
    "TextJp": "それで、[ruby=くじら]鯨[/ruby]は寝てる時も呼吸を我慢してるんだって。すごくない？",
    "TextCN": "所以说，[ruby=くじら]鯨[/ruby]在睡觉的时候也会憋着气哦。不觉得很厉害吗？",
    "TextTw": "也就是說鯨魚睡覺的時候會憋著氣喔，你不覺得很厲害嗎？",
    "TextEn": "So that means whales are holding their breath when they sleep... Isn't that amazing?",
    "VoiceJp": ""
  }
]
```
</details>

### 2. 翻译层转译原始数据为剧情播放器标准格式<a id="player-config-form"></a>

剧情播放器能够接受的单条标准参数如下。

<details>
<summary>初稿</summary>

```json
{
  "Type": [
    "title | place | text | option | live2d"
  ],
  "GroupId": 11000,
  "SelectionGroup": 0,
  "BGMId": 0,
  "Sound": "SE_Typing_02",
  "Transition": 0,
  "BGName": 0,
  "BGEffect": [
    "rain"
  ],
  "PopupFileName": "",
  "Character": [
    {
      "Position": 2,
      "StudentID": 10000,
      "Face": 3,
      "Highlight": True
    },
    {
      "Position": 4,
      "StudentID": 10001,
      "Face": 2,
      "Highlight": False
    }
  ],
  "CharacterEffect": [
    {
      "Target": 1,
      "Effect": "cheer",
      "Async": true
    },
    {
      "Target": 1,
      "Effect": "timeout",
      "Async": false
    }
    {
      "Target": 1,
      "Effect": "hophop",
      "Async": true
    },
    {
      "Target": 1,
      "Effect": "ar",
      "Async": true
    },
    {
      "Target": 3,
      "Effect": "m5",
      "Async": true
    }
  ],
  "Option": [
    {
      "SelectionGroup": 1,
      "Text": {
        "TextJp": "選択肢1",
        "TextCn": "选项1",
        "TextTw": "選項1",
        "TextEn": "Option 1"
      }
    },
    {
      "SelectionGroup": 2,
      "Text": {
        "TextJp": "選択肢2",
        "TextCn": "选项2",
        "TextTw": "選項2",
        "TextEn": "Option 2"
      }
    }
  ],
  "TextEffect": [
    {
      "Name": "st",
      "Value": [
        316,
        566
      ],
      "Async": false
    },
    {
      "Name": "fontsize",
      "Value": 140
    },
    {
      "Name": "smooth",
      "Value": Null
    }
  ],
  "Text": {
    "TextJp": "君日本語本当上手",
    "TextCn": "中文翻译",
    "TextTw": "国际服翻译",
    "TextEN": "English text"
  },
  "VoiceJp": "Main_11000_054"
}
```
</details>

#### 修改后定义:

**注意：命名大小驼峰都有使用, 这是为了区分原始数据属性(大驼峰)和自定义数据属性(小驼峰)。**

```typescript=
type StoryType = "title" | "place" | "text" | "option" | "st" | "effectOnly" | "continue"

interface Text{
    content: string
    waitTime?: number
}
    
interface Character{
      position: number,
      CharacterName: number,
      face: number,
      highlight: boolean
}
    
interface CharacterEffect{
      target: number,
      effect: string,
      async: boolean
}
    
interface Option{
      SelectionGroup: number,
      text: {
        TextJp: string,
        TextCn: string,
        TextTw: string,
        TextEn: string
      }
}
    
interface TextEffect{
      name: string,
      value: string[],
      textIndex: number
}
    
interface Effect{
    type: string
    args: Arrary<string>
}

interface StoryUnit{
  GroupId: number,
  SelectionGroup: number,
  BGMId: number,
  Sound: string,
  Transition: number,
  BGName: number,
  BGEffect: string[]
  PopupFileName: string,
  type: StoryType,
  menuState: boolean,
  characters: Character[],
  characterEffect: CharacterEffect[],
  options: Option[],
  textEffect: TextEffect[],
  text: {
    TextJp: Text[],
    TextCn: Text[],
    TextTw: Text[],
    TextEN: Text[]
  },
  VoiceJp: string,
  stArgs?: string[],
  nextChapterName?:string,
  fight?: number,
  clear?: "st"|"all",
  otherEffect?: Effect[]
}
```

#### 参数定义

##### `Type`: `Array`

定义该条数据的类型。Array 内的元素一般为 `title`, `place`, `text`, `option`, `live2d` 中的一个。如果在 L2D
场景中需要做出选择，可以同时使用 `live2d` 和 `option`。（需不需要做到这一步？棒子没有严格区分普通剧情和 L2D）

<details>
<summary>每个值的具体含义</summary>

##### `title`

该条消息为标题。剧情播放器会在 canvas 中央显示标题。

##### `place`

该条消息为场景名称。剧情播放器会调用 Vue 组件在播放器左上角显示场景名称。

##### `text`

该条消息为文本。剧情播放器会调用 Vue 组件在播放器内部下方显示说话人、说话人所属、文本内容。

##### `option`

该条消息玩家需要做出选择。选择肢会出现在播放器中央。（因为涉及到文字，所以还是交给 Vue 来做比较好）

##### `live2d`

该条消息为 Live2D 场景。
</details>

##### `GroupId`: `Number`

消息组的 ID。在同一个剧情中，这个 ID 总是相同。

##### `SelectionGroup`: `Number`

选择肢的 ID。在 `options` 当中选择了某个选项之后，应该跳转到对应的选择肢 ID 的内容。

##### `BGMId`: `Number`

背景音乐的 ID。剧情播放器根据这个 ID 播放对应的背景音乐。（游戏中 BGM 的命名格式均为 `Theme_000`）

##### `Sound`: `String`

需要播放的音效。播放器根据这个字符串找到对应的音效文件。

##### `Transition`: `Number`

场景切换效果的 ID 。在请求资源的时候，同时请求 `ScenarioTransitionExcelTable.json` 文件，根据这个 ID 找到对应的场景切换效果。

##### `BGName`: `Number`

背景图片的 ID。在请求资源的时候，同时请求 `ScenarioBGExcelTable.json` 文件，根据这个 ID 找到对应的背景图片。

##### `BGEffect`: `Array`

背景图片的特效。这个由特效函数实现。

##### `PopupFileName`: `String`

有的时候会跳出一些小图片，PopupFileName 就是这些图片的文件名。

##### `Character`: `Array`

剧情中出现的人物。

##### `Character.StudentID`: `Number`

人物 ID。通过 ID 载入对应的资源文件。

##### `Character.Position`: `Number`

人物的位置。从左到右为 `1` 到 `5`。

##### `Character.Face`: `Number`

人物的表情。从 atlas 和 skel 当中提取。

##### `Character.Highlight`: `Boolean`

人物是否高亮。`True` 时不做处理，`False` 时加上一层透明度为 0.5 的黑色遮罩。

##### `CharacterEffect`: `Array`

人物的特效。这个由特效函数实现。

##### `CharacterEffect.Target`: `Number`

适用于第几个人物。从左到右为 `1` 到 `5`。（待定，是否用学生 ID 更好一些）

##### `CharacterEffect.Effect`: `String`

特效的名称。这个由特效函数实现。

##### `CharacterEffect.Async`: `Boolean`

特效是否异步。`True` 时特效会立即播放，`False` 时会顺序播放。

##### `Option`: `Array`

选择肢。

##### `Option.SelectionGroup`: `Number`

选择肢的 ID。在 `option` 当中选择了某个选项之后，应该跳转到对应的选择肢 ID 的内容。

##### `Option.Text`: `Object`

选择肢文本。有 `TextJp`, `TextCn`, `TextTw`, `TextEn` 四个 key。

**除了 `TextJp`，其他字段都有可能缺失。前端应该做好 fallback 的准备**

##### `TextEffect`: `Array`

文本特效。

##### `TextEffect.Name`: `String`

特效的名称。

##### `TextEffect.Value`: `Array | Number | undefined`

特效参数。根据特效名称决定类型。

##### `TextEffect.Async`: `Boolean`

特效是否异步。`True` 时特效会立即播放，`False` 时会顺序播放。

##### `Text`: `Object`

剧情文本。有 `TextJp`, `TextCn`, `TextTw`, `TextEn` 四个 key。

**除了 `TextJp`，其他字段都有可能缺失。前端应该做好 fallback 的准备**

##### `VoiceJp`: `String`

日语语音文件名。目前只有序章有语音。

#### 修改后参数定义(仅说明原来参数定义没有的部分)
##### Type: StoryType
`'continue'`: 展示下一章节名

`st`: #st放置文字 
`effectOnly`: 只有特效没有文字的情况
##### menuState
菜单是否显示
##### characters
改为使用`CharacterName`作为唯一标识(剧情里不一定出现的是已实装学生)
##### textEffect
增加了`textIndex`字段指定对哪段文字起作用
##### Text
`content`为具体文字, `waitTime`为该文字播放后需要等待的时间
##### nextChapterName
下一章节名, 可选
##### stArgs
st的参数, 可选
##### fight
是否接下来加入战斗, 值为战斗视频的ID. 可选.
##### clear
是否进行清除操作, `st`清除放置文字, `all`清除所有.
##### otherEffect
无法分类的特效的预留接口


## 播放器

### 0. 定义

```typescript
type NextState = "success" | "blocking" | ...;
interfafce NextArg {
    ...
}
interface NextResult {
    state: NextState;
}
interface Player {
    init: (config: PlayerConfig) => Promise<boolean>;
    next: (args: NextArg) => Promise<NextResult>;
    clear: () => Promise<boolean>;
    select: (selection: number) => Promise<NextResult>; // alias for next ?
    readonly isPlaying: boolean;
}
```
数据交换示意图
![baPlayer](https://cdn.staticaly.com/gh/ourandream/blog_images@master/20221108/baPlayer.4f39upc84g00.webp)


### 1. 本体
本体仅用于更改当前剧情信息.需要支持自动更改剧情信息.

同时, 本体还负责处理各层发出的各种事件.

#### 接口

##### 1. init

初始化播放器状态，提供播放必要配置，配置[见上](#player-config-form)。

`init(config: PlayerConfig): Promise<boolean>;`

##### 2. next

播放下一条脚本，当目前正在播放时，根据脚本类型判断是否直接完成播放或不响应调用。

`next(args: NextArg): Promise<NextResult>;`

##### 3. clear

清屏

`clear(): Promise<boolean>;`

##### 4. select

alias for next ?

`select(selection: number): Promise<NextResult>;`

##### 5. isPlaying

当前是否正在播放?

`readonly isPlaying: boolean;`

#### 事件

播放器应该向容器暴露部分事件，以便 Vue Wrapper 控制播放器的行为。

#### 1. 用于控制快进的事件

##### 「人物语音播放完成」事件，「打字机效果完成」事件，「特效播放完成」事件

其中，如果人物语音没有播放完成（或打字机效果没有完成），则容器强制将这两项事件状态设置为完成。

如果特效没有播放完成，那么在点击时没有反应，不会快进到下一个事件。（L2D 也没有快进选项）

### 2. 特效层
特效层用于播放除人物相关特效外的特效
<details>
<summary>已过时</summary>


#### 1. 提供标准的调用接口用于播放特效
```typescript
playEffect(config:EffectConfig)

type EffectType = 'CharacterEffect' | 'BGEffect' | 'Transition';

interface EffectConfig {
    effectType: EffectType
    name: string
    appInstance: Pixi.Application
    characterInstance?: Pixi-Spine.Spine;
    BgInstance?:Sprite
    args?:Array<string>
}
```

##### `effectType`

type:`'CharacterEffect'|'BGEffect' | 'Transition'`

指定特效的类型, 必需

`characterEffect`指与角色相关的特效, 调用时需要传入可选的角色的pixi-spine实例参数

`BGEffect`指背景特效, 调用时传入背景Sprite实例, 注意如zmc, move, bgshake等也被视为背景特效.

`Transition`指转场
##### `name`
特效名, 必需

##### `appInstance`

type: `Application`

pixi Application实例, 必需

##### `characterInstance`

type: `Spine`

角色pixi-spine实例, 可选
##### `BgInstance`
type: `Sprite`
背景Sprite实例, 可选
##### args
特效可能带有的参数, 数组形式以便传入多个参数. 可选.
</details>
    
#### 事件

`effectDone`: 特效播放完成时发出的事件

##### 补充?

建议用config的形式

`playEffect(config: EffectConfig)`

```typescript
type EffectType = 'CharacterEffect' | 'BGEffect' | 'Transition';

interface EffectConfig {
    effectType: EffectType;
    appInstance: Pixi.Application;
    characterInstance?: PixiSpine;
    args: ...
}
```
### 3. 人物层
人物层负责处理人物的显示, 人物特效, 人物动作.
    
#### 事件
`characterDone`: 人物各种处理已完成

### 4. 背景层
背景层负责背景或live2d的显示. 它在改变的同时会更新背景实例信息.
### 5.声音层
声音层负责背景音乐, 效果音, 语音等的播放, 接口如下:
##### playAudio(config:AudioConfig)
```typescript
interface AudioConfig{
    name: string
    loop: boolean
} 
```
### 6 UI层
UI层负责UI的相关功能
#### 事件
`skip`: 跳过剧情
### 7 文字层
文字层负责有对话框文字, 无对话框文字, 选项的显示.

文字层同时需要更新历史剧情信息与选项选择信息.
#### 事件
`next`: 进入下一语句
    
`select`: 选择后加入下一剧情语句

### 8. 其他接口?

##### 立即完成播放

`skip()`