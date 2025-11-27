<template>
  <view class="container" :class="{ 'dark-mode': settings.isDarkMode }">
    <view class="header">
      <view>
        <text class="app-title">智享生活</text>
      </view>
      <view class="row-center">
        <view class="header-icon" @click="toggleDarkMode">
          <text class="icon-text">{{ settings.isDarkMode ? '☀️' : '🌙' }}</text>
        </view>
      </view>
    </view>

    <scroll-view scroll-y class="content-area custom-scrollbar">
      
      <view v-if="currentTab === 'home'" class="page-animate">
        
        <view v-if="nearestCountdown" class="card gradient-card">
          <view>
            <text class="card-label">
               {{ nearestCountdown.type === 'birthday' ? '🎂 近期生日' : '📅 最近日程' }}
            </text>
            <view class="card-title">{{ nearestCountdown.title }}</view>
            <text class="card-sub">{{ nearestCountdown.date }}</text>
          </view>
          <view class="countdown-box">
            <text class="days-num">{{ nearestCountdown.days }}</text>
            <text class="days-label">天</text>
          </view>
        </view>

        <view class="grid-2">
          <view class="card orange-bg" @click="goToExpiring">
            <view class="row-center text-orange">
              <text>🕒 临期/过期</text>
            </view>
            <view class="stat-num">{{ expiringSoonCount }} <text class="stat-unit">件</text></view>
          </view>
          <view class="card teal-bg-light" @click="switchTab('wallet', 'month')">
            <view class="row-center text-teal">
              <text>💰 本月支出</text>
            </view>
            <view class="stat-num">¥ {{ statsData.month.current }}</view>
          </view>
        </view>

        <view class="card white-bg" @click="switchTab('wallet', 'month')">
          <view class="row-center justify-between mb-2">
             <text class="section-title" style="margin-bottom:0; border:none;">📊 消费分布 (点击详情)</text>
             <text class="text-gray-small">本月</text>
          </view>
          <view class="chart-container" style="justify-content: center;">
            <view class="pie-chart" :style="{ background: homePieGradient }"></view>
            <view class="legend-box" style="flex: 0 0 auto; margin-left: 20px;">
               <view v-for="(item, index) in homeExpensesByCategory.slice(0, 3)" :key="index" class="legend-item">
                 <view class="legend-dot" :style="{ background: item.color }"></view>
                 <text class="legend-text">{{ item.name }} {{ item.percent }}%</text>
               </view>
               <view v-if="homeExpensesByCategory.length > 3" class="legend-item text-gray-small">...</view>
               <view v-if="homeExpensesByCategory.length === 0" class="empty-tip" style="padding:0;">暂无数据</view>
            </view>
          </view>
        </view>

        <view class="card white-bg">
            <view class="row-center justify-between mb-2" @click="switchTab('tools')">
                <text class="section-title" style="margin-bottom:0; border:none;">📝 备忘录 (最新)</text>
                <text class="text-gray-small">查看全部 ></text>
            </view>
            <view v-for="m in memos.slice(0, 3)" :key="m.id" class="memo-card" 
                  @click="openEditModal(m, 'memo')" @longpress="confirmDelete(m.id, 'memo')">
                <text class="text-ellipsis">{{ m.content }}</text>
            </view>
            <view v-if="memos.length === 0" class="empty-tip" style="padding:10px;">暂无待办事项</view>
        </view>

      </view>

      <view v-if="currentTab === 'inventory'" class="page-animate">
        <view class="card white-bg p-2 mb-3 row-center">
          <text class="search-icon" style="margin-right: 10px;">🔍</text>
          <input class="border-input" style="flex: 1;" placeholder="搜索物品名称..." v-model="searchText" />
        </view>
        
        <view class="filter-tabs">
          <view v-for="f in ['all', 'near', 'expired']" :key="f" 
            :class="['filter-chip', filter === f ? 'active-chip' : '']"
            @click="filter = f">
            <text>{{ f === 'all' ? '全部物品' : f === 'near' ? '⚠️ 临期预警' : '❌ 已过期' }}</text>
          </view>
        </view>

        <view v-if="filter === 'expired' && filteredItems.length > 0" class="row-center justify-center mb-3 animate-fade-in">
            <view class="danger-text-btn" @click="confirmClearExpired">
                🗑️ 一键清空所有过期物品
            </view>
        </view>

        <view class="list-container">
          <view v-for="item in filteredItems" :key="item.id" 
                class="list-item" 
                @click="openEditModal(item, 'item')" 
                @longpress="confirmDelete(item.id, 'item')">
            <view v-if="item.imagePath" class="item-thumb">
                <image :src="item.imagePath" mode="aspectFill" class="thumb-img" @click.stop="openCustomPreview(item.imagePath)"></image>
            </view>
            <view class="item-info">
              <view class="item-name">{{ item.name }}</view>
              <view class="item-sub">
                 {{ item.category }} | 生产: {{ item.prodDate || '未知' }}
              </view>
            </view>
            <view class="right-col">
              <view :class="['status-badge', getStatusClass(item.expiryDate)]">
                {{ getDaysText(item.expiryDate) }}
              </view>
              <view class="del-btn" @click.stop="confirmDelete(item.id, 'item')">🗑️</view>
            </view>
          </view>
          <view v-if="filteredItems.length === 0" class="empty-tip">没有找到相关物品</view>
        </view>
      </view>

      <view v-if="currentTab === 'wallet'" class="page-animate">
        
        <view class="stats-grid mb-3">
            <view v-for="key in ['day', 'week', 'month', 'year']" :key="key" 
                  class="stat-card" 
                  :class="{ 'active-stat': walletStatsMode === key }"
                  @click="walletStatsMode = key">
                <text class="stat-label">{{ getStatsLabel(key) }}</text>
                <text class="stat-value">¥{{ statsData[key].current }}</text>
                <view class="trend-row">
                    <text :class="statsData[key].trend >= 0 ? 'trend-up' : 'trend-down'">
                        {{ statsData[key].trend >= 0 ? '↑' : '↓' }} {{ Math.abs(statsData[key].trend) }}%
                    </text>
                </view>
            </view>
        </view>

        <view class="card white-bg mb-3">
            <text class="section-title">📈 {{ getStatsLabel(walletStatsMode) }}消费趋势</text>
            <view class="chart-wrapper">
                <svg width="100%" height="100%" viewBox="0 0 300 100" class="svg-chart">
                    <line x1="0" y1="25" x2="300" y2="25" stroke="#eee" stroke-width="1"/>
                    <line x1="0" y1="50" x2="300" y2="50" stroke="#eee" stroke-width="1"/>
                    <line x1="0" y1="75" x2="300" y2="75" stroke="#eee" stroke-width="1"/>
                    <polyline :points="trendPolyline" fill="none" stroke="#0d9488" stroke-width="2" />
                    <circle v-for="(p, i) in trendPoints" :key="i" :cx="p.x" :cy="p.y" r="2" fill="#0d9488" />
                </svg>
                <view class="chart-labels">
                    <text v-for="(l, i) in trendLabels" :key="i">{{ l }}</text>
                </view>
            </view>
        </view>

        <view class="card white-bg">
          <text class="section-title">📊 {{ getStatsLabel(walletStatsMode) }}分类占比</text>
          <view class="chart-container">
            <view class="pie-chart" :style="{ background: pieGradient }"></view>
            <view class="legend-box">
               <view v-for="(item, index) in expensesByCategory" :key="index" class="legend-item">
                 <view class="legend-dot" :style="{ background: item.color }"></view>
                 <text class="legend-text">{{ item.name }} {{ item.percent }}% (¥{{item.value}})</text>
               </view>
               <view v-if="expensesByCategory.length === 0" class="empty-tip">暂无数据</view>
            </view>
          </view>
        </view>

        <text class="section-title mt-4">📝 快速记账</text>
        <view class="card white-bg p-3 mb-3">
            <view class="row-center mb-3">
                <view class="camera-btn mr-2" @click="chooseWalletPhoto">
                   <image v-if="walletForm.imagePath" :src="walletForm.imagePath" mode="aspectFill" class="mini-thumb"></image>
                   <text v-else class="camera-icon">📷</text>
                </view>

                <input class="border-input flex-1 mr-2" v-model="walletForm.name" placeholder="消费内容" />
                <input class="border-input flex-1" type="number" v-model="walletForm.amount" placeholder="金额" />
            </view>
            <view class="row-center mb-3">
                <picker class="flex-1 mr-2" mode="multiSelector" 
                        :range="pickerRange" 
                        :value="pickerIndex"
                        @columnchange="handleColumnChange" 
                        @change="handleWalletCategoryChange">
                    <view class="picker-view text-center">{{ walletForm.category || '选择分类' }}</view>
                </picker>
                <picker class="flex-1" mode="date" :value="walletForm.date" @change="(e) => walletForm.date = e.detail.value">
                    <view class="picker-view text-center">{{ walletForm.date || '选择日期' }}</view>
                </picker>
            </view>
            <button class="full-btn" @click="addExpense">记一笔</button>
        </view>

        <view class="list-container mt-2">
          <view v-for="e in filteredExpenses" :key="e.id" class="list-item"
                @click="openEditModal(e, 'expense')"
                @longpress="confirmDelete(e.id, 'expense')">
            <view class="row-center">
              <image v-if="e.imagePath" :src="e.imagePath" class="item-thumb" mode="aspectFill" @click.stop="openCustomPreview(e.imagePath)"></image>
              <view v-else class="category-circle">{{ e.category.charAt(0) }}</view>
              
              <view class="ml-2">
                <view class="item-name">{{ e.name }}</view>
                <view class="item-sub">{{ e.date }} | {{ e.category }}</view>
              </view>
            </view>
            <view class="right-align-box">
                <view class="money-text">- {{ e.amount }}</view>
                <view class="del-btn ml-2" @click.stop="confirmDelete(e.id, 'expense')">🗑️</view>
            </view>
          </view>
          <view v-if="filteredExpenses.length === 0" class="empty-tip">本时段无明细</view>
        </view>
      </view>

      <view v-if="currentTab === 'tools'" class="page-animate">
        <text class="section-title">📝 备忘录</text>
        <view class="card white-bg p-3 mb-3">
            <input class="border-input mb-3" v-model="memoText" placeholder="添加待办事项..." />
            <button class="full-btn" @click="addMemo">添加备忘录</button>
        </view>
        <view v-for="m in memos" :key="m.id" class="memo-card" 
              @click="openEditModal(m, 'memo')" @longpress="confirmDelete(m.id, 'memo')">
            <text>{{ m.content }}</text>
            <text class="del-text" @click.stop="confirmDelete(m.id, 'memo')">×</text>
        </view>
        <view class="divider mt-4 mb-4"></view>
        <text class="section-title">🎂 重要日子 / 倒计时</text>
        <view class="card white-bg p-3 mb-3">
             <view class="row-center mb-3">
                <input class="border-input flex-1 mr-2" v-model="cdData.title" placeholder="名称 (如: 生日)" />
                <picker class="flex-1" mode="selector" :range="['节日', '生日', '其他']" @change="(e) => cdData.type = ['holiday','birthday','other'][e.detail.value]">
                    <view class="picker-view text-center">{{ cdData.type === 'birthday' ? '生日' : cdData.type === 'holiday' ? '节日' : '类型' }}</view>
                </picker>
             </view>
             <view class="row-center mb-3">
                <picker mode="date" :value="cdData.date" @change="bindDateChange" style="flex:1">
                  <view class="picker-view">{{ cdData.date || '选择日期' }}</view>
                </picker>
             </view>
             <button class="full-btn" @click="addCountdown">添加倒计时</button>
        </view>
        <view v-for="c in countdowns" :key="c.id" class="list-item"
              @click="openEditModal(c, 'countdown')" @longpress="confirmDelete(c.id, 'countdown')">
             <view>
               <text>{{ c.type === 'birthday' ? '🎂' : '📅' }} {{ c.title }}</text>
               <text class="text-gray-small"> ({{ c.date }})</text>
             </view>
             <view class="right-align-box">
               <text class="days-badge">{{ getDaysLeft(c.date) }}天</text>
               <view class="del-btn ml-2" @click.stop="confirmDelete(c.id, 'countdown')">🗑️</view>
             </view>
        </view>
      </view>

      <view v-if="currentTab === 'manual_add'" class="page-animate">
         <view class="card white-bg p-3">
           <text class="section-title">✏️ {{ manualForm.name ? '完善信息' : '手动录入物品' }}</text>
           <view class="row-center mb-3 photo-row" @click="chooseManualPhoto">
                <view v-if="!manualForm.imagePath" class="row-center" style="color: #999;">
                    <text class="camera-icon-small mr-2">📷</text>
                    <text>点击拍摄或添加图片</text>
                </view>
                <image v-else :src="manualForm.imagePath" mode="aspectFill" class="picked-photo-small"></image>
           </view>
           <view class="row-center mb-3">
               <input class="border-input flex-1 mr-2" v-model="manualForm.name" placeholder="物品名称" />
               <picker class="flex-1" mode="multiSelector" :range="pickerRange" :value="pickerIndex" @columnchange="handleColumnChange" @change="handleManualCategoryChange">
                  <view class="picker-view text-center">{{ manualForm.category || '选择分类' }}</view>
               </picker>
           </view>
           <view class="row-center mb-3">
               <input class="border-input flex-1 mr-2" type="number" v-model="manualForm.price" placeholder="价格 (选填)" />
               <picker class="flex-1" mode="date" :value="manualForm.prodDate" @change="(e) => manualForm.prodDate = e.detail.value">
                  <view class="picker-view text-center">{{ manualForm.prodDate || '生产日期' }}</view>
               </picker>
           </view>
           <view class="row-center mb-3">
               <text class="label-text mr-2" style="width: auto;">保质期(天):</text>
               <input class="border-input flex-1" type="number" v-model="manualForm.shelfLife" placeholder="例如: 180" />
           </view>
           <view class="setting-row mb-4" style="border-top:1px solid #eee; border-bottom:1px solid #eee;">
               <text class="text-sm">🔄 同时记录到账本支出</text>
               <switch :checked="manualForm.syncToWallet" color="#0d9488" style="transform:scale(0.8)" @change="(e) => manualForm.syncToWallet = e.detail.value" />
           </view>
           <button class="full-btn" @click="saveManualItem">确认入库</button>
         </view>
      </view>

    </scroll-view>

    <view class="tab-bar">
      <view class="tab-item" @click="switchTab('home')">
        <text class="tab-icon">{{ currentTab === 'home' ? '🏠' : '🏚️' }}</text>
        <text class="tab-text">首页</text>
      </view>
      <view class="tab-item" @click="switchTab('inventory')">
        <text class="tab-icon">{{ currentTab === 'inventory' ? '📦' : '🥡' }}</text>
        <text class="tab-text">物品</text>
      </view>
      <view class="add-btn-wrapper">
        <view class="add-btn" @click.stop="goDirectManual">
          <text class="plus-icon">＋</text>
        </view>
      </view>
      <view class="tab-item" @click="switchTab('wallet', 'month')">
        <text class="tab-icon">{{ currentTab === 'wallet' ? '💰' : '💸' }}</text>
        <text class="tab-text">记账</text>
      </view>
      <view class="tab-item" @click="switchTab('tools')">
        <text class="tab-icon">{{ currentTab === 'tools' ? '🛠️' : '🔧' }}</text>
        <text class="tab-text">工具</text>
      </view>
    </view>

    <view v-if="showEditModal" class="modal-mask" @click.self="showEditModal = false">
      <view class="dialog-card white-bg">
        <text class="dialog-title">编辑信息</text>
        
        <view v-if="editType === 'item'">
            <view class="row-center mb-3">
                <view class="fixed-thumb-box mr-3" @click="editingData.imagePath ? openCustomPreview(editingData.imagePath) : updateEditImage()">
                     <image v-if="editingData.imagePath" :src="editingData.imagePath" mode="aspectFill" class="fixed-thumb-img"></image>
                     <view v-else class="fixed-thumb-placeholder">📷</view>
                </view>
                <view>
                     <text v-if="editingData.imagePath" class="text-teal" @click="updateEditImage" style="font-size:14px; text-decoration: underline;">点击更换图片</text>
                     <text v-else class="text-gray-small" @click="updateEditImage">点击添加图片</text>
                </view>
            </view>
            
            <input class="border-input mb-2" v-model="editingData.name" placeholder="名称" />
            <picker class="mb-2" mode="multiSelector" :range="pickerRange" :value="pickerIndex" @columnchange="handleColumnChange" @change="(e) => handleEditCategoryChange(e)">
                 <view class="picker-view">分类: {{ editingData.category }}</view>
            </picker>
            <picker mode="date" :value="editingData.prodDate" @change="(e) => editingData.prodDate = e.detail.value">
                <view class="picker-view mb-2">生产日期: {{ editingData.prodDate }}</view>
            </picker>
        </view>

        <view v-if="editType === 'expense'">
             <view class="row-center mb-3">
                 <view class="fixed-thumb-box mr-3" @click="editingData.imagePath ? openCustomPreview(editingData.imagePath) : updateEditImage()">
                      <image v-if="editingData.imagePath" :src="editingData.imagePath" mode="aspectFill" class="fixed-thumb-img"></image>
                      <view v-else class="fixed-thumb-placeholder">📷</view>
                 </view>
                 <view>
                      <text v-if="editingData.imagePath" class="text-teal" @click="updateEditImage" style="font-size:14px; text-decoration: underline;">点击更换照片</text>
                      <text v-else class="text-gray-small" @click="updateEditImage">点击添加照片</text>
                 </view>
             </view>

             <input class="border-input mb-2" v-model="editingData.name" placeholder="消费名称" />
             <input class="border-input mb-2" type="number" v-model="editingData.amount" placeholder="金额" />
             <picker class="mb-2" mode="multiSelector" :range="pickerRange" :value="pickerIndex" @columnchange="handleColumnChange" @change="(e) => handleEditCategoryChange(e)">
                 <view class="picker-view">分类: {{ editingData.category }}</view>
            </picker>
             <picker mode="date" :value="editingData.date" @change="(e) => editingData.date = e.detail.value">
                <view class="picker-view mb-2">日期: {{ editingData.date }}</view>
             </picker>
        </view>

        <view v-if="editType === 'memo'">
             <textarea class="border-input" style="height: 80px; padding: 5px;" v-model="editingData.content" maxlength="-1"></textarea>
        </view>
        <view v-if="editType === 'countdown'">
             <input class="border-input mb-2" v-model="editingData.title" placeholder="标题" />
             <picker mode="date" :value="editingData.date" @change="(e) => editingData.date = e.detail.value">
                <view class="picker-view mb-2">日期: {{ editingData.date }}</view>
             </picker>
             <picker mode="selector" :range="['节日', '生日', '其他']" @change="(e) => editingData.type = ['holiday','birthday','other'][e.detail.value]">
                <view class="picker-view mb-2">类型: {{ editingData.type === 'birthday' ? '生日' : '节日/其他' }}</view>
             </picker>
        </view>

        <view class="dialog-actions">
           <button class="small-btn bg-gray" @click="showEditModal = false">取消</button>
           <button class="small-btn bg-teal" @click="saveEdit">保存修改</button>
        </view>
      </view>
    </view>

    <view v-if="showImagePreview" class="preview-overlay" @click="closeCustomPreview">
        <image :src="previewImageUrl" mode="widthFix" class="preview-img-large" @click.stop></image>
    </view>

    <view v-if="showDeleteDialog" class="modal-mask" @click.self="showDeleteDialog = false">
      <view class="dialog-card white-bg">
        <text class="dialog-title">⚠️ 确认删除</text>
        <text class="dialog-desc">你确定要删除这条记录吗？</text>
        <view class="dialog-actions">
          <button class="small-btn bg-gray" @click="showDeleteDialog = false">取消</button>
          <button class="small-btn bg-red" @click="executeDelete">确认删除</button>
        </view>
      </view>
    </view>

    <view v-if="showClearExpiredDialog" class="modal-mask" @click.self="showClearExpiredDialog = false">
      <view class="dialog-card white-bg">
        <text class="dialog-title">⚠️ 清空过期物品</text>
        <text class="dialog-desc">确定要删除所有【已过期】的物品吗？</text>
        <view class="dialog-actions">
          <button class="small-btn bg-gray" @click="showClearExpiredDialog = false">取消</button>
          <button class="small-btn bg-red" @click="executeClearExpired">确认清空</button>
        </view>
      </view>
    </view>

    <view v-if="notification" class="toast">
      {{ notification }}
    </view>

  </view>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const CATEGORY_TREE = [
    { name: '餐饮美食', children: ['正餐（堂食/外卖）', '零食小吃', '饮品（奶茶/酒水）', '生鲜食材', '粮油调味'] },
    { name: '美妆个护', children: ['美妆彩妆', '护肤保养', '个护清洁', '香水香氛'] },
    { name: '健康医疗', children: ['药品', '保健品', '医疗服务', '医疗器械'] },
    { name: '家居日用', children: ['清洁用品', '厨卫用品', '家纺布艺', '家居装饰'] },
    { name: '数码家电', children: ['大家电', '小家电', '数码产品', '数码配件'] },
    { name: '服饰鞋包', children: ['服装', '鞋靴', '箱包配饰', '内衣袜子'] },
    { name: '出行交通', children: ['公共交通', '私家车', '共享出行', '机票/火车票'] },
    { name: '居住缴费', children: ['房租/房贷', '水电费', '燃气费', '物业费', '网费/话费'] },
    { name: '休闲娱乐', children: ['电影/演出', '旅游度假', '运动健身', '游戏/会员'] },
    { name: '学习教育', children: ['书籍教材', '课程培训', '文具用品', '学费'] },
    { name: '人情社交', children: ['红包礼金', '请客聚餐', '礼物礼品'] },
    { name: '宠物开销', children: ['宠物食品', '宠物用品', '宠物医疗'] },
    { name: '其他杂项', children: ['其他'] }
];
const COLORS = ['#0088FE', '#00C49F', '#FFBB28', '#FF8042', '#8884d8', '#82ca9d', '#ffc658', '#FF6B6B', '#4ECDC4', '#556270'];

// --- 状态 ---
const currentTab = ref('home');
const notification = ref('');
const items = ref([]);
const expenses = ref([]);
const memos = ref([]);
const countdowns = ref([]);
const settings = ref({ isDarkMode: false });

const manualForm = ref({ name: '', category: '食品饮料', price: '', prodDate: '', shelfLife: '', imagePath: '', syncToWallet: false });
const walletForm = ref({ name: '', amount: '', category: '其他', date: '', imagePath: '' });

const searchText = ref('');
const filter = ref('all');
const memoText = ref('');
const cdData = ref({ title: '', date: '', type: 'holiday' });
const walletStatsMode = ref('month');

const pickerRange = ref([[], []]); 
const pickerIndex = ref([0, 0]);

const showDeleteDialog = ref(false); 
const deleteTarget = ref(null); 
const showEditModal = ref(false); 
const editingData = ref({});
const editType = ref('');
const showClearExpiredDialog = ref(false); 

// 自定义预览状态
const showImagePreview = ref(false);
const previewImageUrl = ref('');

onMounted(() => {
  try {
    items.value = uni.getStorageSync('sl_items') || [];
    expenses.value = uni.getStorageSync('sl_expenses') || [];
    memos.value = uni.getStorageSync('sl_memos') || [];
    countdowns.value = uni.getStorageSync('sl_countdowns') || [];
    settings.value = uni.getStorageSync('sl_settings') || { isDarkMode: false };
    initPickerData();
    if (items.value.length === 0) {
      const today = getTodayStr();
      items.value = [{ id: 1, name: '演示物品', category: '零食小吃', prodDate: today, expiryDate: addDays(today, 30), addDate: today, imagePath: '' }];
    }
  } catch (e) { console.error(e); }
});

watch(items, (n) => uni.setStorageSync('sl_items', n), { deep: true });
watch(expenses, (n) => uni.setStorageSync('sl_expenses', n), { deep: true });
watch(memos, (n) => uni.setStorageSync('sl_memos', n), { deep: true });
watch(countdowns, (n) => uni.setStorageSync('sl_countdowns', n), { deep: true });
watch(settings, (n) => uni.setStorageSync('sl_settings', n), { deep: true });

// --- 逻辑 ---
const initPickerData = () => {
    const l1 = CATEGORY_TREE.map(c => c.name);
    const l2 = CATEGORY_TREE[0].children;
    pickerRange.value = [l1, l2];
    pickerIndex.value = [0, 0];
};

const handleColumnChange = (e) => {
    if (e.detail.column === 0) {
        pickerIndex.value[0] = e.detail.value;
        pickerIndex.value[1] = 0; 
        pickerRange.value[1] = CATEGORY_TREE[e.detail.value].children;
    } else {
        pickerIndex.value[1] = e.detail.value;
    }
};

const getSelectedCategory = () => CATEGORY_TREE[pickerIndex.value[0]].children[pickerIndex.value[1]] || '其他';
const handleManualCategoryChange = () => { manualForm.value.category = getSelectedCategory(); };
const handleWalletCategoryChange = () => { walletForm.value.category = getSelectedCategory(); };
const handleEditCategoryChange = () => { editingData.value.category = getSelectedCategory(); };

const getTodayStr = () => new Date().toISOString().split('T')[0];
const goDirectManual = () => {
    manualForm.value = { name: '', category: '零食小吃', price: '', prodDate: getTodayStr(), shelfLife: '', imagePath: '', syncToWallet: false };
    currentTab.value = 'manual_add';
};

const toggleDarkMode = () => settings.value.isDarkMode = !settings.value.isDarkMode;

const switchTab = (tab, mode = null) => {
    currentTab.value = tab;
    if (tab === 'wallet' && mode) walletStatsMode.value = mode;
};

const goToExpiring = () => { currentTab.value = 'inventory'; filter.value = 'near'; };

const chooseManualPhoto = () => {
    uni.chooseImage({ count: 1, sizeType: ['compressed'], success: (res) => {
        uni.saveFile({ tempFilePath: res.tempFilePaths[0], success: (s) => manualForm.value.imagePath = s.savedFilePath });
    }});
};
const chooseWalletPhoto = () => {
    uni.chooseImage({ count: 1, sizeType: ['compressed'], success: (res) => {
        uni.saveFile({ tempFilePath: res.tempFilePaths[0], success: (s) => walletForm.value.imagePath = s.savedFilePath });
    }});
};

const openCustomPreview = (path) => {
    if(!path) return;
    previewImageUrl.value = path;
    showImagePreview.value = true;
};
const closeCustomPreview = () => {
    showImagePreview.value = false;
};

const saveManualItem = () => {
  if (!manualForm.value.name || !manualForm.value.shelfLife) return showToast("名称和保质期必填");
  const expiryDate = addDays(manualForm.value.prodDate || getTodayStr(), manualForm.value.shelfLife);
  items.value.unshift({ 
      id: Date.now(), name: manualForm.value.name, category: manualForm.value.category, 
      prodDate: manualForm.value.prodDate, expiryDate: expiryDate, addDate: getTodayStr(),
      imagePath: manualForm.value.imagePath
  });
  if (manualForm.value.syncToWallet && manualForm.value.price) {
    expenses.value.unshift({ 
        id: Date.now() + 1, name: manualForm.value.name, category: manualForm.value.category, 
        amount: manualForm.value.price, date: getTodayStr(), imagePath: manualForm.value.imagePath
    });
    showToast("物品已入库，且已记账");
  } else {
    showToast("物品入库成功");
  }
  manualForm.value = { name: '', category: '零食小吃', price: '', prodDate: getTodayStr(), shelfLife: '', imagePath: '', syncToWallet: false };
  currentTab.value = 'inventory';
};

const addExpense = () => {
    if(!walletForm.value.name || !walletForm.value.amount) return showToast("请填写内容和金额");
    expenses.value.unshift({
        id: Date.now(), name: walletForm.value.name, amount: walletForm.value.amount,
        category: walletForm.value.category, date: walletForm.value.date || getTodayStr(),
        imagePath: walletForm.value.imagePath 
    });
    showToast("记账成功");
    walletForm.value = { name: '', amount: '', category: '其他', date: getTodayStr(), imagePath: '' };
};

const openEditModal = (item, type) => {
    editType.value = type;
    editingData.value = JSON.parse(JSON.stringify(item));
    showEditModal.value = true;
};
const saveEdit = () => {
    const target = editType.value === 'item' ? items : editType.value === 'expense' ? expenses : editType.value === 'memo' ? memos : countdowns;
    const idx = target.value.findIndex(i => i.id === editingData.value.id);
    if(idx !== -1) target.value[idx] = editingData.value;
    showEditModal.value = false;
    showToast("修改已保存");
};
const updateEditImage = () => {
    uni.chooseImage({ count: 1, success: (res) => {
        uni.saveFile({ tempFilePath: res.tempFilePaths[0], success: (s) => editingData.value.imagePath = s.savedFilePath });
    }});
};

const confirmDelete = (id, type) => { deleteTarget.value = { id, type }; showDeleteDialog.value = true; };
const executeDelete = () => {
    const { id, type } = deleteTarget.value;
    if(type === 'item') items.value = items.value.filter(i => i.id !== id);
    else if(type === 'expense') expenses.value = expenses.value.filter(e => e.id !== id);
    else if(type === 'memo') memos.value = memos.value.filter(m => m.id !== id);
    else if(type === 'countdown') countdowns.value = countdowns.value.filter(c => c.id !== id);
    showDeleteDialog.value = false;
    showToast("已删除");
};

const confirmClearExpired = () => showClearExpiredDialog.value = true;
const executeClearExpired = () => {
    items.value = items.value.filter(i => (new Date(i.expiryDate) - new Date()) / 86400000 >= 0);
    showClearExpiredDialog.value = false;
    showToast("过期物品已清空");
};

const addMemo = () => { if(memoText.value) { memos.value.unshift({ id: Date.now(), content: memoText.value }); memoText.value = ''; }};
const addCountdown = () => { 
    if(cdData.value.title && cdData.value.date) { 
        countdowns.value.push({ id: Date.now(), ...cdData.value }); cdData.value = { title: '', date: '', type: 'holiday' }; showToast("添加成功");
    }
};
const bindDateChange = (e) => cdData.value.date = e.detail.value;

const addDays = (d, days) => { const res = new Date(d); res.setDate(res.getDate() + parseInt(days)); return res.toISOString().split('T')[0]; };
const getDaysLeft = (d) => Math.ceil((new Date(d) - new Date()) / 86400000);
const getStatusClass = (d) => { const diff = (new Date(d) - new Date()) / 86400000; return diff < 0 ? 'tag-red' : diff < 7 ? 'tag-orange' : 'tag-green'; };
const getDaysText = (d) => { const diff = Math.ceil((new Date(d) - new Date()) / 86400000); return diff < 0 ? '已过期' : `${diff}天`; };
const showToast = (msg) => { notification.value = msg; setTimeout(() => notification.value = '', 3000); };

// Computed
const filteredItems = computed(() => {
    return items.value.filter(i => {
        const days = (new Date(i.expiryDate) - new Date()) / 86400000;
        if (filter.value === 'near' && (days < 0 || days > 30)) return false; 
        if (filter.value === 'expired' && days >= 0) return false;
        return i.name.includes(searchText.value);
    });
});
const expiringSoonCount = computed(() => items.value.filter(i => { const diff = (new Date(i.expiryDate) - new Date()) / 86400000; return diff >= -1 && diff <= 30; }).length);
const nearestCountdown = computed(() => {
    const list = countdowns.value.map(c => ({ ...c, days: Math.ceil((new Date(c.date) - new Date()) / 86400000) })).filter(c => c.days >= 0).sort((a,b) => a.days - b.days);
    return list[0] || null;
});

const now = new Date();
const getDateInfo = (dateStr) => {
    const d = new Date(dateStr);
    const firstDay = new Date(d.getFullYear(), 0, 1);
    const w = Math.ceil(((d - firstDay) / 86400000 + firstDay.getDay() + 1) / 7);
    return { y: d.getFullYear(), m: d.getMonth(), day: d.getDate(), w, time: d.getTime() };
};
const calcTotal = (filterFn) => expenses.value.filter(filterFn).reduce((sum, item) => sum + parseFloat(item.amount), 0);

const statsData = computed(() => {
    const todayInfo = getDateInfo(getTodayStr());
    const dayTotal = calcTotal(e => e.date === getTodayStr());
    const weekTotal = calcTotal(e => { const i = getDateInfo(e.date); return i.y === todayInfo.y && i.w === todayInfo.w; });
    const monthTotal = calcTotal(e => { const i = getDateInfo(e.date); return i.y === todayInfo.y && i.m === todayInfo.m; });
    const yearTotal = calcTotal(e => { const i = getDateInfo(e.date); return i.y === todayInfo.y; });
    const calcTrend = (curr, prev) => prev === 0 ? 100 : Math.round(((curr - prev) / prev) * 100);
    return {
        day: { current: dayTotal.toFixed(0), trend: 0 },
        week: { current: weekTotal.toFixed(0), trend: 0 },
        month: { current: monthTotal.toFixed(0), trend: 0 },
        year: { current: yearTotal.toFixed(0), trend: 0 }
    };
});

const filteredExpenses = computed(() => {
    const info = getDateInfo(getTodayStr());
    return expenses.value.filter(e => {
        const i = getDateInfo(e.date);
        if (walletStatsMode.value === 'day') return e.date === getTodayStr();
        if (walletStatsMode.value === 'week') return i.y === info.y && i.w === info.w;
        if (walletStatsMode.value === 'month') return i.y === info.y && i.m === info.m;
        if (walletStatsMode.value === 'year') return i.y === info.y;
        return true;
    });
});

const expensesByCategory = computed(() => {
  const map = {}; let total = 0;
  filteredExpenses.value.forEach(e => {
      const val = parseFloat(e.amount);
      if(!map[e.category]) map[e.category] = 0;
      map[e.category] += val; total += val;
  });
  return Object.keys(map).map((k, i) => ({
      name: k, value: map[k], percent: total ? Math.round((map[k]/total)*100) : 0, color: COLORS[i % COLORS.length]
  })).sort((a,b) => b.value - a.value);
});
const pieGradient = computed(() => {
  if (expensesByCategory.value.length === 0) return '#eee';
  let str = 'conic-gradient('; let curr = 0;
  expensesByCategory.value.forEach(i => { const end = curr + i.percent; str += `${i.color} ${curr}% ${end}%, `; curr = end; });
  return str.slice(0, -2) + ')';
});

const homeExpensesByCategory = computed(() => {
  const info = getDateInfo(getTodayStr());
  const monthExpenses = expenses.value.filter(e => {
      const i = getDateInfo(e.date);
      return i.y === info.y && i.m === info.m;
  });
  const map = {}; let total = 0;
  monthExpenses.forEach(e => {
      const val = parseFloat(e.amount);
      if(!map[e.category]) map[e.category] = 0;
      map[e.category] += val; total += val;
  });
  return Object.keys(map).map((k, i) => ({
      name: k, value: map[k], percent: total ? Math.round((map[k]/total)*100) : 0, color: COLORS[i % COLORS.length]
  })).sort((a,b) => b.value - a.value);
});
const homePieGradient = computed(() => {
  if (homeExpensesByCategory.value.length === 0) return '#eee';
  let str = 'conic-gradient('; let curr = 0;
  homeExpensesByCategory.value.forEach(i => { const end = curr + i.percent; str += `${i.color} ${curr}% ${end}%, `; curr = end; });
  return str.slice(0, -2) + ')';
});

const trendChartData = computed(() => {
    const values = [20, 50, 30, 80, 40, 90, 60];
    const max = 100; const width = 300; const height = 100; const step = width / 6;
    let pointsStr = ''; const dots = []; const labels = ['1','2','3','4','5','6','7'];
    values.forEach((val, i) => {
        const x = i * step; const y = height - (val / max) * height;
        pointsStr += `${x},${y} `; dots.push({ x, y });
    });
    return { points: pointsStr, dots, labels };
});
const trendPolyline = computed(() => trendChartData.value.points);
const trendPoints = computed(() => trendChartData.value.dots);
const trendLabels = computed(() => trendChartData.value.labels);

const getStatsLabel = (key) => ({ day: '今日', week: '本周', month: '本月', year: '本年' }[key]);
const getPrevLabel = (key) => ({ day: '昨日', week: '上周', month: '上月', year: '去年' }[key]);

</script>

<style>
/* 样式部分 */
:root { --bg-color: #F8F9FA; --text-color: #333; --card-bg: white; --sub-text: #999; --header-bg: #0d9488; --header-text: #ffffff; --input-bg: white; --border-color: #eee; }
.dark-mode { --bg-color: #121212; --text-color: #e0e0e0; --card-bg: #1E1E1E; --sub-text: #aaa; --header-bg: #064E3B; --header-text: #e0e0e0; --input-bg: #2C2C2C; --border-color: #333; }
.container { background-color: var(--bg-color); color: var(--text-color); display: flex; flex-direction: column; height: 100vh; }
.card { background-color: var(--card-bg); color: var(--text-color); border-radius: 16px; padding: 20px; margin-bottom: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.03); }
.white-bg { background-color: var(--card-bg); }
.header { padding: 45px 20px 15px 20px; background: var(--header-bg); color: var(--header-text); display: flex; justify-content: space-between; align-items: center; box-shadow: 0 2px 10px rgba(0,0,0,0.2); }
.app-title { font-size: 22px; font-weight: bold; }
.header-icon { width: 36px; height: 36px; background: rgba(255,255,255,0.2); border-radius: 50%; display: flex; align-items: center; justify-content: center; }
.content-area { flex: 1; height: 0; }
.custom-scrollbar ::-webkit-scrollbar { width: 0; height: 0; }
.page-animate { padding: 15px; animation: fadeIn 0.3s; padding-bottom: 100px; }

.gradient-card { background: linear-gradient(135deg, #6366f1, #8b5cf6); color: white; display: flex; justify-content: space-between; align-items: center; }
.orange-bg { background: linear-gradient(135deg, #FFF7ED, #FFEDD5); color: #9A3412; }
.teal-bg-light { background: #F0FDFA; color: #0F766E; }
.grid-2 { display: flex; gap: 15px; margin-bottom: 15px; }
.grid-2 .card { flex: 1; margin-bottom: 0; text-align: center; padding: 15px 10px; }
.stat-num { font-size: 24px; font-weight: bold; margin-top: 5px; }
.stats-grid { display: flex; gap: 8px; justify-content: space-between; }
.stat-card { flex: 1; background: var(--card-bg); padding: 10px 5px; border-radius: 10px; text-align: center; border: 1px solid transparent; transition: all 0.2s; box-shadow: 0 2px 5px rgba(0,0,0,0.02); }
.active-stat { border-color: #0d9488; background: #F0FDFA; transform: translateY(-2px); }
.stat-label { font-size: 11px; color: var(--sub-text); display: block; margin-bottom: 4px; }
.stat-value { font-size: 14px; font-weight: bold; color: var(--text-color); display: block; margin-bottom: 4px; }
.trend-row { font-size: 10px; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.trend-up { color: #EF4444; font-weight: bold; } .trend-down { color: #10B981; font-weight: bold; }
.chart-wrapper { width: 100%; height: 140px; position: relative; padding: 10px 0; }
.svg-chart { overflow: visible; }
.chart-labels { display: flex; justify-content: space-between; font-size: 10px; color: var(--sub-text); margin-top: 5px; }
.list-item { background: var(--card-bg); padding: 12px; border-radius: 12px; margin-bottom: 12px; display: flex; align-items: center; justify-content: space-between; }
.category-circle { width: 40px; height: 40px; background: #f0fdfa; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 16px; color: #0d9488; font-weight: bold; flex-shrink: 0; }
.item-thumb { width: 50px; height: 50px; border-radius: 8px; overflow: hidden; margin-right: 12px; background: #eee; flex-shrink: 0; }
.thumb-img { width: 100%; height: 100%; }
.item-info { flex: 1; }
.item-name { font-weight: bold; font-size: 15px; color: var(--text-color); }
.item-sub { color: var(--sub-text); font-size: 12px; margin-top: 2px; }
.right-align-box { text-align: right; display: flex; align-items: center; justify-content: flex-end; }
.money-text { font-weight: bold; font-size: 16px; color: var(--text-color); }
.status-badge { padding: 4px 8px; border-radius: 6px; font-size: 10px; }
.tag-green { background: #DCFCE7; color: #166534; } .tag-orange { background: #FFEDD5; color: #9A3412; } .tag-red { background: #FEE2E2; color: #991B1B; }
.chart-container { display: flex; align-items: center; margin-top: 15px; gap: 15px; }
.pie-chart { width: 90px; height: 90px; border-radius: 50%; flex-shrink: 0; }
.legend-box { flex: 1; display: flex; flex-direction: column; gap: 6px; }
.legend-item { display: flex; align-items: center; font-size: 11px; color: var(--sub-text); }
.legend-dot { width: 8px; height: 8px; border-radius: 50%; margin-right: 6px; flex-shrink: 0; }
.section-title { font-size: 16px; font-weight: bold; margin-bottom: 12px; display: block; border-left: 4px solid #0d9488; padding-left: 8px; }
.tab-bar { background-color: var(--card-bg); border-top: 1px solid var(--border-color); display: flex; justify-content: space-between; align-items: center; height: 60px; position: fixed; bottom: 0; width: 100%; z-index: 900; padding-bottom: constant(safe-area-inset-bottom); padding-bottom: env(safe-area-inset-bottom); }
.tab-item { flex: 1; text-align: center; display: flex; flex-direction: column; justify-content: center; height: 100%; }
.tab-icon { font-size: 20px; margin-bottom: 2px; } .tab-text { font-size: 10px; color: var(--sub-text); }
.add-btn-wrapper { flex: 1; display: flex; justify-content: center; align-items: flex-end; height: 100%; position: relative; }
.add-btn { width: 52px; height: 52px; background: linear-gradient(135deg, #0d9488, #115e59); border-radius: 50%; position: absolute; top: -20px; left: 50%; transform: translateX(-50%); display: flex; align-items: center; justify-content: center; box-shadow: 0 4px 10px rgba(13, 148, 136, 0.4); border: 4px solid var(--card-bg); }
.plus-icon { color: white; font-size: 26px; }
.filter-tabs { display: flex; gap: 8px; margin-bottom: 15px; overflow-x: auto; }
.filter-chip { padding: 5px 14px; background: var(--card-bg); border-radius: 20px; font-size: 12px; color: var(--sub-text); border: 1px solid var(--border-color); white-space: nowrap; }
.active-chip { background: #0d9488; color: white; border-color: #0d9488; }
.border-input { border-bottom: 1px solid var(--border-color); height: 40px; color: var(--text-color); font-size: 14px; }
.picker-view { border-bottom: 1px solid var(--border-color); height: 40px; line-height: 40px; color: var(--text-color); font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.full-btn { background: #0d9488; color: white; font-size: 15px; margin-top: 15px; border-radius: 8px; line-height: 44px; }
.outline-btn { background: transparent; border: 1px solid #0d9488; color: #0d9488; font-size: 14px; border-radius: 8px; margin-bottom: 10px; width: 100%; padding: 8px 0; }
.danger-btn { background: #FEE2E2; color: #EF4444; margin-top: 20px; font-size: 14px; border-radius: 8px; }
.danger-text-btn { color: #EF4444; font-size: 13px; background: #FEE2E2; padding: 8px 16px; border-radius: 20px; font-weight: bold; }
.photo-row { height: 90px; background: var(--bg-color); border: 1px dashed var(--border-color); border-radius: 8px; justify-content: center; }
.picked-photo-small { width: 80px; height: 80px; border-radius: 6px; }
.camera-btn { width: 40px; height: 40px; background: #f3f4f6; border-radius: 8px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.camera-icon { font-size: 20px; }
.mini-thumb { width: 100%; height: 100%; border-radius: 8px; }
.modal-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.6); z-index: 999; backdrop-filter: blur(2px); display: flex; align-items: center; justify-content: center; }
.dialog-card { width: 75%; padding: 20px; border-radius: 16px; animation: modalScale 0.2s ease-out; }
.dialog-title { font-size: 17px; font-weight: bold; margin-bottom: 10px; display: block; }
.dialog-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.small-btn { font-size: 13px; padding: 0 15px; height: 32px; line-height: 32px; border-radius: 6px; }
.bg-gray { background: #e5e7eb; color: #333; } .bg-teal { background: #0d9488; color: white; } .bg-red { background: #ef4444; color: white; }
.toast { position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.8); color: white; padding: 10px 20px; border-radius: 8px; font-size: 14px; z-index: 3000; }
.empty-tip { text-align: center; color: var(--sub-text); font-size: 12px; padding: 20px; }
.memo-card { background: #FEF9C3; padding: 10px 12px; border-radius: 8px; margin-bottom: 10px; font-size: 14px; color: #4B5563; display: flex; justify-content: space-between; border-left: 4px solid #FCD34D; }
.setting-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; color: var(--text-color); }
.flex-1 { flex: 1; } .mr-2 { margin-right: 10px; } .ml-2 { margin-left: 10px; } .mb-2 { margin-bottom: 8px; } .mb-3 { margin-bottom: 12px; } .mt-4 { margin-top: 16px; } .text-center { text-align: center; } .row-center { display: flex; align-items: center; } .justify-center { justify-content: center; } .justify-between { justify-content: space-between; } .text-gray-small { color: #999; font-size: 12px; }
.text-teal { color: #0d9488; }
.text-ellipsis { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 90%; }
/* [新增] 固定图片区域样式 */
.fixed-thumb-box { width: 80px; height: 80px; border-radius: 8px; overflow: hidden; background: #eee; display: flex; align-items: center; justify-content: center; border: 1px solid #ddd; flex-shrink: 0; }
.fixed-thumb-img { width: 100%; height: 100%; }
.fixed-thumb-placeholder { font-size: 30px; color: #999; }
/* [新增] 全屏图片预览遮罩 */
.preview-overlay { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.9); z-index: 9999; display: flex; align-items: center; justify-content: center; animation: fadeIn 0.2s; }
.preview-img-large { width: 100%; max-height: 90vh; }

@keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }
@keyframes modalScale { from { opacity: 0; transform: scale(0.95); } to { opacity: 1; transform: scale(1); } }
</style>