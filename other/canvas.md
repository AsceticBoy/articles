# learning-canvas

- `canvas` 大小具有 `绘图区域` 和 `画布大小(元素大小)` 两个概念，当你指定 `canvas属性width或height` 时会同时修改 `绘图区域和画布大小`，若通过 `CSS` 方式修改，只会改变 `画布大小`

- `console.profile` 和 `console.profileEnd` 可直接用于调试 `Chrome Tool Profiler`

- `canvas是一个不可获取焦点的元素`，需要监听键盘事件的话，需要挂载到 `document` 或 `window` 上，键盘事件触发顺序：`keydown -> keypress -> keyup`，其中 `keypress` 事件只有在能打印出某个字符的键才会触发

- `canvas` 是一种 `立即模式` 的绘图系统，不会保存图形对象列表，是一种更底层的绘图模式，而 `SVG` 属于 `保留模式` 会将维护一份图形对象列表

- `canvasContext.arc` 方法用来表示在当前路径中增加一段圆弧或者圆形的子路径，所以需要注意点是，当在调用时，如果已经存在其他的子路径，那么该方法会将已有子路径的终点和圆弧的起点连接起来，`beginPath` 会擦除之前的所有子路径，可以防止这种效果

- `canvasContext.fill` 如果当前路径是循环的，或者包含多个相交的子路径(这是前提)，在填充路径时，遵循 `非零环绕规则` 即：若要判断某一区域是否属于填充范围，只需要从该区域朝一个方向发出无限长的射线，从零开始计数，`遇射线与路径相交点处于路径顺时针方向则加一`，`遇射线与路径相交点处于路径逆时针方向则减一`，若最终结果为零，则视为该区域不处于填充范围