# hls.js 源码分析

这篇文章梳理一下 [hls.js](https://github.com/video-dev/hls.js) 播放 m3u8 文件的主流程。

首先，hls 先请求 m3u8文件，获取到每个切片的地址，然后按顺序请求这些 ts，解析出 ts 当中的音视频资源，使用 [Media Source Extensions](https://developer.mozilla.org/en-US/docs/Web/API/MediaSource) 将 buffer 内容进行合流，组成一个可播的媒体资源文件。

### Media Source Extensions

这是浏览器提供的一个 API，让 JS 通过 API 控制 Video 资源。官方例子：

```js
var vidElement = document.querySelector('video');

if (window.MediaSource) {
  var mediaSource = new MediaSource();
  vidElement.src = URL.createObjectURL(mediaSource);
  mediaSource.addEventListener('sourceopen', sourceOpen);
} else {
  console.log("The Media Source Extensions API is not supported.")
}

function sourceOpen(e) {
  URL.revokeObjectURL(vidElement.src);
  var mime = 'video/webm; codecs="opus, vp09.00.10.08"';
  var mediaSource = e.target;
  var sourceBuffer = mediaSource.addSourceBuffer(mime);
  var videoUrl = 'droid.webm';
  fetch(videoUrl)
    .then(function(response) {
      return response.arrayBuffer();
    })
    .then(function(arrayBuffer) {
      sourceBuffer.addEventListener('updateend', function(e) {
        if (!sourceBuffer.updating && mediaSource.readyState === 'open') {
          mediaSource.endOfStream();
        }
      });
      sourceBuffer.appendBuffer(arrayBuffer);
    });
}
```

通过这个 API，就可以把切片 buffer 添加到 Video 里。

### 调用流程

```js
<video id="video"></video>
<script>
  var video = document.getElementById('video');
  var videoSrc = 'https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8';
  if (Hls.isSupported()) {
    var hls = new Hls();
    hls.loadSource(videoSrc);
    hls.attachMedia(video);
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    video.src = videoSrc;
  }
</script>
```

可以看到，外部调用很简单，主要是三步，初始化 Hls 实例，加载 video 资源，挂载 Video DOM。

##### 初始化

```typescript
constructor(userConfig: Partial<HlsConfig> = {}) {
  // 配置 config
  const config = (this.config = mergeConfig(Hls.DefaultConfig, userConfig));
  this.userConfig = userConfig;
  enableLogs(config.debug);
  this._autoLevelCapping = -1;
  if (config.progressive) {
    enableStreamingMode(config);
  }
  // 添加各种 controller
  const abrController = (this.abrController = new config.abrController(this));
  const bufferController = new config.bufferController(this);
  const capLevelController = (this.capLevelController = new config.capLevelController(this));
  const fpsController = new config.fpsController(this);
  const playListLoader = new PlaylistLoader(this);
  const keyLoader = new KeyLoader(this);
  const id3TrackController = new ID3TrackController(this);
  // network controllers
  const levelController = (this.levelController = new LevelController(this));
  // FragmentTracker must be defined before StreamController because the order of event handling is important
  const fragmentTracker = new FragmentTracker(this);
  const streamController = (this.streamController = new StreamController(
    this,
    fragmentTracker
  ));

  // Level Controller initiates loading after all controllers have received MANIFEST_PARSED
  // 调用 startLoad，开始解析资源
  levelController.onParsedComplete = () => {
    if (config.autoStartLoad || streamController.forceStartLoad) {
      this.startLoad(config.startPosition);
    }
  };

  // Cap level controller uses streamController to flush the buffer
  capLevelController.setStreamController(streamController);
  // fpsController uses streamController to switch when frames are being dropped
  fpsController.setStreamController(streamController);
  const networkControllers = [levelController, streamController];
  this.networkControllers = networkControllers;
  const coreComponents = [
    playListLoader,
    keyLoader,
    abrController,
    bufferController,
    capLevelController,
    fpsController,
    id3TrackController,
    fragmentTracker,
  ];
  // 根据 config 生成的 controller
  this.audioTrackController = this.createController(
    config.audioTrackController,
    null,
    networkControllers
  );
  // ...

  this.coreComponents = coreComponents;
}
```

初始化的过程很简单，主要处理传入的 config，以及初始化各种 controller。由于 Hls 内部都是使用事件来进行通知，所以需要在初始化时将各种事件监听挂载上去。

##### loadSource

```typescript
loadSource(url: string) {
  this.stopLoad();
  // ...
  this.trigger(Events.MANIFEST_LOADING, { url: url });
}
```

loadSource 本身并没有做什么，只是触发了 MANIFEST_LOADING 这个事件，那么有哪些 controller 在监听这些事件：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gnw4sgf8a7j30oq0ya7cv.jpg" alt="image-20210222110357146" style="zoom:50%;" />

看一下 stream-controller 做了什么

```typescript
constructor(hls: Hls) {
  hls.on(Events.MANIFEST_LOADING, this.onManifestLoading, this);
}
private onManifestLoading() {
  // reset buffer on manifest loading
  this.log('Trigger BUFFER_RESET');
  this.hls.trigger(Events.BUFFER_RESET, undefined);
  this.fragmentTracker.removeAllFragments();
  this.stalled = false;
  this.startPosition = this.lastCurrentTime = 0;
  this.fragPlaying = null;
}
```

仅是做了重置脏数据的操作，其他的 controller 也是类似操作，真正加载资源的是 playlist-loader 这个 controller，摘取某些重要的代码：

```typescript
// playlist-loader.ts
private onManifestLoading(
  event: Events.MANIFEST_LOADING,
  data: ManifestLoadingData
) {
  const { url } = data;
  this.checkAgeHeader = true;
  this.load({
    id: null,
    groupId: null,
    level: 0,
    responseType: 'text',
    type: PlaylistContextType.MANIFEST,
    url,
    deliveryDirectives: null,
  });
}

private load(context: PlaylistLoaderContext): void {
  // ...
  const loaderCallbacks = {
    onSuccess: this.loadsuccess.bind(this),
    onError: this.loaderror.bind(this),
    onTimeout: this.loadtimeout.bind(this),
  };
	// loader 可以由外部传入，默认使用 xhr
  loader.load(context, loaderConfig, loaderCallbacks);
}
// 请求成功后，执行这个函数
private loadsuccess(
  response: LoaderResponse,
  stats: LoaderStats,
  context: PlaylistLoaderContext,
  networkDetails: any = null
): void {
  const string = response.data as string;
  // 如果不是 #EXTM3U 开头，说明不是 m3u8 文件，抛错
  if (string.indexOf('#EXTM3U') !== 0) {
    this.handleManifestParsingError(
      response,
      context,
      'no EXTM3U delimiter',
      networkDetails
    );
    return;
  }

  stats.parsing.start = performance.now();
	// 如果存在 #EXT-X-TARGETDURATION，说明这是一个包含 ts 的播放列表
	// 如果不存在，说明是一个 master playlist，需要根据用户选择的播放分辨率选择不同质量的子 playlist
  if (
    string.indexOf('#EXTINF:') > 0 ||
    string.indexOf('#EXT-X-TARGETDURATION:') > 0
  ) {
    this.handleTrackOrLevelPlaylist(response, stats, context, networkDetails);
  } else {
    this.handleMasterPlaylist(response, stats, context, networkDetails);
  }
}
// parser m3u8
private handleTrackOrLevelPlaylist(
  response: LoaderResponse,
  stats: LoaderStats,
  context: PlaylistLoaderContext,
  networkDetails: any
): void {
  const hls = this.hls;
  const { id, level, type } = context;
	// ...
  const levelDetails: LevelDetails = M3U8Parser.parseLevelPlaylist(
    response.data as string,
    url,
    levelId!,
    levelType,
    levelUrlId!
  );
  context.levelDetails = levelDetails;
  this.handlePlaylistLoaded(response, stats, context, networkDetails);
}
```

`handlePlaylistLoaded` 函数触发了 `Events.LEVEL_LOADED` 事件，stream-controller 收到事件后开始请求 ts。

```typescript
// stream-controller.ts
class StreamController extends BaseStreamController {
  private onLevelLoaded(event: Events.LEVEL_LOADED, data: LevelLoadedData) {
    // ...
    // trigger handler right now
    this.tick();
  }
  // 调用 tick 后走到这里
  protected doTick() {
    switch (this.state) {
     	// 当前状态 闲置
      case State.IDLE:
        this.doTickIdle();
        break;
      // ...
    }
  }
}
class BaseStreamController extends TaskLoop {
  
}
class TaskLoop {
  private readonly _boundTick: () => void;
  private _tickCallCount = 0;
  private _tickTimer: number | null = null;
  constructor() {
    this._boundTick = this.tick.bind(this);
  }
  public clearNextTick(): boolean {
    if (this._tickTimer) {
      self.clearTimeout(this._tickTimer);
      this._tickTimer = null;
      return true;
    }
    return false;
  }
	/**
   * Will call the subclass doTick implementation in this main loop tick
   * or in the next one (via setTimeout(,0)) in case it has already been called
   * in this tick (in case this is a re-entrant call).
   */
  public tick(): void {
    this._tickCallCount++;
    if (this._tickCallCount === 1) {
      // 由 subclass 实现
      this.doTick();
      // re-entrant call to tick from previous doTick call stack
      // -> schedule a call on the next main loop iteration to process this task processing request
      // 除非多线程环境，否则我理解不会走进这里
      if (this._tickCallCount > 1) {
        // make sure only one timer exists at any time at max
        this.clearNextTick();
        this._tickTimer = self.setTimeout(this._boundTick, 0);
      }
      this._tickCallCount = 0;
    }
  } 
}
```

经过上面一系列的调用之后，函数走到了 doTickIdle，`doTickIdle -> loadFragment -> super.loadFragment -> _loadFragForPlayback` 这里我们先不关心这条调用链做了什么，总之，最后走到了 BaseStreamController  的 loadFragment 方法。

```typescript
class BaseStreamController extends TaskLoop {
  private _loadFragForPlayback(
    frag: Fragment,
    levelDetails: LevelDetails,
    targetBufferTime: number
  ) {
    // 请求 ts 后进到这个 callback
    const progressCallback: FragmentLoadProgressCallback = (
      data: FragLoadedData
    ) => {
      // 由 sub class 实现
      this._handleFragmentLoadProgress(data);
    };
    this._doFragLoad(
      frag,
      levelDetails,
      targetBufferTime,
      progressCallback
    ).then((data) => {
      // ...
      this._handleFragmentLoadComplete(data);
    });
  }
}

class StreamController extends BaseStreamController {
  protected _handleFragmentLoadProgress(data: FragLoadedData) {
    // payload 是 ts 数据
    const { frag, part, payload } = data;

    // transmux the MPEG-TS data to ISO-BMFF segments
    const transmuxer = (this.transmuxer =
      this.transmuxer ||
      new TransmuxerInterface(
        this.hls,
        PlaylistLevelType.MAIN,
      	// transmux 成功后的回调函数
        this._handleTransmuxComplete.bind(this),
        this._handleTransmuxerFlush.bind(this)
      ));
    transmuxer.push(
      payload,
      // ...
    );
  }
}
```

这个 TransmuxerInterface 是 Transmuxer 这个类对外暴露的接口，TransmuxerInterface 通过浏览器的 web worker API，将 Transmuxer 复杂的计算操作放到其他线程处理，以免阻塞 JS 线程。

```typescript
import * as work from 'webworkify-webpack';
class TransmuxerInterface {
  constructor(
    hls: Hls,
    id: PlaylistLevelType,
    onTransmuxComplete: (transmuxResult: TransmuxerResult) => void,
    onFlush: (chunkMeta: ChunkMetadata) => void
  ) {
      // 如果 work 不可用，则使用 transmuxer，二者是一样的操作
      try {
        worker = this.worker = work(
          require.resolve('../demux/transmuxer-worker.ts')
        );
        this.onwmsg = this.onWorkerMessage.bind(this);
        worker.addEventListener('message', this.onwmsg);
        // ...
      } catch (err) {
        this.transmuxer = new Transmuxer(
          this.observer,
          typeSupported,
          config,
          vendor
        );
        this.worker = null;
      }
   	}
  push (...) {
    if (worker) {
      worker.postMessage(
        {
          cmd: 'demux',
          data,
          decryptdata,
          chunkMeta,
          state,
        },
        data instanceof ArrayBuffer ? [data] : []
      );
    } else if (transmuxer) {
      // 
      const transmuxResult = transmuxer.push(
        data,
        decryptdata,
        chunkMeta,
        state
      );
      if (isPromise(transmuxResult)) {
        transmuxResult.then((data) => {
          this.handleTransmuxComplete(data);
        });
      } else {
        // transmux 结束，实际执行的是 new TransmuxerInterface 传入的 onTransmuxComplete
        this.handleTransmuxComplete(transmuxResult as TransmuxerResult);
      }
    }
  }
}
```

先看一下 onTransmuxComplete 做了什么

```typescript
// stream-controller.ts
private _handleTransmuxComplete(transmuxResult: TransmuxerResult) {
  const { remuxResult, chunkMeta } = transmuxResult;

  const { video, text, id3, initSegment } = remuxResult;
  const audio = this.altAudio ? undefined : remuxResult.audio;

  if (video && remuxResult.independent !== false) {
    // 跳过...
    this.bufferFragmentData(video, frag, part, chunkMeta);
  } else if (remuxResult.independent === false) {
    this.backtrack(frag);
    return;
  }

  // audio ...
  // id3...
}
// base-stream-controller.ts
protected bufferFragmentData(
  data: RemuxedTrack,
  frag: Fragment,
  part: Part | null,
  chunkMeta: ChunkMetadata
) {
  const { data1, data2 } = data;
  let buffer = data1;
  if (data1 && data2) {
    // Combine the moof + mdat so that we buffer with a single append
    buffer = appendUint8Array(data1, data2);
  }

  const segment: BufferAppendingData = {
    type: data.type,
    data: buffer,
    frag,
    part,
    chunkMeta,
  };
  this.hls.trigger(Events.BUFFER_APPENDING, segment);
}
// buffer-controller.ts
// hls.on(Events.BUFFER_APPENDING, this.onBufferAppending, this);
protected onBufferAppending(
  event: Events.BUFFER_APPENDING,
  eventData: BufferAppendingData
) {
  // data 是上面经过 transmuxer 处理后的数据
  const { data, type, frag, part, chunkMeta } = eventData;

  const operation: BufferOperation = {
    execute: () => {
      // ...
      this.appendExecutor(data, type);
    },
    // ...
  };
  // append 后，经过一系列操作，会触发 operation.execute
  operationQueue.append(operation, type);
}
// sourcebuffer append
private appendExecutor(data: Uint8Array, type: SourceBufferName) {
  const { operationQueue, sourceBuffer } = this;
  const sb = sourceBuffer[type];
  sb.ended = false;
  sb.appendBuffer(data);
}
```

上面的过程可以简化成：加载 ts 数据 -> 解析 ts 数据，将其中的音视频数据拿出来 -> 调用 MSE 接口，将数据 append 到 sourcebuffer。

### transmuxer

继续刚才的 transmuxer 代码

```typescript
// transmuxer.ts
push(
  data: ArrayBuffer,
  decryptdata: LevelKey | null,
  chunkMeta: ChunkMetadata,
  state?: TransmuxState
): TransmuxerResult | Promise<TransmuxerResult> {
  let uintData: Uint8Array = new Uint8Array(data);
  // ...
	// 最终走到 transmuxUnencrypted 函数
  const result = this.transmux(
    uintData,
    keyData,
    timeOffset,
    accurateTimeOffset,
    chunkMeta
  );
	// ...
  return result;
}

private transmuxUnencrypted(
  data: Uint8Array,
  timeOffset: number,
  accurateTimeOffset: boolean,
  chunkMeta: ChunkMetadata
): TransmuxerResult {
  // 将 ts 格式里的 音视频数据提取出来
  const { audioTrack, avcTrack, id3Track, textTrack } = (this.demuxer as Demuxer).demux(data, timeOffset, false);
  // 在混流成浏览器能播放的格式
  const remuxResult = this.remuxer!.remux(
    audioTrack,
    avcTrack,
    id3Track,
    textTrack,
    timeOffset,
    accurateTimeOffset,
    false
  );
  return {
    remuxResult,
    chunkMeta,
  };
}
```

可以看出，transmuxer 分为两步，Demux 和 Remux。

Mux 是 Multiplex 的缩写，在这里是「混流」的意思，Demux 就是将混流解开的操作，将混流里的音视频，字幕等数据解开，在 Remux 成浏览器能播放的视频格式。而不同格式的 demux 和 remux 方法也不一样

```ts
// transmuxer.ts
const muxConfig: MuxConfig[] = [
  { demux: TSDemuxer, remux: MP4Remuxer },
  { demux: MP4Demuxer, remux: PassThroughRemuxer },
  { demux: AACDemuxer, remux: MP4Remuxer },
  { demux: MP3Demuxer, remux: MP4Remuxer },
];
```

拿 TSDemuxer 简单的分析一下，可以看出和上一篇 [hls 协议详解](https://idmrchan.com/2021/02/19/hls-protocol-analysis/) 描述的一致

```typescript
// tsdemuxer.ts
public demux(
  data: Uint8Array,
  timeOffset: number,
  isSampleAes = false,
  flush = false
): DemuxerResult {
  if (!isSampleAes) {
    this.sampleAes = null;
  }

  let pes: PES | null;

  const avcTrack = this._avcTrack;
  const audioTrack = this._audioTrack;
  const id3Track = this._id3Track;

  let avcId = avcTrack.pid;
  let avcData = avcTrack.pesData;
  let audioId = audioTrack.pid;
  let id3Id = id3Track.pid;
  let audioData = audioTrack.pesData;
  let id3Data = id3Track.pesData;
  let unknownPIDs = false;
  let pmtParsed = this.pmtParsed;
  let pmtId = this._pmtId;

  let len = data.length;
  if (this.remainderData) {
    data = appendUint8Array(this.remainderData, data);
    len = data.length;
    this.remainderData = null;
  }
  // 一个 Packet 最少是 188 字节，如果少于 188，就存下来，加在下个片段
  if (len < 188 && !flush) {
    this.remainderData = data;
    return {
      audioTrack,
      avcTrack,
      id3Track,
      textTrack: this._txtTrack,
    };
  }

  const syncOffset = Math.max(0, TSDemuxer.syncOffset(data));
  // 把剩余不满 188 的部分存下来，放到下一个 ts 一起解析
  len -= (len + syncOffset) % 188;
  if (len < data.byteLength && !flush) {
    this.remainderData = new Uint8Array(
      data.buffer,
      len,
      data.buffer.byteLength - len
    );
  }

  /**
   * loop through TS packets
   * 第1个字节， sync byte，固定为 0x47
   * 第2个字节，前三位分别是 Transport Error Indicator (TEI)，Payload Unit Start Indicator，Transport Priority
   * 第2个字节后 5 位 + 第3个字节 8 位 = 13 位，为 PID
   * 第4个字节，分别是 2位 Transport Scrambling control (TSC)，2位 Adaptation field exist，4位 Continuity counter
   */
  for (let start = syncOffset; start < len; start += 188) {
    if (data[start] === 0x47) {
      // 0x40 0100 0000。判断 Payload Unit Start Indicator 是否为 1
      // 一个 ts 包往往放不下 PES 包的，那么需要截取发送，就是通过这个字段区分，如果为 1，说明这个 ts 是一个包头
      const stt = !!(data[start + 1] & 0x40);
      // 取 data[start + 1] 的后 5 位和 data[str+2] 的 8 位组合成 PID
      // pid is a 13-bit field starting at the last bit of TS[1]
      const pid = ((data[start + 1] & 0x1f) << 8) + data[start + 2];
      // Adaptation field exist 适配域存在标识
      const atf = (data[start + 3] & 0x30) >> 4;

      // if an adaption field is present, its length is specified by the fifth byte of the TS packet header.
      let offset: number;
      // atf 2 bit： 第1位表示是否有 Adaptation field，第2位表示是否有 payload
      // 第1位为1，
      if (atf > 1) {
        // data[start + 4] 表示 Adaptation field 占多少个字节长度
        // demux 不关心 Adaptation field 里的内容，跳过
        offset = start + 5 + data[start + 4];
        // continue if there is only adaptation field
        if (offset === start + 188) {
          continue;
        }
      } else {
        // 跳过前 4 个字节（ TS 包开头的固定部分）
        offset = start + 4;
      }
      switch (pid) {
        // 视频流 PID，通过解 PMT 包得出
        case avcId:
          if (stt) {
            // 如果是包头，需要解 PES 格式
            if (avcData && (pes = parsePES(avcData))) {
              this.parseAVCPES(pes, false);
            }

            avcData = { data: [], size: 0 };
          }
          // Video ES 数据
          if (avcData) {
            avcData.data.push(data.subarray(offset, start + 188));
            avcData.size += start + 188 - offset;
          }
          break;
        case audioId:
          if (stt) {
            if (audioData && (pes = parsePES(audioData))) {
              if (audioTrack.isAAC) {
                this.parseAACPES(pes);
              } else {
                this.parseMPEGPES(pes);
              }
            }
            audioData = { data: [], size: 0 };
          }
          if (audioData) {
            audioData.data.push(data.subarray(offset, start + 188));
            audioData.size += start + 188 - offset;
          }
          break;
        case id3Id:
          if (stt) {
            if (id3Data && (pes = parsePES(id3Data))) {
              this.parseID3PES(pes);
            }

            id3Data = { data: [], size: 0 };
          }
          if (id3Data) {
            id3Data.data.push(data.subarray(offset, start + 188));
            id3Data.size += start + 188 - offset;
          }
          break;
        // 0x0000, PAT 节目关联表 PID
        case 0:
          if (stt) {
            // 当前的 offset 跳过了 TS 包开头的固定部分，接下来的部分是数据包
            // 数据包的第一个字节是 pointer_field（程序特殊信息指针），表示数据从哪里开始
            // 这个字节数据都是 0x00
            offset += data[offset] + 1;
          }

          pmtId = this._pmtId = parsePAT(data, offset);
          break;
        case pmtId: {
          if (stt) {
            offset += data[offset] + 1;
          }

          const parsedPIDs = parsePMT(
            data,
            offset,
            this.typeSupported.mpeg === true ||
              this.typeSupported.mp3 === true,
            isSampleAes
          );

          // only update track id if track PID found while parsing PMT
          // this is to avoid resetting the PID to -1 in case
          // track PID transiently disappears from the stream
          // this could happen in case of transient missing audio samples for example
          // NOTE this is only the PID of the track as found in TS,
          // but we are not using this for MP4 track IDs.
          avcId = parsedPIDs.avc;
          if (avcId > 0) {
            avcTrack.pid = avcId;
          }

          audioId = parsedPIDs.audio;
          if (audioId > 0) {
            audioTrack.pid = audioId;
            audioTrack.isAAC = parsedPIDs.isAAC;
          }
          id3Id = parsedPIDs.id3;
          if (id3Id > 0) {
            id3Track.pid = id3Id;
          }

          if (unknownPIDs && !pmtParsed) {
            logger.log('reparse from beginning');
            unknownPIDs = false;
            // we set it to -188, the += 188 in the for loop will reset start to 0
            start = syncOffset - 188;
          }
          pmtParsed = this.pmtParsed = true;
          break;
        }
        case 17:
        case 0x1fff:
          break;
        default:
          unknownPIDs = true;
          break;
      }
    } else {
      this.observer.emit(Events.ERROR, Events.ERROR, {
        type: ErrorTypes.MEDIA_ERROR,
        details: ErrorDetails.FRAG_PARSING_ERROR,
        fatal: false,
        reason: 'TS packet did not start with 0x47',
      });
    }
  }

  avcTrack.pesData = avcData;
  audioTrack.pesData = audioData;
  id3Track.pesData = id3Data;

  return {
    audioTrack,
    avcTrack,
    id3Track,
    textTrack: this._txtTrack,
  };
}
// 这个库没有去循环调用获取 PMT 节目表，仅取了第一个节目出来
function parsePAT(data, offset) {
  // skip the PSI header and parse the first PMT entry
  // 0x1f -> 0001 1111
  // 取 13 位
  return ((data[offset + 10] & 0x1f) << 8) | data[offset + 11];
}
/**
 * 解析 PMT
 * 第1个字节是 table id
 * 第2个字节后四位 + 第3个字节 = section length
 * 第4，5字节表示 program number
 * 第6，7，8个字节表示固定值，version number，section number 等信息
 * 第9个字节的后5位 + 第10个字节 = PCR PID
 * 第11个字节后4位 + 第12个字节 = 产品描述字节的长度
 * 后面开始循环
 * =====
 * 第1个字节是 stream type，描述流类型
 * 第2字节后5位 + 第3字节 = 流 PID
 * 第4字节后4位 + 第5，6字节 = ES 信息长度
 * ES 信息
 * ====
 * CRC32
 */
function parsePMT(data, offset, mpegSupported, isSampleAes) {
  const result = { audio: -1, avc: -1, id3: -1, isAAC: true };
  // 0x0f -> 0000 1111
  // section 长度取第二个字节后四位 + 第三个字节
  const sectionLength = ((data[offset + 1] & 0x0f) << 8) | data[offset + 2];
  // sectionLength 是从当前算到 CRC 32为止，所以需要 offset +3
  // -4 是为了不包含 CRC32
  const tableEnd = offset + 3 + sectionLength - 4;
  // to determine where the table is, we have to figure out how
  // long the program info descriptors are
  // 产品描述长度
  const programInfoLength =
    ((data[offset + 10] & 0x0f) << 8) | data[offset + 11];
  // advance the offset to the first entry in the mapping table
  offset += 12 + programInfoLength;
  // 循环 N loop
  while (offset < tableEnd) {
    const pid = ((data[offset + 1] & 0x1f) << 8) | data[offset + 2];
    switch (data[offset]) {
      case 0xcf: // SAMPLE-AES AAC
        if (!isSampleAes) {
          logger.log(
            'ADTS AAC with AES-128-CBC frame encryption found in unencrypted stream'
          );
          break;
        }
      /* falls through */
      case 0x0f: // ISO/IEC 13818-7 ADTS AAC (MPEG-2 lower bit-rate audio)
        if (result.audio === -1) {
          result.audio = pid;
        }
        break;
      case 0x15:
        if (result.id3 === -1) {
          result.id3 = pid;
        }
        break;
      case 0xdb: // SAMPLE-AES AVC
        if (!isSampleAes) {
          logger.log(
            'H.264 with AES-128-CBC slice encryption found in unencrypted stream'
          );
          break;
        }
      /* falls through */
      case 0x1b: // ITU-T Rec. H.264 and ISO/IEC 14496-10 (lower bit-rate video)
        if (result.avc === -1) {
          result.avc = pid;
        }
        break;
      // ISO/IEC 11172-3 (MPEG-1 audio)
      // or ISO/IEC 13818-3 (MPEG-2 halved sample rate audio)
      case 0x03:
      case 0x04:
        if (!mpegSupported) {
          logger.log('MPEG audio found, not supported in this browser');
        } else if (result.audio === -1) {
          result.audio = pid;
          result.isAAC = false;
        }
        break;
      case 0x24:
        logger.warn('Unsupported HEVC stream type found');
        break;
      default:
        // logger.log('unknown stream type:' + data[offset]);
        break;
    }
    // move to the next table entry
    // skip past the elementary stream descriptors, if present
    offset += (((data[offset + 3] & 0x0f) << 8) | data[offset + 4]) + 5;
  }
  return result;
}
```





