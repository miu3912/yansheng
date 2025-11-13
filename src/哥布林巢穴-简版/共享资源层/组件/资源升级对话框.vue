<template>
  <div v-if="show" class="dialog-overlay">
    <div class="dialog-content">
      <div class="dialog-header">
        <h3>🌟 经验升级</h3>
        <button class="close-button" @click="closeDialog">×</button>
      </div>

      <div class="dialog-body">
        <!-- 人物选择 -->
        <div class="section">
          <h4>选择人物</h4>
          <select v-model="selectedCharacterId" class="character-select" @change="updatePrediction">
            <option value="">请选择要升级的人物</option>
            <option 
              v-for="character in upgradableCharacters" 
              :key="character.id" 
              :value="character.id"
            >
              {{ character.name }} (等级: {{ getCharacterLevel(character) }})
            </option>
          </select>
        </div>

        <!-- 资源输入 -->
        <div class="section">
          <h4>消耗资源</h4>
          <div class="resource-inputs">
            <div class="resource-input">
              <label>💰 金币数量:</label>
              <input 
                v-model.number="goldAmount" 
                type="number" 
                min="0" 
                :max="availableGold"
                @input="updatePrediction"
              >
              <span class="available">可用: {{ availableGold.toLocaleString() }}</span>
            </div>
            <div class="resource-input">
              <label>🍖 食物数量:</label>
              <input 
                v-model.number="foodAmount" 
                type="number" 
                min="0" 
                :max="availableFood"
                @input="updatePrediction"
              >
              <span class="available">可用: {{ availableFood.toLocaleString() }}</span>
            </div>
          </div>
        </div>

        <!-- 升级预览 -->
        <div v-if="selectedCharacterId" class="section">
          <h4>升级预览</h4>
          <div class="prediction-info">
            <div v-if="prediction.canLevelUp" class="success-prediction">
              <div class="level-change">
                {{ getCharacterName(selectedCharacterId) }}: 
                {{ prediction.oldLevel }} → {{ prediction.predictedLevel }}
                <span class="level-ups">(+{{ prediction.levelUps }}级)</span>
              </div>
              <div class="cost-info">
                消耗: {{ prediction.totalCost.gold.toLocaleString() }}💰 {{ prediction.totalCost.food.toLocaleString() }}🍖
              </div>
            </div>
            <div v-else class="error-prediction">
              {{ prediction.message }}
            </div>
          </div>
        </div>
      </div>

      <div class="dialog-actions">
        <button 
          class="confirm-button" 
          :disabled="!canConfirm"
          @click="confirmUpgrade"
        >
          确认升级
        </button>
        <button class="cancel-button" @click="closeDialog">取消</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';
import { modularSaveManager } from '../核心层/服务/存档系统/模块化存档服务';
import { ExperienceLevelUpService } from '../核心层/服务/通用服务/经验升级服务';
import type { Character } from '../../../功能模块层/人物管理/类型/人物类型';

interface Props {
  show: boolean;
}

interface Emits {
  (e: 'close'): void;
  (e: 'confirm', characterId: string, goldAmount: number, foodAmount: number): void;
}

const props = defineProps<Props>();
const emit = defineEmits<Emits>();

// 响应式数据
const selectedCharacterId = ref('');
const goldAmount = ref(0);
const foodAmount = ref(0);
const upgradableCharacters = ref<Character[]>([]);
const prediction = ref({
  levelUps: 0,
  predictedLevel: 0,
  oldLevel: 0,
  totalCost: { gold: 0, food: 0 },
  canLevelUp: false,
  message: ''
});

// 计算属性
const availableGold = computed(() => modularSaveManager.resources.value.gold || 0);
const availableFood = computed(() => modularSaveManager.resources.value.food || 0);

const canConfirm = computed(() => {
  return selectedCharacterId.value && 
         goldAmount.value > 0 && 
         foodAmount.value > 0 && 
         prediction.value.canLevelUp;
});

// 方法
const closeDialog = () => {
  resetForm();
  emit('close');
};

const confirmUpgrade = () => {
  if (canConfirm.value) {
    emit('confirm', selectedCharacterId.value, goldAmount.value, foodAmount.value);
    resetForm();
  }
};

const resetForm = () => {
  selectedCharacterId.value = '';
  goldAmount.value = 0;
  foodAmount.value = 0;
  prediction.value = {
    levelUps: 0,
    predictedLevel: 0,
    oldLevel: 0,
    totalCost: { gold: 0, food: 0 },
    canLevelUp: false,
    message: ''
  };
};

const updatePrediction = () => {
  if (selectedCharacterId.value && goldAmount.value >= 0 && foodAmount.value >= 0) {
    const result = ExperienceLevelUpService.getLevelUpMessage(
      selectedCharacterId.value, 
      goldAmount.value, 
      foodAmount.value
    );
    prediction.value = result;
  }
};

const getCharacterLevel = (character: Character): number => {
  return character.level ?? Math.floor((character.offspring ?? 0) / 10) ?? 1;
};

const getCharacterName = (characterId: string): string => {
  const character = upgradableCharacters.value.find(c => c.id === characterId);
  return character?.name || '未知人物';
};

// 加载可升级的人物列表
const loadUpgradableCharacters = () => {
  try {
    upgradableCharacters.value = ExperienceLevelUpService.getUpgradableCharacters();
  } catch (error) {
    console.error('加载可升级人物失败:', error);
  }
};

// 监听显示状态
watch(() => props.show, (newVal) => {
  if (newVal) {
    loadUpgradableCharacters();
    resetForm();
  }
});

onMounted(() => {
  loadUpgradableCharacters();
});
</script>

<style lang="scss" scoped>
// 样式部分保持不变...
</style>