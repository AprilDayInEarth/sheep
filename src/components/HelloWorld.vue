<template>
  <div>
    <h1>{{ msg }}</h1>
    <div class="box">
      <div class="pic"><button class="btn1" @click="clickHandler1"></button></div>
      <div class="pic"><button class="btn2" @click="clickHandler2"></button></div>
      <div class="pic"><button class="btn3" @click="clickHandler3"></button></div>
      <div class="pic"><button class="btn4" @click="clickHandler4"></button></div>
      <div class="pic"><button class="btn5" @click="clickHandler5"></button></div>
      <div class="pic"><button class="btn6" @click="clickHandler6"></button></div>
      <div class="pic"><button class="btn7" @click="clickHandler7"></button></div>
      <div class="pic"><button class="btn8" @click="clickHandler8"></button></div>
      <div class="pic"><button class="btn9" @click="clickHandler9"></button></div>
      <div class="container"></div>
    </div>
  </div>
</template>

<script>

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data() {
    return {

    }
  },
  methods: {
    clickHandler1() {
      console.log("点击了1");
    },
    clickHandler2() {
      console.log("点击了2");
    },
    clickHandler3() {
      console.log("点击了3");
    },
    clickHandler4() {
      console.log("点击了4");
    },
    clickHandler5() {
      console.log("点击了5");
    },
    clickHandler6() {
      console.log("点击了6");
    },
    clickHandler7() {
      console.log("点击了7");
    },
    clickHandler8() {
      console.log("点击了8");
    },
    clickHandler9() {
      console.log("点击了9");
    }
  }
}
</script>

<style scoped>
.box {
  width: 500px;
  height: 500px;
  margin: 0 auto;
  border-radius: 5px;
  display: inline-grid;
  grid-template-columns: 60px 60px 60px;
  grid-template-rows: 60px 60px 60px;
  grid-row-gap: 10px;
  grid-column-gap: 20px;
  background-color: rgba(0,0,0,0.1);
}

.box .pic {
  margin-left: 150px;
  margin-top: 100px;
}

.box .pic button {
  width: 60px;
  height: 60px;
  text-align: center;
}

.btn1 {
  background: url("../assets/pic.png") no-repeat -9px -6px;
}
.btn2 {
  background: url("../assets/pic.png") no-repeat -80px -6px;
}
.btn3 {
  background: url("../assets/pic.png") no-repeat -150px -6px;
}
.btn4 {
  background: url("../assets/pic.png") no-repeat -9px -80px;
}
.btn5 {
  background: url("../assets/pic.png") no-repeat -9px -150px;
}
.btn6 {
  background: url("../assets/pic.png") no-repeat -9px -221px;
}
.btn7 {
  background: url("../assets/pic.png") no-repeat -80px -80px;
}
.btn8 {
  background: url("../assets/pic.png") no-repeat -150px -150px;
}
.btn9 {
  background: url("../assets/pic.png") no-repeat -221px -221px;
}

.container {
  width: 400px;
  height: 70px;
  left: 50px;
  top: 150px;
  position: relative;
  background: #fff;
}

</style>
