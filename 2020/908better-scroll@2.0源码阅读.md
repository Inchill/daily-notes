better-scroll@2.0 版本在最近发布了，其初衷来源于社区的一个需求 —— better-scroll能否支持按需加载。在 better-scroll@1.0 版本中，所有的 feature 都是通过 options 选项并且通过一个 BScroll 类以及扩展原型方法来实现的，这显然是不支持按需加载的。

BetterScroll 2.0 采用了插件化的架构设计。CoreScroll 作为最小的滚动单元，暴露了丰富的事件以及钩子，其余的功能都由不同的插件来扩展，这样会让 BetterScroll 更加的灵活，也能解耦不同的场景。下面是整体的架构图：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e940d8a3b1d04c24a356cc5ccb9d9ecd~tplv-k3u1fbpfcp-zoom-1.image" />

## better-scroll 源码

具备完整插件能力的 BetterScroll，不用关心各种插件注册的细节。在这里，core 是核心滚动部分，它作为 BetterScroll 的最小使用单元，压缩体积比 1.0 小了将近三分之一。在 core 的基础上，注册了所有插件，对外暴露了一个完整的 better-scroll。

看到这里我有一个疑问：为什么 BS 通过 use 方法注册了所有插件，还要通过 export 暴露一次（export 导出后，导入需要花括号按需引入）。问了 2.0 作者，他说因为这里不用的话，typescript不会生成对应的d.ts。想了想，自己还得去系统学习一下 ts。

```typescript
import BScroll from '@better-scroll/core'
import MouseWheel from '@better-scroll/mouse-wheel'
import ObserveDom from '@better-scroll/observe-dom'
import PullDownRefresh from '@better-scroll/pull-down'
import PullUpLoad from '@better-scroll/pull-up'
import ScrollBar from '@better-scroll/scroll-bar'
import Slide from '@better-scroll/slide'
import Wheel from '@better-scroll/wheel'
import Zoom from '@better-scroll/zoom'
import NestedScroll from '@better-scroll/nested-scroll'
import InfinityScroll from '@better-scroll/infinity'
import Movable from '@better-scroll/movable'

export {
  createBScroll,
  BScrollInstance,
  Options,
  CustomOptions,
  TranslaterPoint,
  MountedBScrollHTMLElement,
  Behavior,
  Boundary,
  CustomAPI
} from '@better-scroll/core'

export {
  MouseWheel,
  ObserveDom,
  PullDownRefresh,
  PullUpLoad,
  ScrollBar,
  Slide,
  Wheel,
  Zoom,
  NestedScroll,
  InfinityScroll,
  Movable
}

BScroll.use(MouseWheel)
  .use(ObserveDom)
  .use(PullDownRefresh)
  .use(PullUpLoad)
  .use(ScrollBar)
  .use(Slide)
  .use(Wheel)
  .use(Zoom)
  .use(NestedScroll)
  .use(InfinityScroll)
  .use(Movable)

export default BScroll
```
