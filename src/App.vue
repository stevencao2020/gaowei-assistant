<script setup lang="ts">
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue';
import dayjs from 'dayjs';
import * as solarLunar from 'solarlunar';
// PDF & 截图（静态引入，避免动态导入失败）
import { jsPDF } from 'jspdf';
import html2canvas from 'html2canvas';

// -------- Day.js plugins (同步加载) --------
// 用同步方式加载，确保在 setup 阶段就能使用 dayjs.tz / dayjs.dayOfYear
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
import dayOfYear from 'dayjs/plugin/dayOfYear';
dayjs.extend(utc);
dayjs.extend(timezone);
dayjs.extend(dayOfYear);


// ---------------- 日历 & 干支/节气 ----------------
export type CalendarData = {
  solarDate: string;
  lunarDate: string;
  gzYear: string;
  gzMonth: string;
  gzDay: string;
  jieqi: string;
};

function getCalendarData(d: Date = new Date()): CalendarData {
  const s = dayjs(d);
  const l: any = solarLunar.solar2lunar(s.year(), s.month() + 1, s.date());
  const monthCn = l.monthCn || l.lMonthCn || l.IMonthCn || '';
  const dayCn   = l.dayCn   || l.lDayCn   || l.IDayCn   || '';
  const term    = l.term || l.jieqi || '';
  return {
    solarDate: s.format('YYYY-MM-DD'),
    lunarDate: `${monthCn}${dayCn}${l.isLeap ? '（闰）' : ''}`,
    gzYear: l.gzYear,
    gzMonth: l.gzMonth,
    gzDay: l.gzDay,
    jieqi: term || '无节气',
  };
}

// ---------------- 真太阳时校准 ----------------
function equationOfTimeMinutes(n: number) {
  const B = (2 * Math.PI * (n - 81)) / 364;
  return 9.87 * Math.sin(2 * B) - 7.53 * Math.cos(B) - 1.5 * Math.sin(B);
}
function toTrueSolar(dateLocalISO: string, tz: string, longitude: number) {
  const m = dayjs.tz(dateLocalISO, tz);
  const N = (m as any).dayOfYear();
  const tzOffsetHours = -m.utcOffset() / 60;
  const eot = equationOfTimeMinutes(N);
  const offset = eot + 4 * (longitude - 15 * tzOffsetHours);
  return m.add(offset, 'minute');
}

// ---------------- 真实时柱推导 ----------------
const hourBranches = ['子','丑','寅','卯','辰','巳','午','未','申','酉','戌','亥'];
const hourStemTable: Record<string, string[]> = {
  '甲': ['甲','丙','戊','庚','壬','甲','丙','戊','庚','壬','甲','丙'],
  '己': ['甲','丙','戊','庚','壬','甲','丙','戊','庚','壬','甲','丙'],
  '乙': ['乙','丁','己','辛','癸','乙','丁','己','辛','癸','乙','丁'],
  '庚': ['乙','丁','己','辛','癸','乙','丁','己','辛','癸','乙','丁'],
  '丙': ['丙','戊','庚','壬','甲','丙','戊','庚','壬','甲','丙','戊'],
  '辛': ['丙','戊','庚','壬','甲','丙','戊','庚','壬','甲','丙','戊'],
  '丁': ['丁','己','辛','癸','乙','丁','己','辛','癸','乙','丁','己'],
  '壬': ['丁','己','辛','癸','乙','丁','己','辛','癸','乙','丁','己'],
  '戊': ['戊','庚','壬','甲','丙','戊','庚','壬','甲','丙','戊','庚'],
  '癸': ['戊','庚','壬','甲','丙','戊','庚','壬','甲','丙','戊','庚'],
};
function getHourPillar(dayStem: string, hour: number) {
  const idx = Math.floor(((hour + 1) % 24) / 2);
  const branch = hourBranches[idx];
  const stem = (hourStemTable[dayStem] || hourStemTable['甲'])[idx];
  return stem + branch;
}// —— 五行映射：天干/地支 → 五行索引（0金 1木 2水 3火 4土）——
const elemIdx: Record<'金'|'木'|'水'|'火'|'土', number> = { '金':0,'木':1,'水':2,'火':3,'土':4 };
const stemElem: Record<string, number> = {
  '甲': elemIdx['木'], '乙': elemIdx['木'],
  '丙': elemIdx['火'], '丁': elemIdx['火'],
  '戊': elemIdx['土'], '己': elemIdx['土'],
  '庚': elemIdx['金'], '辛': elemIdx['金'],
  '壬': elemIdx['水'], '癸': elemIdx['水'],
};
const branchElem: Record<string, number> = {
  '子': elemIdx['水'], '丑': elemIdx['土'], '寅': elemIdx['木'], '卯': elemIdx['木'],
  '辰': elemIdx['土'], '巳': elemIdx['火'], '午': elemIdx['火'], '未': elemIdx['土'],
  '申': elemIdx['金'], '酉': elemIdx['金'], '戌': elemIdx['土'], '亥': elemIdx['水'],
};

// —— 归一化为 100%（四舍五入+余数分配，避免总和≠100）——
function normalizePercents(vals: number[]): number[] {
  const sum = vals.reduce((a,b)=>a+b,0) || 1;
  const raw = vals.map(v => (v / sum) * 100);
  const rounded = raw.map(v => Math.round(v));
  let diff = 100 - rounded.reduce((a,b)=>a+b, 0);
  const fracs = raw.map((v,i)=>({i, frac: v - Math.floor(v)})).sort((a,b)=>b.frac-a.frac);
  let k = 0;
  while (diff !== 0 && fracs.length) {
    const idx = fracs[k % fracs.length].i;
    rounded[idx] += (diff > 0 ? 1 : -1);
    diff += (diff > 0 ? -1 : 1);
    k++;
  }
  return rounded;
}

// —— 从四柱计算五行占比（可调权重：干 0.6 / 支 0.4）——
function computeFiveFromBazi(gzYear: string, gzMonth: string, gzDay: string, gzHour: string, stemWeight=0.6, branchWeight=0.4): number[] {
  const pillars = [gzYear, gzMonth, gzDay, gzHour].filter(Boolean);
  const acc = [0,0,0,0,0]; // 金木水火土
  for (const p of pillars) {
    const s = p.charAt(0); const b = p.charAt(1);
    const si = stemElem[s]; const bi = branchElem[b];
    if (si!=null) acc[si] += stemWeight;
    if (bi!=null) acc[bi] += branchWeight;
  }
  return normalizePercents(acc);
}

// ---------------- 运势占位逻辑 ----------------
type Bazi = { dayStem: string };
const tenGodsMap: Record<string, Record<string,string>> = {
  '丁': { '甲': '印星', '乙': '偏印', '丙': '比肩', '丁': '劫财', '戊': '食神', '己': '伤官', '庚':'官杀','辛':'七杀','壬':'偏财','癸':'正财' },
  '甲': { '庚': '官杀', '辛': '七杀', '甲': '比肩', '乙': '劫财', '丙': '食神', '丁': '伤官', '戊':'偏财','己':'正财','壬':'印星','癸':'偏印' },
};
function analyzeFortune(bazi: Bazi, dayStem: string) {
  const relation = tenGodsMap[bazi.dayStem]?.[dayStem] ?? '中性';
  let rating: '吉' | '中平' | '凶' = '中平';
  if (relation.includes('印') || relation.includes('食') || relation.includes('财')) rating = '吉';
  if (relation.includes('伤') || relation.includes('杀')) rating = '凶';
  return { relation, rating, advice: `今日 ${relation}，宜稳步推进计划。` };
}

// ---------------- 档案（含出生信息 & 真太阳时） ----------------
export type Profile = { id: string; name: string; birthPlace?: string; birthDate?: string; birthTime?: string; tz?: string; longitude?: number; latitude?: number; useTrueSolar?: boolean; dayStem?: string };
const LS_KEY = 'gw_profiles_v2';
const LS_ACTIVE = 'gw_active_profile_v2';
const localTZ = Intl.DateTimeFormat().resolvedOptions().timeZone || 'Asia/Shanghai';
const defaultProfiles: Profile[] = [
  { id: 'u1', name: '用户A', birthPlace: '北京', dayStem: '丁', birthDate: '1990-01-01', birthTime: '08:00', tz: localTZ, longitude: 116.38, latitude: 39.90, useTrueSolar: false },
  { id: 'u2', name: '用户B', birthPlace: '上海', dayStem: '甲', birthDate: '1992-06-06', birthTime: '14:30', tz: localTZ, longitude: 121.47, latitude: 31.23, useTrueSolar: true  },
];
function loadProfiles(){ try{const list=JSON.parse(localStorage.getItem(LS_KEY)||'null') as Profile[]|null; const active=localStorage.getItem(LS_ACTIVE)||''; if(list&&list.length) return {list,activeId:active||list[0].id};}catch{} return {list:defaultProfiles,activeId:defaultProfiles[0].id}; }
const profiles = ref<Profile[]>(loadProfiles().list);
const activeProfileId = ref<string>(loadProfiles().activeId);
const activeProfile = computed(()=> profiles.value.find(p=>p.id===activeProfileId.value) || profiles.value[0]);
watch([profiles,activeProfileId],()=>{ localStorage.setItem(LS_KEY,JSON.stringify(profiles.value)); localStorage.setItem(LS_ACTIVE,activeProfileId.value); },{deep:true});

const form = ref<Profile>({ id:'', name:'', birthPlace:'', dayStem:'丁', tz: localTZ, birthDate:'', birthTime:'', longitude:116.38, latitude:39.90, useTrueSolar:false });
const editing = ref(false);
function resetForm(){ form.value = { id:'', name:'', birthPlace:'', dayStem:'丁', tz: localTZ, birthDate:'', birthTime:'', longitude:116.38, latitude:39.90, useTrueSolar:false }; editing.value=false; }
// —— 出生地选项（常见城市；可按需继续扩充） ——
const places = [
  { name: '北京', tz: 'Asia/Shanghai', lon: 116.38, lat: 39.90 },
  { name: '上海', tz: 'Asia/Shanghai', lon: 121.47, lat: 31.23 },
  { name: '广州', tz: 'Asia/Shanghai', lon: 113.27, lat: 23.13 },
  { name: '深圳', tz: 'Asia/Shanghai', lon: 114.06, lat: 22.55 },
  { name: '杭州', tz: 'Asia/Shanghai', lon: 120.16, lat: 30.25 },
  { name: '成都', tz: 'Asia/Shanghai', lon: 104.07, lat: 30.67 },
  { name: '重庆', tz: 'Asia/Shanghai', lon: 106.55, lat: 29.56 },
  { name: '武汉', tz: 'Asia/Shanghai', lon: 114.30, lat: 30.60 },
  { name: '西安', tz: 'Asia/Shanghai', lon: 108.94, lat: 34.34 },
  { name: '南京', tz: 'Asia/Shanghai', lon: 118.80, lat: 32.06 },
  { name: '天津', tz: 'Asia/Shanghai', lon: 117.20, lat: 39.13 },
  { name: '香港', tz: 'Asia/Hong_Kong', lon: 114.17, lat: 22.28 },
  { name: '台北', tz: 'Asia/Taipei', lon: 121.56, lat: 25.05 },
  { name: '东京', tz: 'Asia/Tokyo', lon: 139.69, lat: 35.69 },
  { name: '首尔', tz: 'Asia/Seoul', lon: 126.98, lat: 37.57 },
  { name: '新加坡', tz: 'Asia/Singapore', lon: 103.82, lat: 1.35 },
  { name: '纽约', tz: 'America/New_York', lon: -74.00, lat: 40.71 },
  { name: '洛杉矶', tz: 'America/Los_Angeles', lon: -118.24, lat: 34.05 },
  { name: '伦敦', tz: 'Europe/London', lon: -0.1276, lat: 51.5072 },
  { name: '巴黎', tz: 'Europe/Paris', lon: 2.35, lat: 48.86 },
] as const;

function onBirthPlaceChange(name: string){
  const p = places.find(x=>x.name===name);
  if (!p) return;
  form.value.birthPlace = p.name;
  form.value.tz = p.tz;
  form.value.longitude = p.lon;
  form.value.latitude = p.lat;
  manualLongitudeTouched.value = false;
}
// —— 新增/编辑时的实时八字/能量/神煞预览 ——
const formPreviewReady = computed(()=> !!form.value.birthDate && !!form.value.birthTime);
const formPreview = computed(()=>{
  if (!formPreviewReady.value) return null as any;
  const tz = form.value.tz || localTZ;
  const iso = `${form.value.birthDate}T${form.value.birthTime}`;
  const m = form.value.useTrueSolar && form.value.longitude!=null
    ? toTrueSolar(iso, tz, form.value.longitude!)
    : (dayjs as any).tz(iso, tz);
  const l:any = solarLunar.solar2lunar(m.year(), m.month()+1, m.date());
  const gzYear = l.gzYear, gzMonth = l.gzMonth, gzDay = l.gzDay;
  const dStem = (gzDay||'').charAt(0) || '丁';
  const hourPillar = getHourPillar(dStem, m.hour());
  const energy = computeFiveFromBazi(gzYear, gzMonth, gzDay, hourPillar);
  const shensha = computeShenSha(gzYear, gzMonth, gzDay, hourPillar);
  return { gzYear, gzMonth, gzDay, hourPillar, dStem, energy, shensha };
});

function computeShenSha(gzYear: string, gzMonth: string, gzDay: string, gzHour: string){
  const yStem = gzYear?.charAt(0) || ''; const yBr = gzYear?.charAt(1) || '';
  const mStem = gzMonth?.charAt(0) || ''; const mBr = gzMonth?.charAt(1) || '';
  const dStem = gzDay?.charAt(0) || ''; const dBr = gzDay?.charAt(1) || '';
  const hStem = gzHour?.charAt(0) || ''; const hBr = gzHour?.charAt(1) || '';
  const branches = [yBr, mBr, dBr, hBr].filter(Boolean);

  const res: string[] = [];

  // 天乙贵人（以日干查）
  const tianyiguiren: Record<string,string[]> = {
    '甲': ['丑','未'], '戊': ['丑','未'], '庚': ['丑','未'],
    '乙': ['子','申'], '己': ['子','申'],
    '丙': ['亥','酉'], '丁': ['亥','酉'], '辛': ['亥','酉'],
    '壬': ['卯','巳'], '癸': ['卯','巳'],
  };
  if ((tianyiguiren[dStem] || []).some(b => branches.includes(b))) res.push('天乙贵人');

  // 四局判类函数
  const groupOf = (br:string) => {
    if ('申子辰'.includes(br)) return '申子辰';
    if ('寅午戌'.includes(br)) return '寅午戌';
    if ('亥卯未'.includes(br)) return '亥卯未';
    if ('巳酉丑'.includes(br)) return '巳酉丑';
    return '';
  };

  // 桃花/咸池（以年支或日支所属三合局查）
  const peachByGroup: Record<string,string> = { '申子辰':'酉', '寅午戌':'卯', '亥卯未':'子', '巳酉丑':'午' };
  {
    const baseBr = dBr || yBr;
    const peachBr = peachByGroup[groupOf(baseBr)];
    if (peachBr && branches.includes(peachBr)) res.push('桃花(咸池)');
  }

  // 驿马（以日/年支所属三合局查）
  const yimaByGroup: Record<string,string> = { '申子辰':'寅', '寅午戌':'申', '亥卯未':'巳', '巳酉丑':'亥' };
  {
    const baseBr = dBr || yBr;
    const yimaBr = yimaByGroup[groupOf(baseBr)];
    if (yimaBr && branches.includes(yimaBr)) res.push('驿马');
  }

  // 华盖（以日/年支所属三合局查）
  const huagaiByGroup: Record<string,string> = { '申子辰':'辰', '寅午戌':'戌', '亥卯未':'未', '巳酉丑':'丑' };
  {
    const baseBr = dBr || yBr;
    const hg = huagaiByGroup[groupOf(baseBr)];
    if (hg && branches.includes(hg)) res.push('华盖');
  }

  // 禄神 & 羊刃（以日干查）
  const luByStem: Record<string,string> = { '甲':'寅','乙':'卯','丙':'巳','丁':'午','戊':'巳','己':'午','庚':'申','辛':'酉','壬':'亥','癸':'子' };
  const renByStem: Record<string,string> = { '甲':'卯','乙':'辰','丙':'午','丁':'未','戊':'午','己':'未','庚':'酉','辛':'戌','壬':'子','癸':'丑' };
  if (branches.includes(luByStem[dStem])) res.push('禄神');
  if (branches.includes(renByStem[dStem])) res.push('羊刃');

  // 文昌（以年干查）
  const wenchangByYearStem: Record<string,string> = { '甲':'巳','乙':'午','丙':'申','丁':'酉','戊':'申','己':'酉','庚':'亥','辛':'子','壬':'寅','癸':'卯' };
  if (wenchangByYearStem[yStem] && branches.includes(wenchangByYearStem[yStem])) res.push('文昌');

  // 将星（以月支所属三合局查）
  const jiangxingByMonthGroup: Record<string,string> = { '寅午戌':'午','申子辰':'子','亥卯未':'卯','巳酉丑':'酉' };
  {
    const jx = jiangxingByMonthGroup[groupOf(mBr)];
    if (jx && branches.includes(jx)) res.push('将星');
  }

  // 红艳（以日干查）
  const hongyanByStem: Record<string,string[]> = {
    '甲':['午'],'戊':['午'],'庚':['午'],
    '乙':['子'],'己':['子'],'辛':['子'],
    '丙':['酉'],'丁':['酉'],
    '壬':['卯'],'癸':['卯'],
  };
  if ((hongyanByStem[dStem]||[]).some(b=>branches.includes(b))) res.push('红艳');

  return Array.from(new Set(res));
}
function startEdit(){ const p=activeProfile.value; form.value = { ...p }; editing.value=true; }
function submitForm(){
  const raw = (form.value.name ?? '');
  // 去掉空格、零宽字符、BOM 等不可见字符
  const n = String(raw).replace(/[\s\u200b\u200c\u200d\ufeff]/g, '');
  if (!n) {
    alert('请输入姓名');
    return;
  }
  // 将标准化后的姓名写回
  form.value.name = n;

  if (editing.value && form.value.id) {
    const idx = profiles.value.findIndex(p => p.id === form.value.id);
    if (idx >= 0) profiles.value[idx] = { ...form.value } as Profile;
    activeProfileId.value = form.value.id as string;
  } else {
    const id = 'u' + Math.random().toString(36).slice(2, 8);
    profiles.value.push({ ...(form.value as Profile), id });
    activeProfileId.value = id;
  }
  resetForm();
}
function removeActive(){ const p=activeProfile.value; if(!p) return; if(!confirm(`确认删除档案：${p.name}？`)) return; const idx=profiles.value.findIndex(x=>x.id===p.id); if(idx>=0){ profiles.value.splice(idx,1); if(profiles.value.length) activeProfileId.value = profiles.value[0].id; } }

// ---------------- 推导日主/时柱 ----------------
const derivedDayStem = computed(()=>{ const p=activeProfile.value; if(!p||!p.birthDate||!p.birthTime||!p.tz) return p.dayStem||'丁'; const iso=`${p.birthDate}T${p.birthTime}`; const m = p.useTrueSolar && p.longitude!=null ? toTrueSolar(iso,p.tz,p.longitude) : (dayjs as any).tz(iso,p.tz); const l:any = solarLunar.solar2lunar(m.year(), m.month()+1, m.date()); const dayStem=(l.gzDay||'').charAt(0) || p.dayStem || '丁'; return dayStem; });
const derivedHourPillar = computed(()=>{ const p=activeProfile.value; if(!p||!p.birthDate||!p.birthTime||!p.tz) return ''; const iso=`${p.birthDate}T${p.birthTime}`; const m = p.useTrueSolar && p.longitude!=null ? toTrueSolar(iso,p.tz,p.longitude) : (dayjs as any).tz(iso,p.tz); const l:any = solarLunar.solar2lunar(m.year(), m.month()+1, m.date()); const dayStem=(l.gzDay||'').charAt(0) || p.dayStem || '丁'; return getHourPillar(dayStem, m.hour()); });
// —— 当前档案的四柱（用于五行图 & 展示）——
const activeBazi = computed(() => {
  const p = activeProfile.value;
  if (!p) return null;
  if (!p.birthDate || !p.birthTime || !p.tz) return null;
  const iso = `${p.birthDate}T${p.birthTime}`;
  const m = p.useTrueSolar && p.longitude != null
    ? toTrueSolar(iso, p.tz, p.longitude)
    : (dayjs as any).tz(iso, p.tz);
  const l: any = solarLunar.solar2lunar(m.year(), m.month()+1, m.date());
  const gzYear = l.gzYear ?? '';
  const gzMonth = l.gzMonth ?? '';
  const gzDay = l.gzDay ?? '';
  const gzHour = getHourPillar((gzDay||'丁').charAt(0), m.hour());
  return { gzYear, gzMonth, gzDay, gzHour };
});
// ---------------- 页面主状态 ----------------
const date = ref<Date>(new Date());
const cal = computed(()=> getCalendarData(date.value));
const dayStemToday = computed(()=> cal.value.gzDay.charAt(0));
const fortune = computed(()=> analyzeFortune({ dayStem: derivedDayStem.value }, dayStemToday.value));
function prevDay(){ const d=new Date(date.value); d.setDate(d.getDate()-1); date.value=d; }
function nextDay(){ const d=new Date(date.value); d.setDate(d.getDate()+1); date.value=d; }
function today(){ date.value=new Date(); }

// ---------------- 五行图（Chart.js 动态导入） ----------------
let Chart: any = null; let fiveChart: any = null; const fiveCanvas = ref<HTMLCanvasElement|null>(null); const chartOk = ref(false);
function fiveFromDayStem(stem:string){ switch(stem){ case '丁': return [15,15,15,40,15]; case '甲': return [15,40,15,15,15]; case '乙': return [15,35,15,20,15]; default: return [20,20,20,20,20]; } }
function renderFiveChart() {
  if (!Chart || !fiveCanvas.value) return;
  let data = [20,20,20,20,20];
if (activeBazi.value) {
  const { gzYear, gzMonth, gzDay, gzHour } = activeBazi.value;
  data = computeFiveFromBazi(gzYear, gzMonth, gzDay, gzHour);
} else {
  // 兼容：若档案信息不全，降级为日主示例
  data = fiveFromDayStem(derivedDayStem.value);
}
  if (fiveChart) fiveChart.destroy();
  fiveChart = new Chart(fiveCanvas.value, {
    type: 'doughnut',
    data: {
      labels: ['金', '木', '水', '火', '土'],
      datasets: [{
        data,
        backgroundColor: ['#60a5fa', '#34d399', '#93c5fd', '#f87171', '#fbbf24'],
        borderWidth: 0,
      }],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: { animateRotate: true, duration: 800, easing: 'easeOutQuart' },
      plugins: { legend: { position: 'bottom' } },
    },
  });
}

// ---------------- 趋势图（ECharts 动态导入） ----------------
let echartsMod:any=null; let trendChart:any=null; const trendDiv=ref<HTMLDivElement|null>(null); const echartsOk=ref(false);
function renderTrendChart(){
  if(!echartsMod||!trendDiv.value) return;
  if(!trendChart) trendChart = echartsMod.init(trendDiv.value);
  const months=Array.from({length:12},(_,i)=>dayjs(date.value).add(i,'month').format('MM月'));
  const values=months.map((_,i)=>60+Math.round(20*Math.sin(i/2)));
  trendChart.setOption({
    animation: true,
    animationDuration: 800,
    animationEasing: 'cubicOut',
    grid:{left:40,right:16,top:24,bottom:28},
    xAxis:{type:'category',data:months},
    yAxis:{type:'value',min:0,max:100},
    series:[{type:'line',data:values,smooth:true}],
    tooltip:{trigger:'axis'}
  });
  trendChart.resize();
}

onMounted(async()=>{ await nextTick(); try{ const mod=await import('chart.js'); Chart=mod.Chart; Chart.register(mod.ArcElement,mod.Tooltip,mod.Legend); chartOk.value=true; renderFiveChart(); }catch{ chartOk.value=false; } try{ const mod=await import('echarts'); echartsMod=(mod as any).default ?? mod; echartsOk.value=true; renderTrendChart(); }catch{ echartsOk.value=false; } window.addEventListener('resize', handleResize); });
onBeforeUnmount(()=>{ window.removeEventListener('resize', handleResize); if(fiveChart){ fiveChart.destroy(); fiveChart=null; } if(trendChart){ trendChart.dispose(); trendChart=null; } });
function handleResize(){ if(trendChart) trendChart.resize(); }
watch([derivedDayStem, activeBazi], ()=>{ renderFiveChart(); });
watch([date],()=>{ renderTrendChart(); });

// ---------------- 择日（按事项） ----------------
const eventTypes = ['签约','开业','搬家','面试'] as const;
const eventType = ref<typeof eventTypes[number]>('签约');
const favorByEvent: Record<string,{gods:string[];branches:string[]}> = {
  '签约': { gods:['印','财'], branches:['子','亥','卯'] },
  '开业': { gods:['食','财'], branches:['寅','午','辰'] },
  '搬家': { gods:['印'],     branches:['丑','未','亥'] },
  '面试': { gods:['印','食'], branches:['子','申','酉'] },
};
function scoreByRating(r:'吉'|'中平'|'凶'){ return r==='吉'?2 : r==='中平'?1 : 0; }
function scoreByEvent(relation:string, gzDay:string){ const fav=favorByEvent[eventType.value]; const br=gzDay.charAt(1); let add=0; if(fav.gods.some(g=>relation.includes(g))) add+=2; if(fav.branches.includes(br)) add+=1; return add; }
const bestDays = computed(()=>{ const res:{date:string;gzDay:string;rating:'吉'|'中平'|'凶';score:number}[]=[]; const p=derivedDayStem.value; for(let i=0;i<30;i++){ const d=dayjs().add(i,'day').toDate(); const c=getCalendarData(d); const dayStem=c.gzDay.charAt(0); const f=analyzeFortune({dayStem:p}, dayStem); const base=scoreByRating(f.rating as any); const extra=scoreByEvent(f.relation,c.gzDay); res.push({date:c.solarDate,gzDay:c.gzDay,rating:f.rating as any,score: base+extra}); } res.sort((a,b)=>b.score-a.score); return { top3: res.slice(0,3), backup5: res.slice(3,8) }; });
// ---------------- 时区选项（常用 IANA） ----------------
const tzOptionsBase = [
  'Asia/Shanghai', 'Asia/Tokyo', 'Asia/Hong_Kong', 'Asia/Singapore', 'Asia/Taipei', 'Asia/Seoul', 'Asia/Bangkok',
  'Europe/London', 'Europe/Berlin', 'Europe/Paris',
  'America/Los_Angeles', 'America/New_York', 'America/Chicago', 'America/Toronto',
  'Australia/Sydney', 'Pacific/Auckland',
  'UTC'
];
const tzOptions = ref<string[]>(tzOptionsBase.includes(localTZ) ? tzOptionsBase : [localTZ, ...tzOptionsBase]);

// —— 时区默认经度映射（城市中心近似值，用于预填）——
const tzLongitude: Record<string, number> = {
  'Asia/Shanghai': 116.38,
  'Asia/Tokyo': 139.69,
  'Asia/Hong_Kong': 114.17,
  'Asia/Singapore': 103.82,
  'Asia/Taipei': 121.56,
  'Asia/Seoul': 126.98,
  'Asia/Bangkok': 100.50,
  'Europe/London': -0.1276,
  'Europe/Berlin': 13.40,
  'Europe/Paris': 2.35,
  'America/Los_Angeles': -118.24,
  'America/New_York': -74.00,
  'America/Chicago': -87.62,
  'America/Toronto': -79.38,
  'Australia/Sydney': 151.21,
  'Pacific/Auckland': 174.76,
  'UTC': 0.0,
};

// —— 时区下拉搜索 ——
const tzQuery = ref('');
const tzFiltered = computed(() => {
  const q = tzQuery.value.trim().toLowerCase();
  if (!q) return tzOptions.value;
  return tzOptions.value.filter(tz => tz.toLowerCase().includes(q));
});

// —— 经度“是否手动修改”标记，用于自动预填 ——
const manualLongitudeTouched = ref(false);
function onLongitudeInput() { manualLongitudeTouched.value = true; }

// 选择时区时，若经度未被手动改过，则自动预填推荐经度
watch(() => form.value.tz, (tz) => {
  if (!manualLongitudeTouched.value && tz) {
    const v = tzLongitude[tz];
    if (typeof v === 'number') form.value.longitude = v;
  }
});

// 提供一个方法可一键按时区重置经度
function resetLongitudeByTZ() {
  const v = tzLongitude[form.value.tz || ''];
  if (typeof v === 'number') {
    form.value.longitude = v;
    manualLongitudeTouched.value = false;
  }
  else {
    alert('当前时区无默认经度，请手动输入');
  }
}

// —— 自动定位（经纬度） ——
function autoLocate(){
  if (!navigator.geolocation) { alert('此设备不支持定位'); return; }
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      form.value.longitude = Number(pos.coords.longitude.toFixed(4));
      form.value.latitude  = Number(pos.coords.latitude.toFixed(4));
    },
    (err) => { alert('定位失败：' + (err?.message || '未知错误')); }
  );
}

// ---------------- 表单校验（基础必填与范围） ----------------
const formErrors = ref<{[k:string]: string}>({});
function validateForm(): boolean {
  const e: {[k:string]: string} = {};
  const name = String(form.value.name ?? '').replace(/[\s\u200b\u200c\u200d\ufeff]/g, '');
  if (!name) e.name = '请输入姓名';
  if (form.value.birthDate && !/^\d{4}-\d{2}-\d{2}$/.test(form.value.birthDate)) e.birthDate = '日期格式应为 YYYY-MM-DD';
  if (form.value.birthTime && !/^\d{2}:\d{2}$/.test(form.value.birthTime)) e.birthTime = '时间格式应为 HH:mm';
  // 取消时区必填
  if (form.value.longitude != null && (form.value.longitude < -180 || form.value.longitude > 180)) e.longitude = '经度范围 -180 ~ 180';
  if (form.value.latitude != null && form.value.latitude !== undefined && form.value.latitude !== null) {
    const lat = Number(form.value.latitude);
    if (!Number.isNaN(lat) && (lat < -90 || lat > 90)) e.latitude = '纬度范围 -90 ~ 90';
  }
  formErrors.value = e;
  return Object.keys(e).length === 0;
}
// ---------------- 报告导出容器 ----------------
const reportRef = ref<HTMLDivElement | null>(null);

// ---------------- 导出选项（页眉/页脚/封面 & 导出范围） ----------------
const exportOptions = ref({
  includeCover: true,
  includeFooter: true,
  range: 'all' as 'all' | 'basic-dates', // all：整页截图；basic-dates：仅“基本信息 + 择日”
});
// ---------------- 导出加载态（按钮禁用 & 提示） ----------------
const exporting = ref(false);
async function withExporting<T>(fn: () => Promise<T>): Promise<T | undefined> {
  if (exporting.value) return; // 防抖
  exporting.value = true;
  try { return await fn(); }
  finally { exporting.value = false; }
}

function addHeaderFooter(doc: jsPDF, opts: { includeFooter: boolean; coverPages?: number }) {
  const total = doc.getNumberOfPages();
  const cover = opts.coverPages ?? 0;
  if (!opts.includeFooter) return;
  for (let i = 1; i <= total; i++) {
    doc.setPage(i);
    // 页脚页码（从 1 开始计数，封面不计或不显示）
    if (i > cover) {
      (doc as any).setFont('NotoSansSC');
      doc.setFontSize(10);
      const label = `第 ${i - cover} / ${total - cover} 页`;
      const w = doc.getTextWidth(label);
      const pageW = doc.internal.pageSize.getWidth();
      const pageH = doc.internal.pageSize.getHeight();
      doc.text(label, pageW - w - 24, pageH - 20);
    }
  }
}

async function addCoverPage(doc: jsPDF) {
  await ensureChineseFont();
  (doc as any).setFont('NotoSansSC');
  const marginX = 48;
  const lineH = 24;
  let y = 140;
  doc.setFontSize(28); doc.setFont(undefined, 'bold');
  doc.text('高维助理 · 万年历报告', marginX, y); y += lineH * 1.5;
  doc.setFontSize(14); doc.setFont(undefined, 'normal');
  const lines = [
    `档案：${activeProfile.value?.name || ''}`,
    `公历：${cal.value.solarDate}    农历：${cal.value.lunarDate}`,
    `干支：${cal.value.gzYear} · ${cal.value.gzMonth} · ${cal.value.gzDay}`,
    `节气：${cal.value.jieqi}`,
    `生成时间：${dayjs().format('YYYY-MM-DD HH:mm')}`,
  ];
  lines.forEach(t => { doc.text(t, marginX, y); y += lineH; });
  // 封底空白，下一页由正文/截图填充
  doc.addPage();
}

// ---------------- 中文字体（可复制文本）支持：从 assets 或 CDN 加载 TTF 并注册到 jsPDF ----------------
let fontReady = false;
async function ensureChineseFont() {
  if (fontReady) return;
  // 依次尝试：本地 assets -> 多个 CDN（国内/海外）
  const candidates = [
    new URL('./assets/NotoSansSC-Regular.ttf', import.meta.url).toString(),
    // jsDelivr（npm）
    'https://cdn.jsdelivr.net/npm/@fontsource/noto-sans-sc/files/noto-sans-sc-chinese-simplified-400-normal.ttf',
    'https://cdn.jsdelivr.net/npm/@fontsource/noto-sans-sc@5.0.8/files/noto-sans-sc-chinese-simplified-400-normal.ttf',
    // UNPKG（npm）
    'https://unpkg.com/@fontsource/noto-sans-sc@latest/files/noto-sans-sc-chinese-simplified-400-normal.ttf',
    // 备选：GitHub raw（可能偶发 403，放最后）
    'https://raw.githubusercontent.com/fontsource/font-files/main/fonts/noto-sans-sc/chinese-simplified-400-normal.ttf'
  ];

  let buf: ArrayBuffer | null = null;
  let lastErr: any = null;
  for (const url of candidates) {
    try {
      const res = await fetch(url, { mode: 'cors' });
      if (!res.ok) throw new Error('HTTP ' + res.status);
      buf = await res.arrayBuffer();
      break;
    } catch (e) {
      lastErr = e;
      // 尝试下一个源
    }
  }
  if (!buf) {
    console.error('加载中文字体失败：', lastErr);
    throw new Error('无法加载中文字体。请将 NotoSansSC-Regular.ttf 放到 src/assets/ 后重试。');
  }

  // 转 base64 并注册
  const b64 = btoa(String.fromCharCode(...new Uint8Array(buf)));
  (jsPDF as any).API.addFileToVFS('NotoSansSC-Regular.ttf', b64);
  (jsPDF as any).API.addFont('NotoSansSC-Regular.ttf', 'NotoSansSC', 'normal');
  fontReady = true;
}

// ---------------- 导出 PDF 报告（文本，可复制中文） ----------------
async function exportPdf() {
  return withExporting(async () => {
    try {
      await ensureChineseFont();
      const doc = new jsPDF({ unit: 'pt', format: 'a4' });
      (doc as any).setFont('NotoSansSC');

      const marginX = 48;
      const lineH = 18;
      let y = 64;

      doc.setFontSize(20); doc.setFont(undefined, 'bold');
      doc.text('高维助理 · 万年历报告（文本版）', marginX, y); y += lineH * 1.5;

      doc.setFontSize(12); doc.setFont(undefined, 'normal');
      const basicLines = [
        `档案：${activeProfile.value?.name || ''}`,
        `公历：${cal.value.solarDate}`,
        `农历：${cal.value.lunarDate}`,
        `干支：${cal.value.gzYear} · ${cal.value.gzMonth} · ${cal.value.gzDay}`,
        `节气：${cal.value.jieqi}`,
        `推导日主：${derivedDayStem.value}，出生时柱：${derivedHourPillar.value || '（信息不全）'}`,
        `当日关系：${fortune.value.relation}，评级：${fortune.value.rating}`,
      ];
      for (const t of basicLines) { doc.text(t, marginX, y); y += lineH; }
      y += 6;

      const best3 = bestDays.value.top3.map(d => `${d.date}（${d.gzDay}）：${d.rating}`);
      const backup5 = bestDays.value.backup5.map(d => `${d.date}（${d.gzDay}）：${d.rating}`);

      doc.setFontSize(16); doc.setFont(undefined, 'bold'); doc.text('30天内最优 3 日', marginX, y); y += lineH + 2;
      doc.setFontSize(12); doc.setFont(undefined, 'normal');
      for (const t of best3) { doc.text(t, marginX, y); y += lineH; }
      y += 6;

      doc.setFontSize(16); doc.setFont(undefined, 'bold'); doc.text('30天内备选 5 日', marginX, y); y += lineH + 2;
      doc.setFontSize(12); doc.setFont(undefined, 'normal');
      for (const t of backup5) { doc.text(t, marginX, y); y += lineH; }

      addHeaderFooter(doc, { includeFooter: exportOptions.value.includeFooter, coverPages: 0 });
      const filename = `万年历报告_文本_${activeProfile.value?.name || '未命名'}_${cal.value.solarDate}.pdf`;
      doc.save(filename);
    } catch (err: any) {
      alert(err?.message || '导出失败（文本版）');
    }
  });
}
// ---------------- 导出 图文版 PDF（含图表与中文） ----------------
async function exportPdfFull() {
  return withExporting(async () => {
    try {
      const pdf = new jsPDF({ unit: 'pt', format: 'a4' });
      let coverAdded = false;

      // 1) 可选封面
      if (exportOptions.value.includeCover) { await addCoverPage(pdf); coverAdded = true; }

      // 2) 选择截图源
      let targetEl: HTMLElement | null = null;
      if (exportOptions.value.range === 'all') {
        targetEl = reportRef.value as HTMLElement;
      } else {
        // 仅“基本信息 + 择日”：拼一个临时容器
        const basic = document.getElementById('sec-basic');
        const dates = document.getElementById('sec-dates');
        const tmp = document.createElement('div');
        tmp.style.width = (reportRef.value?.clientWidth || 600) + 'px';
        tmp.style.padding = '16px';
        tmp.style.background = '#fff';
        if (basic) tmp.appendChild(basic.cloneNode(true));
        if (dates) tmp.appendChild(dates.cloneNode(true));
        document.body.appendChild(tmp);
        targetEl = tmp;

        // 渲染并清理
        const canvas = await html2canvas(targetEl, { scale: window.devicePixelRatio > 1 ? 2 : 1, backgroundColor: '#ffffff', useCORS: true, logging: false });
        const pageW = pdf.internal.pageSize.getWidth();
        const pageH = pdf.internal.pageSize.getHeight();
        const imgW = pageW; const imgH = canvas.height * (imgW / canvas.width);
        let offsetY = 0; const ratio = imgW / canvas.width;
        while (offsetY < imgH - 1) {
          const slicePx = Math.min(Math.round(pageH / ratio), canvas.height - Math.round(offsetY / ratio));
          const canvasPage = document.createElement('canvas');
          canvasPage.width = canvas.width; canvasPage.height = slicePx;
          const ctx = canvasPage.getContext('2d')!;
          ctx.drawImage(canvas, 0, Math.round(offsetY / ratio), canvas.width, slicePx, 0, 0, canvas.width, slicePx);
          const pageData = canvasPage.toDataURL('image/png');
          const renderH = slicePx * ratio;
          if (pdf.getNumberOfPages() > 0) pdf.addPage();
          pdf.addImage(pageData, 'PNG', 0, 0, imgW, renderH);
          offsetY += renderH;
        }
        document.body.removeChild(tmp);

        addHeaderFooter(pdf, { includeFooter: exportOptions.value.includeFooter, coverPages: coverAdded ? 1 : 0 });
        const filename = `万年历报告(图文版)_${activeProfile.value?.name || '未命名'}_${cal.value.solarDate}.pdf`;
        pdf.save(filename);
        return;
      }

      // 3) 整页截图流程
      if (!targetEl) { alert('找不到报告内容区域'); return; }
      const canvas = await html2canvas(targetEl as HTMLElement, { scale: window.devicePixelRatio > 1 ? 2 : 1, backgroundColor: '#ffffff', useCORS: true, logging: false, windowWidth: document.documentElement.scrollWidth });
      const pageW = pdf.internal.pageSize.getWidth();
      const pageH = pdf.internal.pageSize.getHeight();
      const imgW = pageW; const imgH = canvas.height * (imgW / canvas.width);
      let offsetY = 0; const ratio = imgW / canvas.width;
      while (offsetY < imgH - 1) {
        const slicePx = Math.min(Math.round(pageH / ratio), canvas.height - Math.round(offsetY / ratio));
        const canvasPage = document.createElement('canvas');
        canvasPage.width = canvas.width; canvasPage.height = slicePx;
        const ctx = canvasPage.getContext('2d')!;
        ctx.drawImage(canvas, 0, Math.round(offsetY / ratio), canvas.width, slicePx, 0, 0, canvas.width, slicePx);
        const pageData = canvasPage.toDataURL('image/png');
        const renderH = slicePx * ratio;
        if (pdf.getNumberOfPages() > 0) pdf.addPage();
        pdf.addImage(pageData, 'PNG', 0, 0, imgW, renderH);
        offsetY += renderH;
      }

      addHeaderFooter(pdf, { includeFooter: exportOptions.value.includeFooter, coverPages: coverAdded ? 1 : 0 });
      const filename = `万年历报告(图文版)_${activeProfile.value?.name || '未命名'}_${cal.value.solarDate}.pdf`;
      pdf.save(filename);
    } catch (e) {
      console.error(e);
      alert('导出失败：请在控制台查看错误信息');
    }
  });
}
</script>

<template>
  <main class="min-h-screen p-6" :style="{ backgroundImage: 'var(--gw-bg-1)' }">
    <div class="mx-auto max-w-screen-sm space-y-6">
      <h1 class="text-2xl font-semibold text-center">高维助理 · 万年历 MVP</h1>

      <!-- 导出按钮 -->
      <div class="flex justify-end gap-2">
        <button class="btn-orange disabled:opacity-60" :disabled="exporting" @click="exportPdf">
          <span v-if="!exporting">导出 PDF 报告（文本，可复制中文）</span>
          <span v-else>正在导出（文本）…</span>
        </button>
        <button class="btn-amber disabled:opacity-60" :disabled="exporting" @click="exportPdfFull">
          <span v-if="!exporting">导出 PDF 报告（图文）</span>
          <span v-else>正在导出（图文）…</span>
        </button>
      </div>
      <div class="flex flex-wrap items-center justify-end gap-3 text-sm text-gray-700 mt-1">
        <label class="flex items-center gap-1"><input type="checkbox" v-model="exportOptions.includeCover" /> 封面</label>
        <label class="flex items-center gap-1"><input type="checkbox" v-model="exportOptions.includeFooter" /> 页脚页码</label>
        <div class="flex items-center gap-2">
          <span>范围：</span>
          <select v-model="exportOptions.range" class="border rounded px-2 py-1">
            <option value="all">全部页面</option>
            <option value="basic-dates">仅 “基本信息 + 择日”</option>
          </select>
        </div>
      </div>

      <div ref="reportRef">
        <!-- 档案切换 -->
        <div class="flex items-center justify-center gap-3 flex-wrap">
          <label class="text-sm text-gray-600">档案：</label>
          <select v-model="activeProfileId" class="border rounded px-2 py-1">
            <option v-for="p in profiles" :key="p.id" :value="p.id">{{ p.name }}</option>
          </select>
          <button class="px-2 py-1 text-sm bg-gray-200 rounded" @click="startEdit">编辑当前</button>
          <button class="px-2 py-1 text-sm bg-red-200 rounded" @click="removeActive">删除当前</button>
        </div>

        <!-- 档案表单 -->
        <section class="rounded-xl border p-4 glass">
          <div class="font-medium mb-2">{{ editing ? '编辑档案' : '新增档案' }}</div>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
            <input class="border rounded px-2 py-1" placeholder="姓名" v-model="form.name" />

            <!-- 隐藏 12地支/天干选择 -->
            <select v-if="false" v-model="form.dayStem" class="border rounded px-2 py-1">
              <option>甲</option><option>乙</option><option>丙</option><option>丁</option><option>戊</option>
              <option>己</option><option>庚</option><option>辛</option><option>壬</option><option>癸</option>
            </select>

            <input class="border rounded px-2 py-1" type="date" v-model="form.birthDate" />
            <input class="border rounded px-2 py-1" type="time" step="60" v-model="form.birthTime" />

            <!-- 出生地选择器 -->
            <select class="border rounded px-2 py-1" v-model="form.birthPlace" @change="onBirthPlaceChange((form.birthPlace as any))">
              <option value="" disabled>选择出生地</option>
              <option v-for="p in places" :key="p.name" :value="p.name">{{ p.name }}</option>
            </select>

            <!-- 经纬度只读显示 -->
            <div class="md:col-span-2">
              <div class="flex items-center gap-2">
                <input class="border rounded px-2 py-1 w-32" aria-label="经度" placeholder="东经" type="text" :value="form.longitude ?? ''" readonly />
                <input class="border rounded px-2 py-1 w-32" aria-label="纬度" placeholder="北纬" type="text" :value="form.latitude ?? ''" readonly />
                <span class="text-xs text-gray-500">（根据出生地自动带出）</span>
              </div>
              <p v-if="formErrors.longitude" class="text-xs text-red-600 mt-1">{{ formErrors.longitude }}</p>
              <p v-if="formErrors.latitude" class="text-xs text-red-600">{{ formErrors.latitude }}</p>
            </div>

            <!-- 隐藏 时区搜索 + 下拉 -->
            <div v-if="false" class="md:col-span-2 grid grid-cols-1 md:grid-cols-2 gap-2">
              <input class="border rounded px-2 py-1" placeholder="搜索时区（输入关键字，如 shanghai/los_angeles）" v-model="tzQuery" />
              <select v-model="form.tz" class="border rounded px-2 py-1">
                <option v-for="tz in tzFiltered" :key="tz" :value="tz">{{ tz }}</option>
              </select>
            </div>

            <label class="flex items-center gap-2 md:col-span-2 text-sm">
              <input type="checkbox" v-model="form.useTrueSolar" /> 使用真太阳时校准
            </label>
          </div>
          <div class="mt-3 flex items-center gap-2">
            <button class="btn-orange" @click="submitForm">{{ editing ? '保存' : '添加' }}</button>
            <button class="btn-ghost" @click="resetForm">重置</button>
          </div>

          <div v-if="formPreviewReady" class="mt-4 rounded-lg border glass p-3 text-sm space-y-1">
            <div class="font-medium mb-1">实时预览（依据“新增/编辑档案”当前输入）</div>
            <div>八字：<b>{{ formPreview?.gzYear }}</b> · <b>{{ formPreview?.gzMonth }}</b> · <b>{{ formPreview?.gzDay }}</b> · <b>{{ formPreview?.hourPillar }}</b></div>
            <div>推导日主：{{ formPreview?.dStem }}；五行占比：金{{ formPreview?.energy[0] }}%、木{{ formPreview?.energy[1] }}%、水{{ formPreview?.energy[2] }}%、火{{ formPreview?.energy[3] }}%、土{{ formPreview?.energy[4] }}%</div>
            <div>神煞（专业·核心）：{{ Array.isArray(formPreview?.shensha) ? (formPreview!.shensha as string[]).join('、') : formPreview?.shensha }}</div>
          </div>
        </section>
      </div>

      <!-- 日期切换 -->
      <div class="flex items-center justify-center gap-3">
        <button class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300" @click="prevDay">前一天</button>
        <button class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300" @click="today">今天</button>
        <button class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300" @click="nextDay">后一天</button>
      </div>

      <!-- 基本信息 -->
      <section id="sec-basic" class="rounded-xl border p-4 glass space-y-2">
        <div class="text-lg">公历：{{ cal.solarDate }}</div>
        <div>农历：{{ cal.lunarDate }}</div>
        <div>干支：{{ cal.gzYear }} · {{ cal.gzMonth }} · {{ cal.gzDay }}</div>
        <div>节气：{{ cal.jieqi }}</div>
        <div class="mt-2 text-sm text-gray-600">当前档案：{{ activeProfile.name }}；推导日主：{{ derivedDayStem }}；出生时柱：{{ derivedHourPillar || '未填写完整信息' }}</div>
        <div class="text-sm text-gray-600">当日关系：{{ fortune.relation }}；评级：{{ fortune.rating }}</div>
      </section>

      <!-- 五行图 -->
      <section class="rounded-xl border glass p-4 h-56">
        <div class="mb-2 font-medium">五行占比（基于档案八字）</div>
        <div v-if="!chartOk" class="text-sm text-gray-500">未检测到 Chart.js。可执行：<code>npm i chart.js</code></div>
        <canvas v-else ref="fiveCanvas" class="w-full h-full"></canvas>
      </section>

      <!-- 趋势图 -->
      <section class="rounded-xl border bg-white/70 shadow-sm p-4 h-64">
        <div class="mb-2 font-medium">未来12个月趋势（示例）</div>
        <div v-if="!echartsOk" class="text-sm text-gray-500">未检测到 ECharts。可执行：<code>npm i echarts</code></div>
        <div v-else ref="trendDiv" class="w-full h-full"></div>
      </section>

      <!-- 择日 -->
      <section id="sec-dates" class="rounded-xl border bg-white/70 shadow-sm p-4">
        <div class="mb-2 font-medium">30天内推荐日期</div>
        <div class="flex items-center gap-2 mb-3">
          <span class="text-sm text-gray-600">事项：</span>
          <select v-model="eventType as any" class="border rounded px-2 py-1">
            <option v-for="et in eventTypes" :key="et" :value="et">{{ et }}</option>
          </select>
        </div>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div>
            <div class="font-medium mb-1">最优 3 日</div>
            <ul class="list-disc pl-5">
              <li v-for="d in bestDays.top3" :key="'top'+d.date">
                {{ d.date }}（{{ d.gzDay }}）：<b>{{ d.rating }}</b>
              </li>
            </ul>
          </div>
          <div>
            <div class="font-medium mb-1">备选 5 日</div>
            <ul class="list-disc pl-5">
              <li v-for="d in bestDays.backup5" :key="'bk'+d.date">
                {{ d.date }}（{{ d.gzDay }}）：{{ d.rating }}
              </li>
            </ul>
          </div>
        </div>
      </section>
    </div>
  </main>
</template>

<style scoped>
/* 可选：局部样式 */
</style>
