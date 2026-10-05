# GeoStudy (MVP 1.0) 前端业务与系统开发文档

## 1. 系统架构与技术选型

为了确保针对海外 K12 课堂与 Homeschool 场景实现**零延迟、免登录、免后端依赖**的轻量化体验，MVP 采用纯静态架构。

* **渲染引擎**：MapLibre GL JS（免费、开源、高性能矢量 WebGL 地图，支持流畅缩放与平移）。
* **底图样式 (Tile Source)**：OpenStreetMap 矢量瓦片（通过 CARTO / Stadia / MapTiler 样式转换为简约学术风风格）。
* **数据解析**：`PapaParse`（解析 CSV 题库）或原生 `fetch`（加载 JSON 题库）。
* **截图生成**：`html2canvas`（支持将成绩结算 DOM 节点转换为 PNG 动态下载）。
* **UI & 样式框架**：Tailwind CSS + Headless UI（响应式、简约学术风，适配 PC / 平板）。

---

## 2. 核心数据结构与题库规范 (Data Specification)

题库既可存储于本地 `questions.json`，也可通过 Google Sheets 导出为 CSV 链接。每道题目包含跨学科线索与知识点卡片。

### 2.1 数据 JSON 结构示例
```json
[
  {
    "id": "hist-geo-001",
    "subject": "Both",
    "topic": "Silk Road & Trade Networks",
    "target_name": "Samarkand",
    "lat": 39.6542,
    "lng": 66.9597,
    "clue_1": "I was the legendary capital of Timur's empire and a critical oasis city where Sogdian merchants traded silk, spices, and ideas.",
    "clue_2": "Located in the fertile valley of the Zarafshan River, I sit at the crossroads between China, Persia, and the Mediterranean.",
    "clue_3": "Today, I am the second-largest city in Uzbekistan, famous for my turquoise-tiled Registan Mosque.",
    "fact_sheet": {
      "history": "Flourished as a primary Silk Road trading hub from 1000 BCE. Destroyed by Genghis Khan in 1220 and rebuilt as the grand capital of the Timurid Empire.",
      "geography": "Its location in a mountain oasis provided fertile farmland and vital water sources in an otherwise arid region."
    },
    "grade_level": "Grade 6-9"
  }
]
```

---

## 3. 核心计算与业务逻辑模块

### 3.1 距离与计分算法 (Haversine Formula)

基于球面大圆距离算法计算用户点击坐标 $(lat_1, lng_1)$ 与目标坐标 $(lat_2, lng_2)$ 的实际公里数，并通过衰减映射计算得出 0~100 分。

```javascript
// 计算两点间的球面大圆距离（单位：公里）
function calculateDistance(lat1, lon1, lat2, lon2) {
  const R = 6371; // 地球平均半径 (km)
  const dLat = (lat2 - lat1) * (Math.PI / 180);
  const dLon = (lon2 - lon1) * (Math.PI / 180);
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(lat1 * (Math.PI / 180)) *
      Math.cos(lat2 * (Math.PI / 180)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c; // 返回 km
}

// 计分函数：误差 50km 内得 100分，超出 2000km 得 0分
function calculateScore(distanceKm) {
  if (distanceKm <= 50) return 100;
  if (distanceKm >= 2000) return 0;
  // 线性衰减逻辑（可调整为指数衰减曲线）
  const score = 100 - ((distanceKm - 50) / (2000 - 50)) * 100;
  return Math.round(score);
}
```

### 3.2 每日一练与 Cookie/LocalStorage 换题机制

为了保证学生同一天访问看到的是同一套固定题目、次日自动刷新的逻辑：

```javascript
function getDailySeed() {
  const today = new Date();
  // 生成 YYYY-MM-DD 格式作为 Hash 种子
  return `${today.getFullYear()}-${today.getMonth() + 1}-${today.getDate()}`;
}

function getDailyQuestions(allQuestions, count = 5) {
  const seedStr = getDailySeed();
  // 基于 seed 伪随机打乱数组
  let hash = 0;
  for (let i = 0; i < seedStr.length; i++) {
    hash = seedStr.charCodeAt(i) + ((hash << 5) - hash);
  }
  
  const shuffled = [...allQuestions].sort((a, b) => {
    const pseudoRandom = Math.sin(hash++) * 10000;
    return (pseudoRandom - Math.floor(pseudoRandom)) - 0.5;
  });

  return shuffled.slice(0, count);
}
```

### 3.3 成绩单 PNG 生成与 WebGL 避坑方案

因为 MapLibre 的 Canvas 启用了 WebGL，如果把 WebGL 地图节点直接打入截图容易导致黑屏。**最佳实践**是只对“独立 DOM 渲染的成绩结算卡片”做截图生成，不把地图 Canvas 打进 PNG 里：

```javascript
import html2canvas from 'html2canvas';

async function downloadReportCard(elementId) {
  const cardElement = document.getElementById(elementId);
  if (!cardElement) return;

  const canvas = await html2canvas(cardElement, {
    scale: 2, // 提高 2 倍分辨率，保障高清打印/保存
    useCORS: true,
    backgroundColor: '#ffffff'
  });

  const image = canvas.toDataURL('image/png');
  const link = document.createElement('a');
  link.href = image;
  link.download = `GeoStudy_Score_${new Date().toISOString().slice(0, 10)}.png`;
  link.click();
}
```

---

## 4. 前端交互页面流 (Page Workflow)

```
[首页 Header/Hero] 
   └── 点击 "Start Today's Quiz"
          ↓
[答题页 Main Workspace] 
   ├── 侧边/顶部栏: 渲染 3 阶梯 Clues (Clue 1 -> Clue 2 -> Clue 3 逐级解锁)
   ├── 中央区域: MapLibre 交互地图
   └── 地图点击事件触发:
          ├── 在地图标记用户点击点与真实点，画虚线连线
          ├── 计算距离与得分，弹出 [知识点卡片弹窗 (Fact Sheet Modal)]
          └── 点击 "Next Question" (重复 5 题)
          ↓
[结算页 Results Screen]
   ├── 显示总分、平均误差公里数、评语 (如: "Master Geographer!")
   ├── 功能按钮区:
   │      ├── 一键复制练习链接 (Clipboard API)
   │      ├── 一键复制成绩文案 (Clipboard API)
   │      └── 点击生成下载成绩单图片 (html2canvas)
   └── 底部/侧边: 渲染非打扰式 Banner 广告位 (无弹窗)
```

---

## 5. Playwright 自动化测试与交付方案 (E2E Test)

为了确保本地部署和代码打包交付后不是“空架子”，可配套使用 Playwright 脚本进行全功能覆盖测试。

### 5.1 `tests/geostudy.spec.js` 脚本示例

```javascript
const { test, expect } = require('@playwright/test');

test.describe('GeoStudy MVP 全流程自动化测试', () => {
  
  test.beforeEach(async ({ page }) => {
    // 假设本地服务启动在 8080 端口
    await page.goto('http://localhost:8080/puzzle/geostudy/');
  });

  test('1. 页面基本加载与无 JS 报错校验', async ({ page }) => {
    await expect(page).toHaveTitle(/GeoStudy/);
    const mapContainer = page.locator('[data-testid="map-container"]');
    await expect(mapContainer).toBeVisible();
  });

  test('2. 地图点击交互与答题逻辑', async ({ page }) => {
    // 点击开始答题按钮
    await page.click('[data-testid="start-btn"]');
    
    // 校验线索 1 是否正确渲染
    const clueText = page.locator('[data-testid="clue-1"]');
    await expect(clueText).not.toBeEmpty();

    // 模拟点击地图容器（点击中心区域）
    const map = page.locator('[data-testid="map-container"]');
    await map.click({ position: { x: 300, y: 200 } });

    // 校验知识点卡片弹窗是否弹出
    const factModal = page.locator('[data-testid="fact-card-modal"]');
    await expect(factModal).toBeVisible();

    // 校验分数显示
    const scoreText = page.locator('[data-testid="question-score"]');
    await expect(scoreText).toContainText(/Score:/);
  });

  test('3. 完成 5 题后，结算页与分享校验', async ({ page, context }) => {
    await page.click('[data-testid="start-btn"]');

    // 循环模拟完成 5 题
    for (let i = 0; i < 5; i++) {
      await page.locator('[data-testid="map-container"]').click({ position: { x: 300, y: 200 } });
      await page.click('[data-testid="next-btn"]');
    }

    // 校验是否跳转至结果页
    const resultPage = page.locator('[data-testid="result-screen"]');
    await expect(resultPage).toBeVisible();

    // 校验复制链接逻辑
    await context.grantPermissions(['clipboard-read', 'clipboard-write']);
    await page.click('[data-testid="copy-link-btn"]');
    const clipboardText = await page.evaluate(() => navigator.clipboard.readText());
    expect(clipboardText).toContain('http');
  });
  
});
```

### 5.2 本地运行命令
1. 安装依赖：`npm install -D @playwright/test`
2. 执行全自动化测试：`npx playwright test`
3. 查看测试报告：`npx playwright show-report`

---

## 6. 上线验收与交付清单 (Checklist)

1. **DOM 节点 testid 标注**：确保关键元素标注了 `data-testid`（如 `map-container`、`start-btn`、`next-btn`、`fact-card-modal`），便于 Playwright 稳定抓取。
2. **知识点校验 (Content QA)**：确保第一批 20 道题的 `lat`/`lng` 经过 Google Maps 精确核对，且文本阅读难度控制在 Lexile 800L~1100L 内。
3. **响应式断点**：针对平板（768px - 1024px）进行 CSS 样式适配，保证课堂投屏无错位。