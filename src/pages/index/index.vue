<template>
  <!--  -->
  <view
    class="container"
    v-for="(arrayItem, arrayIndex) in array"
    :key="arrayIndex"
  >
    <!-- 分数显示 -->
    <view class="score" style="position: absolute; top: 120rpx; left: 480rpx">
      {{ score }}<span @click="scoreAdd" style="font-size: 24rpx">+</span>
    </view>
    <!-- 遮罩层 -->
    <!-- (index + flashLightindex) % 2 === 0 -->
    <view v-for="(item, index) in 24" :key="index">
      <view
        v-if="flash"
        :style="{
          position: 'absolute',
          top: topCalculate(item) + 'rpx',
          left: leftCalculate(item) + 'rpx',
          width: '106rpx',
          height: '106rpx',
          backgroundColor: StrobeCalculation(index)
            ? 'hsla(0, 0%, 100%, 0.8)'
            : 'transparent',
          zIndex: 10000,
          animation: 'all 0.5s linear',
        }"
      ></view>
    </view>
    <view
      v-if="num !== -1"
      :style="{
        position: 'absolute',
        top: topCalculate((num % 24) + 1) + 'rpx',
        left: leftCalculate((num % 24) + 1) + 'rpx',
        width: '106rpx',
        height: '106rpx',
        backgroundColor: 'hsla(0, 0%, 100%, 0.8)',
        zIndex: 1000,
      }"
    >
    </view>
    <view
      v-for="(item, index) in arrayItem.top"
      :key="index"
      class="item"
      :style="{
        position: 'absolute',
        top: `0rpx`,
        left: `${index * 106}rpx`,
      }"
    >
      {{ item.number }}
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
    <!-- 右层10-18 -->
    <view
      v-for="(item, index) in arrayItem.right"
      :key="index"
      class="item"
      :style="{
        position: 'absolute',
        top: `${index * 106}rpx`,
        right: '0',
      }"
    >
      {{ item.number }}
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
    <!-- 底层19-27 -->
    <view
      v-for="(item, index) in arrayItem.bottom"
      :key="index"
      class="item"
      :style="{
        position: 'absolute',
        bottom: '0',
        right: `${index * 106}rpx`,
      }"
    >
      {{ item.number }}
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
      v-for="(item, index) in arrayItem.left"
      :key="index"
      class="item"
      :style="{
        position: 'absolute',
        bottom: `${index * 106}rpx`,
        left: '0',
      }"
    >
      {{ item.number }}
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
      <view
        v-for="(item, index) in bet"
        :key="index"
        style="width: 60rpx; height: 60rpx; display: inline-block"
      >
        <image
          :src="`/static/image/${item.name}.png`"
          style="width: 60rpx; height: 60rpx"
        />
      </view>
      <view v-if="betNum">
        <view
          v-for="(item, index) in bet"
          :key="index"
          style="
            width: 60rpx;
            height: 60rpx;
            display: inline-block;
            text-align: center;
          "
        >
          {{ item.number }}
        </view>
      </view>
      <view v-else>
        <view
          v-for="(item, index) in result.bet"
          :key="index"
          style="
            width: 60rpx;
            height: 60rpx;
            display: inline-block;
            text-align: center;
          "
        >
          {{ item.number }}
        </view>
      </view>
    </view>
    <view class="bet-item-bottom">
      <view
        v-for="(item, index) in bet"
        :key="index"
        style="width: 83rpx; height: 83rpx; display: inline-block"
      >
        <image
          :src="`/static/image/${item.name}.png`"
          style="width: 83rpx; height: 83rpx"
          @click="buttonClick(item)"
        />
      </view>
      <!-- 测试数据位置 -->
    </view>
  </view>
  <!-- 输入框 -->
</template>
<script setup>
import { watch } from "vue";
import CryptoJS from "crypto-js";
import { ref } from "vue";
import { onUnload, onLoad } from "@dcloudio/uni-app";

// 押注数量
const bet = ref([
  { name: "king", number: 0 },
  { name: "doubleSeven", number: 0 },
  { name: "doubleStar", number: 0 },
  { name: "watermelon", number: 0 },
  { name: "tangerine", number: 0 },
  { name: "bell", number: 0 },
  { name: "mango", number: 0 },
  { name: "apple", number: 0 },
]);
// 闪灯数组
let flashLightindex = ref(0);
// 闪灯定时器
let flashLightTimeout = null;
// 关闭闪灯
let flash = ref(false);
// 倍率显示
let ratenum = ref(0);
// 倍率;
let rate = ref([1, 10, 100]);
// 开火车
let train = ref([1, 2, 3, 4, 5, 6, 7]);
// 投注号码
let betNum = ref(true);
// 当前中号
let num = ref(-1);
// 记录加分次数和加分时间

let scoreAddarr = ref({ Count: 0, Time: 0 });
// 分数增加
const scoreAdd = function () {
  let scoreAddarrStr = uni.getStorageSync("scoreAddarr");
  // 如果不是今天的日期，重置加分次数
  if (scoreAddarrStr) {
    scoreAddarr.value = JSON.parse(scoreAddarrStr);
    if (scoreAddarr.value.Time !== new Date().toLocaleDateString()) {
      scoreAddarr.value.Count = 0;
      uni.setStorageSync("scoreAddarr", JSON.stringify(scoreAddarr.value));
      scoreAddarrStr = uni.getStorageSync("scoreAddarr");
    }
    scoreAddarr.value = JSON.parse(scoreAddarrStr);
    if (scoreAddarr.value.Count >= 10) {
      uni.showToast({
        title: "您今日已达加分上限",
        icon: "none",
      });
      return;
    }
  }
  scoreStraa.value = true;
  scoreAddarr.value.Count++;
  score.value += 100;
  // 只记录日期，不记录时间
  scoreAddarr.value.Time = new Date().toLocaleDateString();
  uni.setStorageSync("scoreAddarr", JSON.stringify(scoreAddarr.value));
};
// watch(scoreAddarr, (newVal, oldVal) => {
//   // 储存到本地
// });
// 什么灯
let isStroboscopicaaa = ref(2);
// 显示逻辑
const StrobeCalculation = function (index) {
  if (isStroboscopicaaa.value === 1) {
    return (index + +flashLightindex.value) % 2 === 0;
  }
  if (isStroboscopicaaa.value === 2) {
    return isStroboscopic.value;
  }
  if (isStroboscopicaaa.value === 3) {
    return isNum3.value
      ? isNum.value > index
      : isNum.value < index || index === 0;
  }
  if (isStroboscopicaaa.value === 4) {
    // 当负的那边到3时停止  正的那边到24时从123开始
    return (
      (Math.abs(index - 15) <= isNum4.value && index >= 3) ||
      (isNum4.value - 8 > 0 ? index < isNum4.value - 8 : index === flash)
    );
  }
};
// 倍率增加
const ratenumAdd = function () {
  if (ratenum.value < rate.value.length - 1) {
    ratenum.value++;
  } else {
    ratenum.value = 0;
  }
};
// 是否为频闪闪灯
let isStroboscopic = ref(false);
// 从1开始全亮
// 3号闪灯
let isNum = ref(0);
let isNum3 = ref(true);
// 4号
let isNum4 = ref(0);
// 闪灯频率
let flashLightFrequency = ref();
const flashLight = function (numaa) {
  isStroboscopicaaa.value = numaa;
  if (numaa === 1) {
    flashLightFrequency.value = 400;
  }
  if (numaa === 2) {
    flashLightFrequency.value = 200;
  }
  if (numaa === 3) {
    flashLightFrequency.value = 20;
  }
  if (numaa === 4) {
    flashLightFrequency.value = 50;
  }

  const flashLightRecursive = function () {
    if (!flash.value) {
      clearInterval(flashLightTimeout);
      return;
    }
    if (numaa === 1) {
      if (flashLightindex.value === 0) {
        flashLightindex.value = 1;
      } else {
        flashLightindex.value = 0;
      }
    }
    if (numaa === 2) {
      isStroboscopic.value = !isStroboscopic.value;
    }
    if (numaa === 3) {
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
    if (numaa === 4) {
      if (isNum4.value === 12) {
        isNum4.value = 1;
      } else {
        isNum4.value++;
      }
    }
  };
  flashLightTimeout = setInterval(
    flashLightRecursive,
    flashLightFrequency.value
  );
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
}, 3000);
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
  // 下注数不能超过1000
  if (item.number + Number(rate.value[ratenum.value]) > 1000) {
    let distance = item.number + Number(rate.value[ratenum.value]) - 1000;
    uni.showToast({
      title: `下注数不能超过1000，数据容易出错`,
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
// 计算当前押注号码的位置
const leftCalculate = function (num) {
  if (num > 0 && num < 7) {
    return num * 106 - 106;
  } else if (num > 6 && num < 13) {
    return 6 * 106;
  } else if (num > 12 && num < 20) {
    return 7 * 106 - (num % 12) * 106;
  } else if (num > 18 && num < 27) {
    return 0;
  }
};
// 计算当前押注号码的顶部位置
const topCalculate = function (num) {
  if (num > 0 && num < 7) {
    return 0;
  } else if (num > 6 && num < 13) {
    return (num % 7) * 106;
  } else if (num > 12 && num < 19) {
    return 106 * 6;
  } else if (num > 18 && num < 27) {
    return 106 * 7 - (num % 18) * 106;
  }
};
// 用于存储每个号码的信息
// 每个号码的信息包括：号码(number)、名称(name)、倍率(magnification)、额外倍数(double)
const array = ref([
  {
    top: [
      { number: 1, name: "tangerine", magnification: 2, double: 2 },
      { number: 2, name: "bell", magnification: 2, double: 2 },
      { number: 3, name: "king", magnification: 30, double: 0 },
      { number: 4, name: "king", magnification: 1000, double: 0 },
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
// 一个用于储存上次的押注数量用于在下次直接开始
let result = ref({
  bet: [
    { name: "king", number: 0 },
    { name: "doubleSeven", number: 0 },
    { name: "doubleStar", number: 0 },
    { name: "watermelon", number: 0 },
    { name: "tangerine", number: 0 },
    { name: "bell", number: 0 },
    { name: "mango", number: 0 },
    { name: "apple", number: 0 },
  ],
});

// 分数
let score = ref(100);
// 随机生成
let speed = ref(200);
// 定时器用于控制num的增加
let timer = null; // 提升作用域
// 用于开始进行运行
const aaa = function () {
  const aaaRecursive = function () {
    if (num.value + 1 === stopNum.value) {
      start.value = false;
      // 关闭所有定时器
      clearTimeout(timer);
      clearTimeout(timer1);
      bbbb.value = true;
      speed.value = 200;
      timer1 = null;
      scoreCalculate();
      return;
    }
    num.value++;
    timer = setTimeout(aaaRecursive, speed.value);
  };
  clearTimeout(timer); // 先清除旧定时器，再创建新的
  aaaRecursive();
};
// 用于控制速度的变化
let bbbb = ref(true);
// 用于控制速度变化的定时器
let timer1 = null;
// 停止位置(范围120-144)
let stopNum = ref(135);
// 用于控制速度变化的函数
const guan = function () {
  const guanRecursive = function () {
    if (num.value + 10 >= stopNum.value) {
      clearTimeout(timer1);
      return;
    }
    if (bbbb.value) {
      speed.value -= 15;
    } else {
      speed.value = Number(speed.value) + 15;
    }
    if (speed.value <= 15) {
      bbbb.value = false;
    }
    clearTimeout(timer);
    clearTimeout(timer1);
    aaa();
    timer1 = setTimeout(guanRecursive, 200);
  };
  guanRecursive();
};
// 记分函数
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
      (item) => item.number === (num.value % 24) + 1
    );
  } else if (
    // 筛选出当前右侧号码的信息
    (num.value % 24) + 1 > 6 &&
    (num.value % 24) + 1 < 13 &&
    (num.value % 24) + 1 !== 10
  ) {
    filterNum = array.value[0].right.find(
      (item) => item.number === (num.value % 24) + 1
    );
  } else if (
    // 筛选出当前底部号码的信息
    (num.value % 24) + 1 > 12 &&
    (num.value % 24) + 1 < 19 &&
    (num.value % 24) + 1 !== 16
  ) {
    filterNum = array.value[0].bottom.find(
      (item) => item.number === (num.value % 24) + 1
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
        (item) => item.number === (num.value % 24) + 1
      );
    }
  } else if ((num.value % 24) + 1 === 16) {
    // 筛选出当前全中的信息
    // 开灯
    flash.value = true;
    flashLight(4);
    // 全中加分
    scoreStraa.value = false;
    for (let i = 0; i < bet.value.length; i++) {
      let number = bet.value[i].number * 4 * 10;
      for (let index = 0; index < +number; index += 10) {
        // 三秒内加分
        let speedIndex = 3000 / number;
        // 每10毫米加分一次
        setTimeout(() => {
          score.value += 10;
          if (index >= +number - 20) {
            scoreStraa.value = true;
          } else {
            scoreStraa.value = false;
          }
        }, index * speedIndex);
        // 执行完所有加分后，scoreStraa为true
      }
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
    // 百分之40概率什么都不执行
    const randomNum = Math.floor(Math.random() * (100 - 0 + 1)) + 0;
    if (randomNum < 80) {
      // 提示被小鸟叼走了
      uni.showToast({
        title: "被小鸟叼走了",
        icon: "none",
      });
      // 清除当前押注数量
      for (let i = 0; i < bet.value.length; i++) {
        bet.value[i].number = 0;
      }
      return;
    }
    // 百分之五的概率转为大王(中)
    if (randomNum < 85) {
      filterNum = array.value[0].top.find((item) => item.number === 3);
      num.value = 2;
      // 延时300毫秒后执行闪烁;
      setTimeout(() => {
        flash.value = true;
        flashLight(2);
      }, 3000);
      setTimeout(() => {
        flash.value = false;
      }, 6000);
    }
    // 百分之十五的概率转为全中
    if (randomNum < 60 && randomNum > 100) {
      filterNum = array.value[0].top.find((item) => item.number === 4);
      num.value = 3;
      // 延时300毫秒后执行闪烁;
      setTimeout(() => {
        flash.value = true;
        flashLight(4);
      }, 3000);
      setTimeout(() => {
        flash.value = false;
      }, 6000);
    }
  }
  // 根据filterNum的magnification判断是否加分bet;
  for (let i = 0; i < bet.value.length; i++) {
    if (bet.value[i].name === filterNum.name) {
      if (Math.floor(Math.random() * (100 - 0 + 1)) + 0 < 20) {
        // 闪灯
        if (!flash.value) {
          flash.value = true;
          flashLight(4);
        }
        let number =
          bet.value[i].number *
          (filterNum.magnification + filterNum.double) *
          80;
        scoreStraa.value = false;
        for (let index = 0; index < +number; index += 10) {
          // 每10毫米加分一次
          // 确保在3秒内完成加分
          let speedIndex = 3000 / number;
          setTimeout(() => {
            score.value += 10;
            if (index >= +number - 20) {
              scoreStraa.value = true;
            } else {
              scoreStraa.value = false;
            }
          }, index * speedIndex);
        }
        // 三秒后关闭闪烁
        setTimeout(() => {
          flash.value = false;
        }, 3000);
      } else {
        let number =
          bet.value[i].number *
          (filterNum.magnification + filterNum.double) *
          10;
        scoreStraa.value = false;
        for (let index = 0; index < +number; index += 10) {
          // 确保在3秒内完成加分
          let speedIndex = 3000 / number;
          setTimeout(() => {
            score.value += 10;
            if (index >= +number - 20) {
              scoreStraa.value = true;
            } else {
              scoreStraa.value = false;
            }
          }, index * speedIndex);
        }
      }
    }
  }

  for (let i = 0; i < bet.value.length; i++) {
    bet.value[i].number = 0;
  }
};
// 开始状态
let start = ref(false);
const click = (M, N) => {
  // 10分之一的概率进入开火车
  // if (Math.floor(Math.random() * (100 - 0 + 1)) + 0 < 10) {
  //   // 开火车
  // }
  if (start.value) {
    uni.showToast({
      title: "还未结束",
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

  let array = generateProbabilityArray();
  // 清理浮点误差
  array = array.map((item) => Number(item.toFixed(2)));
  // 当前概率数组
  // 清理浮点误差
  let randomIndex = Math.floor(Math.random() * array.length);
  stopNum.value = 120 + 4
  // array[randomIndex];
  num.value = Math.floor(Math.random() * (M - N + 1)) + N;
  aaa(0);
  guan();
};
// 监听所有下注中的变化
// 概率数组(probabilityArray)根据下注数量实时计算probability
let probabilityArray = ref([
  // 小三元*2
  { num: [1, 2, 7, 13, 14, 19], probability: 200 },
  // 小三元
  { num: [12, 18, 24], probability: 300 },
  // 大三元*2
  { num: [8, 17, 20], probability: 100 },
  // 大三元
  { num: [9, 15, 21], probability: 200 },
  // 苹果
  { num: [6, 11, 23], probability: 100 },
  // 王
  { num: [3, 4, 5], probability: 20 },
  // 全中
  { num: [16], probability: 40 },
  // 其他
  { num: [10, 22], probability: 40 },
]);

let probability = ref([]);
const generateProbabilityArray = () => {
  // 原概率
  // 王
  probabilityArray.value[5].probability =
    probabilityArray.value[5].probability -
    probabilityArray.value[5].probability * (bet.value[0].number / 1000);
  // 全中
  const allBetNumber = bet.value.reduce((acc, cur) => acc + cur.number, 0);
  probabilityArray.value[6].probability =
    probabilityArray.value[6].probability -
    probabilityArray.value[6].probability * (allBetNumber / 10000);
  // 小三元押注总数
  // const smallBetNumber = bet.value
  //   .filter(
  //     (item) =>
  //       item.name === "tangerine" ||
  //       item.name === "bell" ||
  //       item.name === "mango"
  //   )
  //   .reduce((acc, cur) => acc + cur.number, 0);
  // // 小三元
  // probabilityArray.value[1].probability =
  //   probabilityArray.value[1].probability -
  //   probabilityArray.value[1].probability * (smallBetNumber / 100000);
  // // 小三元*2
  // probabilityArray.value[0].probability =
  //   probabilityArray.value[0].probability -
  //   probabilityArray.value[0].probability * (smallBetNumber / 10000);
  // 大三元押注总数
  const bigBetNumber = bet.value
    .filter(
      (item) =>
        item.name === "doubleStar" ||
        item.name === "doubleSeven" ||
        item.name === "watermelon"
    )
    .reduce((acc, cur) => acc + cur.number, 0);
  // 大三元
  // probabilityArray.value[3].probability =
  //   probabilityArray.value[3].probability -
  //   probabilityArray.value[3].probability * (bigBetNumber / 100000);
  // 大三元*2
  probabilityArray.value[2].probability =
    probabilityArray.value[2].probability -
    probabilityArray.value[2].probability * (bigBetNumber / 10000);
  // 修改后概率
  const resultaa = [];
  for (let index = 0; index < probabilityArray.value.length; index++) {
    const { num, probability } = probabilityArray.value[index];
    for (let i = 0; i < probability; i++) {
      const randomItem = num[Math.floor(Math.random() * num.length)];
      resultaa.push(randomItem);
    }
  }
  return resultaa;
};
// 关闭页面时清除定时器
onUnload(() => {
  clearTimeout(timer);
  uni.setStorageSync("score", score.value);
});
// 打开页面时从本地存储获取分数
onLoad(() => {
  try {
    const storedScore = uni.getStorageSync("score");
    // 解密
    const bytes = CryptoJS.AES.decrypt(storedScore, "h0911925");
    const originalText = bytes.toString(CryptoJS.enc.Utf8);
    // const alphabet = ["f", "p", "j", "n", "b", "y", "m", "a", "s", "q"];
    // 根据字母表将加密后的分数转换为数字
    let scoreStr = "";

    for (let i = 0; i < originalText.length; i++) {
      const digit = alphabet.indexOf(originalText[i]);
      if (digit !== -1) {
        scoreStr += digit.toString();
      }
    }
    score.value = +scoreStr;

    // 验证数据格式是否正确（必须是数字且非负）
    if (typeof +scoreStr === "number" && +scoreStr > 0) {
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
});

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
// let只有在分数加完之后才储存本地
let scoreStraa = ref(false);

// 监听分数变化，自动保存到本地存储
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
  const ciphertext = CryptoJS.AES.encrypt(
    encryptedScore,
    "h0911925"
  ).toString();
  // 储存加密后的分数
  uni.setStorageSync("score", ciphertext);
  // 添加加密方式
  // 加密方式：将原分数数字类型转换为字符串后，将每个数字改为其对应的字母（从1开始）
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
.item {
  display: inline-block;
  width: 106rpx;
  height: 106rpx;
  border: 1rpx solid rgb(133, 131, 255);
  box-sizing: border-box;
  text-align: center;
  line-height: 106rpx;
}
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
.bet-item-top {
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%);
  bottom: 20rpx;
  width: 480rpx;
  height: 150rpx;
  padding: 20rpx 20rpx;
  display: inline-block;
}
.bet-item-bottom {
  position: absolute;
  bottom: -120rpx;
  text-align: center;
  width: 100%;
  display: inline-block;
  margin-top: 40rpx;
}
</style>
