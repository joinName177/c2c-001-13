<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue';
import { LocalTimeRepository } from '../adapters/local-time.repository';
import { CATEGORIES, CATEGORY_META, UNDO_WINDOW_MS, type InvestmentRecord, type TimeCategory, type UndoEntry } from '../core/models';
import { balanceSheet, cloneSnapshot, metricsFor, portfolioScore, record, sampleSnapshot, suggestions, totalInvested } from '../core/time-engine';

const repo = new LocalTimeRepository();
const tab = ref<'总览' | '记录' | '组合' | '分析'>('总览');
const notice = ref('');
const state = reactive(repo.load());
const accountDraft = reactive({ dailyHours: state.account.dailyHours, weeklyHours: state.account.weeklyHours });
const draft = reactive({ category: '事业' as TimeCategory, hours: 2, note: '' });
const metrics = computed(() => metricsFor(state.records));
const invested = computed(() => totalInvested(state.records));
const score = computed(() => portfolioScore(metrics.value, state.account));
const sheet = computed(() => balanceSheet(metrics.value, state.account));
const tips = computed(() => suggestions(metrics.value, state.account));
const recent = computed(() => [...state.records].sort((a, b) => b.date.localeCompare(a.date)).slice(0, 5));

const pendingDelete = ref<InvestmentRecord | null>(null);
const undoEntry = ref<UndoEntry | null>(null);
const undoChanges = ref<string[]>([]);
const undoRemaining = ref(0);
const undoConflictOpen = ref(false);
let undoTimer: number | undefined;

function save() { repo.save({ account: state.account, records: state.records }); }
function markUndoChange(label: string) { if (undoEntry.value) undoChanges.value.push(label); }
function updateAccount() { state.account.dailyHours = Math.max(1, Number(accountDraft.dailyHours) || 1); state.account.weeklyHours = Math.max(1, Number(accountDraft.weeklyHours) || 1); state.account.updatedAt = new Date().toISOString(); save(); markUndoChange('更新了时间账户'); notice.value = '时间账户已更新'; }
function addRecord() { const item = record(draft.category, draft.hours, draft.note); state.records.unshift(item); save(); markUndoChange(`新增「${item.note}」`); draft.note = ''; notice.value = '一笔时间投资已入账'; tab.value = '记录'; }
function askDelete(item: InvestmentRecord) { pendingDelete.value = item; }
function confirmDelete() {
  const item = pendingDelete.value;
  if (!item) return;
  undoEntry.value = { snapshot: cloneSnapshot(state), deleted: { ...item }, expiresAt: Date.now() + UNDO_WINDOW_MS };
  undoChanges.value = [];
  state.records = state.records.filter((entry) => entry.id !== item.id);
  save();
  pendingDelete.value = null;
  notice.value = '';
  startUndoTimer();
}
function startUndoTimer() {
  window.clearInterval(undoTimer);
  tickUndoTimer();
  undoTimer = window.setInterval(tickUndoTimer, 1000);
}
function tickUndoTimer() {
  const entry = undoEntry.value;
  if (!entry) { window.clearInterval(undoTimer); return; }
  const remain = Math.ceil((entry.expiresAt - Date.now()) / 1000);
  if (remain <= 0) clearUndo(); else undoRemaining.value = remain;
}
function clearUndo() {
  undoEntry.value = null;
  undoChanges.value = [];
  undoConflictOpen.value = false;
  undoRemaining.value = 0;
  window.clearInterval(undoTimer);
}
function requestUndo() { if (!undoEntry.value) return; if (undoChanges.value.length) undoConflictOpen.value = true; else performUndo(); }
function performUndo() {
  const entry = undoEntry.value;
  if (!entry) return;
  state.account = { ...entry.snapshot.account };
  state.records = entry.snapshot.records.map((item) => ({ ...item }));
  accountDraft.dailyHours = state.account.dailyHours;
  accountDraft.weeklyHours = state.account.weeklyHours;
  save();
  clearUndo();
  notice.value = '已撤销删除，流水、组合与建议已还原';
}
function resetDemo() { clearUndo(); const fresh = sampleSnapshot(); state.account = fresh.account; state.records = fresh.records; save(); notice.value = '已恢复初始示例组合'; }
function onKeydown(event: KeyboardEvent) { if (event.key !== 'Escape') return; if (undoConflictOpen.value) undoConflictOpen.value = false; else if (pendingDelete.value) pendingDelete.value = null; }
onMounted(() => window.addEventListener('keydown', onKeydown));
onBeforeUnmount(() => { window.removeEventListener('keydown', onKeydown); window.clearInterval(undoTimer); });
</script>

<template>
  <div class="app-shell investment-app">
    <header class="topbar">
      <div class="brand"><div class="brand-mark">时</div><div><span class="eyebrow">LIFE CAPITAL / 01</span><h1>人生时间投资所</h1></div></div>
      <div class="top-actions"><span class="live-dot">本周账期进行中</span><button class="ghost-button" type="button" @click="resetDemo">↻ 重置演示</button></div>
    </header>
    <main class="page-wrap">
      <section class="hero-row">
        <div><p class="eyebrow accent-text">个人时间资产管理台</p><h2>把今天的小时，<em>配置成明天的底气。</em></h2><p class="hero-copy">你的时间不是被消耗的余额，而是一支每天都在重新开盘的投资组合。</p></div>
        <div class="hero-date"><span>当前周期</span><strong>WEEK 40</strong><small>2026 · 10 / 01 — 10 / 07</small></div>
      </section>
      <nav class="tabs" aria-label="功能模块"><button v-for="item in ['总览', '记录', '组合', '分析']" :key="item" type="button" :class="{ active: tab === item }" @click="tab = item as typeof tab">{{ item }}<span v-if="item === '记录'" class="tab-count">{{ state.records.length }}</span></button></nav>

      <section v-if="tab === '总览'" class="content-grid overview-grid">
        <article class="panel account-panel"><div class="panel-heading"><div><span class="eyebrow">TIME ACCOUNT</span><h3>时间账户</h3></div><span class="panel-index">01 / 05</span></div><div class="account-number"><strong>{{ state.account.weeklyHours }}</strong><span>可配置小时 / 周</span></div><div class="meter"><span :style="{ width: `${Math.min(100, invested / state.account.weeklyHours * 100)}%` }"></span></div><div class="split-line"><span>已配置 <b>{{ invested.toFixed(1) }}h</b></span><span>剩余 <b>{{ Math.max(0, state.account.weeklyHours - invested).toFixed(1) }}h</b></span></div><div class="account-form"><label>每日可支配<input v-model.number="accountDraft.dailyHours" type="number" min="1" step="0.5" />小时</label><label>每周可配置<input v-model.number="accountDraft.weeklyHours" type="number" min="1" step="1" />小时</label><button class="primary-button" type="button" @click="updateAccount">更新账户 →</button></div></article>
        <article class="panel score-panel"><div class="panel-heading"><div><span class="eyebrow">PORTFOLIO HEALTH</span><h3>组合健康度</h3></div><span class="panel-index">02 / 05</span></div><div class="score-layout"><div class="score-ring" :style="{ '--score': `${Math.min(100, score * 20)}%` }"><div><strong>{{ score }}</strong><span>/ 10</span></div></div><div><p class="score-label">复利潜力良好</p><p class="muted">基于 {{ state.records.length }} 笔投资记录与各资产类别的历史回报估算。</p><div class="legend-list"><span><i class="dot green"></i>配置平衡</span><span><i class="dot yellow"></i>可继续优化</span></div></div></div></article>
        <article class="panel chart-panel"><div class="panel-heading"><div><span class="eyebrow">ALLOCATION MAP</span><h3>时间资产配置</h3></div><button class="text-button" type="button" @click="tab = '组合'">查看组合 →</button></div><div class="allocation-layout"><div class="donut" :style="{ background: `conic-gradient(${metrics.map((item) => `${item.color} ${item.share * 100}%`).join(', ')})` }"><div><strong>{{ Math.round(invested) }}</strong><span>hours</span></div></div><div class="metric-list"><div v-for="item in metrics" :key="item.category" class="metric-row"><i class="category-symbol" :style="{ color: item.color }">{{ item.icon }}</i><span>{{ item.category }}</span><strong>{{ item.hours.toFixed(1) }}h</strong><small>{{ Math.round(item.share * 100) }}%</small></div></div></div></article>
        <article class="panel ledger-panel"><div class="panel-heading"><div><span class="eyebrow">RECENT ACTIVITY</span><h3>最近投资</h3></div><button class="text-button" type="button" @click="tab = '记录'">全部记录 →</button></div><div class="activity-list"><div v-for="item in recent" :key="item.id" class="activity-row"><span class="activity-icon" :style="{ background: CATEGORY_META[item.category].color }">{{ CATEGORY_META[item.category].icon }}</span><div><strong>{{ item.note }}</strong><small>{{ item.category }} · {{ new Date(item.date).toLocaleDateString('zh-CN', { month: 'numeric', day: 'numeric' }) }}</small></div><b>+{{ item.hours }}h</b></div></div></article>
      </section>

      <section v-else-if="tab === '记录'" class="content-grid record-grid"><article class="panel form-panel"><div class="panel-heading"><div><span class="eyebrow">NEW INVESTMENT</span><h3>记录一笔时间</h3></div><span class="panel-index">02 / 05</span></div><p class="muted intro">每一次记录，都是在告诉未来的自己：这段时间值得。</p><label>投资到<select v-model="draft.category"><option v-for="item in CATEGORIES" :key="item" :value="item">{{ item }} · {{ CATEGORY_META[item].label }}</option></select></label><label>投入小时<input v-model.number="draft.hours" type="number" min="0.5" step="0.5" /></label><label>这段时间做了什么<textarea v-model="draft.note" rows="4" placeholder="例如：完成了季度复盘"></textarea></label><button class="primary-button full-button" type="button" @click="addRecord">确认入账 →</button><p class="notice">{{ notice }}</p></article><article class="panel table-panel"><div class="panel-heading"><div><span class="eyebrow">INVESTMENT LEDGER</span><h3>投资流水</h3></div><span class="total-pill">{{ state.records.length }} 笔记录</span></div><div class="ledger-table"><div class="table-head"><span>资产类别</span><span>备注</span><span>小时</span><span>日期</span><span></span></div><div v-for="item in recent" :key="item.id" class="table-row"><span class="category-cell"><i :style="{ color: CATEGORY_META[item.category].color }">{{ CATEGORY_META[item.category].icon }}</i>{{ item.category }}</span><span>{{ item.note }}</span><strong>{{ item.hours }}h</strong><span>{{ new Date(item.date).toLocaleDateString('zh-CN') }}</span><button class="delete-button" type="button" title="删除记录" @click="askDelete(item)">×</button></div></div></article></section>

      <section v-else-if="tab === '组合'" class="content-grid portfolio-grid"><article class="panel wide-panel"><div class="panel-heading"><div><span class="eyebrow">ASSET ALLOCATION</span><h3>投资组合</h3></div><span class="panel-index">03 / 05</span></div><div class="portfolio-cards"><div v-for="item in metrics" :key="item.category" class="portfolio-card" :style="{ '--category-color': item.color }"><div class="portfolio-card-top"><span class="portfolio-icon">{{ item.icon }}</span><span>{{ Math.round(item.share * 100) }}%</span></div><h4>{{ item.category }}</h4><p>{{ CATEGORY_META[item.category].label }}</p><div class="mini-meter"><i :style="{ width: `${Math.min(100, item.share * 200)}%` }"></i></div><div class="portfolio-card-foot"><strong>{{ item.hours.toFixed(1) }}h</strong><span>ROI {{ item.roi.toFixed(2) }}x</span></div></div></div></article><article class="panel balance-panel"><div class="panel-heading"><div><span class="eyebrow">LIFE BALANCE SHEET</span><h3>人生资产负债表</h3></div><span class="panel-index">04 / 05</span></div><div class="balance-figure"><div><span>时间资产</span><strong>{{ sheet.assets }} <small>价值点</small></strong></div><div><span>未配置负债</span><strong class="liability">{{ sheet.liabilities }} <small>小时</small></strong></div><div class="balance-total"><span>净人生资产</span><strong>{{ sheet.equity }}</strong></div></div></article></section>

      <section v-else class="content-grid analysis-grid"><article class="panel suggestions-panel"><div class="panel-heading"><div><span class="eyebrow">REBALANCE SIGNALS</span><h3>配置优化建议</h3></div><span class="panel-index">05 / 05</span></div><div class="suggestion-list"><div v-for="(tip, index) in tips" :key="tip" class="suggestion"><span>0{{ index + 1 }}</span><p>{{ tip }}</p><b>→</b></div></div></article><article class="panel roi-panel"><div class="panel-heading"><div><span class="eyebrow">RETURN ON INVESTMENT</span><h3>时间 ROI 分析</h3></div></div><div class="roi-list"><div v-for="item in metrics" :key="item.category" class="roi-row"><div class="roi-name"><i :style="{ background: item.color }"></i><span>{{ item.category }}</span></div><div class="roi-bar"><span :style="{ width: `${item.roi / 2.5 * 100}%`, background: item.color }"></span></div><strong>{{ item.roi.toFixed(2) }}x</strong></div></div><p class="footnote">ROI 是基于精力恢复、能力复利、关系质量与长期机会成本的个人化估算，不代表财务回报。</p></article></section>
      <p v-if="notice && tab !== '记录'" class="toast">{{ notice }}</p>
    </main>

    <div v-if="pendingDelete" class="modal-mask" @click.self="pendingDelete = null">
      <div class="modal-card" role="dialog" aria-modal="true" aria-label="删除确认">
        <span class="eyebrow">CONFIRM DELETE</span>
        <h3>删除这笔时间投资？</h3>
        <div class="delete-summary"><i :style="{ color: CATEGORY_META[pendingDelete.category].color }">{{ CATEGORY_META[pendingDelete.category].icon }}</i><div><strong>{{ pendingDelete.note }}</strong><small>{{ pendingDelete.category }} · {{ pendingDelete.hours }}h · {{ new Date(pendingDelete.date).toLocaleDateString('zh-CN') }}</small></div></div>
        <p class="modal-copy">删除后 {{ UNDO_WINDOW_MS / 1000 }} 秒内可撤销并原样恢复；超时将永久删除，无法找回。</p>
        <div class="modal-actions"><button class="outline-button" type="button" @click="pendingDelete = null">取消</button><button class="danger-button" type="button" @click="confirmDelete">确认删除</button></div>
      </div>
    </div>

    <div v-if="undoEntry" class="undo-bar" role="status">
      <span class="undo-text">已删除「{{ undoEntry.deleted.note }}」</span><span class="undo-countdown">{{ undoRemaining }}s</span><button class="undo-button" type="button" @click="requestUndo">↩ 撤销</button><button class="undo-close" type="button" title="放弃撤销" @click="clearUndo">×</button>
    </div>

    <div v-if="undoConflictOpen && undoEntry" class="modal-mask" @click.self="undoConflictOpen = false">
      <div class="modal-card" role="alertdialog" aria-modal="true" aria-label="撤销冲突提示">
        <span class="eyebrow">UNDO CONFLICT</span>
        <h3>撤销可能覆盖后续变更</h3>
        <p class="modal-copy">删除「{{ undoEntry.deleted.note }}」之后，你又完成了 {{ undoChanges.length }} 项变更：</p>
        <ul class="change-list"><li v-for="(change, index) in undoChanges" :key="index">{{ change }}</li></ul>
        <p class="modal-copy">继续撤销会把流水、组合、健康度与建议整体还原到删除前的状态，以上变更将被覆盖。</p>
        <div class="modal-actions"><button class="outline-button" type="button" @click="undoConflictOpen = false">保留当前状态</button><button class="danger-button" type="button" @click="performUndo">仍然撤销</button></div>
      </div>
    </div>
  </div>
</template>
