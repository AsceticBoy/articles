# learning-canvas

- `canvas` 大小具有 `绘图区域` 和 `画布大小(元素大小)` 两个概念，当你指定 `canvas属性width或height` 时会同时修改 `绘图区域和画布大小`，若通过 `CSS` 方式修改，只会改变 `画布大小`

- `console.profile` 和 `console.profileEnd` 可直接用于调试 `Chrome Tool Profiler`

- `canvas是一个不可获取焦点的元素`，需要监听键盘事件的话，需要挂载到 `document` 或 `window` 上，键盘事件触发顺序：`keydown -> keypress -> keyup`，其中 `keypress` 事件只有在能打印出某个字符的键才会触发

- `canvas` 是一种 `立即模式` 的绘图系统，不会保存图形对象列表，是一种更底层的绘图模式，而 `SVG` 属于 `保留模式` 会将维护一份图形对象列表