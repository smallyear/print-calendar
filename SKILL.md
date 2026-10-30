---
name: print-calendar
description: 生成可打印在一张A4纸上的中国工作日历网页(HTML),含农历、节气、法定节假日、调休补班、周末,并在底部附披露易上传时间提醒。当用户想打印/生成某个时间段(如"下个月到三月底")的日历、工作日历、节假日日历时使用。
---

# 打印用工作日历生成技能

为用户生成一个**单文件 HTML 日历**,可直接浏览器打印到一张 A4 纸上,方便贴在桌前查看工作日、周末、节假日。

## 固定的格式要求(用户已确认的偏好)

- A4 **竖版**,**两个月一行**(4个月=2行,6个月=3行)
- 每格显示**公历 + 农历**(初一显示月份;节日红字,节气绿字)
- 颜色:工作日白、周末蓝、法定节假日红、调休上班橙、预计假期(官方未公布)红色虚线框
- 使用**北京时间**,星期以周日开头
- 日期范围外的格子留空(例如从 1月21日开始,则1月1-20日为空白格)
- **页面最底部必须加这条提醒**(蓝色提示框):
  `其他:披露易每天 6:00-8:30、12:00-12:30、16:30-23:00 允许发行人上传报告`

## 工作流程

1. **确定日期范围**:从用户话里取起止日期。用系统给出的当前日期理解"下个月""下一段时间"。范围不明确时,只问一个问题确认起止日期。年份一定以当前日期为准,不要凭印象。
2. **查官方放假安排**:用网络搜索查"国务院办公厅关于 XXXX 年部分节假日安排的通知",覆盖范围涉及的每一年。取得:放假日期、调休上班日。
   - 官方尚未公布(通常 11 月才公布次年安排)时,不要猜调休日;把按规则推算的假期放进 `tentative`(虚线框),并在底部说明里写明"以官方通知为准"。
3. **查节气**:搜索该范围内的二十四节气公历日期,填入 `solarTerms`。
4. **生成 HTML**:用下面的模板,只修改"可编辑配置"区域(`RANGE`、`holidays`、`tentative`、`makeupDays`、`solarTerms`、`NOTES`),其余代码不要动。用 artifact 输出。
5. **核对**(必做,过去出过错):
   - 范围内任选 2-3 个已知日期(如节假日首日、补班日),核对星期几与官方通知写的一致
   - 补班日不在周末则说明有误;补班日应落在周六/周日
   - 春节、中秋等农历节日的公历日期与搜索结果一致
6. **回复用户**:简短说明范围、节假日/补班概要;如有"预计假期",明确告诉用户官方通知出来后需更新,并说明改哪个配置项。

## 常见坑

- 星期一律由代码用 `Date.UTC` 计算,**不要在代码里硬编码"某年1月1日是星期几"**
- 春节假期通知里"除夕"可能是腊月二十八或二十九,以通知原文为准,不要按固定天数套
- 部分公民假日(妇女节、青年节、儿童节等)不算全体放假,不标红
- 圣诞节等非大陆法定节日不标为假期

## HTML 模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>工作日历</title>
<style>
    @page { size: A4 portrait; margin: 8mm; }
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    :root { --cell-h: 15mm; }
    body { font-family: 'Microsoft YaHei','PingFang SC',Arial,sans-serif; background:#f0f0f0; padding:12px; color:#222; }
    .container { width:194mm; margin:0 auto; background:#fff; padding:4mm; }
    h1 { text-align:center; font-size:18px; margin-bottom:8px; }
    .legend { display:flex; justify-content:center; flex-wrap:wrap; gap:14px; margin-bottom:8px; font-size:11px; }
    .legend-item { display:flex; align-items:center; gap:4px; }
    .legend-box { width:14px; height:14px; border:1px solid #bbb; }
    .grid { display:grid; grid-template-columns:1fr 1fr; gap:4mm; }
    .month { border:1.5px solid #2c3e50; }
    .month-header { background:#2c3e50; color:#fff; text-align:center; font-weight:bold; font-size:14px; padding:5px 0; }
    .weekdays { display:grid; grid-template-columns:repeat(7,1fr); background:#34495e; color:#fff; }
    .weekdays div { text-align:center; font-size:11px; font-weight:bold; padding:4px 0; }
    .weekdays div.we { color:#ffcdd2; }
    .days { display:grid; grid-template-columns:repeat(7,1fr); }
    .day { height:var(--cell-h); border:0.5px solid #ccc; display:flex; flex-direction:column; align-items:center; justify-content:center; line-height:1.15; background:#fff; }
    .day .num { font-size:14px; font-weight:bold; }
    .day .sub { font-size:8px; color:#777; margin-top:1px; white-space:nowrap; }
    .day.empty { background:#fafafa; }
    .day.weekend { background:#e3f2fd; }
    .day.weekend .num { color:#1565c0; }
    .day.holiday { background:#ffebee; }
    .day.holiday .num, .day.holiday .sub { color:#c62828; }
    .day.holiday .sub { font-weight:bold; }
    .day.tentative { background:#fff1f1; outline:1.5px dashed #e57373; outline-offset:-2px; }
    .day.tentative .num, .day.tentative .sub { color:#c62828; }
    .day.makeup { background:#fff3e0; }
    .day.makeup .num, .day.makeup .sub { color:#e65100; }
    .day.makeup .sub { font-weight:bold; }
    .day .sub.fest { color:#c62828; font-weight:bold; }
    .day .sub.term { color:#2e7d32; font-weight:bold; }
    .day.today { box-shadow:inset 0 0 0 2px #ff9800; }
    .notes { margin-top:8px; display:grid; gap:5px; }
    .note { padding:6px 8px; font-size:11px; line-height:1.5; border-left:4px solid #c62828; background:#fdecea; }
    .note.blue { border-left-color:#0277bd; background:#e8f4f8; }
    .print-button { position:fixed; top:16px; right:16px; padding:10px 18px; background:#2c3e50; color:#fff; border:none; border-radius:5px; cursor:pointer; font-size:14px; }
    @media print { body{background:#fff;padding:0;} .container{padding:0;width:100%;} .no-print{display:none;} }
</style>
</head>
<body>
<button class="print-button no-print" onclick="window.print()">🖨️ 打印日历</button>
<div class="container">
    <h1 id="title"></h1>
    <div class="legend">
        <div class="legend-item"><div class="legend-box" style="background:#fff"></div>工作日</div>
        <div class="legend-item"><div class="legend-box" style="background:#e3f2fd"></div>周末</div>
        <div class="legend-item"><div class="legend-box" style="background:#ffebee"></div>法定节假日</div>
        <div class="legend-item"><div class="legend-box" style="background:#fff3e0"></div>调休上班</div>
        <div class="legend-item" id="legend-tent" style="display:none"><div class="legend-box" style="background:#fff1f1;outline:1.5px dashed #e57373;outline-offset:-2px"></div>预计假期(待官方通知)</div>
        <div class="legend-item"><span style="color:#2e7d32;font-weight:bold">绿字</span>节气 / <span style="color:#c62828;font-weight:bold">红字</span>节日</div>
    </div>
    <div class="grid" id="grid"></div>
    <div class="notes" id="notes"></div>
</div>

<script>
// ================= 可编辑配置(只改这里) =================
// 日期范围(含首尾)
const RANGE = { start: '2026-11-01', end: '2027-02-28' };
// 法定节假日 { 'YYYY-MM-DD': '名称' }
const holidays = {};
// 预计假期(官方未公布时使用,虚线框) { 'YYYY-MM-DD': 1 }
const tentative = {};
// 调休上班日 { 'YYYY-MM-DD': '补班' }
const makeupDays = {};
// 节气 { 'YYYY-MM-DD': '立冬' }
const solarTerms = {};
// 底部提示(蓝色框那条为固定必加项,不要删)
const NOTES = [
    // { text: '说明文字,可含 <strong>', blue: false },
    { text: '<strong>📢 其他:</strong> 披露易每天 6:00-8:30、12:00-12:30、16:30-23:00 允许发行人上传报告', blue: true }
];
// ==========================================================

const weekdayNames = ['日','一','二','三','四','五','六'];
const lunarDayNames = ['','初一','初二','初三','初四','初五','初六','初七','初八','初九','初十',
    '十一','十二','十三','十四','十五','十六','十七','十八','十九','二十',
    '廿一','廿二','廿三','廿四','廿五','廿六','廿七','廿八','廿九','三十'];
const lunarMonthNames = ['','正','二','三','四','五','六','七','八','九','十','冬','腊'];
const pad = n => String(n).padStart(2, '0');
const key = (y, m, d) => `${y}-${pad(m + 1)}-${pad(d)}`;
const parse = s => { const [y, m, d] = s.split('-').map(Number); return [y, m - 1, d]; };

function lunarOf(y, m, d) {
    const dt = new Date(Date.UTC(y, m, d, 4)); // 北京时间中午
    const parts = new Intl.DateTimeFormat('en-US-u-ca-chinese', {
        timeZone: 'Asia/Shanghai', month: 'numeric', day: 'numeric'
    }).formatToParts(dt);
    const mo = parts.find(p => p.type === 'month').value;
    const da = parts.find(p => p.type === 'day').value;
    return { m: parseInt(mo, 10), d: parseInt(da, 10), leap: /bis/i.test(mo) };
}
function lunarInfo(y, m, d) {
    const l = lunarOf(y, m, d);
    const t = new Date(Date.UTC(y, m, d + 1, 4));
    const n = lunarOf(t.getUTCFullYear(), t.getUTCMonth(), t.getUTCDate());
    let festival = '';
    if (n.m === 1 && n.d === 1 && !n.leap) festival = '除夕';
    else if (!l.leap && l.m === 1 && l.d === 1) festival = '春节';
    else if (!l.leap && l.m === 1 && l.d === 15) festival = '元宵';
    else if (!l.leap && l.m === 5 && l.d === 5) festival = '端午';
    else if (!l.leap && l.m === 7 && l.d === 7) festival = '七夕';
    else if (!l.leap && l.m === 8 && l.d === 15) festival = '中秋';
    else if (!l.leap && l.m === 9 && l.d === 9) festival = '重阳';
    else if (!l.leap && l.m === 12 && l.d === 8) festival = '腊八';
    else if (!l.leap && l.m === 12 && l.d === 23) festival = '小年';
    let text = lunarDayNames[l.d];
    if (l.d === 1) text = (l.leap ? '闰' : '') + lunarMonthNames[l.m] + '月';
    return { text, festival };
}

const [sy, sm, sd] = parse(RANGE.start);
const [ey, em, ed] = parse(RANGE.end);
const startKey = RANGE.start, endKey = RANGE.end;
const todayStr = new Date().toLocaleDateString('en-CA', { timeZone: 'Asia/Shanghai' });

function buildMonth(y, m) {
    const box = document.createElement('div');
    box.className = 'month';
    const head = document.createElement('div');
    head.className = 'month-header';
    head.textContent = `${y}年 ${m + 1}月`;
    box.appendChild(head);

    const wk = document.createElement('div');
    wk.className = 'weekdays';
    weekdayNames.forEach((n, i) => {
        const c = document.createElement('div');
        c.textContent = n;
        if (i === 0 || i === 6) c.className = 'we';
        wk.appendChild(c);
    });
    box.appendChild(wk);

    const days = document.createElement('div');
    days.className = 'days';
    const first = new Date(Date.UTC(y, m, 1)).getUTCDay();
    const total = new Date(Date.UTC(y, m + 1, 0)).getUTCDate();
    const addEmpty = () => { const e = document.createElement('div'); e.className = 'day empty'; days.appendChild(e); };
    for (let i = 0; i < first; i++) addEmpty();

    for (let d = 1; d <= total; d++) {
        const k = key(y, m, d);
        if (k < startKey || k > endKey) { addEmpty(); continue; }
        const dow = new Date(Date.UTC(y, m, d)).getUTCDay();
        const cell = document.createElement('div');
        cell.className = 'day';
        const info = lunarInfo(y, m, d);
        let subText = info.text, subCls = 'sub';

        if (holidays[k]) {
            cell.classList.add('holiday');
            const prev = new Date(Date.UTC(y, m, d - 1));
            const prevKey = key(prev.getUTCFullYear(), prev.getUTCMonth(), prev.getUTCDate());
            if (info.festival) subText = info.festival;
            else if (holidays[k] !== holidays[prevKey]) subText = holidays[k];
        } else if (makeupDays[k]) {
            cell.classList.add('makeup');
            subText = makeupDays[k];
        } else if (tentative[k]) {
            cell.classList.add('tentative');
            if (info.festival) { subText = info.festival; subCls += ' fest'; }
        } else {
            if (dow === 0 || dow === 6) cell.classList.add('weekend');
            if (info.festival) { subText = info.festival; subCls += ' fest'; }
            else if (solarTerms[k]) { subText = solarTerms[k]; subCls += ' term'; }
        }
        if (k === todayStr) cell.classList.add('today');

        const num = document.createElement('div'); num.className = 'num'; num.textContent = d;
        const sub = document.createElement('div'); sub.className = subCls; sub.textContent = subText;
        cell.appendChild(num); cell.appendChild(sub);
        days.appendChild(cell);
    }
    box.appendChild(days);
    return box;
}

// 生成月份列表
const months = [];
for (let y = sy, m = sm; y < ey || (y === ey && m <= em); m++) {
    if (m > 11) { m = 0; y++; }
    months.push([y, m]);
    if (y === ey && m === em) break;
}
const rows = Math.ceil(months.length / 2);
document.documentElement.style.setProperty('--cell-h', rows <= 2 ? '15mm' : (rows === 3 ? '10mm' : '8mm'));

const fmt = s => { const [y, m, d] = parse(s); return `${y}年${m + 1}月${d}日`; };
document.getElementById('title').textContent = `工作日历 ${fmt(RANGE.start)} - ${fmt(RANGE.end)}`;
if (Object.keys(tentative).length) document.getElementById('legend-tent').style.display = 'flex';

const grid = document.getElementById('grid');
months.forEach(([y, m]) => grid.appendChild(buildMonth(y, m)));

const notes = document.getElementById('notes');
NOTES.forEach(n => {
    const el = document.createElement('div');
    el.className = 'note' + (n.blue ? ' blue' : '');
    el.innerHTML = n.text;
    notes.appendChild(el);
});
</script>
</body>
</html>
```

## 使用示例

- "帮我生成 3月到6月的打印日历"
- "下个月的日历打印一下"
- "2027年7月21日到12月31日的日历"

先按流程查官方放假安排和节气,再填入模板配置输出。
