<template>
  <div v-if="show" class="json-editor-overlay">
    <div class="json-editor-modal" @click.stop>
      <div class="json-editor-header">
        <h4>编辑人物JSON（测试性功能，不确定项不要乱改！！）</h4>
        <button class="close-btn" @click="close">×</button>
      </div>
      <div class="json-editor-body">
        <textarea v-model="jsonText" class="json-textarea" spellcheck="false"></textarea>
        <div v-if="jsonError" class="json-error">{{ jsonError }}</div>
      </div>
      <div class="json-editor-footer">
        <button class="json-btn json-btn-cancel" @click="close">取消</button>
        <button class="json-btn json-btn-save" :disabled="!!jsonError || isSaving" @click="saveJson">
          {{ isSaving ? '保存中...' : '保存' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';
import type { Character } from '../../功能模块层/人物管理/类型/人物类型';
import { WorldbookService } from '../../核心层/服务/世界书管理/服务/世界书服务';
import { modularSaveManager } from '../../核心层/服务/存档系统/模块化存档服务';
import { toast } from '../../核心层/服务/通用服务/弹窗提示服务';

// 定义组件属性
interface Props {
  show: boolean;
  character: Character | null;
}

// 定义组件事件
interface Emits {
  (e: 'close'): void;
  (e: 'character-updated', character: Character): void;
}

const props = defineProps<Props>();
const emit = defineEmits<Emits>();

// 字段名中英文映射表
const fieldNameMap: Record<string, string> = {
  // 基础信息
  id: '唯一标识符（勿动）',
  name: '姓名（勿动）',
  title: '身份',
  avatar: '头像（不确定勿动）',
  corruptedAvatar: '半堕落头像（不确定勿动）',
  fullyCorruptedAvatar: '完全堕落头像（不确定勿动）',

  // 状态信息
  status: '状态（imprisoned[关押中], training[调教中], breeding[交配中]）',
  originalStatus: '原始状态（勿动）',
  locationId: '位置ID（勿动）',
  capturedAt: '捕获时间',
  canCombat: '可战斗',

  // 属性信息
  loyalty: '堕落值',
  stamina: '当前体力',
  fertility: '当前生育力',
  offspring: '产卵数量',
  maxStamina: '最大体力（不超过200）',
  maxFertility: '最大生育力（不超过200）',
  rating: '评级（S/A/B/C/D）',
  favorite: '是否收藏',

  // 战斗属性
  level: '等级',
  attributes: '基础属性',
  deployedAttributes: '部署属性',
  troopDeployment: '部队编制',
  formationPosition: '编制位置',

  // 训练信息
  lastTraining: '最后训练时间',

  // 详细人物信息
  race: '种族',
  age: '年龄',
  country: '国家',
  background: '出身等级（勿动）',
  unitType: '单位类型（physical[物理], magical[魔法]）',
  canLeadRaces: '可带领种族',
  sexExperience: '性经验',
  sensitivePoints: '敏感点',
  sensitivePointsDetail: '敏感点详情',
  lifeStory: '人生经历',
  personality: '性格特征',
  fears: '恐惧',
  secrets: '秘密',
  appearance: '外观信息',

  // 属性子字段
  attack: '攻击力',
  defense: '防御力',
  intelligence: '智力',
  speed: '速度',
  health: '血量',

  // 外观子字段
  height: '身高',
  weight: '体重',
  measurements: '三围',
  cupSize: '罩杯',
  description: '描述',
  clothing: '衣着',
  originalClothing: '原始衣着（不确定勿动）',
  corruptedClothing: '堕落衣着（不确定勿动）',
  head: '头部',
  top: '上装',
  bottom: '下装',
  socks: '袜子',
  shoes: '鞋子',
  underwear: '内衣',
  accessories: '装饰品',
  toys: '玩具',

  // 敏感点子字段
  part: '部位',
  isSensitive: '是否敏感',

  // 人生经历子字段
  childhood: '童年经历',
  adolescence: '青少年经历',
  adulthood: '成年经历',
  currentState: '当前状态',

  // 部队编制子字段
  normalGoblins: '普通衍生物',
  warriorGoblins: '衍生物战士',
  shamanGoblins: '衍生物萨满',
  paladinGoblins: '衍生物圣骑士',

  // 额外附加信息
  additionalInformation: '额外附加信息',
  Notes: '玩家备注',
};

// 需要隐藏的字段列表（不显示在编辑器中，但保存时会保留原始值）
const hiddenFields = new Set([
  'id', // 唯一标识符
  'favorite', // 是否收藏
  'clothing', // 衣着
  'isSensitive', // 是否敏感
  'originalClothing', // 原始衣着
  'corruptedClothing', // 堕落衣着
  'locationId', // 位置ID
  'attributes', // 基础属性
]);

// 创建反向映射表（中文 -> 英文）
const reverseFieldNameMap: Record<string, string> = {};
Object.entries(fieldNameMap).forEach(([en, zh]) => {
  reverseFieldNameMap[zh] = en;
});

// 递归转换对象字段名为中文（并过滤隐藏字段）
function translateKeysToChinese(obj: any, parentKey?: string): any {
  if (obj === null || obj === undefined) {
    return obj;
  }

  if (Array.isArray(obj)) {
    return obj.map(item => translateKeysToChinese(item, parentKey));
  }

  if (typeof obj === 'object') {
    const translated: any = {};
    for (const [key, value] of Object.entries(obj)) {
      // 如果是隐藏字段，跳过不显示
      if (hiddenFields.has(key)) {
        continue;
      }

      const chineseKey = fieldNameMap[key] || key;
      // 对于嵌套对象，传递父级key用于判断是否在appearance内部
      translated[chineseKey] = translateKeysToChinese(value, key);
    }
    return translated;
  }

  return obj;
}

// 递归转换对象字段名为英文
function translateKeysToEnglish(obj: any): any {
  if (obj === null || obj === undefined) {
    return obj;
  }

  if (Array.isArray(obj)) {
    return obj.map(item => translateKeysToEnglish(item));
  }

  if (typeof obj === 'object') {
    const translated: any = {};
    for (const [key, value] of Object.entries(obj)) {
      const englishKey = reverseFieldNameMap[key] || key;
      translated[englishKey] = translateKeysToEnglish(value);
    }
    return translated;
  }

  return obj;
}

// JSON编辑器状态
const jsonText = ref('');
const jsonError = ref('');
const isSaving = ref(false);

// 监听show变化，初始化JSON文本
watch(
  () => props.show,
  newShow => {
    if (newShow && props.character) {
      // 确保 additionalInformation 字段存在（如果不存在则添加默认值）
      const characterWithDefaults = {
        ...props.character,
        additionalInformation: props.character.additionalInformation || {
          Notes: '',
        },
      };

      // 将字段名转换为中文后显示
      const chineseJson = translateKeysToChinese(characterWithDefaults);
      jsonText.value = JSON.stringify(chineseJson, null, 2);
      jsonError.value = '';
      validateJson();
    } else {
      jsonText.value = '';
      jsonError.value = '';
    }
  },
  { immediate: true },
);

// 关闭编辑器
const close = () => {
  emit('close');
};

// 验证JSON格式
const validateJson = () => {
  jsonError.value = '';
  if (!jsonText.value.trim()) {
    jsonError.value = 'JSON不能为空';
    return;
  }
  try {
    const parsed = JSON.parse(jsonText.value);

    // 尝试将中文字段名转换为英文，验证是否可以转换
    try {
      translateKeysToEnglish(parsed);
    } catch (transError) {
      jsonError.value = `字段名转换错误: ${transError instanceof Error ? transError.message : '未知错误'}`;
      return;
    }
  } catch (error) {
    jsonError.value = `JSON格式错误: ${error instanceof Error ? error.message : '未知错误'}`;
  }
};

// 监听JSON文本变化，实时验证
watch(jsonText, () => {
  validateJson();
});

// 保存JSON
const saveJson = async () => {
  if (!props.character || jsonError.value || isSaving.value) return;

  try {
    isSaving.value = true;

    // 解析JSON（此时字段名是中文）
    const chineseJson = JSON.parse(jsonText.value);

    // 在转换之前，先保存原始appearance的深拷贝（用于后续恢复服装信息）
    const originalAppearance = props.character.appearance
      ? JSON.parse(JSON.stringify(props.character.appearance))
      : null;

    // 将字段名转换回英文
    const updatedCharacter = translateKeysToEnglish(chineseJson) as Character;

    // 处理attributes.Unittype到unitType的转换（向后兼容）
    const attributes = updatedCharacter.attributes as any;
    if (attributes?.Unittype && !updatedCharacter.unitType) {
      // 如果attributes中有Unittype，将其转换为unitType
      updatedCharacter.unitType = attributes.Unittype as 'physical' | 'magical';
      // 从attributes中移除Unittype，因为正确的字段名是unitType（不在attributes中）
      delete attributes.Unittype;
    }

    // 确保unitType字段存在（如果缺失则从原始数据恢复或使用默认值）
    if (!updatedCharacter.unitType) {
      updatedCharacter.unitType = props.character.unitType || 'physical';
    }

    // 特殊处理appearance中的服装信息（必须在其他隐藏字段恢复之前处理）
    if (originalAppearance) {
      console.log('🔍 [人物编辑] 开始恢复服装信息...', {
        原始clothing存在: !!originalAppearance.clothing,
        原始originalClothing存在: !!originalAppearance.originalClothing,
        原始corruptedClothing存在: !!originalAppearance.corruptedClothing,
        用户appearance存在: !!updatedCharacter.appearance,
      });

      // 确保appearance对象存在
      if (!updatedCharacter.appearance) {
        // 如果用户删除了整个 appearance，完全恢复原始数据
        console.log('📦 [人物编辑] 用户删除了appearance，完全恢复原始数据');
        updatedCharacter.appearance = JSON.parse(JSON.stringify(originalAppearance));
      } else {
        // 用户保留了appearance，需要合并字段并强制恢复服装信息
        // 先合并用户修改的其他字段（height、weight、description等）
        const mergedAppearance: any = {
          ...updatedCharacter.appearance,
        };

        console.log('🔄 [人物编辑] 合并appearance字段，恢复服装信息', {
          用户appearance字段: Object.keys(updatedCharacter.appearance),
          合并前clothing存在: !!mergedAppearance.clothing,
        });

        // 强制恢复服装信息（无论用户是否修改了appearance的其他字段）
        // 这些字段在JSON编辑器中是隐藏的，所以必须从原始数据恢复
        if (originalAppearance.clothing) {
          mergedAppearance.clothing = JSON.parse(JSON.stringify(originalAppearance.clothing));
          console.log('✅ [人物编辑] 已恢复clothing:', Object.keys(mergedAppearance.clothing));
        }
        if (originalAppearance.originalClothing) {
          mergedAppearance.originalClothing = JSON.parse(JSON.stringify(originalAppearance.originalClothing));
          console.log('✅ [人物编辑] 已恢复originalClothing:', Object.keys(originalAppearance.originalClothing));
        }
        if (originalAppearance.corruptedClothing) {
          mergedAppearance.corruptedClothing = JSON.parse(JSON.stringify(originalAppearance.corruptedClothing));
          console.log('✅ [人物编辑] 已恢复corruptedClothing:', Object.keys(originalAppearance.corruptedClothing));
        }

        updatedCharacter.appearance = mergedAppearance;

        console.log('✅ [人物编辑] 服装信息恢复完成', {
          最终clothing存在: !!mergedAppearance.clothing,
          最终originalClothing存在: !!mergedAppearance.originalClothing,
          最终corruptedClothing存在: !!mergedAppearance.corruptedClothing,
        });
      }
    } else {
      console.warn('⚠️ [人物编辑] 原始人物没有appearance数据，无法恢复服装信息');
    }

    // 特殊处理attributes对象（战斗属性是计算出来的，必须完整保留）
    if (props.character.attributes) {
      console.log('🔍 [人物编辑] 恢复attributes战斗属性...');
      updatedCharacter.attributes = JSON.parse(JSON.stringify(props.character.attributes));
      // 【旧存档兼容性处理】确保恢复的attributes中没有Unittype字段
      if ((updatedCharacter.attributes as any).Unittype) {
        console.log('⚠️ [人物编辑] 发现恢复的attributes中有Unittype字段，已删除');
        delete (updatedCharacter.attributes as any).Unittype;
      }
      console.log('✅ [人物编辑] attributes已恢复:', updatedCharacter.attributes);
    }

    // 特殊处理sensitivePointsDetail数组中的isSensitive字段
    if (props.character.sensitivePointsDetail && props.character.sensitivePointsDetail.length > 0) {
      console.log('🔍 [人物编辑] 检查sensitivePointsDetail数组...');

      // 如果用户修改了sensitivePointsDetail，需要恢复每个元素中的isSensitive字段
      if (updatedCharacter.sensitivePointsDetail) {
        const originalDetailMap = new Map(
          props.character.sensitivePointsDetail.map(item => [item.part, item.isSensitive]),
        );

        // 恢复每个敏感点的isSensitive状态
        updatedCharacter.sensitivePointsDetail = updatedCharacter.sensitivePointsDetail.map((item: any) => {
          const originalIsSensitive = originalDetailMap.get(item.part);
          if (originalIsSensitive !== undefined) {
            return {
              ...item,
              isSensitive: originalIsSensitive,
            };
          }
          return item;
        });

        console.log('✅ [人物编辑] sensitivePointsDetail中的isSensitive已恢复');
      } else {
        // 如果用户删除了sensitivePointsDetail，完全恢复原始数据
        updatedCharacter.sensitivePointsDetail = JSON.parse(JSON.stringify(props.character.sensitivePointsDetail));
        console.log('✅ [人物编辑] sensitivePointsDetail已完全恢复');
      }
    }

    // 处理 additionalInformation 字段（确保结构正确）
    if (updatedCharacter.additionalInformation) {
      // 确保 additionalInformation 对象有 Notes 字段
      if (!updatedCharacter.additionalInformation.Notes) {
        updatedCharacter.additionalInformation.Notes = '';
      }
    } else {
      // 如果用户删除了 additionalInformation，保留原始值或设为默认值
      updatedCharacter.additionalInformation = props.character.additionalInformation || {
        Notes: '',
      };
    }

    // 恢复其他隐藏字段的原始值（避免丢失）
    for (const hiddenField of hiddenFields) {
      // 跳过已特殊处理的字段
      if (
        hiddenField === 'clothing' ||
        hiddenField === 'originalClothing' ||
        hiddenField === 'corruptedClothing' ||
        hiddenField === 'attributes' ||
        hiddenField === 'isSensitive' // isSensitive在数组中，不在顶层
      ) {
        continue;
      }

      const originalValue = (props.character as any)[hiddenField];
      if (originalValue !== undefined) {
        console.log(`🔍 [人物编辑] 恢复隐藏字段: ${hiddenField}`);
        // 直接设置隐藏字段的原始值（使用深拷贝）
        (updatedCharacter as any)[hiddenField] = JSON.parse(JSON.stringify(originalValue));
        console.log(`✅ [人物编辑] ${hiddenField}已恢复`);
      }
    }

    // 获取训练数据
    const trainingData = modularSaveManager.getModuleData({ moduleName: 'training' }) as any;
    const characters = (trainingData?.characters || []) as Character[];

    // 更新人物数据
    const updatedCharacters = characters.map(char => {
      if (char.id === updatedCharacter.id) {
        return updatedCharacter;
      }
      return char;
    });

    // 更新存档
    modularSaveManager.updateModuleData({
      moduleName: 'training',
      data: {
        ...trainingData,
        characters: updatedCharacters,
      },
    });

    // 保存到数据库
    await modularSaveManager.saveCurrentGameData(0);

    // 更新世界书
    await WorldbookService.updateCharacterEntry(updatedCharacter);

    // 通知父组件更新人物数据
    emit('character-updated', updatedCharacter);

    // toast.success(`人物 ${updatedCharacter.name} 的JSON数据已保存并更新世界书`);

    // 关闭编辑器
    close();
  } catch (error) {
    console.error('保存JSON失败:', error);
    toast.error(`保存失败: ${error instanceof Error ? error.message : '未知错误'}`);
  } finally {
    isSaving.value = false;
  }
};
</script>

<style scoped lang="scss">
// JSON编辑器样式
.json-editor-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20000;
  backdrop-filter: blur(4px);
}

.json-editor-modal {
  background: linear-gradient(135deg, rgba(40, 26, 20, 0.98), rgba(26, 19, 19, 0.98));
  border: 2px solid rgba(205, 133, 63, 0.6);
  border-radius: 16px;
  width: 90%;
  max-width: 900px;
  max-height: 85vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.7);
  animation: slideIn 0.3s ease;

  .json-editor-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 24px;
    border-bottom: 2px solid rgba(205, 133, 63, 0.4);

    h4 {
      margin: 0;
      color: #ffd7a1;
      font-size: 18px;
      font-weight: 700;
    }

    .close-btn {
      background: none;
      border: none;
      color: #9ca3af;
      font-size: 28px;
      cursor: pointer;
      padding: 4px 8px;
      border-radius: 4px;
      transition: all 0.2s ease;
      line-height: 1;

      &:hover {
        background: rgba(239, 68, 68, 0.1);
        color: #ef4444;
      }
    }
  }

  .json-editor-body {
    flex: 1;
    padding: 20px;
    overflow: hidden;
    display: flex;
    flex-direction: column;

    .json-textarea {
      flex: 1;
      width: 100%;
      min-height: 400px;
      padding: 12px;
      background: rgba(30, 20, 16, 0.9);
      border: 2px solid rgba(205, 133, 63, 0.4);
      border-radius: 8px;
      color: #f0e6d2;
      font-family: 'Courier New', Consolas, monospace;
      font-size: 13px;
      line-height: 1.6;
      resize: none;
      overflow-y: auto;
      transition: border-color 0.2s ease;

      &:focus {
        outline: none;
        border-color: rgba(205, 133, 63, 0.8);
      }

      &::placeholder {
        color: rgba(240, 230, 210, 0.4);
      }
    }

    .json-error {
      margin-top: 12px;
      padding: 12px;
      background: rgba(239, 68, 68, 0.1);
      border: 1px solid rgba(239, 68, 68, 0.5);
      border-radius: 6px;
      color: #fca5a5;
      font-size: 13px;
      line-height: 1.5;
    }
  }

  .json-editor-footer {
    display: flex;
    justify-content: flex-end;
    gap: 12px;
    padding: 16px 24px;
    border-top: 2px solid rgba(205, 133, 63, 0.4);

    .json-btn {
      padding: 10px 20px;
      border-radius: 8px;
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.2s ease;
      border: 2px solid transparent;

      &.json-btn-cancel {
        background: rgba(107, 114, 128, 0.2);
        color: #d1d5db;
        border-color: rgba(107, 114, 128, 0.4);

        &:hover {
          background: rgba(107, 114, 128, 0.3);
          border-color: rgba(107, 114, 128, 0.6);
        }
      }

      &.json-btn-save {
        background: linear-gradient(135deg, rgba(34, 197, 94, 0.8), rgba(22, 163, 74, 0.9));
        color: #dcfce7;
        border-color: rgba(34, 197, 94, 0.6);

        &:hover:not(:disabled) {
          background: linear-gradient(135deg, rgba(34, 197, 94, 0.9), rgba(22, 163, 74, 1));
          border-color: rgba(34, 197, 94, 0.8);
          transform: translateY(-1px);
        }

        &:disabled {
          opacity: 0.6;
          cursor: not-allowed;
        }
      }
    }
  }
}

@keyframes slideIn {
  from {
    transform: translateY(-20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}
</style>
