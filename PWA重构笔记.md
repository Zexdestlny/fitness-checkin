# 健身打卡（时砺）PWA 重构笔记

## 项目背景

原项目是一个 React Native / Expo 原生 App，目标是发布到 iPhone 使用，但被苹果开发者计划阻挡。于是决定将项目重构为 PWA（Progressive Web App），通过 Safari 的"添加到主屏幕"功能实现类似原生 App 的体验。

**核心目标不变**：所有数据仍在本地存储，不需要服务器。

---

## 整体架构思路

### 从 React Native 到 PWA 的路径选择

有两条路：

1. **完全重写** — 用纯 HTML/CSS/JS 重写整个项目
2. **复用现有代码** — 利用 Expo 已有的 Web 支持，适配兼容性问题

选择了方案 2，原因：
- 项目已经有 `react-native-web` 和 `react-dom` 依赖
- 5 个页面 + 多个组件的代码量不小，重写成本高
- Expo Web 构建已经能生成 `dist/` 静态文件
- 只需要解决几个关键的 Web 兼容问题

### 技术栈

| 层 | 技术 |
|---|---|
| UI 框架 | React Native + react-native-web |
| 导航 | @react-navigation/bottom-tabs |
| 存储 | AsyncStorage → Web 端自动降级为 localStorage |
| 图表 | react-native-svg（替代 react-native-chart-kit） |
| 图标 | @expo/vector-icons (Ionicons) |
| 构建 | Expo export --platform web |
| 部署 | GitHub Pages（免费 HTTPS） |

---

## 遇到的问题与解决思路

### 问题 1：原生日期选择器不兼容 Web

**现象**：`@react-native-community/datetimepicker` 是原生模块，Web 端无法使用。

**思路**：
- 利用 Metro 打包器的 **平台文件扩展名** 机制（`.web.js` / `.native.js`）
- 创建两个同名组件 `DatePicker.web.js` 和 `DatePicker.native.js`
- Metro 会根据构建目标自动选择正确的文件

**实现**：
- `DatePicker.native.js`：保留原有的 `DateTimePicker`
- `DatePicker.web.js`：使用 HTML 原生 `<input type="date">`，这是浏览器内置的日期选择器，在 Safari 上有原生级体验

**注意点**：
- HTML `<input>` 可以直接在 React Native Web 的 JSX 中使用，因为 react-native-web 的 JSX 运行时允许 HTML 元素和 RN 组件混用
- `<input>` 上的 `style` 属性需要同时兼容 RN 样式和 DOM 样式

### 问题 2：图表库不兼容 Web

**现象**：`react-native-chart-kit` 依赖 `Dimensions.get('window')` 硬编码宽度，且未验证 Web 兼容性。

**思路**：
- 直接用 `react-native-svg`（项目已有此依赖）自绘图表
- SVG 是 Web 原生支持的，不需要额外适配
- 使用 `onLayout` 动态获取容器宽度，替代硬编码的 `Dimensions`

**实现**：
- `TrendLineChart`：用 `<Polyline>` + `<Circle>` 绘制折线图，底部用 `<Path>` 绘制填充区域
- `StatsBarChart`：用 `<Rect>` 绘制柱状图，顶部标注数值
- `TypePieChart`：用 SVG `<Path>` 的弧线命令 `A` 绘制饼图扇区，附带图例

**核心技巧**：
```
// 用 onLayout 获取容器宽度，替代 Dimensions
function useChartLayout() {
  const [width, setWidth] = useState(300);
  const onLayout = (e) => setWidth(e.nativeEvent.layout.width);
  return { width, onLayout };
}
```

### 问题 3：启动动画在 Web 端崩溃

**现象**：`SplashScreen.js` 使用 `useNativeDriver: true`，Web 端不支持原生驱动动画。

**思路**：
- RN 的 Animated API 支持两种驱动：Native Driver（原生线程）和 JS Driver（JS 线程）
- Web 端只有 JS 线程，所以 `useNativeDriver: true` 会报错
- 用 `Platform.OS !== 'web'` 动态切换

**修改**：将所有 `useNativeDriver: true` 替换为 `useNativeDriver: Platform.OS !== 'web'`

### 问题 4：Vercel 域名在国内无法访问

**现象**：部署到 Vercel 后，`vercel.app` 域名在国内超时。

**思路**：Vercel 的默认域名在国内 DNS 解析有问题。换成 GitHub Pages：
- `github.io` 域名在国内通常可访问
- 同样免费、自带 HTTPS
- 适合静态站点

### 问题 5：GitHub Pages 上页面空白（404 资源加载失败）

**现象**：HTML 加载了但 JS bundle 返回 404。

**根因**：JS bundle 放在 `_expo/` 文件夹下，GitHub Pages 默认启用 Jekyll，会忽略以 `_` 开头的文件夹。

**修复**：在根目录创建一个空的 `.nojekyll` 文件，告诉 GitHub Pages 不要用 Jekyll 处理。

### 问题 6：绝对路径 vs 相对路径

**现象**：部署到 GitHub Pages 项目页（`username.github.io/repo-name/`）后，资源路径 `/icon.png` 会解析到 `username.github.io/icon.png` 而非 `username.github.io/repo-name/icon.png`。

**思路**：将所有绝对路径改为相对路径。
- `/icon.png` → `./icon.png`
- `/manifest.json` → `./manifest.json`
- `/_expo/static/...` → `./_expo/static/...`

同时修改 `manifest.json` 的 `start_url` 和 `sw.js` 的缓存路径。

### 问题 7：底部导航图标和关闭图标丢失

**现象**：页面出现了但图标都显示为方块/空白。

**根因**：`@expo/vector-icons` (Ionicons) 在 Web 上通过 CSS `@font-face` 加载图标字体。静态导出的 HTML 中没有 `@font-face` 声明，浏览器不知道去哪里加载字体文件。

**修复**：在 HTML `<head>` 中添加 `@font-face` 声明，指向 dist 中的字体文件：
```css
@font-face {
  font-family: Ionicons;
  src: url(./assets/node_modules/expo/node_modules/@expo/vector-icons/build/vendor/react-native-vector-icons/Fonts/Ionicons.xxx.ttf) format('truetype');
}
```

需要为每个使用的图标集（Ionicons、MaterialIcons、MaterialCommunityIcons 等）添加对应的声明。

---

## PWA 配置要点

### manifest.json

```json
{
  "name": "时砺 - 健身打卡",
  "short_name": "时砺",
  "start_url": "./",
  "display": "standalone",    // 全屏无浏览器 UI
  "orientation": "portrait",   // 竖屏
  "background_color": "#1a1a2e",
  "theme_color": "#1a1a2e",
  "icons": [
    { "src": "./icon-192.png", "sizes": "192x192" },
    { "src": "./icon-512.png", "sizes": "512x512", "purpose": "any maskable" }
  ]
}
```

### Apple 专用 Meta 标签

```html
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="apple-mobile-web-app-title" content="时砺" />
<link rel="apple-touch-icon" href="./icon-192.png" />
```

### Service Worker

提供离线缓存策略（Cache-First）：
1. `install` 事件：预缓存核心资源
2. `fetch` 事件：优先从缓存返回，同时更新缓存

---

## 项目文件结构

```
fitness-checkin/
├── App.js                    # 入口组件，Tab 导航
├── app.json                  # Expo 配置
├── package.json              # 依赖 & build-pwa 脚本
├── src/
│   ├── screens/              # 5 个页面
│   ├── components/           # 可复用组件
│   │   ├── DatePicker.native.js  # 原生日期选择器
│   │   ├── DatePicker.web.js     # Web 日期选择器
│   │   ├── ChartView.js          # SVG 图表（折线/柱状/饼图）
│   │   └── SplashScreen.js       # 启动动画
│   ├── storage/              # AsyncStorage 数据层
│   └── utils/                # 常量和工具函数
├── web/                      # Web 静态资源
│   ├── manifest.json
│   └── sw.js
├── scripts/
│   └── pwa-postbuild.js      # 构建后自动化脚本
└── dist/                     # 构建产物（GitHub Pages 根目录）
```

---

## 日常使用命令

```bash
# 构建 PWA
npm run build-pwa

# 推送到 GitHub Pages
cd dist
git add -A && git commit -m "update" && git push
```

---

## 关键经验总结

1. **Expo Web 是 RN 转 PWA 的捷径** — 不需要重写代码，只需解决兼容性问题
2. **平台文件扩展名**（`.web.js` / `.native.js`）是处理平台差异的最优雅方式
3. **react-native-svg** 是 Web/Native 双端都完美支持的图表方案
4. **相对路径**是部署到子目录（GitHub Pages 项目页）的关键
5. **`.nojekyll`** 文件对 GitHub Pages 托管 Expo 项目至关重要
6. **`@font-face`** 是 Web 端图标字体加载的必要配置
7. **`useNativeDriver: Platform.OS !== 'web'`** 是处理动画兼容性的标准写法
