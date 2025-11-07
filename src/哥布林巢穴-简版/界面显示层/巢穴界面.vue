<template>
  <div class="nest-container">
    <!-- 巢穴头部信息 -->
    <div class="nest-header">
      <h3 class="nest-title">🏰 衍生圣巢</h3>
      <div class="income-summary">
        <div v-if="totalIncome.gold > 0" class="income-item">
          <span class="income-icon">💰</span>
          <span class="income-amount">+{{ totalIncome.gold }}</span>
        </div>
        <div v-if="totalIncome.food > 0" class="income-item">
          <span class="income-icon">🍖</span>
          <span class="income-amount">+{{ totalIncome.food }}</span>
        </div>
        <div class="income-label">每回合</div>
      </div>
    </div>

    <!-- 建筑分类标签 -->
    <div class="building-tabs">
      <button class="tab-button" :class="{ active: activeTab === 'breeding' }" @click="activeTab = 'breeding'">
        👶 产卵室
      </button>
      <button class="tab-button" :class="{ active: activeTab === 'resource' }" @click="activeTab = 'resource'">
        🏗️ 资源建筑
      </button>
    </div>

    <!-- 建筑内容区域 -->
    <div class="building-content">
      <!-- 产卵室建筑槽位 -->
      <div v-if="activeTab === 'breeding'" class="building-section">
        <div class="building-scroll-container">
          <div class="building-grid">
            <div
              v-for="(slot, index) in breedingSlots"
              :key="`breeding-${index}`"
              class="building-slot"
              :class="getSlotClasses(slot, index, 'breeding')"
              @click="handleSlotClick(index, 'breeding')"
            >
              <!-- 已建设建筑 -->
              <div v-if="slot.building" class="building">
                <div class="building-icon">{{ slot.building.icon }}</div>
                <div class="building-name">{{ slot.building.name }}</div>

                <!-- 显示交配间占用情况 -->
                <div v-if="slot.building.id === 'breeding'" class="breeding-status">
                  <div v-if="getBreedingRoomOccupant(index)" class="occupied-status">
                    <span class="occupant-name">{{ getBreedingRoomOccupant(index)?.name }}</span>
                    <span class="occupant-status">{{
                      getBreedingRoomOccupant(index)?.status === 'breeding' ? '交配中' : '待命'
                    }}</span>
                  </div>
                  <div v-else class="available-status">
                    <span class="available-text">空闲</span>
                  </div>
                </div>

                <button class="remove-button" title="拆除建筑" @click.stop="removeBuilding(index, 'breeding')">
                  ×
                </button>
              </div>

              <!-- 空槽位 -->
              <div v-else-if="slot.unlocked" class="empty-slot">
                <div class="empty-icon">🏗️</div>
                <div class="empty-text">空槽位</div>
              </div>

              <!-- 可开通槽位 -->
              <div v-else-if="isNextUnlockSlot(index, 'breeding')" class="next-unlock-slot">
                <div class="expand-icon">+</div>
                <div class="expand-text">开通槽位</div>
                <div class="expand-cost">{{ getSlotCost(index).gold }}💰 {{ getSlotCost(index).food }}🍖</div>
              </div>

              <!-- 锁定槽位 -->
              <div v-else class="locked-slot">
                <div class="locked-icon">🔒</div>
                <div class="locked-text">锁定</div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 资源建筑槽位 -->
      <div v-if="activeTab === 'resource'" class="building-section">
        <div class="building-scroll-container">
          <div class="building-grid">
            <div
              v-for="(slot, index) in resourceSlots"
              :key="`resource-${index}`"
              class="building-slot"
              :class="getSlotClasses(slot, index, 'resource')"
              @click="handleSlotClick(index, 'resource')"
            >
              <!-- 已建设建筑 -->
              <div v-if="slot.building" class="building">
                <div class="building-icon">{{ slot.building.icon }}</div>
                <div class="building-name">{{ slot.building.name }}</div>

                <!-- 建筑收入显示 -->
                <div v-if="slot.building.income" class="building-income">
                  <div v-if="slot.building.income.gold" class="income-display">
                    <span class="income-icon">💰</span>
                    <span class="income-text">+{{ slot.building.income.gold }}</span>
                  </div>
                  <div v-if="slot.building.income.food" class="income-display">
                    <span class="income-icon">🍖</span>
                    <span class="income-text">+{{ slot.building.income.food }}</span>
                  </div>
                </div>

                <!-- 献祭祭坛特殊交互 -->
                <div v-if="slot.building.id === 'sacrifice_altar'" class="sacrifice-button-container">
                  <button class="sacrifice-button" @click.stop="openSacrificeDialog(index)">献祭</button>
                </div>

                <button class="remove-button" title="拆除建筑" @click.stop="removeBuilding(index, 'resource')">
                  ×
                </button>
              </div>

              <!-- 空槽位 -->
              <div v-else-if="slot.unlocked" class="empty-slot">
                <div class="empty-icon">🏗️</div>
                <div class="empty-text">空槽位</div>
              </div>

              <!-- 可开通槽位 -->
              <div v-else-if="isNextUnlockSlot(index, 'resource')" class="next-unlock-slot">
                <div class="expand-icon">+</div>
                <div class="expand-text">开通槽位</div>
                <div class="expand-cost">{{ getSlotCost(index).gold }}💰 {{ getSlotCost(index).food }}🍖</div>
              </div>

              <!-- 锁定槽位 -->
              <div v-else class="locked-slot">
                <div class="locked-icon">🔒</div>
                <div class="locked-text">锁定</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 建筑选择菜单 -->
    <div v-if="showMenu" class="building-menu">
      <div class="menu-header">
        <h4>选择建筑</h4>
        <button class="close-menu" title="关闭菜单" @click="closeMenu">×</button>
      </div>
      <div class="building-options">
        <div
          v-for="building in availableBuildings"
          :key="building.id"
          class="building-option"
          :class="{ disabled: !canBuild(building) }"
          @click="selectBuilding(building)"
        >
          <div class="option-icon">{{ building.icon }}</div>
          <div class="option-texts">
            <div class="option-name">{{ building.name }}</div>
            <div class="option-desc">{{ building.description }}</div>
          </div>
          <div class="option-cost">{{ building.cost.gold }}💰 {{ building.cost.food }}🍖</div>
        </div>
      </div>
    </div>

    <!-- 献祭对话框 -->
    <SacrificeDialog :show="showSacrificeDialog" @close="closeSacrificeDialog" @confirm="handleSacrificeConfirm" />
  </div>
</template>

<script setup lang="ts">
import { computed, onActivated, onMounted, ref, watch } from 'vue';
import SacrificeDialog from '../共享资源层/组件/献祭对话框.vue';
import { modularSaveManager } from '../核心层/服务/存档系统/模块化存档服务';
import type { NestModuleData } from '../核心层/服务/存档系统/模块化存档类型';
import { SacrificeService, type SacrificeAmounts } from '../核心层/服务/通用服务/献祭服务';
import { PlayerLevelService } from '../核心层/服务/通用服务/玩家等级服务';
import { ConfirmService } from '../核心层/服务/通用服务/确认框服务';

// ==================== 类型定义 ====================

/**
 * 建筑效果接口
 */
interface BuildingEffect {
  type: string;
  icon: string;
  description: string;
}

/**
 * 建筑成本接口
 */
interface BuildingCost {
  gold: number;
  food: number;
}

/**
 * 建筑收入接口
 */
interface BuildingIncome {
  gold?: number;
  food?: number;
}

/**
 * 建筑接口定义
 */
interface Building {
  id: string;
  name: string;
  icon: string;
  description: string;
  cost: BuildingCost;
  category: 'breeding' | 'resource';
  income?: BuildingIncome; // 每回合收入
  effects: BuildingEffect[];
}

/**
 * 建筑槽位接口定义
 */
interface BuildingSlot {
  building: Building | null;
  unlocked: boolean;
}

/**
 * 槽位类型
 */
type SlotType = 'breeding' | 'resource';

/**
 * 槽位成本接口
 */
interface SlotCost {
  gold: number;
  food: number;
}

// ==================== 资源管理 ====================

// 直接使用 modularSaveManager 获取错误提示功能
const getInsufficientResourcesMessage = modularSaveManager.getInsufficientResourcesMessage.bind(modularSaveManager);

// ==================== 建筑和槽位资源管理 ====================

/**
 * 检查是否能负担建筑成本
 */
const canAffordBuilding = (cost: { gold: number; food: number }): boolean => {
  return modularSaveManager.hasEnoughResources([
    { type: 'gold', amount: cost.gold, reason: '建筑成本' },
    { type: 'food', amount: cost.food, reason: '建筑成本' },
  ]);
};

/**
 * 支付建筑成本
 */
const payForBuilding = (cost: { gold: number; food: number }, buildingName: string): boolean => {
  return modularSaveManager.consumeResources([
    { type: 'gold', amount: cost.gold, reason: `建设${buildingName}` },
    { type: 'food', amount: cost.food, reason: `建设${buildingName}` },
  ]);
};

/**
 * 检查是否能负担槽位开通成本
 */
const canAffordSlotExpansion = (cost: { gold: number; food: number }): boolean => {
  return modularSaveManager.hasEnoughResources([
    { type: 'gold', amount: cost.gold, reason: '槽位开通' },
    { type: 'food', amount: cost.food, reason: '槽位开通' },
  ]);
};

/**
 * 支付槽位开通成本
 */
const payForSlotExpansion = (cost: { gold: number; food: number }): boolean => {
  return modularSaveManager.consumeResources([
    { type: 'gold', amount: cost.gold, reason: '开通槽位' },
    { type: 'food', amount: cost.food, reason: '开通槽位' },
  ]);
};

// ==================== 响应式数据 ====================

// 界面状态
const activeTab = ref<SlotType>('breeding');
const showMenu = ref(false);
const selectedSlotIndex = ref(-1);
const selectedSlotType = ref<SlotType>('breeding');

// 建筑槽位数据
const breedingSlots = ref<BuildingSlot[]>([]);
const resourceSlots = ref<BuildingSlot[]>([]);

// 人物数据
const characters = ref<any[]>([]);

// ==================== 献祭相关数据 ====================

// 献祭对话框状态
const showSacrificeDialog = ref(false);
const currentSacrificeSlotIndex = ref(-1);

// ==================== 建筑数据定义 ====================

/**
 * 产卵室建筑列表
 */
const breedingBuildings: Building[] = [
  {
    id: 'breeding',
    name: '产卵室',
    icon: '👶',
    description: '用于俘虏孵蛋产卵衍生物',
    cost: { gold: 50, food: 30 },
    category: 'breeding',
    effects: [{ type: 'breeding', icon: '👶', description: '俘虏生育衍生物' }],
  },
];

/**
 * 资源建筑列表
 */
const resourceBuildings: Building[] = [
  {
    id: 'food',
    name: '食物间',
    icon: '🍖',
    description: '每回合+20食物',
    cost: { gold: 100, food: 50 },
    category: 'resource',
    income: { food: 20 },
    effects: [{ type: 'food', icon: '🍖', description: '每回合+20食物' }],
  },
  {
    id: 'trade',
    name: '贸易间',
    icon: '💰',
    description: '每回合+30金钱',
    cost: { gold: 150, food: 30 },
    category: 'resource',
    income: { gold: 30 },
    effects: [{ type: 'gold', icon: '💰', description: '每回合+30金钱' }],
  },
  {
    id: 'food_warehouse',
    name: '食物仓库',
    icon: '🏚️',
    description: '提高食物储存，食物总收入+10%',
    cost: { gold: 200, food: 120 },
    category: 'resource',
    effects: [{ type: 'food_multiplier', icon: '🍖', description: '食物收入+10%' }],
  },
  {
    id: 'gold_hall',
    name: '金币大厅',
    icon: '🏦',
    description: '改善金币储存，金币总收入+10%',
    cost: { gold: 260, food: 80 },
    category: 'resource',
    effects: [{ type: 'gold_multiplier', icon: '💰', description: '金钱收入+10%' }],
  },
  {
    id: 'sacrifice_altar',
    name: '献祭祭坛',
    icon: '🔥',
    description: '献祭衍生物升级人物等级',
    cost: { gold: 3000, food: 1500 },
    category: 'resource',
    effects: [{ type: 'sacrifice', icon: '🔥', description: '献祭衍生物升级等级' }],
  },
];

// ==================== 计算属性 ====================

/**
 * 当前可用建筑列表（根据选中的标签页）
 */
const availableBuildings = computed(() => {
  const buildings = activeTab.value === 'breeding' ? breedingBuildings : resourceBuildings;

  // 为产卵室计算动态成本
  if (activeTab.value === 'breeding') {
    return buildings.map(building => {
      if (building.id === 'breeding') {
        const existingBreedingCount = breedingSlots.value.filter(slot => slot.building?.id === 'breeding').length;
        return {
          ...building,
          cost: {
            gold: building.cost.gold + existingBreedingCount * 25,
            food: building.cost.food + existingBreedingCount * 15,
          },
        };
      }
      return building;
    });
  }

  // 资源建筑：过滤掉已存在的献祭祭坛（只允许建造1个）
  return buildings.filter(building => {
    if (building.id === 'sacrifice_altar') {
      // 检查是否已经有献祭祭坛
      const existingAltarCount = resourceSlots.value.filter(slot => slot.building?.id === 'sacrifice_altar').length;
      return existingAltarCount === 0; // 如果已经有1个或以上，则不显示
    }
    return true;
  });
});

/**
 * 计算所有建筑的总收入
 */
const totalIncome = computed(() => {
  let totalGold = 0;
  let totalFood = 0;

  // 计算产卵室建筑收入
  breedingSlots.value.forEach(slot => {
    if (slot.building && slot.building.income) {
      if (slot.building.income.gold) totalGold += slot.building.income.gold;
      if (slot.building.income.food) totalFood += slot.building.income.food;
    }
  });

  // 计算资源建筑收入
  resourceSlots.value.forEach(slot => {
    if (slot.building && slot.building.income) {
      if (slot.building.income.gold) totalGold += slot.building.income.gold;
      if (slot.building.income.food) totalFood += slot.building.income.food;
    }
  });

  // 应用加成：每座食物仓库使食物收入+10%，每座金币大厅使金钱收入+10%
  const foodWarehouseCount = resourceSlots.value.filter(s => s.building?.id === 'food_warehouse').length;
  const goldHallCount = resourceSlots.value.filter(s => s.building?.id === 'gold_hall').length;

  if (foodWarehouseCount > 0) {
    totalFood = Math.round(totalFood * Math.pow(1.1, foodWarehouseCount));
  }
  if (goldHallCount > 0) {
    totalGold = Math.round(totalGold * Math.pow(1.1, goldHallCount));
  }

  return { gold: totalGold, food: totalFood };
});

// ==================== 槽位管理 ====================

/**
 * 初始化建筑槽位
 */
const initializeSlots = () => {
  console.log('开始初始化槽位...');

  // 初始化产卵室槽位
  breedingSlots.value = [];
  // 前两个槽位默认开通，首槽位放置产卵室
  breedingSlots.value.push({
    building: breedingBuildings.find(b => b.id === 'breeding') || null,
    unlocked: true,
  });
  breedingSlots.value.push({
    building: null,
    unlocked: true,
  });

  // 初始化资源建筑槽位
  resourceSlots.value = [];
  // 第一个槽位默认开通并放置食物间
  resourceSlots.value.push({
    building: resourceBuildings.find(b => b.id === 'food') || null,
    unlocked: true,
  });
  // 第二个槽位默认开通并放置贸易间
  resourceSlots.value.push({
    building: resourceBuildings.find(b => b.id === 'trade') || null,
    unlocked: true,
  });
  // 添加一个可开通的槽位
  resourceSlots.value.push({
    building: null,
    unlocked: false,
  });

  console.log('槽位初始化完成:');
  console.log('产卵室槽位:', breedingSlots.value);
  console.log('资源建筑槽位:', resourceSlots.value);
};

/**
 * 添加新槽位
 */
const addNewSlot = (type: SlotType) => {
  if (type === 'breeding') {
    breedingSlots.value.push({
      building: null,
      unlocked: false,
    });
  } else {
    resourceSlots.value.push({
      building: null,
      unlocked: false,
    });
  }
};

/**
 * 获取槽位开通成本
 */
const getSlotCost = (index: number): SlotCost => {
  // 产卵室和资源建筑使用相同的槽位开通成本逻辑：前2个槽位免费，其后逐渐增加
  const baseGold = 200;
  const baseFood = 100;
  const multiplier = Math.max(0, index - 1); // 前2个槽位免费
  return {
    gold: baseGold + multiplier * 50,
    food: baseFood + multiplier * 20,
  };
};

// ==================== 槽位状态管理 ====================

/**
 * 处理槽位点击事件
 */
const handleSlotClick = (index: number, type: SlotType) => {
  const slots = type === 'breeding' ? breedingSlots.value : resourceSlots.value;
  const slot = slots[index];

  if (!slot.unlocked) {
    // 检查是否可以开通（按顺序开通）
    if (canUnlockSlot(index, type)) {
      const cost = getSlotCost(index);

      // 检查资源是否足够
      if (canAffordSlotExpansion(cost)) {
        // 消耗资源并开通槽位
        if (payForSlotExpansion(cost)) {
          slot.unlocked = true;
          // 开通槽位后，添加一个新的可开通槽位
          addNewSlot(type);
          // 立即保存，确保数据不丢失
          saveBuildingData();
          console.log('槽位开通成功，数据已保存');
        }
      } else {
        // 显示资源不足提示
        const message = getInsufficientResourcesMessage([
          { type: 'gold', amount: cost.gold, reason: '槽位开通' },
          { type: 'food', amount: cost.food, reason: '槽位开通' },
        ]);
        console.log(message);
        // 这里可以显示toast提示
      }
    }
    // 如果不能开通，不显示任何提示，保持界面简洁
  } else if (!slot.building) {
    // 选择建筑
    showBuildingMenu(index, type);
  }
};

/**
 * 检查是否可以开通槽位（按顺序开通）
 */
const canUnlockSlot = (index: number, type: SlotType) => {
  const slots = type === 'breeding' ? breedingSlots.value : resourceSlots.value;

  if (type === 'breeding') {
    // 产卵室：与资源建筑相同，前2个槽位默认开通
    if (index < 2) return true;

    // 检查前面的槽位是否都已开通
    for (let i = 2; i < index; i++) {
      if (!slots[i].unlocked) {
        return false;
      }
    }
    return true;
  } else {
    // 资源建筑：前2个槽位默认开通
    if (index < 2) return true;

    // 检查前面的槽位是否都已开通
    for (let i = 2; i < index; i++) {
      if (!slots[i].unlocked) {
        return false;
      }
    }
    return true;
  }
};

/**
 * 检查是否是下一个可开通的槽位
 */
const isNextUnlockSlot = (index: number, type: SlotType) => {
  const slots = type === 'breeding' ? breedingSlots.value : resourceSlots.value;
  if (slots[index].unlocked) return false;

  if (type === 'breeding') {
    // 产卵室：与资源建筑相同，从索引2开始查找第一个未开通的槽位
    for (let i = 2; i < slots.length; i++) {
      if (!slots[i].unlocked) {
        return i === index;
      }
    }
  } else {
    // 资源建筑：从索引2开始查找第一个未开通的槽位
    for (let i = 2; i < slots.length; i++) {
      if (!slots[i].unlocked) {
        return i === index;
      }
    }
  }
  return false;
};

/**
 * 获取槽位CSS类名
 */
const getSlotClasses = (slot: BuildingSlot, index: number, type: SlotType) => {
  return {
    occupied: !!slot.building,
    empty: !slot.building && slot.unlocked,
    locked: !slot.unlocked,
    nextUnlock: isNextUnlockSlot(index, type),
  };
};

// ==================== 建筑菜单管理 ====================

/**
 * 显示建筑选择菜单
 */
const showBuildingMenu = (slotIndex: number, type: SlotType) => {
  selectedSlotIndex.value = slotIndex;
  selectedSlotType.value = type;
  showMenu.value = true;
};

/**
 * 关闭建筑菜单
 */
const closeMenu = () => {
  showMenu.value = false;
  selectedSlotIndex.value = -1;
};

// ==================== 建筑建设管理 ====================

/**
 * 检查是否可以建设指定建筑
 */
const canBuild = (building: Building) => {
  // 检查献祭祭坛是否已存在（只允许建造1个）
  if (building.id === 'sacrifice_altar') {
    const existingAltarCount = resourceSlots.value.filter(slot => slot.building?.id === 'sacrifice_altar').length;
    if (existingAltarCount >= 1) {
      return false; // 已经有一个献祭祭坛，不能再建造
    }
    return canAffordBuilding(building.cost);
  }

  if (building.id === 'breeding') {
    // 产卵室成本基于现有数量
    const existingBreedingCount = breedingSlots.value.filter(slot => slot.building?.id === 'breeding').length;
    const dynamicCost = {
      gold: building.cost.gold + existingBreedingCount * 25,
      food: building.cost.food + existingBreedingCount * 15,
    };
    return canAffordBuilding(dynamicCost);
  } else {
    return canAffordBuilding(building.cost);
  }
};

/**
 * 选择建筑进行建设
 */
const selectBuilding = (building: Building) => {
  // 检查献祭祭坛是否已存在
  if (building.id === 'sacrifice_altar') {
    const existingAltarCount = resourceSlots.value.filter(slot => slot.building?.id === 'sacrifice_altar').length;
    if (existingAltarCount >= 1) {
      console.log('献祭祭坛只能建造1个');
      // 可以在这里显示提示消息
      return;
    }
  }

  if (!canBuild(building)) {
    // 显示资源不足提示
    let cost = building.cost;
    if (building.id === 'breeding') {
      // 产卵室使用动态成本
      const existingBreedingCount = breedingSlots.value.filter(slot => slot.building?.id === 'breeding').length;
      cost = {
        gold: building.cost.gold + existingBreedingCount * 25,
        food: building.cost.food + existingBreedingCount * 15,
      };
    }
    const message = getInsufficientResourcesMessage([
      { type: 'gold', amount: cost.gold, reason: `建设${building.name}` },
      { type: 'food', amount: cost.food, reason: `建设${building.name}` },
    ]);
    console.log(message);
    return;
  }

  if (selectedSlotIndex.value >= 0) {
    // 计算实际成本
    let actualCost = building.cost;
    if (building.id === 'breeding') {
      // 产卵室使用动态成本
      const existingBreedingCount = breedingSlots.value.filter(slot => slot.building?.id === 'breeding').length;
      actualCost = {
        gold: building.cost.gold + existingBreedingCount * 25,
        food: building.cost.food + existingBreedingCount * 15,
      };
    }

    // 消耗资源并建设建筑
    if (payForBuilding(actualCost, building.name)) {
      const slots = selectedSlotType.value === 'breeding' ? breedingSlots.value : resourceSlots.value;
      slots[selectedSlotIndex.value].building = building;
      // 立即保存，确保数据不丢失
      saveBuildingData();
      console.log('建筑建设成功，数据已保存');
      closeMenu();
    }
  }
};

/**
 * 拆除建筑
 */
const removeBuilding = async (slotIndex: number, type: SlotType) => {
  const slots = type === 'breeding' ? breedingSlots.value : resourceSlots.value;
  const building = slots[slotIndex].building;
  if (!building) return;

  const confirmed = await ConfirmService.showWarning(
    `确定要拆除 ${building.name} 吗？`,
    '确认拆除',
    `拆除后将失去该建筑的所有效果，且无法恢复。`,
  );

  if (confirmed) {
    slots[slotIndex].building = null;
    // 立即保存，确保数据不丢失
    saveBuildingData();
    console.log('建筑拆除成功，数据已保存');
  }
};

// ==================== 数据持久化 ====================

/**
 * 保存建筑数据到模块化存档系统
 */
const saveBuildingData = (): void => {
  try {
    // 计算当前总收入
    const currentTotalIncome = totalIncome.value;

    const nestData: NestModuleData = {
      breedingSlots: breedingSlots.value,
      resourceSlots: resourceSlots.value,
      activeTab: activeTab.value,
      totalIncome: currentTotalIncome,
      breedingRoomInfo: [], // 产卵室信息由调教界面同步管理
    };

    console.log('保存巢穴数据到模块化存档系统:', nestData);

    // 使用模块化存档服务更新巢穴数据
    modularSaveManager.updateModuleData({
      moduleName: 'nest',
      data: nestData,
    });

    console.log('巢穴数据保存成功');
  } catch (error) {
    console.error('保存巢穴数据失败:', error);
    // 可以在这里添加用户提示
  }
};

/**
 * 从模块化存档系统加载建筑数据
 */
const loadBuildingData = (): void => {
  try {
    console.log('从模块化存档系统加载巢穴数据');

    // 获取当前游戏数据
    const currentGameData = modularSaveManager.getCurrentGameData();

    if (currentGameData && currentGameData.nest) {
      const nestData = currentGameData.nest;
      console.log('加载到巢穴数据:', nestData);

      // 更新界面数据
      breedingSlots.value = nestData.breedingSlots || [];
      resourceSlots.value = nestData.resourceSlots || [];
      activeTab.value = nestData.activeTab || 'breeding';

      console.log('巢穴数据加载成功');
    } else {
      console.log('没有找到巢穴数据，使用初始数据');
      // 如果没有数据，使用初始数据
      const initialNestData = modularSaveManager.getInitialNestData();
      if (initialNestData) {
        breedingSlots.value = initialNestData.breedingSlots;
        resourceSlots.value = initialNestData.resourceSlots;
        activeTab.value = initialNestData.activeTab;
        console.log('使用初始巢穴数据');
      } else {
        console.warn('无法获取初始巢穴数据');
      }
    }
  } catch (error) {
    console.error('加载巢穴数据失败:', error);
    // 发生错误时使用初始数据作为后备
    try {
      const initialNestData = modularSaveManager.getInitialNestData();
      if (initialNestData) {
        breedingSlots.value = initialNestData.breedingSlots;
        resourceSlots.value = initialNestData.resourceSlots;
        activeTab.value = initialNestData.activeTab;
        console.log('使用初始数据作为后备方案');
      }
    } catch (fallbackError) {
      console.error('后备方案也失败:', fallbackError);
    }
  }
};

// ==================== 自动保存机制 ====================

/**
 * 监听建筑数据变化，自动保存
 */
watch(
  [breedingSlots, resourceSlots, activeTab],
  () => {
    // 延迟保存，避免频繁保存
    setTimeout(() => {
      saveBuildingData();
    }, 100);
  },
  { deep: true },
);

// ==================== 组件生命周期 ====================

/**
 * 组件挂载时初始化
 */
onMounted(() => {
  console.log('巢穴界面挂载');
  // 初始化槽位
  initializeSlots();
  // 直接加载建筑数据，简单可靠
  loadBuildingData();
  // 加载人物数据
  loadCharacters();
});

/**
 * 获取交配间占用者
 */
const getBreedingRoomOccupant = (roomIndex: number) => {
  const roomId = `breeding-${roomIndex}`;

  // 首先从巢穴模块的产卵室信息中查找
  try {
    const nestData = modularSaveManager.getModuleData({ moduleName: 'nest' }) as any;
    if (nestData && nestData.breedingRoomInfo) {
      const roomInfo = nestData.breedingRoomInfo.find((room: any) => room.roomId === roomId);
      if (roomInfo) {
        return {
          id: roomInfo.characterId,
          name: roomInfo.characterName,
          status: roomInfo.status,
        };
      }
    }
  } catch (error) {
    console.error('从巢穴模块获取产卵室信息失败:', error);
  }

  // 如果巢穴模块中没有，则从人物数据中查找（兼容性）
  return characters.value.find(
    char => char.locationId === roomId && (char.status === 'breeding' || char.status === 'imprisoned'),
  );
};

/**
 * 加载人物数据
 */
const loadCharacters = () => {
  try {
    const trainingData = modularSaveManager.getModuleData({ moduleName: 'training' }) as any;
    if (trainingData && trainingData.characters) {
      characters.value = trainingData.characters;
    }
  } catch (error) {
    console.error('加载人物数据失败:', error);
  }
};

/**
 * 同步产卵室占用信息
 */
const syncBreedingRoomInfo = () => {
  try {
    const breedingRoomInfo: any[] = [];

    // 遍历所有人物，找出占用产卵室的人物
    characters.value.forEach(char => {
      if (char.locationId && char.locationId.startsWith('breeding-')) {
        breedingRoomInfo.push({
          roomId: char.locationId,
          characterId: char.id,
          characterName: char.name,
          status: char.status === 'breeding' ? 'breeding' : 'imprisoned',
          occupiedAt: new Date(),
        });
      }
    });

    // 获取当前巢穴数据
    const currentNestData = modularSaveManager.getModuleData({ moduleName: 'nest' }) as any;

    // 更新巢穴数据
    modularSaveManager.updateModuleData({
      moduleName: 'nest',
      data: {
        ...currentNestData,
        breedingRoomInfo: breedingRoomInfo,
      },
    });

    console.log('巢穴界面：产卵室占用信息已同步:', breedingRoomInfo);
  } catch (error) {
    console.error('巢穴界面：同步产卵室信息失败:', error);
  }
};

/**
 * 组件激活时重新加载数据（防止从其他页面返回时数据不同步）
 */
onActivated(() => {
  console.log('巢穴界面激活');
  loadBuildingData();
  loadCharacters();
  // 同步产卵室信息，确保显示最新状态
  syncBreedingRoomInfo();
});

// ==================== 献祭相关方法 ====================

/**
 * 打开献祭对话框
 */
const openSacrificeDialog = (slotIndex: number) => {
  currentSacrificeSlotIndex.value = slotIndex;
  showSacrificeDialog.value = true;
};

/**
 * 关闭献祭对话框
 */
const closeSacrificeDialog = () => {
  showSacrificeDialog.value = false;
  currentSacrificeSlotIndex.value = -1;
};

/**
 * 处理献祭确认
 */
const handleSacrificeConfirm = async (characterId: string, sacrificeAmounts: SacrificeAmounts) => {
  // 计算献祭总数和提示信息
  const totalAmount =
    sacrificeAmounts.normalGoblins +
    sacrificeAmounts.warriorGoblins +
    sacrificeAmounts.shamanGoblins +
    sacrificeAmounts.paladinGoblins;
  const sacrificeMessage = SacrificeService.getSacrificeMessage(characterId, sacrificeAmounts);

  // 确认献祭
  const confirmed = await ConfirmService.showWarning(
    `确定要献祭 ${totalAmount} 个衍生物吗？`,
    '确认献祭',
    `将消耗 ${totalAmount} 个衍生物，${sacrificeMessage.message}`,
  );

  if (!confirmed) {
    return;
  }

  // 执行献祭
  const result = SacrificeService.performSacrifice(characterId, sacrificeAmounts);

  if (result.success) {
    if (result.newLevel > result.oldLevel) {
      console.log(result.message);
      // 献祭成功后，更新玩家等级（因为人物等级提升了）
      PlayerLevelService.updatePlayerLevel();
      // 触发事件通知调教界面刷新人物数据
      eventEmit('人物等级更新');
      // 可以在这里显示成功提示
    } else {
      console.log(result.message);
      // 即使等级没有提升，也更新玩家等级（确保玩家等级是最新的）
      PlayerLevelService.updatePlayerLevel();
      // 可以在这里显示提示
    }
  } else {
    console.error(result.message);
    // 可以在这里显示错误提示
    return;
  }

  // 关闭对话框
  closeSacrificeDialog();
};
</script>

<style lang="scss" scoped>
// ==================== 基础容器样式 ====================

.nest-container {
  height: calc(100vh - 90px);
  width: 100%;
  max-width: 100%;
  padding: 16px;
  background: linear-gradient(180deg, rgba(40, 26, 20, 0.6), rgba(25, 17, 14, 0.85));
  border: 1px solid rgba(205, 133, 63, 0.25);
  border-radius: 12px;
  box-shadow:
    inset 0 1px 0 rgba(255, 200, 150, 0.08),
    0 8px 18px rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  position: relative;
  box-sizing: border-box;
  overflow: hidden;
}

// ==================== 头部样式 ====================

.nest-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;

  .nest-title {
    margin: 0;
    color: #ffd7a1;
    text-shadow:
      0 2px 6px rgba(0, 0, 0, 0.6),
      0 0 10px rgba(255, 120, 40, 0.2);
    font-size: 20px;
  }
}

.income-summary {
  display: flex;
  align-items: center;
  gap: 8px;
  background: linear-gradient(180deg, rgba(34, 197, 94, 0.15), rgba(28, 20, 17, 0.9));
  border: 1px solid rgba(34, 197, 94, 0.3);
  border-radius: 8px;
  padding: 6px 12px;

  .income-item {
    display: flex;
    align-items: center;
    gap: 4px;

    .income-icon {
      font-size: 14px;
    }

    .income-amount {
      color: #22c55e;
      font-size: 12px;
      font-weight: 600;
      @media (min-width: 769px) {
        font-size: 16px;
      }
    }
  }

  .income-label {
    color: #9ca3af;
    font-size: 10px;
    font-weight: 500;
    margin-left: 4px;
    @media (min-width: 769px) {
      font-size: 12px;
    }
  }
}

// ==================== 标签页样式 ====================

.building-tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;

  .tab-button {
    background: linear-gradient(180deg, rgba(44, 30, 24, 0.8), rgba(28, 20, 17, 0.9));
    color: #f0e6d2;
    border: 1px solid rgba(205, 133, 63, 0.3);
    border-radius: 8px;
    padding: 8px 16px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 600;
    transition: all 0.2s ease;
    flex: 1;
    @media (min-width: 769px) {
      font-size: 16px;
    }

    &:hover {
      background: linear-gradient(180deg, rgba(44, 30, 24, 0.9), rgba(28, 20, 17, 0.95));
      border-color: rgba(205, 133, 63, 0.5);
      transform: translateY(-1px);
    }

    &.active {
      background: linear-gradient(180deg, rgba(34, 197, 94, 0.2), rgba(28, 20, 17, 0.9));
      border-color: rgba(34, 197, 94, 0.5);
      color: #22c55e;
    }
  }
}

// ==================== 内容区域样式 ====================

.building-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.building-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

// ==================== 滚动容器样式 ====================

.building-scroll-container {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  padding-right: 8px;

  // 自定义滚动条样式
  &::-webkit-scrollbar {
    width: 8px;
  }

  &::-webkit-scrollbar-track {
    background: rgba(44, 30, 24, 0.3);
    border-radius: 4px;
  }

  &::-webkit-scrollbar-thumb {
    background: linear-gradient(180deg, rgba(205, 133, 63, 0.6), rgba(139, 69, 19, 0.8));
    border-radius: 4px;
    transition: all 0.2s ease;

    &:hover {
      background: linear-gradient(180deg, rgba(205, 133, 63, 0.8), rgba(139, 69, 19, 1));
    }
  }

  // Firefox 滚动条样式
  scrollbar-width: thin;
  scrollbar-color: rgba(205, 133, 63, 0.6) rgba(44, 30, 24, 0.3);
}

// ==================== 网格布局样式 ====================

.building-grid {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 8px;
  padding: 8px;
  width: 100%;
  min-height: fit-content;

  // 移动端优化
  @media (max-width: 768px) {
    grid-template-columns: repeat(2, 1fr);
    gap: 8px;
    padding: 4px;
  }
}

// ==================== 建筑槽位样式 ====================

.building-slot {
  aspect-ratio: 1;
  background: linear-gradient(180deg, rgba(44, 30, 24, 0.72), rgba(28, 20, 17, 0.92));
  border: 2px solid rgba(205, 133, 63, 0.25);
  border-radius: 8px;
  padding: 8px;
  box-shadow:
    inset 0 1px 0 rgba(255, 200, 150, 0.08),
    0 4px 10px rgba(0, 0, 0, 0.4);
  transition: all 0.3s ease;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;
  min-width: 0;
  min-height: 0;

  // 移动端优化
  @media (max-width: 768px) {
    padding: 8px;
    border-radius: 8px;
  }

  &:hover {
    transform: translateY(-2px);
    box-shadow:
      inset 0 1px 0 rgba(255, 200, 150, 0.12),
      0 8px 18px rgba(0, 0, 0, 0.5);
  }

  // 槽位状态样式
  &.occupied {
    border-color: rgba(34, 197, 94, 0.5);
    background: linear-gradient(180deg, rgba(34, 197, 94, 0.1), rgba(28, 20, 17, 0.92));
  }

  &.empty {
    border-color: rgba(107, 114, 128, 0.5);
    background: linear-gradient(180deg, rgba(107, 114, 128, 0.1), rgba(28, 20, 17, 0.92));
  }

  &.locked {
    border-color: rgba(107, 114, 128, 0.3);
    background: linear-gradient(180deg, rgba(107, 114, 128, 0.05), rgba(28, 20, 17, 0.92));
    opacity: 0.6;
  }

  &.nextUnlock {
    border-color: rgba(34, 197, 94, 0.5);
    background: linear-gradient(180deg, rgba(34, 197, 94, 0.1), rgba(28, 20, 17, 0.92));
  }
}

// ==================== 建筑内容样式 ====================

.building {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;

  .building-icon {
    font-size: 40px;
    margin-bottom: 8px;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 24px;
      margin-bottom: 4px;
    }
  }

  .building-name {
    color: #ffd7a1;
    font-weight: 700;
    font-size: 16px;
    text-align: center;
    line-height: 1.2;
    margin-bottom: 4px;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 12px;
    }
    @media (min-width: 769px) {
      font-size: 18px;
    }
  }

  .building-income {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
    margin-bottom: 4px;

    .income-display {
      display: flex;
      align-items: center;
      gap: 2px;
      background: rgba(34, 197, 94, 0.1);
      border: 1px solid rgba(34, 197, 94, 0.3);
      border-radius: 4px;
      padding: 2px 4px;

      .income-icon {
        font-size: 10px;
      }

      .income-text {
        color: #22c55e;
        font-size: 10px;
        font-weight: 600;
        @media (min-width: 769px) {
          font-size: 16px;
        }
      }
    }
  }

  .remove-button {
    position: absolute;
    top: 4px;
    right: 4px;
    width: 20px;
    height: 20px;
    background: linear-gradient(180deg, #dc2626, #b91c1c);
    color: #ffffff;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    font-size: 12px;
    font-weight: bold;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;

    &:hover {
      background: linear-gradient(180deg, #ef4444, #dc2626);
      transform: scale(1.1);
    }
  }

  .breeding-status {
    position: absolute;
    bottom: 4px;
    left: 4px;
    right: 4px;
    background: rgba(0, 0, 0, 0.7);
    border-radius: 4px;
    padding: 2px 4px;
    font-size: 10px;

    .occupied-status {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 1px;

      .occupant-name {
        color: #ffd7a1;
        font-weight: 600;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        max-width: 100%;
        @media (min-width: 769px) {
          font-size: 16px;
        }
      }

      .occupant-status {
        color: #ff6b6b;
        font-size: 8px;
        @media (min-width: 769px) {
          font-size: 16px;
        }
      }
    }

    .available-status {
      display: flex;
      justify-content: center;
      align-items: center;

      .available-text {
        color: #90ee90;
        font-weight: 600;
        font-size: 8px;
        @media (min-width: 769px) {
          font-size: 16px;
        }
      }
    }
  }
}

// ==================== 槽位状态样式 ====================

.empty-slot {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  .empty-icon {
    font-size: 40px;
    margin-bottom: 8px;
    opacity: 0.6;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 24px;
      margin-bottom: 4px;
    }
  }

  .empty-text {
    color: #9ca3af;
    font-size: 14px;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 10px;
    }
  }
}

.next-unlock-slot {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  .expand-icon {
    font-size: 40px;
    margin-bottom: 8px;
    color: #22c55e;
    font-weight: bold;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 24px;
      margin-bottom: 4px;
    }
  }

  .expand-text {
    color: #22c55e;
    font-size: 14px;
    font-weight: 600;
    margin-bottom: 4px;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 10px;
    }
    @media (min-width: 769px) {
      font-size: 16px;
    }
  }

  .expand-cost {
    color: #fbbf24;
    font-size: 12px;
    font-weight: 600;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 8px;
    }
  }
}

.locked-slot {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  .locked-icon {
    font-size: 40px;
    margin-bottom: 8px;
    color: #6b7280;
    opacity: 0.6;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 24px;
      margin-bottom: 4px;
    }
  }

  .locked-text {
    color: #6b7280;
    font-size: 14px;
    opacity: 0.6;

    // 移动端优化
    @media (max-width: 768px) {
      font-size: 10px;
    }
    @media (min-width: 769px) {
      font-size: 16px;
    }
  }
}

// ==================== 建筑菜单样式 ====================

.building-menu {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: linear-gradient(180deg, rgba(40, 26, 20, 0.95), rgba(25, 17, 14, 0.98));
  border: 2px solid rgba(205, 133, 63, 0.5);
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.6);
  z-index: 1000;
  max-width: 500px;
  width: 90%;

  .menu-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;

    h4 {
      color: #ffd7a1;
      margin: 0;
      font-size: 18px;
    }

    .close-menu {
      background: none;
      border: none;
      color: #f0e6d2;
      font-size: 24px;
      cursor: pointer;
      padding: 0;
      width: 30px;
      height: 30px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 50%;

      &:hover {
        background: rgba(255, 255, 255, 0.1);
      }
    }
  }

  .building-options {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .building-option {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px;
    background: linear-gradient(180deg, rgba(44, 30, 24, 0.8), rgba(28, 20, 17, 0.9));
    border: 1px solid rgba(205, 133, 63, 0.3);
    border-radius: 10px;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover:not(.disabled) {
      background: linear-gradient(180deg, rgba(44, 30, 24, 0.9), rgba(28, 20, 17, 0.95));
      border-color: rgba(205, 133, 63, 0.5);
      transform: translateY(-2px);
    }

    &.disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    .option-icon {
      font-size: 24px;
    }

    .option-texts {
      display: flex;
      flex-direction: column;
      gap: 2px;
      flex: 1;

      .option-name {
        color: #ffd7a1;
        font-weight: 600;
        font-size: 16px;
        line-height: 1.1;
        @media (min-width: 769px) {
          font-size: 18px;
        }
      }

      .option-desc {
        color: #9ca3af;
        font-size: 12px;
        line-height: 1.2;
        @media (min-width: 769px) {
          font-size: 14px;
        }
      }
    }

    .option-cost {
      color: #87ceeb;
      font-size: 13px;
      @media (min-width: 769px) {
        font-size: 15px;
      }
    }
  }
}

// ==================== 献祭祭坛相关样式 ====================

.sacrifice-button-container {
  margin-top: 4px;
}

.sacrifice-button {
  background: linear-gradient(180deg, #dc2626, #b91c1c);
  color: #ffffff;
  border: none;
  border-radius: 6px;
  padding: 4px 8px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  width: 100%;

  &:hover {
    background: linear-gradient(180deg, #ef4444, #dc2626);
    transform: translateY(-1px);
    box-shadow: 0 2px 8px rgba(220, 38, 38, 0.4);
  }

  &:active {
    transform: translateY(0);
  }
}
</style>
