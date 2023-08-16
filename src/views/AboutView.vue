<template>
  <div class="bg"></div>
  <div class="box">
    <h3>第 {{ level }}关 </h3>
    <div class="app">
      <div class="scene-container">
        <div class="scene-inner">
          <div v-for="(item, index) in scene" class="symbol" :key="item.id" :style="{
            transform: `translateX(${item.status === 0 ? item.x : item.status === 1 ? sortedQueue[item.id] : -1000}%) translateY(${item.status === 0 ? item.y : 895}%)`,
            opacity: item.status < 2 ? 1 : 0,
            }" @click="clickSymbol(index)">
            <div class="symbol-inner" :style="{ backgroundColor: item.isCover ? '#999' : 'white' }">
              <img style="width:100% ;" :src="item.icon" alt="">
            </div>
          </div>
        </div>
      </div>
      <div class="queue-container flex-container flex-center"></div>
      <div class="flex-container flex-between">
        <button class="flex-grow" @click="pop">
          弹出
        </button>
        <button class="flex-grow" @click="undo">
          撤销
        </button>
        <button class="flex-grow" @click="wash">
          洗牌
        </button>
        <button class="flex-grow" @click="levelUp">
          下一关
        </button>
      </div>
      <div class="modal" v-if="finished">
        <h1>{{ tipText }}</h1>
        <button @click="restart">再来一次</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watchEffect } from 'vue'
// 生成随机id
const randomString = (len) => {
  const pool = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  let res = '';
  while (len >= 0) {
    res += pool[Math.floor(pool.length * Math.random())];
    len--;
  }
  return res;
};
//延时函数
const waitTimeout = (timeout) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve();
    }, timeout);
  });
};
const icons = []
// 设置最大随机关卡
const maxLevel = 50;
// 以下感谢大佬们提供的算法
const makeScene = (level) => {
  // 获取当前关卡
  const curLevel = Math.min(maxLevel, level);
  // 获取当前关卡应该拥有的icon数量
  const iconPool = icons.slice(0, 2 * curLevel);
  // 算出偏移量范围具体细节范围
  const offsetPool = [0, 25, -25, 50, -50].slice(0, 1 + curLevel);
  //  最终的元数据数组
  const scene = [];
  // 确定范围
  //在一般情下 translate 的偏移量，如果是百分比的话，是按照自身的宽度或者高度去计算的，所以最大的偏移范围是百分800%
  // 然后通过Math.random 会小于百分之八百
  // 所以就会形成当前区间的随机数
  const range = [
    [2, 6],
    [1, 6],
    [1, 7],
    [0, 7],
    [0, 8],
  ][Math.min(4, curLevel - 1)];
  const randomSet = (icon) => {
    // 求偏移量
    const offset = offsetPool[Math.floor(offsetPool.length * Math.random())];
    // 偏移求列数
    const row = range[0] + Math.floor((range[1] - range[0]) * Math.random());
    // 求偏移行数
    const column = range[0] + Math.floor((range[1] - range[0]) * Math.random());
    console.log(offset, row, column);
    // 生成元数据对象
    scene.push({
      isCover: false, // 默认都是没有被覆盖的
      status: 0,// 是否被选中的状态
      icon,// 图标
      id: randomString(4), // 生成随机id
      x: column * 100 + offset, //x 坐标
      y: row * 100 + offset,// y坐标
    });
  };

  // 如果级别高了就加点icon 花哨一点
  let compareLevel = curLevel;
  while (compareLevel > 0) {
    iconPool.push(...iconPool.slice(0, Math.min(10, 2 * (compareLevel - 5))));
    compareLevel -= 5;
  }
  // 生成元数据，初始状态下 iconPool的内容少生 随着增加，就会越来越难
  for (const icon of iconPool) {
    for (let i = 0; i < 6; i++) {
      randomSet(icon);
    }
  }
  // 返回元数据
  return scene;
};
// 洗牌
const washScene = (level, scene) => {
  // 洗牌的随机逻辑与初始化相同,不在赘述
  const updateScene = scene.slice().sort(() => Math.random() - 0.5);
  const offsetPool = [0, 25, -25, 50, -50].slice(0, 1 + level);
  const range = [
    [2, 6],
    [1, 6],
    [1, 7],
    [0, 7],
    [0, 8],
  ][Math.min(4, level - 1)];
  const randomSet = (symbol) => {
    const offset = offsetPool[Math.floor(offsetPool.length * Math.random())];
    const row = range[0] + Math.floor((range[1] - range[0]) * Math.random());
    const column = range[0] + Math.floor((range[1] - range[0]) * Math.random());
    symbol.x = column * 100 + offset;
    symbol.y = row * 100 + offset;
    symbol.isCover = false;
  };
  //遍历初始化数组
  for (const symbol of updateScene) {
    // 碰见已经选中的不处理
    if (symbol.status !== 0) continue;
    randomSet(symbol);
  }

};
const scene = ref(makeScene(1)) // 元数据
const level = ref(1); // 等级
const queue = ref([]);// 下方选中数据
const sortedQueue = ref([]);// 排序内容
const finished = ref(false);// 是否完成
const tipText = ref('');// 提示
const animating = ref(false);// 动画
// 检查是否被覆盖
const checkCover = (value) => {
  // 深拷贝一份
  const updateScene = value.slice();
  // 是否覆盖算法
  // 遍历所有的元数据
  // 双重for循环来找到每个元素的覆盖情况
  for (let i = 0; i < updateScene.length; i++) {
    // 当前item对角坐标
    const cur = updateScene[i];
    // 先假设他都不是覆盖的
    cur.isCover = false;
    // 如果status 不为0 说明已经被选中了，不用再判断了
    if (cur.status !== 0) continue;
    // 拿到坐标
    const { x: x1, y: y1 } = cur;
    // 为了拿到他们的对角坐标，所以要加上100
    //之所以要加上100 是由于 他的总体是800% 也就是一个格子的换算宽度是100
    const x2 = x1 + 100,
      y2 = y1 + 100;
    // 第二个来循环来判断他的覆盖情况
    for (let j = i + 1; j < updateScene.length; j++) {
      const compare = updateScene[j];
      if (compare.status !== 0) continue;
      const { x, y } = compare;
      // 处理交集也就是选中情况
      // 两区域有交集视为选中
      // 两区域不重叠情况取反即为交集
      if (!(y + 100 <= y1 || y >= y2 || x + 100 <= x1 || x >= x2)) {
        // 由于后方出现的元素会覆盖前方的元素，所以只要后方的元素被选中了，前方的元素就不用再判断了
        // 又由于双层循环第二层从j 开始，所以不用担心会重复判断
        cur.isCover = true;
        break;
      }
    }
  }
  scene.value = updateScene;
};

// 弹出
const pop = () => {
  // 如果选中队列中没有数据那么就退出
  if (!queue.value.length) return;
  const updateQueue = queue.value.slice();
  const symbol = updateQueue.shift();
  if (!symbol) return;
  const find = scene.value.find((s) => s.id === symbol.id);
  if (find) {
    // 随机一个位置
    queue.value = updateQueue;
    find.status = 0;
    find.x = 100 * Math.floor(8 * Math.random());
    find.y = 700;
    // 遮挡检测
    checkCover(scene.value);
  }
};
// 撤销
const undo = () => {
  // 和弹出相同逻辑
  // 只是返回的位置为原来位置
  if (!queue.value.length) return;
  const updateQueue = queue.value.slice();
  const symbol = updateQueue.pop();
  if (!symbol) return;
  const find = scene.value.find((s) => s.id === symbol.id);
  if (find) {
    queue.value = updateQueue;
    find.status = 0;
    checkCover(scene);
  }
};
// 洗牌
const wash = () => {
  checkCover(washScene(level.value, scene.value));
};

// 下一关
const levelUp = () => {
  if (level.value >= maxLevel) {
    return;
  }
  finished.value = false;
  level.value = level.value + 1;
  queue.value = [];
  // 初始化场景并且检查是否被覆盖
  checkCover(makeScene(level.value + 1));
};
// 重开
const restart = () => {
  // 初始化内容
  finished.value = false;
  level.value = 1;
  queue.value = [];
  // 第一关初始化，并开启遮挡检查
  checkCover(makeScene(1));
};

// 点击item
const clickSymbol = async (idx) => {
  // 如果已经完成了，就不处理
  if (finished.value || animating.value) return;
  // 拷贝一份Scene
  const symbol = scene.value[idx];
  // 覆盖了和已经在队列里的也不处理
  if (symbol.isCover || symbol.status !== 0) return;
  // 置为可以选中状态
  symbol.status = 1;
  queue.value.push(symbol);
  // 制造动画效果中防止点击
  animating.value = true;
  // 三百毫秒的延迟
  await waitTimeout(300);
  // 拿到与他匹配的所有icon
  const filterSame = queue.value.filter((sb) => sb.icon === symbol.icon);

  // 选中的三个配对成功表示已经是三连了
  if (filterSame.length === 3) {
    // 由于icon的类型一样，留下队列中的不一样的剩余内容重新赋值
    queue.value = queue.value.filter((sb) => sb.icon !== symbol.icon);
    // 隐藏iocn，dom
    for (const sb of filterSame) {
      const find = scene.value.find((i) => i.id === sb.id);
      // 将他们的状态变为2 通过opacity 属性 来隐藏icon
      if (find) find.status = 2;
    }
  }

  // 当格子沾满了，那么久表示已经失败了
  if (queue.value.length === 7) {
    tipText.value = '失败了'
    finished.value = true;
  }

  if (!scene.value.find((s) => s.status !== 2)) {
    // 如果完成所有关卡，那就过了所有关了
    if (level.value === maxLevel) {
      tipText.value = '完成挑战';
      finished.value = true
      return;
    }
    //否则加一关
    level.value = level.value + 1;
    queue.value = []
    // 重新初始化
    checkCover(makeScene(level.value + 1));
  } else {
    // 处理覆盖情况
    checkCover(scene.value);
  }
  // 动画结束
  animating.value = false;
};
// 队列区排序
watchEffect(() => {
  const cache = [];
  // 通过当前的icon的标识，将相同的icon归纳到一块
  // 方便后续排序
  for (const symbol of queue.value) {
    if (cache[symbol.icon]) {
      cache[symbol.icon].push(symbol);
    } else {
      cache[symbol.icon] = [symbol];
    }
  }
  const temp = [];
  for (const symbols of Object.values(cache)) {
    temp.push(...(symbols));
  }
  const updateSortedQueue = [];
  let x = 50;
  // 拿到更新后的队列区数据，计算权重
  for (const symbol of temp) {
    updateSortedQueue[symbol.id] = x;
    x += 100;
  }
  //赋值 ，这个是为了将选中的排序后的内容移动到队列区
  sortedQueue.value = updateSortedQueue
  // 检查覆盖情况
  checkCover(scene.value);
})
</script>
<style>
a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}

a:hover {
  color: #535bf2;
}


button {
  border-radius: 8px;
  border: 1px solid transparent;
  padding: 0.6em 1.2em;
  font-size: 1em;
  font-weight: 500;
  font-family: inherit;
  background-color: #1a1a1a;
  cursor: pointer;
  transition: border-color 0.25s;
  word-break: keep-all;
}

button:hover {
  border-color: #646cff;
}

button:focus,
button:focus-visible {
  outline: 4px auto -webkit-focus-ring-color;
}

#app {
  text-align: center;
  width: 100vw;
  height: 100vh;
  max-width: 500px;
  position: relative;

}

.bg {
  position: absolute;
  background-image: url();
  background-size: 100% 100%;
  background-repeat: no-repeat;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
}

.box {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  padding: 30px;
}

.app {
  width: 100%;
  margin: 0 auto;
}

.scene-container {
  width: 100%;
  padding-bottom: 100%;
  position: relative;
  margin: 10% 0;
}

.scene-inner {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  overflow: visible;
  font-size: 28px;
}

.symbol {
  width: 12.5%;
  padding-bottom: 12.5%;
  position: absolute;
  transition: 0.3s;
  left: 0;
  top: 0;
}

.symbol-inner {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 8px;
  border: 2px solid #444;
  transition: 0.3s;
  padding: 5px;
}

.queue-container {
  border-radius: 8px;
  width: 100%;
  padding-bottom: 15%;
  border: 2px solid gray;
  margin-bottom: 16px;
  box-sizing: border-box;
}

.flex-container {
  display: flex;
  gap: 8px;
}

.flex-center {
  justify-content: center;
  align-items: center;
}

.flex-grow {
  flex-grow: 1;
  color: #fff
}

.flex-between {
  justify-content: space-between;
  align-items: center;
}

.modal {
  position: fixed;
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(10px);
  background-color: rgb(0 0 0 / 10%);
  top: 0;
  left: 0;
}
</style>

