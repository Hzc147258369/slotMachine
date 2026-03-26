<!--
 * 水果机游戏页面
 * 功能说明：实现水果机游戏的核心逻辑，包括押注、开奖、分数计算、闪灯效果等
 * 同时在数据变换时使用节流函数，避免频繁触发导致性能问题
 * 第三方库：使用 CryptoJS 进行分数加密存储并使用 AES 加密算法
 * CryptoJS 开源协议：MIT License
 * 论文提交专用加密密钥：fruitMachine2026
-->
<template>
  <view class="container">
    <!-- 分数显示 -->
    <view class="score" style="position: absolute; top: 120rpx; left: 480rpx">
      {{ score }}<span @click="scoreAdd" style="font-size: 24rpx">+</span>
    </view>
    <!-- 遮罩层 -->
    <view v-if="flash" class="flash-container">
      <view
        v-for="index in 24"
        :key="index"
        class="flash-item"
        :class="{ 'flash-active': StrobeCalculation(index - 1) }"
        :style="{
          top: topCalculate(index) + 'rpx',
          left: leftCalculate(index) + 'rpx',
        }"
      ></view>
    </view>
    <view
      v-if="num !== -1"
      class="current-item"
      :style="{
        top: topCalculate((num % 24) + 1) + 'rpx',
        left: leftCalculate((num % 24) + 1) + 'rpx',
      }"
    >
    </view>
    <view
      v-for="(item, index) in array[0].top"
      :key="index"
      class="item"
      :class="['item-top', `item-top-${index}`]"
    >
      <image
        :src="`/static/image/${item.name}${item.double ? 'big' : ''}.png`"
        class="item-image"
      />
    </view>
    <!-- 右层10-18 -->
    <view
      v-for="(item, index) in array[0].right"
      :key="index"
      class="item"
      :class="['item-right', `item-right-${index}`]"
    >
      <image
        :src="`/static/image/${item.name}${item.double ? 'big' : ''}.png`"
        class="item-image"
      />
    </view>
    <!-- 底层19-27 -->
    <view
      v-for="(item, index) in array[0].bottom"
      :key="index"
      class="item"
      :class="['item-bottom', `item-bottom-${index}`]"
    >
      <image
        :src="`/static/image/${item.name}${item.double ? 'big' : ''}.png`"
        style="
          position: absolute;
          top: 0;
          left: 0;
          width: 103rpx;
          height: 103rpx;
        "
      />
    </view>
    <!-- 左层28-36 -->
    <view
      v-for="(item, index) in array[0].left"
      :key="index"
      class="item"
      :class="['item-left', `item-left-${index}`]"
    >
      <image
        :src="`/static/image/${item.name}${item.double ? 'big' : ''}.png`"
        style="
          position: absolute;
          top: 0;
          left: 0;
          width: 103rpx;
          height: 103rpx;
        "
      />
    </view>
    <!-- 按钮 -->
    <view class="btn" @click="click(24, 1)"
      >开始
      <!-- 先去除原点击事件 -->
      <span @click.stop="ratenumAdd">x{{ rate[ratenum] }}</span></view
    >
    <!-- 分数显示 -->
    <view class="bet-item-top">
      <view v-for="(item, index) in bet" :key="index" class="bet-top-item">
        <image :src="`/static/image/${item.name}.png`" class="bet-top-image" />
      </view>
      <view v-if="betNum" class="bet-numbers">
        <view v-for="(item, index) in bet" :key="index" class="bet-number-item">
          {{ item.number }}
        </view>
      </view>
      <view v-else class="bet-numbers">
        <view
          v-for="(item, index) in result.bet"
          :key="index"
          class="bet-number-item"
        >
          {{ item.number }}
        </view>
      </view>
    </view>
    <view class="bet-item-bottom">
      <view v-for="(item, index) in bet" :key="index" class="bet-bottom-item">
        <image
          :src="`/static/image/${item.name}.png`"
          class="bet-bottom-image"
          @click="buttonClick(item)"
        />
      </view>
      <!-- 测试数据位置 -->
    </view>
  </view>
  <!-- 输入框 -->
</template>
<script setup>
import { watch, onMounted, ref } from "vue";
import CryptoJS from "crypto-js";
import { onUnload, onLoad } from "@dcloudio/uni-app";

/**
 * 押注数量数组
 * 每个元素包含物品名称和押注数量
 */
const bet = ref([
  { name: "king", number: 0 }, // 王
  { name: "doubleSeven", number: 0 }, // 双七
  { name: "doubleStar", number: 0 }, // 双星
  { name: "watermelon", number: 0 }, // 西瓜
  { name: "tangerine", number: 0 }, // 橘子
  { name: "bell", number: 0 }, // 铃铛
  { name: "mango", number: 0 }, // 芒果
  { name: "apple", number: 0 }, // 苹果
]);

/**
 * 定时器管理数组
 * 用于存储所有创建的定时器，方便统一清理
 */
let timers = [];

/**
 * 闪灯索引
 * 用于控制错位闪灯效果
 */
let flashLightindex = ref(0);

/**
 * 闪灯状态
 * true: 闪灯开启, false: 闪灯关闭
 */
let flash = ref(false);

/**
 * 倍率索引
 * 用于控制当前选中的倍率
 */
let ratenum = ref(0);

/**
 * 倍率数组
 * 包含可选的倍率值：1倍、10倍、100倍
 */
let rate = ref([1, 10, 100]);

/**
 * 开火车模式的步骤数组
 * 用于控制开火车模式的中奖顺序
 */
let train = ref([1, 2, 3, 4, 5, 6, 7]);

/**
 * 投注号码状态
 * true: 使用当前押注, false: 使用上局记录的押注
 */
let betNum = ref(true);

/**
 * 当前中奖号码
 * -1: 未中奖, 其他值: 中奖号码
 */
let num = ref(-1);

/**
 * 记录加分次数和时间
 * Count: 今日加分次数, Time: 加分日期
 */
let scoreAddarr = ref({ Count: 0, Time: 0 });
/**
 * 分数增加函数
 * 功能：给玩家增加分数，并限制每日加分次数
 * 逻辑：
 * 1. 从本地存储获取加分记录
 * 2. 如果不是今天的日期，重置加分次数
 * 3. 检查是否达到每日加分上限（10次）
 * 4. 增加分数和加分次数
 * 5. 保存加分记录到本地存储
 */
const scoreAdd = function () {
  let scoreAddarrStr = uni.getStorageSync("scoreAddarr");

  // 如果不是今天的日期，重置加分次数
  if (scoreAddarrStr) {
    scoreAddarr.value = JSON.parse(scoreAddarrStr);

    // 检查日期是否为今天
    if (scoreAddarr.value.Time !== new Date().toLocaleDateString()) {
      scoreAddarr.value.Count = 0;
      uni.setStorageSync("scoreAddarr", JSON.stringify(scoreAddarr.value));
      scoreAddarrStr = uni.getStorageSync("scoreAddarr");
    }

    scoreAddarr.value = JSON.parse(scoreAddarrStr);

    // 检查是否达到每日加分上限
    if (scoreAddarr.value.Count >= 10) {
      uni.showToast({
        title: "您今日已达加分上限",
        icon: "none",
      });
      return;
    }
  }

  // 标记分数可以保存
  scoreStraa.value = true;
  // 增加加分次数
  scoreAddarr.value.Count++;
  // 增加分数
  score.value += 100;
  // 只记录日期，不记录时间
  scoreAddarr.value.Time = new Date().toLocaleDateString();
  // 保存加分记录到本地存储
  uni.setStorageSync("scoreAddarr", JSON.stringify(scoreAddarr.value));
};
// watch(scoreAddarr, (newVal, oldVal) => {
//   // 储存到本地
// });
/**
 * 闪灯类型
 * 1: 错位闪灯, 2: 频闪, 3: 从1开始全亮, 4: 特殊闪灯
 */
let isStroboscopicaaa = ref(2);

/**
 * 闪灯显示逻辑函数
 * 功能：根据闪灯类型和索引计算闪灯状态
 * @param {number} index - 闪灯索引
 * @returns {boolean} 闪灯是否激活
 */
const StrobeCalculation = function (index) {
  // 错位闪灯模式
  if (isStroboscopicaaa.value === 1) {
    return (index + +flashLightindex.value) % 2 === 0;
  }

  // 频闪模式
  if (isStroboscopicaaa.value === 2) {
    return isStroboscopic.value;
  }

  // 从1开始全亮模式
  if (isStroboscopicaaa.value === 3) {
    return isNum3.value ? isNum.value > index : isNum.value < index;
  }

  // 特殊闪灯模式
  if (isStroboscopicaaa.value === 4) {
    // 当负的那边到3时停止，正的那边到24时从123开始
    return (
      (Math.abs(index - 15) <= isNum4.value && index >= 3) ||
      (isNum4.value - 8 > 0 ? index < isNum4.value - 8 : false)
    );
  }
};
/**
 * 倍率增加函数
 * 功能：循环切换选中的倍率
 * 逻辑：
 * 1. 如果当前倍率不是最大的，增加倍率索引
 * 2. 如果当前倍率是最大的，重置为0（回到最小倍率）
 */
const ratenumAdd = function () {
  if (ratenum.value < rate.value.length - 1) {
    ratenum.value++;
  } else {
    ratenum.value = 0;
  }
};
/**
 * 频闪闪灯状态
 * true: 频闪开启, false: 频闪关闭
 */
let isStroboscopic = ref(false);

/**
 * 3号闪灯的当前数字
 * 用于控制从1开始全亮的效果
 */
let isNum = ref(0);

/**
 * 3号闪灯的方向
 * true: 递增, false: 递减
 */
let isNum3 = ref(true);

/**
 * 4号闪灯的当前值
 * 用于控制特殊闪灯的效果
 */
let isNum4 = ref(0);

/**
 * 定时器管理函数
 * 功能：创建定时器并添加到管理数组
 * @param {string} type - 定时器类型："interval"或"timeout"
 * @param {function} func - 定时器回调函数
 * @param {number} delay - 定时器延迟时间（毫秒）
 * @returns {number} 定时器ID
 */
const setTimer = (type, func, delay) => {
  let timer;
  if (type === "interval") {
    timer = setInterval(func, delay);
  } else {
    timer = setTimeout(func, delay);
  }
  timers.push(timer);
  return timer;
};

/**
 * 清理所有定时器
 * 功能：清除所有创建的定时器，避免内存泄漏
 */
const clearTimers = () => {
  timers.forEach((timer) => {
    clearTimeout(timer);
    clearInterval(timer);
  });
  timers = [];
};

/**
 * 控制闪灯效果
 * @param {number} flashType - 闪灯类型：1-错位闪灯, 2-频闪, 3-从1开始全亮, 4-特殊闪灯
 */
let flashLightFrequency = ref();
let lastFlashTime = 0;
const flashLight = function (flashType) {
  isStroboscopicaaa.value = flashType;
  if (flashType === 1) {
    flashLightFrequency.value = 400;
  }
  if (flashType === 2) {
    flashLightFrequency.value = 200;
  }
  if (flashType === 3) {
    flashLightFrequency.value = 20;
  }
  if (flashType === 4) {
    flashLightFrequency.value = 50;
  }

  const flashLightRecursive = function (timestamp) {
    if (!flash.value) {
      clearTimers();
      return;
    }

    // 控制动画帧率
    if (
      !lastFlashTime ||
      timestamp - lastFlashTime >= flashLightFrequency.value
    ) {
      if (flashType === 1) {
        if (flashLightindex.value === 0) {
          flashLightindex.value = 1;
        } else {
          flashLightindex.value = 0;
        }
      }
      if (flashType === 2) {
        isStroboscopic.value = !isStroboscopic.value;
      }
      if (flashType === 3) {
        if (isNum3.value) {
          isNum.value++;
        } else {
          isNum.value--;
        }
        if (isNum.value === 24) {
          isNum3.value = false;
        } else if (isNum.value === 0) {
          isNum3.value = true;
        }
      }
      if (flashType === 4) {
        if (isNum4.value === 12) {
          isNum4.value = 1;
        } else {
          isNum4.value++;
        }
      }
      lastFlashTime = timestamp;
    }

    // 使用 requestAnimationFrame 代替 setInterval
    requestAnimationFrame(flashLightRecursive);
  };

  // 开始动画
  requestAnimationFrame(flashLightRecursive);
};
flash.value = true;
// 错位闪灯
// flashLight(1);
// 频闪
// flashLight(2);
// 从1开始全亮
// flashLight(3);
// 4号
// flashLight(4);

// 三秒后关闭
setTimeout(() => {
  flash.value = false;
}, 1000);
// 橙子  tangerine
// 铃铛  bell
// 国王  king
// 苹果  apple
// 芒果  mango
// 西瓜  watermelon
// 特殊  special
// 七倍  doubleSeven
// 星倍  doubleStar

// 用于押注
const buttonClick = function (item) {
  scoreStraa.value = true;
  if (score.value < 10 * rate.value[ratenum.value]) {
    uni.showToast({
      title: "余额不足",
      icon: "none",
    });
    return;
  }
  if (start.value) {
    uni.showToast({
      title: "还未结束",
      icon: "none",
    });
    return;
  }
  // 下注数不能超过5000
  if (item.number + Number(rate.value[ratenum.value]) > 1000) {
    uni.showToast({
      title: `下注数不能超过1000!!!`,
      icon: "none",
    });
    // distance;
    return;
  }
  // 清空结果
  result.value.bet.forEach((item) => {
    item.number = 0;
  });
  betNum.value = true;
  // 转为数字
  item.number = Number(item.number) + Number(rate.value[ratenum.value]);
  score.value -= Number(rate.value[ratenum.value]) * 10;
};
/**
 * 计算当前押注号码的左侧位置
 * 功能：根据号码计算其在水果机上的水平位置
 * @param {number} num - 押注号码
 * @returns {number} 左侧位置（rpx）
 */
const leftCalculate = function (num) {
  // 顶部一行（1-6）
  if (num > 0 && num < 7) {
    return num * 106 - 106;
  }
  // 右侧一列（7-12）
  else if (num > 6 && num < 13) {
    return 6 * 106;
  }
  // 底部一行（13-19）
  else if (num > 12 && num < 20) {
    return 7 * 106 - (num % 12) * 106;
  }
  // 左侧一列（20-24）
  else if (num > 18 && num < 27) {
    return 0;
  }
};

/**
 * 计算当前押注号码的顶部位置
 * 功能：根据号码计算其在水果机上的垂直位置
 * @param {number} num - 押注号码
 * @returns {number} 顶部位置（rpx）
 */
const topCalculate = function (num) {
  // 顶部一行（1-6）
  if (num > 0 && num < 7) {
    return 0;
  }
  // 右侧一列（7-12）
  else if (num > 6 && num < 13) {
    return (num % 7) * 106;
  }
  // 底部一行（13-19）
  else if (num > 12 && num < 19) {
    return 106 * 6;
  }
  // 左侧一列（20-24）
  else if (num > 18 && num < 27) {
    return 106 * 7 - (num % 18) * 106;
  }
};
/**
 * 存储每个号码的信息
 * 结构：
 * - top: 顶部一行号码信息
 * - right: 右侧一列号码信息
 * - bottom: 底部一行号码信息
 * - left: 左侧一列号码信息
 * 每个号码的属性：
 * - number: 号码
 * - name: 物品名称
 * - magnification: 倍率
 * - double: 额外倍数
 */
const array = ref([
  {
    top: [
      { number: 1, name: "tangerine", magnification: 2, double: 2 },
      { number: 2, name: "bell", magnification: 2, double: 2 },
      { number: 3, name: "king", magnification: 30, double: 0 },
      { number: 4, name: "king", magnification: 100, double: 0 },
      { number: 5, name: "king", magnification: 50, double: 0 },
      { number: 6, name: "apple", magnification: 5, double: 0 },
    ],
    right: [
      { number: 7, name: "mango", magnification: 2, double: 2 },
      { number: 8, name: "watermelon", magnification: 4, double: 4 },
      { number: 9, name: "watermelon", magnification: 4, double: 0 },
      { number: 10, name: "special", magnification: "special" },
      { number: 11, name: "apple", magnification: 5, double: 0 },
      { number: 12, name: "tangerine", magnification: 2, double: 0 },
    ],
    bottom: [
      { number: 13, name: "tangerine", magnification: 2, double: 2 },
      { number: 14, name: "bell", magnification: 2, double: 2 },
      { number: 15, name: "doubleSeven", magnification: 4, double: 0 },
      { number: 16, name: "fullCenter", magnification: "All hit", double: 0 },
      { number: 17, name: "doubleSeven", magnification: 4, double: 8 },
      { number: 18, name: "mango", magnification: 2, double: 0 },
    ],
    left: [
      { number: 19, name: "mango", magnification: 2, double: 2 },
      { number: 20, name: "doubleStar", magnification: 4, double: 4 },
      { number: 21, name: "doubleStar", magnification: 4, double: 0 },
      { number: 22, name: "special", magnification: "special", double: 0 },
      { number: 23, name: "apple", magnification: 5, double: 0 },
      { number: 24, name: "bell", magnification: 2, double: 0 },
    ],
  },
]);
/**
 * 存储上次的押注数量
 * 用于在下次直接开始时使用
 * 结构：
 * - bet: 上次的押注数量数组
 * - num: 上次的中奖号码
 */
let result = ref({
  bet: [
    { name: "king", number: 0 }, // 王
    { name: "doubleSeven", number: 0 }, // 双七
    { name: "doubleStar", number: 0 }, // 双星
    { name: "watermelon", number: 0 }, // 西瓜
    { name: "tangerine", number: 0 }, // 橘子
    { name: "bell", number: 0 }, // 铃铛
    { name: "mango", number: 0 }, // 芒果
    { name: "apple", number: 0 }, // 苹果
  ],
});

/**
 * 玩家分数
 * 初始值为100
 */
let score = ref(100);

/**
 * 水果机滚动速度
 * 初始值为200毫秒
 */
let speed = ref(200);
/**
 * 开始游戏运行逻辑
 * 控制水果机的滚动效果
 */
const startGame = function () {
  const gameRecursive = function () {
    if (num.value + 1 === stopNum.value) {
      start.value = false;
      // 关闭所有定时器
      clearTimers();
      isSpeedIncreasing.value = true;
      speed.value = 200;
      scoreCalculate();
      return;
    }
    num.value++;
    setTimer("timeout", gameRecursive, speed.value);
  };
  clearTimers();
  gameRecursive();
};
/**
 * 控制速度变化的状态
 * true: 速度增加中, false: 速度减少中
 */
let isSpeedIncreasing = ref(true);
// 停止位置(范围120-144)
let stopNum = ref(135);
/**
 * 控制速度变化的函数
 * 实现水果机滚动速度的加速和减速效果
 */
const controlSpeed = function () {
  const speedRecursive = function () {
    if (num.value + 10 >= stopNum.value) {
      return;
    }
    if (isSpeedIncreasing.value) {
      speed.value -= 15;
    } else {
      speed.value = Number(speed.value) + 15;
    }
    if (speed.value <= 15) {
      isSpeedIncreasing.value = false;
    }
    clearTimers();
    startGame();
    setTimer("timeout", speedRecursive, 200);
  };
  speedRecursive();
};
/**
 * 计算分数函数
 * 功能：根据中奖结果计算玩家得分
 * 逻辑：
 * 1. 深拷贝押注数据，避免引用关联
 * 2. 记录押注数量
 * 3. 根据中奖号码筛选对应的物品信息
 * 4. 根据物品类型和押注情况计算得分
 * 5. 处理特殊情况（全中、特殊符号）
 */
const scoreCalculate = function () {
  // 深拷贝 bet.value，避免引用关联
  const betCopy = JSON.parse(JSON.stringify(bet.value));
  // 记录押注数量
  result.value.bet = betCopy;
  result.value.num = (num.value % 24) + 1;
  betNum.value = false;
  // 筛选出当前号码的信息
  let filterNum = null;
  // 筛选出当前顶部号码的信息
  if ((num.value % 24) + 1 > 0 && (num.value % 24) + 1 < 7) {
    filterNum = array.value[0].top.find(
      (item) => item.number === (num.value % 24) + 1,
    );
  } else if (
    // 筛选出当前右侧号码的信息
    (num.value % 24) + 1 > 6 &&
    (num.value % 24) + 1 < 13 &&
    (num.value % 24) + 1 !== 10
  ) {
    filterNum = array.value[0].right.find(
      (item) => item.number === (num.value % 24) + 1,
    );
  } else if (
    // 筛选出当前底部号码的信息
    (num.value % 24) + 1 > 12 &&
    (num.value % 24) + 1 < 19 &&
    (num.value % 24) + 1 !== 16
  ) {
    filterNum = array.value[0].bottom.find(
      (item) => item.number === (num.value % 24) + 1,
    );
  } else if (
    // 筛选出当前左侧号码的信息
    (num.value % 24) + 1 > 18 &&
    (num.value % 24) + 1 < 25 &&
    (num.value % 24) + 1 !== 22
  ) {
    if ((num.value % 24) + 1 === 0) {
      filterNum = array.value[0].left[5];
    } else {
      filterNum = array.value[0].left.find(
        (item) => item.number === (num.value % 24) + 1,
      );
    }
  } else if ((num.value % 24) + 1 === 16) {
    // 筛选出当前全中的信息
    // 开灯
    flash.value = true;
    flashLight(4);
    // 全中加分 - 优化为统一的分数增长动画
    scoreStraa.value = false;

    // 计算总加分
    const totalAddScore = bet.value.reduce((sum, item) => {
      return sum + item.number * 4 * 10;
    }, 0);

    if (totalAddScore > 0) {
      const duration = 3000; // 动画持续时间
      const startTime = Date.now();
      let addedScore = 0;

      // 使用requestAnimationFrame实现平滑动画
      const updateScore = () => {
        const elapsed = Date.now() - startTime;
        const progress = Math.min(elapsed / duration, 1);

        // 计算当前应加的分数
        const currentScore = Math.floor((progress * totalAddScore) / 10) * 10;
        const scoreToAdd = currentScore - addedScore;

        if (scoreToAdd > 0) {
          score.value += scoreToAdd;
          addedScore = currentScore;
        }

        // 更新scoreStraa状态
        if (progress >= 1) {
          scoreStraa.value = true;
        } else {
          scoreStraa.value = false;
        }

        // 继续动画直到完成
        if (progress < 1) {
          requestAnimationFrame(updateScore);
        }
      };

      requestAnimationFrame(updateScore);
    } else {
      scoreStraa.value = true;
    }
    // 三秒后关闭闪烁
    setTimeout(() => {
      flash.value = false;
    }, 3000);
    // 清除当前押注数量
    for (let i = 0; i < bet.value.length; i++) {
      bet.value[i].number = 0;
    }
    return;
  } else if ((num.value % 24) + 1 === 22 || (num.value % 24) + 1 === 10) {
    // 筛选出当前特殊中奖的信息
    // 开灯
    flash.value = true;
    flashLight(3);
    // 确保闪烁之后再执行关闭闪烁
    setTimeout(() => {
      flash.value = false;
    }, 3000);
    // 简化随机逻辑，使用更清晰的概率分布
    const randomNum = Math.random();

    // 80%概率：被小鸟叼走
    if (randomNum < 0.8) {
      // 提示被小鸟叼走了
      uni.showToast({
        title: "被小鸟叼走了",
        icon: "none",
      });
      // 清除当前押注数量
      bet.value.forEach((item) => {
        item.number = 0;
      });
      return;
    }
    // 5%概率：转为大王(中)
    else if (randomNum < 0.85) {
      filterNum = array.value[0].top.find((item) => item.number === 3);
      num.value = 2;
      // 使用统一的定时器管理
      setTimer(
        "timeout",
        () => {
          flash.value = true;
          flashLight(2);
        },
        3000,
      );
      setTimer(
        "timeout",
        () => {
          flash.value = false;
        },
        6000,
      );
    }
    // 15%概率：转为全中
    else {
      filterNum = array.value[0].top.find((item) => item.number === 4);
      num.value = 3;
      // 使用统一的定时器管理
      setTimer(
        "timeout",
        () => {
          flash.value = true;
          flashLight(4);
        },
        3000,
      );
      setTimer(
        "timeout",
        () => {
          flash.value = false;
        },
        6000,
      );
    }
  }
  // 根据filterNum的magnification判断是否加分bet;
  for (let i = 0; i < bet.value.length; i++) {
    if (bet.value[i].name === filterNum.name) {
      // 大王特殊处理：总是闪灯
      const isKing = filterNum.name === "king";

      // 闪灯逻辑
      if (isKing || Math.random() < 0.2) {
        // 强制打开闪灯
        flash.value = true;
        flashLight(2); // 使用频闪效果，更醒目

        // 计算加分
        const multiplier = isKing ? 80 : Math.random() < 0.2 ? 80 : 10;
        let totalAddScore =
          bet.value[i].number *
          (filterNum.magnification + filterNum.double) *
          multiplier;

        if (totalAddScore > 0) {
          const duration = 3000; // 动画持续时间
          const startTime = Date.now();
          let addedScore = 0;

          // 使用requestAnimationFrame实现平滑动画
          const updateScore = () => {
            const elapsed = Date.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);

            // 计算当前应加的分数
            const currentScore =
              Math.floor((progress * totalAddScore) / 10) * 10;
            const scoreToAdd = currentScore - addedScore;

            if (scoreToAdd > 0) {
              score.value += scoreToAdd;
              addedScore = currentScore;
            }

            // 更新scoreStraa状态
            if (progress >= 1) {
              scoreStraa.value = true;
            } else {
              scoreStraa.value = false;
            }

            // 继续动画直到完成
            if (progress < 1) {
              requestAnimationFrame(updateScore);
            }
          };

          requestAnimationFrame(updateScore);
        } else {
          scoreStraa.value = true;
        }

        // 三秒后关闭闪烁
        setTimer(
          "timeout",
          () => {
            flash.value = false;
          },
          3000,
        );
      } else {
        let totalAddScore =
          bet.value[i].number *
          (filterNum.magnification + filterNum.double) *
          10;

        if (totalAddScore > 0) {
          const duration = 3000; // 动画持续时间
          const startTime = Date.now();
          let addedScore = 0;
          // 使用requestAnimationFrame实现平滑动画
          const updateScore = () => {
            const elapsed = Date.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);

            // 计算当前应加的分数
            const currentScore =
              Math.floor((progress * totalAddScore) / 10) * 10;
            const scoreToAdd = currentScore - addedScore;

            if (scoreToAdd > 0) {
              score.value += scoreToAdd;
              addedScore = currentScore;
            }

            // 更新scoreStraa状态
            if (progress >= 1) {
              scoreStraa.value = true;
            } else {
              scoreStraa.value = false;
            }

            // 继续动画直到完成
            if (progress < 1) {
              requestAnimationFrame(updateScore);
            }
          };

          requestAnimationFrame(updateScore);
        } else {
          scoreStraa.value = true;
        }
      }
    }
  }

  for (let i = 0; i < bet.value.length; i++) {
    bet.value[i].number = 0;
  }
};
/**
 * 游戏开始状态
 * true: 游戏进行中, false: 游戏未开始
 */
let start = ref(false);
/**
 * 点击开始按钮的处理函数
 * @param {number} M - 随机数范围的最大值
 * @param {number} N - 随机数范围的最小值
 */
const click = (M, N) => {
  // 10%的概率进入开火车模式（暂未实现）
  // if (Math.floor(Math.random() * 100) < 10) {
  //   // 开火车模式：连续中奖
  //   // 实现逻辑：连续生成多个中奖结果，每次中奖后短暂停留
  // }
  if (start.value) {
    uni.showToast({
      title: "游戏还未结束",
      icon: "none",
    });
    return;
  }
  // 不能所有的押注都为0

  // 当原押注数全部为空时则以上局记录结果开始
  if (!betNum.value) {
    // 将记录的押注数量赋值给当前押注数量
    bet.value = result.value.bet.map((item) => ({
      ...item,
      number: item.number,
    }));
    // 从分数扣减当前押注数量
    if (
      score.value <
      bet.value.reduce((acc, cur) => acc + cur.number, 0) * 10
    ) {
      // 清理所有押注
      for (let i = 0; i < bet.value.length; i++) {
        bet.value[i].number = 0;
      }
      // 清理所有押注记录
      result.value.bet = bet.value.map((item) => ({
        ...item,
        number: 0,
      }));
      return uni.showToast({
        title: "余额不足",
        icon: "none",
      });
    }
    score.value -= bet.value.reduce((acc, cur) => acc + cur.number, 0) * 10;
  }
  if (bet.value.every((item) => item.number === 0)) {
    uni.showToast({
      title: "请先押注",
      icon: "none",
    });
    return -1;
  }
  start.value = true;
  // 直接获取结果数字，而不是数组
  const resultNum = generateProbabilityArray();
  console.log(resultNum);
  stopNum.value = 120 + resultNum;
  num.value = Math.floor(Math.random() * (M - N + 1)) + N;
  startGame();
  controlSpeed();
};
// 监听所有下注中的变化
/**
 * 基础概率数组
 * 说明：每个对象包含一组数字和对应的概率值
 * 概率值越大，出现的可能性越高
 */
const baseProbabilityArray = [
  // 小三元*2（橘子、铃铛、芒果）
  { num: [1, 2, 7, 13, 14, 19], probability: 200 },
  // 小三元（橘子、铃铛、芒果）
  { num: [12, 18, 24], probability: 300 },
  // 大三元*2（西瓜、双七、双星）
  { num: [8, 17, 20], probability: 100 },
  // 大三元（西瓜、双七、双星）
  { num: [9, 15, 21], probability: 200 },
  // 苹果
  { num: [6, 11, 23], probability: 100 },
  // 王（高倍率）
  { num: [3, 4, 5], probability: 20 },
  // 全中
  { num: [16], probability: 40 },
  // 其他（特殊符号）
  { num: [10, 22], probability: 40 },
];

/**
 * 生成概率数组并计算中奖结果
 * 算法流程：
 * 1. 根据押注情况调整各项的概率
 * 2. 计算总概率
 * 3. 生成随机数并根据概率分布确定中奖结果
 * 返回值： {number} 中奖数字
 */
const generateProbabilityArray = () => {
  // 复制基础概率数组，避免修改原始数据
  const probabilityArray = JSON.parse(JSON.stringify(baseProbabilityArray));

  // 王的概率调整
  probabilityArray[5].probability =
    probabilityArray[5].probability -
    probabilityArray[5].probability * (bet.value[0].number / 1000);

  // 全中的概率调整
  const allBetNumber = bet.value.reduce((acc, cur) => acc + cur.number, 0);
  probabilityArray[6].probability =
    probabilityArray[6].probability -
    probabilityArray[6].probability * (allBetNumber / 10000);

  // 大三元押注总数
  const bigBetNumber = bet.value
    .filter(
      (item) =>
        item.name === "doubleStar" ||
        item.name === "doubleSeven" ||
        item.name === "watermelon",
    )
    .reduce((acc, cur) => acc + cur.number, 0);

  // 大三元*2的概率调整
  probabilityArray[2].probability =
    probabilityArray[2].probability -
    probabilityArray[2].probability * (bigBetNumber / 10000);

  // 计算总概率
  const totalProbability = probabilityArray.reduce(
    (sum, item) => sum + item.probability,
    0,
  );

  // 生成随机数
  let random = Math.random() * totalProbability;
  // 确定随机结果
  let cumulativeProbability = 0;
  for (const { num, probability } of probabilityArray) {
    cumulativeProbability += probability;
    if (random <= cumulativeProbability) {
      // 从该组中随机选择一个数字
      return num[Math.floor(Math.random() * num.length)];
    }
  }

  // 默认返回第一个组的第一个数字
  return probabilityArray[0].num[0];
};
// 关闭页面时清除定时器
onUnload(() => {
  clearTimers();
});
/**
 * 页面加载完毕时从本地存储获取分数
 * 解密方式：使用AES解密，然后将字母转换为对应的数字
 * 延时执行，确保DOM渲染完成
 */
setTimeout(() => {
  try {
    const storedScore = uni.getStorageSync("score");
    if (!storedScore) {
      // 如果本地存储为空，使用默认值
      score.value = 100;
      return;
    }
    // 使用论文提交专用密钥进行AES解密
    const bytes = CryptoJS.AES.decrypt(storedScore, "fruitMachine2026");
    const originalText = bytes.toString(CryptoJS.enc.Utf8);
    // 根据字母表将加密后的分数转换为数字
    let scoreStr = "";

    for (let i = 0; i < originalText.length; i++) {
      const digit = alphabet.indexOf(originalText[i]);
      if (digit !== -1) {
        scoreStr += digit.toString();
      }
    }
    // 验证数据格式是否正确（必须是数字且非负）
    if (typeof +scoreStr === "number" && +scoreStr >= 0) {
      score.value = +scoreStr;
    } else {
      // 数据格式不正确，清理本地存储并使用默认值
      uni.removeStorageSync("score");
      score.value = 100;
      // 可以添加提示信息
      uni.showToast({
        title: "数据格式有误，已重置分数",
        icon: "none",
      });
    }
  } catch (error) {
    // 读取过程中发生错误，清理本地存储并使用默认值
    uni.removeStorageSync("score");
    score.value = 100;
    console.error("读取本地存储失败：", error);
  }
}, 100);

// 通用的本地存储清理函数
defineExpose({
  clearLocalStorage: () => {
    try {
      // 清理所有游戏相关的本地存储
      uni.removeStorageSync("score");
      // 如果有其他游戏数据，也可以在这里清理
      // uni.removeStorageSync("otherGameData");
      uni.showToast({
        title: "本地数据已清理",
        icon: "success",
      });
    } catch (error) {
      console.error("清理本地存储失败：", error);
      uni.showToast({
        title: "清理失败",
        icon: "error",
      });
    }
  },
});
// 先随机对应英文字母表中每个字母的位置
const alphabet = ["f", "p", "j", "n", "b", "y", "m", "a", "s", "q"];
/**
 * 分数保存状态
 * true: 分数可以保存, false: 分数正在变化中，暂不保存
 * 用于控制分数变化动画期间不保存分数，避免频繁操作本地存储
 */
let scoreStraa = ref(false);

/**
 * 监听分数变化，自动保存到本地存储
 * 加密方式：将原分数数字类型转换为字符串后，将每个数字改为其对应的字母，然后使用AES加密
 */
watch(score, (newScore) => {
  if (!scoreStraa.value) {
    return;
  }
  let encryptedScore = "";
  // 转为字符串
  const scoreStr = newScore.toString();
  for (let i = 0; i < scoreStr.length; i++) {
    encryptedScore += alphabet[scoreStr[i]];
  }
  // 使用论文提交专用密钥进行AES加密
  const ciphertext = CryptoJS.AES.encrypt(
    encryptedScore,
    "fruitMachine2026",
  ).toString();
  // 储存加密后的分数
  uni.setStorageSync("score", ciphertext);
});
</script>
<style scoped>
.container {
  position: relative;
  top: 100rpx;
  width: 746rpx;
  height: 746rpx;
  border: 1rpx solid rgb(220, 235, 255);
  box-sizing: border-box;
  margin: 0 auto;
}

/* Score Display */
.score {
  position: absolute;
  top: 120rpx;
  left: 480rpx;
}

.score span {
  font-size: 24rpx;
}

/* Flash Effect */
.flash-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10000;
}

.flash-item {
  position: absolute;
  width: 106rpx;
  height: 106rpx;
  background-color: transparent;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border-radius: 8rpx;
  overflow: hidden;
}

.flash-active {
  background-color: hsla(0, 0%, 100%, 0.6);
  box-shadow:
    0 0 20rpx rgba(255, 255, 255, 0.8),
    inset 0 0 20rpx rgba(255, 255, 255, 0.6),
    0 0 40rpx rgba(255, 215, 0, 0.5);
  border: 2rpx solid rgba(255, 255, 255, 0.9);
  transform: scale(1.05);
  animation: pulse 0.6s ease-in-out;
}

/* 脉冲动画 */
@keyframes pulse {
  0% {
    transform: scale(1);
    box-shadow:
      0 0 10rpx rgba(255, 255, 255, 0.5),
      inset 0 0 10rpx rgba(255, 255, 255, 0.3),
      0 0 20rpx rgba(255, 215, 0, 0.3);
  }
  50% {
    transform: scale(1.1);
    box-shadow:
      0 0 30rpx rgba(255, 255, 255, 1),
      inset 0 0 30rpx rgba(255, 255, 255, 0.8),
      0 0 60rpx rgba(255, 215, 0, 0.7);
  }
  100% {
    transform: scale(1.05);
    box-shadow:
      0 0 20rpx rgba(255, 255, 255, 0.8),
      inset 0 0 20rpx rgba(255, 255, 255, 0.6),
      0 0 40rpx rgba(255, 215, 0, 0.5);
  }
}

/* Current Item */
.current-item {
  position: absolute;
  width: 106rpx;
  height: 106rpx;
  background-color: hsla(50, 100%, 80%, 0.7);
  z-index: 1000;
  border-radius: 8rpx;
  box-shadow:
    0 0 25rpx rgba(255, 215, 0, 0.9),
    inset 0 0 15rpx rgba(255, 215, 0, 0.5);
  border: 3rpx solid rgba(255, 215, 0, 1);
  transform: scale(1.08);
  animation: currentItemPulse 0.8s ease-in-out infinite alternate;
}

/* 当前选中项脉冲动画 */
@keyframes currentItemPulse {
  from {
    box-shadow:
      0 0 20rpx rgba(255, 215, 0, 0.7),
      inset 0 0 12rpx rgba(255, 215, 0, 0.4);
    transform: scale(1.05);
  }
  to {
    box-shadow:
      0 0 30rpx rgba(255, 215, 0, 1),
      inset 0 0 20rpx rgba(255, 215, 0, 0.6);
    transform: scale(1.1);
  }
}

/* Game Items */
.item {
  position: absolute;
  width: 106rpx;
  height: 106rpx;
  border: 1rpx solid rgb(133, 131, 255);
  box-sizing: border-box;
  text-align: center;
  line-height: 106rpx;
}

.item-image {
  position: absolute;
  top: 0;
  left: 0;
  width: 103rpx;
  height: 103rpx;
}

/* Item Positioning */
.item-top {
  top: 0rpx;
}

.item-top-0 {
  left: 0rpx;
}
.item-top-1 {
  left: 106rpx;
}
.item-top-2 {
  left: 212rpx;
}
.item-top-3 {
  left: 318rpx;
}
.item-top-4 {
  left: 424rpx;
}
.item-top-5 {
  left: 530rpx;
}

.item-right {
  right: 0rpx;
}

.item-right-0 {
  top: 0rpx;
}
.item-right-1 {
  top: 106rpx;
}
.item-right-2 {
  top: 212rpx;
}
.item-right-3 {
  top: 318rpx;
}
.item-right-4 {
  top: 424rpx;
}
.item-right-5 {
  top: 530rpx;
}

.item-bottom {
  bottom: 0rpx;
}

.item-bottom-0 {
  right: 0rpx;
}
.item-bottom-1 {
  right: 106rpx;
}
.item-bottom-2 {
  right: 212rpx;
}
.item-bottom-3 {
  right: 318rpx;
}
.item-bottom-4 {
  right: 424rpx;
}
.item-bottom-5 {
  right: 530rpx;
}

.item-left {
  left: 0rpx;
}

.item-left-0 {
  bottom: 0rpx;
}
.item-left-1 {
  bottom: 106rpx;
}
.item-left-2 {
  bottom: 212rpx;
}
.item-left-3 {
  bottom: 318rpx;
}
.item-left-4 {
  bottom: 424rpx;
}
.item-left-5 {
  bottom: 530rpx;
}

/* Button */
.btn {
  position: absolute;
  width: 200rpx;
  height: 100rpx;
  background-color: hsla(212, 100%, 86%, 0.788);
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  line-height: 100rpx;
}

/* Bet Items */
.bet-item-top {
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%);
  bottom: 20rpx;
  width: 480rpx;
  height: 150rpx;
  padding: 20rpx;
}

.bet-item-bottom {
  position: absolute;
  bottom: -120rpx;
  text-align: center;
  width: 100%;
  margin-top: 40rpx;
}

/* Bet Top */
.bet-top-item {
  width: 60rpx;
  height: 60rpx;
  display: inline-block;
}

.bet-top-image {
  width: 60rpx;
  height: 60rpx;
}

.bet-numbers {
  margin-top: 10rpx;
}

.bet-number-item {
  width: 60rpx;
  height: 60rpx;
  display: inline-block;
  text-align: center;
}

/* Bet Bottom */
.bet-bottom-item {
  width: 83rpx;
  height: 83rpx;
  display: inline-block;
}

.bet-bottom-image {
  width: 83rpx;
  height: 83rpx;
}
</style>
