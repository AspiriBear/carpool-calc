<script setup>
import { ref, computed, nextTick, onMounted, watch } from 'vue';
import * as XLSX from 'xlsx';
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';

// 新配色
const theme = {
  '--primary-100': '#8FBF9F',
  '--primary-200': '#68a67d',
  '--primary-300': '#24613b',
  '--accent-100': '#F18F01',
  '--accent-200': '#833500',
  '--text-100': '#353535',
  '--text-200': '#5f5f5f',
  '--bg-100': '#F5ECD7',
  '--bg-200': '#ebe2cd',
  '--bg-300': '#c2baa6',
};
Object.entries(theme).forEach(([k, v]) => document.documentElement.style.setProperty(k, v));

// 欢迎页状态
const showWelcome = ref(true);
const title = ref('');
const exchangeRate = ref(0.052);

// 主流程数据
const members = ref([]); // 只在list区体现
const items = ref([]);
let itemId = 1;

// 输入栏
const inputMember = ref('CN');
const newItem = ref({
  name: '',
  originalPriceJPY: '',
  quantity: 1,
  discountType: '85-percent',
  customFinalPriceJPY: '',
});

const discountOptions = [
  { value: '85-percent', label: '85折' },
  { value: 'no-discount', label: '不打折' },
  { value: 'custom', label: '自定义折后价' },
];

const memberListAreaRef = ref(null);
const listAreaMarginTop = ref('280px'); // Initial estimation

function getDiscountedItemPriceJPY(item) {
  if (item.discountType === '85-percent') return item.originalPriceJPY * 0.85 * item.quantity;
  if (item.discountType === 'no-discount') return item.originalPriceJPY * item.quantity;
  if (item.discountType === 'custom') return item.customFinalPriceJPY * item.quantity;
  return 0;
}
function getItemUnitPriceCNY(item) {
  return (getDiscountedItemPriceJPY(item) / item.quantity) * exchangeRate.value;
}
const totalDiscountedJPY = computed(() => items.value.reduce((sum, item) => sum + getDiscountedItemPriceJPY(item), 0));
const totalCNY = computed(() => totalDiscountedJPY.value * exchangeRate.value);

// 按成员分组
const memberList = computed(() => {
  // 以昵称分组
  const map = {};
  items.value.forEach(item => {
    if (!map[item.member]) map[item.member] = [];
    map[item.member].push(item);
  });
  return Object.entries(map).map(([nickname, items]) => ({ nickname, items }));
});

function getMemberOriginalPriceCNY(member) {
  return member.items.reduce((sum, i) => sum + i.originalPriceJPY * i.quantity, 0) * exchangeRate.value;
}
function getMemberDiscountedPriceCNY(member) {
  return member.items.reduce((sum, i) => sum + getDiscountedItemPriceJPY(i), 0) * exchangeRate.value;
}
function getMemberRefundAmountCNY(member) {
  return getMemberOriginalPriceCNY(member) - getMemberDiscountedPriceCNY(member);
}

function calculateAndSetMargin() {
  if (showWelcome.value) return; // Only calculate in main page

  const headerEl = document.querySelector('.main-header');
  const infoBarEl = document.querySelector('.info-bar');
  const inputAreaEl = document.querySelector('.input-area');

  if (headerEl && infoBarEl && inputAreaEl && memberListAreaRef.value) {
    const headerHeight = headerEl.offsetHeight;
    const infoBarHeight = infoBarEl.offsetHeight;
    const inputAreaHeight = inputAreaEl.offsetHeight;

    // For mobile fixed positioning
    if (window.innerWidth <= 600) {
      // Ensure Info Bar starts exactly after Header with a small buffer
      infoBarEl.style.top = `${headerEl.getBoundingClientRect().bottom + 1}px`; // Add 1px buffer
      // Ensure Input Area starts exactly after Info Bar with a small buffer
      inputAreaEl.style.top = `${infoBarEl.getBoundingClientRect().bottom + 1}px`; // Add 1px buffer

      // Calculate total height for mobile list area margin
      const totalHeight = headerEl.offsetHeight + infoBarEl.offsetHeight + inputAreaEl.offsetHeight;
      listAreaMarginTop.value = `${totalHeight}px`;
    } else {
      // For desktop sticky positioning
      // Ensure Info Bar starts exactly after Header
      infoBarEl.style.top = `${headerHeight}px`;
      // Ensure Input Area starts exactly after Info Bar
      inputAreaEl.style.top = `${headerHeight + infoBarHeight}px`;

      // Calculate total height for desktop list area margin
      const totalHeight = headerEl.offsetHeight + infoBarEl.offsetHeight + inputAreaEl.offsetHeight;
      listAreaMarginTop.value = `${totalHeight}px`;
    }
  }
}

function addItem() {
  if (!inputMember.value || !newItem.value.originalPriceJPY || newItem.value.quantity < 1) return;
  items.value.push({
    id: itemId++,
    member: inputMember.value,
    name: newItem.value.name && newItem.value.name.trim() ? newItem.value.name : '拼车物品',
    originalPriceJPY: Number(newItem.value.originalPriceJPY),
    quantity: Number(newItem.value.quantity),
    discountType: newItem.value.discountType,
    customFinalPriceJPY: newItem.value.discountType === 'custom' ? Number(newItem.value.customFinalPriceJPY) : 0,
  });
  // 清空输入栏
  newItem.value = { name: '', originalPriceJPY: '', quantity: 1, discountType: '85-percent', customFinalPriceJPY: '' };
}
function removeItem(itemId) {
  const idx = items.value.findIndex(i => i.id === itemId);
  if (idx !== -1) items.value.splice(idx, 1);
}

function startApp() {
  if (!title.value || !exchangeRate.value) return;
  showWelcome.value = false;
}

// 导出为 XLSX
async function handleExportXLSX() {
  // 构造数据
  const aoa = [['成员', '物品名称', '原价(日元)', '数量', '折扣', '折后价(日元)', '折后单价(￥)']];
  const merges = [];
  let row = 1;
  memberList.value.forEach(member => {
    const startRow = row;
    member.items.forEach(item => {
      aoa.push([
        member.nickname,
        item.name,
        item.originalPriceJPY,
        item.quantity,
        discountOptions.find(opt => opt.value === item.discountType)?.label,
        getDiscountedItemPriceJPY(item).toFixed(2),
        getItemUnitPriceCNY(item).toFixed(2),
      ]);
      row++;
    });
    // 合并A列
    if (member.items.length > 1) {
      merges.push({ s: { r: startRow, c: 0 }, e: { r: row - 1, c: 0 } });
    }
    // 小计行
    aoa.push([
      '',
      '个人小计',
      '折前总价',
      getMemberOriginalPriceCNY(member).toFixed(2),
      '折后总价',
      getMemberDiscountedPriceCNY(member).toFixed(2),
      '应退差价',
      getMemberRefundAmountCNY(member).toFixed(2),
    ]);
    merges.push({ s: { r: row, c: 1 }, e: { r: row, c: 1 } }); // 只合并"个人小计"
    row++;
  });
  const ws = XLSX.utils.aoa_to_sheet(aoa);
  ws['!merges'] = merges;
  // 设置样式（底色+居中）
  const memberColor = { fill: { fgColor: { rgb: 'D9EAD3' } }, alignment: { vertical: 'center', horizontal: 'center' } };
  const subtotalColor = {
    fill: { fgColor: { rgb: 'F9CB9C' } },
    alignment: { vertical: 'center', horizontal: 'center' },
  };
  let r = 1;
  memberList.value.forEach(member => {
    // 成员名行
    ws[`A${r + 1}`].s = memberColor;
    // 小计行
    ws[`B${r + member.items.length + 1}`].s = subtotalColor;
    r += member.items.length + 1;
  });
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, '拼车清单');
  XLSX.writeFile(wb, `${title.value || '拼车清单'}.xlsx`);
}

// 导出为 PDF
async function handleExportPDF() {
  await nextTick();
  const el = document.querySelector('.member-list-area');
  if (!el) return;
  const canvas = await html2canvas(el, { backgroundColor: null, scale: 2 });
  const imgData = canvas.toDataURL('image/png');
  // 以A4宽度自适应缩放
  const pdf = new jsPDF('p', 'mm', 'a4');
  const pageWidth = pdf.internal.pageSize.getWidth();
  const pageHeight = pdf.internal.pageSize.getHeight();
  const imgWidth = pageWidth;
  const imgHeight = canvas.height * (imgWidth / canvas.width);
  let position = 0;
  let remainHeight = imgHeight;
  while (remainHeight > 0) {
    pdf.addImage(imgData, 'PNG', 0, position, imgWidth, imgHeight);
    remainHeight -= pageHeight;
    if (remainHeight > 0) {
      pdf.addPage();
      position = 0;
    }
  }
  pdf.save(`${title.value || '拼车清单'}.pdf`);
}

// 导出为 PNG
async function handleExportPNG() {
  await nextTick();
  const el = document.querySelector('.member-list-area');
  if (!el) return;
  const canvas = await html2canvas(el, { backgroundColor: null, scale: 2 });
  const link = document.createElement('a');
  link.href = canvas.toDataURL('image/png');
  link.download = `${title.value || '拼车清单'}.png`;
  link.click();
}

function confirmExport(type) {
  let msg = '';
  if (type === 'xlsx') msg = '确定要导出为 Excel 吗？';
  if (type === 'pdf') msg = '确定要导出为 PDF 吗？';
  if (type === 'png') msg = '确定要导出为 PNG 吗？';
  if (window.confirm(msg)) {
    if (type === 'xlsx') handleExportXLSX();
    if (type === 'pdf') handleExportPDF();
    if (type === 'png') handleExportPNG();
  }
}

// Mobile export options visibility
const showMobileExportOptions = ref(false);

function toggleMobileExportOptions() {
  showMobileExportOptions.value = !showMobileExportOptions.value;
}

function handleMobileExport(type) {
  confirmExport(type);
  showMobileExportOptions.value = false; // Close options after selection
}

onMounted(() => {
  calculateAndSetMargin();
});

watch(
  items,
  () => {
    nextTick(() => {
      calculateAndSetMargin();
    });
  },
  { deep: true },
);

watch(showWelcome, newValue => {
  if (!newValue) {
    nextTick(() => {
      calculateAndSetMargin();
    });
  }
});
</script>

<template>
  <div class="carpool-app">
    <!-- 欢迎页 -->
    <div v-if="showWelcome" class="welcome-page">
      <div class="welcome-card">
        <h1>欢迎使用熊矾小工具</h1>
        <div class="welcome-form">
          <label>拼车名称</label>
          <input v-model="title" placeholder="如：骏河屋35" />
          <label>日元兑人民币汇率</label>
          <input v-model.number="exchangeRate" type="number" step="0.0001" min="0" placeholder="如：0.052" />
          <button class="start-btn" @click="startApp">开始</button>
        </div>
      </div>
    </div>

    <!-- 主流程页面 -->
    <div v-else class="main-page">
      <!-- 顶部固定/吸顶区，所有端只渲染一套 -->
      <div class="main-header">
        <div class="main-title">{{ title }}</div>
        <div class="main-rate">汇率：1日元 = {{ exchangeRate }} 人民币</div>
      </div>
      <div class="info-bar">
        <div class="info-main">
          日元总价 <span class="info-main-value">{{ totalDiscountedJPY.toFixed(0) }}</span>
        </div>
        <div class="info-main">
          人民币总价 <span class="info-main-value">{{ totalCNY.toFixed(2) }}</span>
        </div>
      </div>
      <div class="input-area">
        <!-- 桌面端原有布局 -->
        <div class="input-row input-row-desktop">
          <div class="input-group">
            <label>成员昵称</label>
            <input v-model="inputMember" placeholder="如：苯磺酸熊" />
          </div>
          <div class="input-group">
            <label>物品名称</label>
            <input v-model="newItem.name" placeholder="如：吧唧" />
          </div>
          <div class="input-group">
            <label>日元原价</label>
            <input v-model.number="newItem.originalPriceJPY" type="number" min="0" placeholder="如：1999" />
          </div>
          <div class="input-group">
            <label>数量</label>
            <input v-model.number="newItem.quantity" type="number" min="1" placeholder="1" />
          </div>
          <div class="input-group">
            <label>享受折扣</label>
            <select v-model="newItem.discountType">
              <option v-for="opt in discountOptions" :value="opt.value">{{ opt.label }}</option>
            </select>
          </div>
          <div class="input-group" v-if="newItem.discountType === 'custom'">
            <label>自定义折后价</label>
            <input v-model.number="newItem.customFinalPriceJPY" type="number" min="0" placeholder="折后日元价" />
          </div>
          <button class="add-btn" @click="addItem"><i class="fa-solid fa-plus"></i> 新增物品</button>
        </div>
        <!-- 移动端两列布局 -->
        <div class="input-row input-row-mobile">
          <div class="input-group">
            <label>成员昵称</label>
            <input v-model="inputMember" placeholder="如：苯磺酸熊" />
          </div>
          <div class="input-group">
            <label>物品名称</label>
            <input v-model="newItem.name" placeholder="如：吧唧" />
          </div>
        </div>
        <div class="input-row input-row-mobile">
          <div class="input-group">
            <label>日元原价</label>
            <input v-model.number="newItem.originalPriceJPY" type="number" min="0" placeholder="如：1999" />
          </div>
          <div class="input-group">
            <label>数量</label>
            <input v-model.number="newItem.quantity" type="number" min="1" placeholder="1" />
          </div>
        </div>
        <div class="input-row input-row-mobile">
          <div class="input-group" style="flex: 2">
            <label>享受折扣</label>
            <select v-model="newItem.discountType">
              <option v-for="opt in discountOptions" :value="opt.value">{{ opt.label }}</option>
            </select>
            <div v-if="newItem.discountType === 'custom'" style="margin-top: 0.3em">
              <input v-model.number="newItem.customFinalPriceJPY" type="number" min="0" placeholder="折后日元价" />
            </div>
          </div>
          <button class="add-btn" @click="addItem" style="flex: 1"><i class="fa-solid fa-plus"></i> 新增物品</button>
        </div>
      </div>
      <!-- list区域 -->
      <div class="member-list-area" :style="{ marginTop: listAreaMarginTop }" ref="memberListAreaRef">
        <div v-for="member in memberList" :key="member.nickname" class="member-info-card">
          <div class="member-title">{{ member.nickname }}</div>
          <div class="member-block">
            <table class="item-table">
              <thead>
                <tr>
                  <th class="item-name-col">
                    <span class="item-name-label-desktop"
                      ><!-- 桌面端 --><i class="fa-solid fa-box" style="margin-right: 0.3em"></i>物品名称</span
                    >
                    <span class="item-name-label-mobile"><!-- 移动端 --><i class="fa-solid fa-box"></i></span>
                  </th>
                  <th>原价(日元)</th>
                  <th>数量</th>
                  <th>折扣</th>
                  <th>折后价(日元)</th>
                  <th>折后单价(￥)</th>
                  <th>操作</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in member.items" :key="item.id">
                  <td data-label="物品名称" class="item-name-col">
                    <span class="item-name-desktop">{{ item.name }}</span>
                    <span class="item-name-mobile"><i class="fa-solid fa-box"></i></span>
                  </td>
                  <td data-label="原价(日元)">{{ item.originalPriceJPY }}</td>
                  <td data-label="数量">{{ item.quantity }}</td>
                  <td data-label="折扣">{{ discountOptions.find(opt => opt.value === item.discountType)?.label }}</td>
                  <td data-label="折后价(日元)">{{ getDiscountedItemPriceJPY(item).toFixed(2) }}</td>
                  <td data-label="折后单价(￥)">{{ getItemUnitPriceCNY(item).toFixed(2) }}</td>
                  <td data-label="操作">
                    <button class="item-remove-btn" @click="removeItem(item.id)">
                      <i class="fa-solid fa-trash"></i>
                    </button>
                  </td>
                </tr>
                <tr v-if="member.items.length === 0">
                  <td colspan="7" style="color: var(--text-200)">暂无物品</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="member-summary">
            <span>折前总价：￥{{ getMemberOriginalPriceCNY(member).toFixed(2) }}</span>
            <span>折后总价：￥{{ getMemberDiscountedPriceCNY(member).toFixed(2) }}</span>
            <span>应退差价：￥{{ getMemberRefundAmountCNY(member).toFixed(2) }}</span>
          </div>
        </div>
      </div>
      <!-- 底部导出按钮区（桌面端） -->
      <div class="export-bar">
        <button class="export-btn" @click="handleExportXLSX"><i class="fa-solid fa-file-excel"></i> 导出为 XLSX</button>
        <button class="export-btn" @click="handleExportPDF"><i class="fa-solid fa-file-pdf"></i> 导出为 PDF</button>
        <button class="export-btn" @click="handleExportPNG"><i class="fa-solid fa-image"></i> 导出为 PNG</button>
      </div>
      <!-- 移动端悬浮导出按钮 -->
      <div class="fab-export-group">
        <!-- Export options -->
        <div class="fab-options" v-if="showMobileExportOptions">
          <button class="fab-btn fab-xlsx" @click="handleMobileExport('xlsx')">
            <i class="fa-solid fa-file-excel"></i>
          </button>
          <button class="fab-btn fab-pdf" @click="handleMobileExport('pdf')">
            <i class="fa-solid fa-file-pdf"></i>
          </button>
          <button class="fab-btn fab-png" @click="handleMobileExport('png')"><i class="fa-solid fa-image"></i></button>
        </div>
        <!-- Main export button -->
        <button class="fab-btn fab-main" @click="toggleMobileExportOptions">
          <i class="fa-solid fa-file-export"></i>
        </button>
      </div>
    </div>
    <!-- 点缀星星装饰 -->
    <i class="fa-solid fa-star star-deco star-deco-1"></i>
    <i class="fa-solid fa-star star-deco star-deco-2"></i>
    <i class="fa-solid fa-star star-deco star-deco-3"></i>
    <i class="fa-solid fa-star-half-stroke star-deco star-deco-4"></i>
    <i class="fa-regular fa-star star-deco star-deco-5"></i>
  </div>
</template>

<style>
@import url('https://fontsapi.zeoseven.com/244/main/result.css');
html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  background: #f8f6ed;
  box-sizing: border-box;
  overflow-x: hidden;
  font-family: 'QiushuiShotai', system-ui, Avenir, Helvetica, Arial, sans-serif;
}
* {
  box-sizing: border-box;
}
input:focus,
select:focus,
textarea:focus {
  outline: none;
  border-color: var(--primary-200);
  box-shadow: 0 0 0 2px var(--primary-100), 0 2px 8px #c2baa633;
  background: #fffbe9;
  transition: box-shadow 0.2s, border-color 0.2s;
}
</style>

<style scoped>
.carpool-app {
  padding: 0 0 48px 0; /* Keep desktop padding */
  display: flex;
  flex-direction: column;
  align-items: center;
}
.main-header,
.info-bar,
.input-area,
.member-list-area {
  max-width: 900px;
  width: 100%;
  margin: 0 auto 0 auto;
}
.info-bar {
  max-width: 900px;
  margin: 1.2em auto 0 auto;
  width: 100%;
  display: flex;
  align-items: center;
  background: var(--primary-100);
  border-radius: 18px;
  box-shadow: 0 2px 8px #c2baa6cc;
  padding: 1em 2em;
  gap: 2em;
  font-size: 1.05em;
  color: #fff;
  justify-content: space-between;
  margin-bottom: 0.8em; /* Add space below info bar */
}
.welcome-page,
.main-page {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  box-sizing: border-box;
}
.welcome-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
}
.welcome-card {
  background: rgba(235, 226, 205, 0.92);
  border-radius: 18px;
  box-shadow: 0 2px 16px #c2baa6cc;
  padding: 2.5em 2.5em 2em 2.5em;
  min-width: 320px;
  max-width: 90vw;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.welcome-card h1 {
  color: var(--primary-200);
  margin-bottom: 1.5em;
}
.welcome-form {
  display: flex;
  flex-direction: column;
  gap: 1.2em;
  width: 100%;
}
.welcome-form label {
  color: var(--text-200);
  font-size: 1em;
  margin-bottom: 0.2em;
}
.welcome-form input {
  border: 1px solid var(--primary-100);
  border-radius: 8px;
  padding: 0.5em 1em;
  background: var(--bg-100);
  color: var(--text-100);
  font-size: 1.1em;
}
.start-btn {
  background: var(--primary-100);
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 0.7em 2em;
  font-size: 1.1em;
  cursor: pointer;
  margin-top: 1em;
  transition: background 0.2s;
}
.start-btn:hover {
  background: var(--primary-200);
}

.main-header {
  max-width: 900px;
  margin: 0 auto 0 auto;
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: var(--bg-200);
  border-radius: 18px;
  box-shadow: 0 2px 8px #c2baa6cc;
  padding: 1.2em 2em;
  font-size: 1.2em;
  color: var(--primary-300);
}
.main-title {
  font-weight: bold;
  font-size: 1.3em;
  color: var(--primary-300);
}
.main-rate {
  color: var(--text-200);
  font-size: 1em;
}
.input-area {
  max-width: 900px;
  margin: 1.2em auto 0 auto;
  width: 100%;
  background: var(--bg-200);
  border-radius: 18px;
  box-shadow: 0 2px 8px #c2baa6cc;
  padding: 1.2em 2em 0.7em 2em;
}
.input-row {
  display: flex;
  flex-wrap: wrap;
  gap: 1.2em;
  align-items: flex-end;
  justify-content: flex-start;
}
.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.2em;
}
.input-group label {
  font-size: 0.95em;
  color: var(--text-200);
  margin-bottom: 0.1em;
}
.input-group input,
.input-group select {
  border: 1px solid var(--primary-100);
  border-radius: 8px;
  padding: 0.4em 0.8em;
  background: var(--bg-100);
  color: var(--text-100);
  font-size: 1em;
}
.add-btn {
  background: var(--accent-100);
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 0.7em 1.8em;
  font-size: 1.1em;
  cursor: pointer;
  margin-left: 1em;
  transition: background 0.2s;
}
.add-btn:hover {
  background: var(--accent-200);
}

.info-main {
  font-weight: bold;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 0.2em 0.8em;
  font-size: 1em;
  display: flex;
  align-items: center;
  gap: 0.5em;
}
.info-main-value {
  font-size: 1.1em;
  letter-spacing: 1px;
}

.member-list-area {
  max-width: 900px;
  width: 100%;
  margin: 1.2em auto 0 auto;
  padding-bottom: 16px;
  display: flex;
  flex-direction: column;
  gap: 1.2em;
}
.member-info-card {
  background: var(--bg-200);
  border-radius: 18px;
  box-shadow: 0 2px 8px #c2baa6cc;
  padding: 1.2em 2em;
  display: flex;
  flex-direction: column;
  gap: 0.7em;
}
.member-title {
  font-weight: bold;
  color: var(--primary-300);
  font-size: 1.1em;
  margin-bottom: 0.5em;
}
.member-block {
  background: var(--bg-100);
  border-radius: 8px;
  padding: 0.5em 0.5em 0.5em 0.5em;
  margin-bottom: 0.5em;
}
.item-table {
  width: 100%;
  border-collapse: collapse;
  background: var(--bg-100);
  border-radius: 8px;
  overflow: hidden;
  margin-bottom: 0.5em;
}
.item-table th,
.item-table td {
  border: 1px solid var(--primary-100);
  padding: 0.4em 0.7em;
  text-align: center;
}
.item-name-label-mobile,
.item-name-mobile {
  display: none;
}
.item-name-label-desktop,
.item-name-desktop {
  display: inline;
}
.member-summary {
  display: flex;
  gap: 2em;
  color: var(--accent-200);
  font-size: 1.08em;
  margin-top: 0.5em;
}

.export-bar {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  max-width: 900px;
  margin: 0 auto;
  background: var(--primary-100);
  border-radius: 0 0 18px 18px;
  box-shadow: 0 -2px 8px #c2baa6cc;
  padding: 1.2em 0;
  display: flex;
  justify-content: center;
  gap: 2em;
  z-index: 100;
}
.export-btn {
  background: var(--accent-100);
  color: #fff;
  border: none;
  border-radius: 18px;
  padding: 0.8em 2.5em;
  font-size: 1.15em;
  cursor: pointer;
  transition: background 0.2s;
}
.export-btn:hover {
  background: var(--accent-200);
}

@media (max-width: 900px) {
  .main-header,
  .info-bar,
  .member-list-area,
  .input-area,
  .input-form {
    max-width: 98vw;
    padding: 1em 0.5em;
  }
  .input-row {
    flex-wrap: wrap;
    gap: 0.7em;
  }
  .member-summary {
    flex-direction: column;
    gap: 0.5em;
  }
  .info-bar {
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 0.7em 0.5em;
    gap: 0.5em;
    font-size: 0.98em;
    margin-bottom: 0.8em; /* Add space below info bar */
    word-break: break-all;
    overflow-wrap: break-word;
  }
  .info-main {
    width: 100%;
    text-align: center;
    word-break: break-all;
    overflow-wrap: break-word;
    font-size: 1em;
  }
  .info-main-value {
    font-size: 0.95em;
    word-break: break-all;
    overflow-wrap: break-word;
    display: inline-block;
    max-width: 90vw;
    white-space: pre-wrap;
  }
  /* Item name column display adjustment for mobile */
  .item-name-label-desktop,
  .item-name-desktop {
    display: none;
  }
  .item-name-label-mobile,
  .item-name-mobile {
    display: inline;
  }
  .member-list-area {
    position: relative;
    margin-top: 280px;
    height: calc(100vh - 280px - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 16px;
    max-width: 100vw;
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 100%; /* Use 100% width of parent */
    padding: 0.8em 0.5em; /* Further reduced horizontal padding for mobile */
  }
}

@media (max-width: 600px) {
  .carpool-app {
    padding: 0 !important; /* Keep vertical padding */
    min-height: 100vh;
  }
  .main-page {
    width: 100%; /* Ensure main-page takes full width on mobile */
    padding: 0 0.5em; /* Keep horizontal padding for content inside main-page */
    box-sizing: border-box;
  }
  .main-header {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 101;
    border-radius: 0;
    max-width: 100vw;
    box-shadow: 0 2px 8px #c2baa6cc;
    height: 56px;
    min-height: 56px;
    background: var(--bg-200);
    padding: 0.2em 0.5em;
    display: flex;
    align-items: center;
  }
  .info-bar {
    position: fixed;
    left: 0;
    right: 0;
    z-index: 102;
    border-radius: 0;
    max-width: 100vw;
    box-shadow: 0 2px 8px #c2baa6cc;
    height: 66px;
    min-height: 66px;
    background: var(--primary-100);
    background-clip: padding-box;
    color: #fff;
    padding: 0.4em 0.5em; /* Keep padding */
    display: flex; /* Re-add flex properties */
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.3em;
    width: 100%; /* Ensure full width */
    margin: 0 auto; /* Center horizontally */
    margin-bottom: 0.8em; /* Add space below info bar */
  }
  .input-area {
    display: flex;
    flex-direction: column;
    gap: 0.2em;
    position: fixed;
    left: 0;
    right: 0;
    z-index: 103;
    max-width: 100vw;
    border-radius: 0;
    box-shadow: 0 2px 8px #c2baa6cc;
    background: rgba(235, 226, 205, 0.96);
    height: auto;
    min-height: 120px;
    padding: 0.7em 0.5em 0.7em 0.5em;
    margin: 0 !important; /* Force remove all margin */
    margin-top: 0 !important; /* Explicitly remove top margin */
  }
  .input-row-desktop {
    display: none !important; /* Hide desktop input row in mobile */
  }
  .input-row-mobile {
    display: flex !important;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 0.5em;
    margin-bottom: 0.3em;
    width: 100%;
  }
  .input-group {
    flex: 1 1 48%;
    min-width: 0;
    margin-bottom: 0.2em;
  }
  .add-btn {
    width: 100%;
    min-width: 0;
    margin-left: 0;
    margin-top: 0.8em;
    box-sizing: border-box;
    padding: 0.8em 1.8em;
  }
  .export-bar {
    display: none !important;
  }
  .fab-export-group {
    position: fixed;
    right: 16px;
    bottom: 80px;
    z-index: 999;
    display: flex;
    flex-direction: column;
    gap: 16px;
    align-items: flex-end; /* Align items to the right */
  }
  .fab-btn {
    width: 54px;
    height: 54px;
    border-radius: 50%;
    background: var(--accent-100);
    color: #fff;
    border: none;
    box-shadow: 0 2px 8px #c2baa6cc;
    font-size: 1.6em;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.2s;
  }
  .fab-btn:active {
    background: var(--accent-200);
  }
  .fab-xlsx {
    background: #1d6f42;
  }
  .fab-pdf {
    background: #b71c1c;
  }
  .fab-png {
    background: #f18f01;
  }
  .fab-main {
    background: var(--primary-200);
  }
  .fab-options {
    display: flex;
    flex-direction: column;
    gap: 16px;
    margin-bottom: 16px; /* Space between options and main button */
  }
  .member-list-area {
    position: relative;
    margin-top: 280px;
    height: calc(100vh - 280px - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 0 !important; /* Remove padding-bottom on mobile */
    max-width: 100vw;
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 100%; /* Use 100% width of parent */
    padding: 0.8em 0.5em; /* Further reduced horizontal padding for mobile */
  }
  .member-block {
    overflow-x: auto; /* Enable horizontal scrolling for the table */
    padding: 0.5em 0.8em; /* Adjusted padding for mobile */
  }
  .item-table {
    width: 100%;
    border-collapse: collapse;
    background: var(--bg-100);
    border-radius: 8px;
    overflow: hidden;
    margin-bottom: 0.5em;
  }
  /* Table as blocks on mobile */
  .item-table,
  .item-table thead,
  .item-table tbody,
  .item-table tr {
    display: block;
    width: 100%;
  }
  .item-table thead {
    display: none; /* Hide table header */
  }
  .item-table tr {
    margin-bottom: 0.8em; /* Add space between item blocks */
    border: 1px solid var(--primary-100); /* Add border for clarity */
    border-radius: 8px;
    background: var(--bg-100);
    padding: 0.8em;
    display: flex; /* Use flex for layout inside tr */
    flex-direction: column; /* Stack table cells vertically */
    gap: 0.5em; /* Space between cells */
  }
  .item-table td {
    border: none; /* Remove cell borders */
    text-align: left; /* Align text left */
    padding: 0.2em 0; /* Adjust padding */
    display: flex; /* Use flex to align label and value */
    justify-content: space-between; /* Space between label and value */
    align-items: center;
  }
  /* Add labels to table cells using data attributes or span if needed */
  .item-table td::before {
    content: attr(data-label) ':'; /* Use data-label for column titles */
    font-weight: bold;
    color: var(--text-200);
    margin-right: 1em;
    flex-shrink: 0; /* Prevent label from shrinking */
  }
  /* Adjust specific column styles if needed */
  .item-name-col {
    font-weight: bold;
    color: var(--primary-300);
  }
}
@media (min-width: 601px) {
  .input-row-desktop {
    display: flex !important;
  }
  .input-row-mobile {
    display: none !important;
  }
  .main-header {
    position: sticky;
    top: 0;
    z-index: 101;
  }
  .info-bar {
    position: sticky;
    z-index: 102;
  }
  .input-area {
    position: sticky;
    z-index: 103;
  }
  .member-list-area {
    position: relative;
    margin-top: 362px;
    height: calc(100vh - 362px - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 16px;
    max-width: 900px;
    width: 100vw;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 98vw;
    max-width: 900px;
  }
}
/* 点缀星星装饰 */
.star-deco {
  position: fixed;
  color: #f7d774;
  opacity: 0.7;
  pointer-events: none;
  z-index: 20;
  filter: drop-shadow(0 2px 4px #fff6c6cc);
}
.star-deco-1 {
  left: 2vw;
  top: 2vh;
  font-size: 1.8em;
  opacity: 0.6;
  transform: rotate(-15deg);
}
.star-deco-2 {
  right: 4vw;
  top: 8vh;
  font-size: 2.2em;
  opacity: 0.8;
  transform: rotate(10deg);
}
.star-deco-3 {
  left: 8vw;
  bottom: 6vh;
  font-size: 1.3em;
  opacity: 0.5;
  transform: rotate(20deg);
}
.star-deco-4 {
  right: 10vw;
  bottom: 4vh;
  font-size: 2.5em;
  opacity: 0.7;
  color: #ffe9a7;
  transform: rotate(-8deg);
}
.star-deco-5 {
  left: 50vw;
  bottom: 2vh;
  font-size: 1.1em;
  opacity: 0.4;
  color: #f7d774;
  transform: rotate(12deg);
}

.welcome-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-100);
}
.welcome-card {
  background: var(--bg-200);
  border-radius: 18px;
  box-shadow: 0 2px 16px #c2baa6cc;
  padding: 2.5em 2.5em 2em 2.5em;
  min-width: 320px;
  max-width: 90vw;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.fixed-top-area {
  width: 100vw;
  max-width: 100vw;
  left: 0;
  right: 0;
  top: 0;
  z-index: 100;
  position: fixed;
  background: transparent;
}
@media (min-width: 601px) {
  .main-header {
    position: sticky;
    top: 0;
    z-index: 101;
  }
  .info-bar {
    position: sticky;
    z-index: 102;
  }
  .input-area {
    position: sticky;
    z-index: 103;
  }
  .member-list-area {
    position: relative;
    margin-top: 362px;
    height: calc(100vh - 362px - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 16px;
    max-width: 900px;
    width: 100vw;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 98vw;
    max-width: 900px;
  }
}
@media (max-width: 600px) {
  .fixed-top-area {
    position: fixed;
    width: 100vw;
    left: 0;
    right: 0;
    top: 0;
    z-index: 100;
    background: transparent;
  }
  .member-list-area {
    position: relative;
    margin-top: 280px;
    height: calc(100vh - v-bind(listAreaMarginTop) - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 0 !important; /* Remove padding-bottom on mobile */
    max-width: 100vw;
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 100%; /* Use 100% width of parent */
    padding: 1em 0.5em; /* Further reduced horizontal padding for mobile */
  }
  .member-block {
    overflow-x: auto; /* Enable horizontal scrolling for the table */
    padding: 0.5em 0.8em; /* Adjusted padding for mobile */
  }
  .item-table {
    width: 100%;
    border-collapse: collapse;
    background: var(--bg-100);
    border-radius: 8px;
    overflow: hidden;
    margin-bottom: 0.5em;
  }
}

/* Default hide both, show specific one in media queries */
.input-row-desktop,
.input-row-mobile {
  display: none;
}

@media (min-width: 601px) {
  .input-row-desktop {
    display: flex !important;
  }
  .input-row-mobile {
    display: none !important;
  }
  /* ... existing desktop styles ... */
}
@media (max-width: 600px) {
  .input-row-desktop {
    display: none !important; /* Ensure desktop is hidden on mobile */
  }
  .input-row-mobile {
    display: flex !important; /* Ensure mobile is shown on mobile */
    flex-direction: row;
    flex-wrap: wrap;
    gap: 0.5em;
    margin-bottom: 0.3em;
    width: 100%;
  }
  /* ... existing mobile styles ... */
}

.fab-export-group {
  display: none; /* Hide by default */
}

@media (max-width: 600px) {
  .fixed-top-area {
    position: fixed;
    width: 100vw;
    left: 0;
    right: 0;
    top: 0;
    z-index: 100;
    background: transparent;
  }
  .member-list-area {
    position: relative;
    margin-top: 280px;
    height: calc(100vh - v-bind(listAreaMarginTop) - 90px);
    overflow-y: auto;
    padding-top: 0.7em;
    padding-bottom: 0 !important; /* Remove padding-bottom on mobile */
    max-width: 100vw;
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 1.2em;
    align-items: center;
  }
  .member-info-card {
    margin: 0 auto 1.2em auto;
    width: 100%; /* Use 100% width of parent */
    padding: 1em 0.5em; /* Further reduced horizontal padding for mobile */
  }
  .member-block {
    overflow-x: auto; /* Enable horizontal scrolling for the table */
    padding: 0.5em 0.8em; /* Adjusted padding for mobile */
  }
  .item-table {
    width: 100%;
    border-collapse: collapse;
    background: var(--bg-100);
    border-radius: 8px;
    overflow: hidden;
    margin-bottom: 0.5em;
  }
  .fab-export-group {
    position: fixed;
    right: 16px;
    bottom: 80px;
    z-index: 999;
    display: flex; /* Show on mobile */
    flex-direction: column;
    gap: 16px;
    align-items: flex-end; /* Align items to the right */
  }
}
</style>
