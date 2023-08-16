<template>
  <div class="container">
    <div class="bg"></div>
    <div class="box">
      <h3>第 {{ level }}关</h3>
      <div class="app">
        <div class="scene-container">
          <div class="scene-inner">
            <div v-for="(item, index) in scene" class="symbol" :key="item.id" :style="{
              transform: `translateX(${item.status === 0 ? item.x : item.status === 1 ? sortedQueue[item.id] : -850}%) translateY(${item.status === 0 ? item.y : 745}%)`,
              opacity: item.status < 2 ? 1 : 0 }" @click="clickSymbol(index)">
              <div class="symbol-inner" :style="{ backgroundColor: item.isCover ? '#999' : 'white' }">
                <img style="width:100%;" :src="item.icon" alt="">
              </div>
            </div>
          </div>
        </div>
        <div class="queue-container flex-container flex-center"></div>
        <div class="flex-container flex-between">
          <button class="flex-grow" @click="pop">弹出</button>
          <button class="flex-grow" @click="undo">撤销</button>
          <button class="flex-grow" @click="wash">洗牌</button>
          <button class="flex-grow" @click="levelUp">下一关</button>
        </div>
        <div class="modal" v-if="finished">
          <h1>{{ tipText }}</h1>
          <button @click="restart">再来一次</button>
        </div>
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
    len --;
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
// 初始icon数组
// 羊了个羊图片素材
const icons = [
  'data:image/gif;base64,R0lGODlhagBhAPf/AHZ2dsTExF0xEZRRIHE9FvDu7bZlKllZWcS7tObm5pCQ%0AkHl5eWJiYmVkZJSUlPDw8D4iDIx6baJZJJlUIlIqDWZmZnt6equqquHh4XBw%0AcKKiotzc3J6enj0fCJ2cnGxsbLBiKJyNgu7u7tXV1WBgYIKCguXh3lU7Kb6+%0AvuTk5KGgoGpqakEgCdHLxvj4+JeWlo5OHszMzFpaWqyiml5FM2hoaI2MjHhj%0AVMHBwdra2oSEhLKmnr2zq/r6+mQ1E8rKyoqKioZzZezs7Ovo5o+Ojunp6Wk4%0AFOnm5JKSkm1XRtjY2Lqvp7WpoWhRQNDQ0EwxHKmoqEomC3pBGbi4uIeGhrCw%0AsOPf3K6urtXOyaemplJOS0MnEpODd5iYmHJycq5gJ8vDvcLCwl5eXn9rXX9F%0AGru7u25ubrOzs5uamol3aYJHG4ZJHFJRUKuelUYpFapeJqldJmw6FUUjCjwd%0AB1E3I3x8fKCShmVNPGNLOk0oDJ5XI1I4JZBPHzseCVguD1BIQlRUVFdXV66t%0ArV1dXVZWVlxcXIiIiP39/aWkpPT09K2srN/f339/f7poK5iXl4CAgJybm7hm%0AKrlnKv7+/qyrq1VVVaqpqaKhoaSjo+rq6m9ZSfLy8tzX1NLS0vf394SDg/b2%0A9szFv/38/Pr6+ff29bNkKUAjDolLHba2tvf19PX19X59fYmJiblnK62trXRz%0Ac3diU7pnK6alpWdnZ/z7+6eZj8fHx59YI5GRkZJQIHZAGPb08459cO/s65qZ%0AmdfX13FxcfPy8Ojk4peHfKickqWlpd/b17S0tLCkm9PMyImIiN7e3qysrKen%0Ap6eakKSkpKOViuPj44yLi87HwXBvb52dnbespPn493tnWMC2r3d3d3JdTfLw%0A7oB/f2tUQ/r5+YFuX4+Pj0kuGWppaUU1KJ+fn3t7e6tfJ5WVlahhLYKBgdfR%0AzEtBOmBHNmFIN2xMNYeHh8i/ue3r6VZVVY2NjUEkD4xMHqRbJZB/c5JaMfz8%0A/LtoKzsdB6+urlNTU////yH/C1hNUCBEYXRhWE1QPD94cGFja2V0IGJlZ2lu%0APSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1w%0AbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUg%0AWE1QIENvcmUgNy4xLWMwMDAgNzkuZGFiYWNiYiwgMjAyMS8wNC8xNC0wMDoz%0AOTo0NCAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3%0ALnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNj%0AcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRv%0AYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2Jl%0ALmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9i%0AZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9y%0AVG9vbD0iQWRvYmUgUGhvdG9zaG9wIDIzLjAgKFdpbmRvd3MpIiB4bXBNTTpJ%0AbnN0YW5jZUlEPSJ4bXAuaWlkOkIyRUM0N0Y3MzY3NzExRUQ4ODE1RTgwMzRE%0AMzZGOTlFIiB4bXBNTTpEb2N1bWVudElEPSJ4bXAuZGlkOkIyRUM0N0Y4MzY3%0ANzExRUQ4ODE1RTgwMzREMzZGOTlFIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0%0AUmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6QjJFQkExOTAzNjc3MTFFRDg4MTVF%0AODAzNEQzNkY5OUUiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6QjJFQkEx%0AOTEzNjc3MTFFRDg4MTVFODAzNEQzNkY5OUUiLz4gPC9yZGY6RGVzY3JpcHRp%0Ab24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIi%0APz4B//79/Pv6+fj39vX08/Lx8O/u7ezr6uno5+bl5OPi4eDf3t3c29rZ2NfW%0A1dTT0tHQz87NzMvKycjHxsXEw8LBwL++vby7urm4t7a1tLOysbCvrq2sq6qp%0AqKempaSjoqGgn56dnJuamZiXlpWUk5KRkI+OjYyLiomIh4aFhIOCgYB/fn18%0Ae3p5eHd2dXRzcnFwb25tbGtqaWhnZmVkY2JhYF9eXVxbWllYV1ZVVFNSUVBP%0ATk1MS0pJSEdGRURDQkFAPz49PDs6OTg3NjU0MzIxMC8uLSwrKikoJyYlJCMi%0A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',
  'data:image/gif;base64,R0lGODlhaABnAPf/AObt5SyEJSNyHN7n3lWCURNSDvn7+XWacks5C7u0pChG%0ACiqAI7fKtc7bzarBqT1xOR9qGRhdEuDo4Cd6ICl/IhRUDhZYEFmFVVNDGv39%0A/Jy3mmdTEPb185WxkzVrMYKjfxleExphFChiJSV3HmGLXrGpljFoLEJ0PtHN%0Awi2GJl2IWiZ4H0Y0Bn6ge1RBCqW9o/r8+iFuGhxlFih8IXueeLrNuR1mFyZf%0AIF5LDS+KJ9XRxjluNUh5ROnv6H9zVBtiFSBtGrPHsUV3QazCq9vk2hJSDSxk%0AKE19SmePY3mddsDRvpiOdCuCJB5pGBlWE5u1mfDv62mRZoyqiRZZEVtKIVlV%0AENfUylF/TmJTLVKATnKXb1E+CSJxHJKvkBVRDYmohhpVFh5aGRVXEEBzPFpH%0ADGJODhRVD25eMyBbGyqBI0QyBWpWETGPKTGOKS2GJTCMKBdbErKrmPb49i6J%0AJyJwGyR0HS6IJi+LKDKPKvD08BlgFDGNKe7z7hpYFnebdPf59v3+/Sd7ICR1%0AHjCNKSZ5HxNUDhNRECBsGRdUEiZgIWSNYGWOYbDFrjCNKPH18SReHzGPKuPr%0A48zZy8/czi+JJ0d4Q9fi1tHe0BtjFdXg1Pj690x7SCFcGvj39WyTacXVxJGG%0AbDCLKC2HJsjXx8fBtOfk38nXyB5oF9ji19/o3xxZF8rYyT42Buvw6vX39fL1%0A8hdcEqigi3xvUOju5yuDJOzx7O3y7RNRDRZUEUUzBi5lKE9+TGuTaG2UamhV%0AEcfWxtHd0LTIst/c1Eo4Bx1XF7zOu+zq5rWunNPf0iNzHYOQb56Vfk49Exlf%0AE7jLt9nj2KKVaop+YqG6nzdTD0MyBa7ErO/t6Uk3B8/Lv764qfv6+ae+pY+t%0AjcLSwMTTwqefiR1SDuHp4RVYEG+VbFA/FLOsmnOYcCdSDmOMYPP28+ro5Pv8%0A+5m0l0VFC+Lg2aCWgKKZghdODPv7+tbh1XJkO3VoR3tuT4uqiI6si6O7oWhZ%0ALsbVxcC4mxJRDTKQKv///yH/C1hNUCBEYXRhWE1QPD94cGFja2V0IGJlZ2lu%0APSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1w%0AbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUg%0AWE1QIENvcmUgNy4xLWMwMDAgNzkuZGFiYWNiYiwgMjAyMS8wNC8xNC0wMDoz%0AOTo0NCAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3%0ALnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNj%0AcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRv%0AYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2Jl%0ALmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9i%0AZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9y%0AVG9vbD0iQWRvYmUgUGhvdG9zaG9wIDIzLjAgKFdpbmRvd3MpIiB4bXBNTTpJ%0AbnN0YW5jZUlEPSJ4bXAuaWlkOkIyRUQ0MDNCMzY3NzExRUQ4ODE1RTgwMzRE%0AMzZGOTlFIiB4bXBNTTpEb2N1bWVudElEPSJ4bXAuZGlkOkIyRUQ0MDNDMzY3%0ANzExRUQ4ODE1RTgwMzREMzZGOTlFIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0%0AUmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6QjJFQzQ4MDEzNjc3MTFFRDg4MTVF%0AODAzNEQzNkY5OUUiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6QjJFRDQw%0AM0EzNjc3MTFFRDg4MTVFODAzNEQzNkY5OUUiLz4gPC9yZGY6RGVzY3JpcHRp%0Ab24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIi%0APz4B//79/Pv6+fj39vX08/Lx8O/u7ezr6uno5+bl5OPi4eDf3t3c29rZ2NfW%0A1dTT0tHQz87NzMvKycjHxsXEw8LBwL++vby7urm4t7a1tLOysbCvrq2sq6qp%0AqKempaSjoqGgn56dnJuamZiXlpWUk5KRkI+OjYyLiomIh4aFhIOCgYB/fn18%0Ae3p5eHd2dXRzcnFwb25tbGtqaWhnZmVkY2JhYF9eXVxbWllYV1ZVVFNSUVBP%0ATk1MS0pJSEdGRURDQkFAPz49PDs6OTg3NjU0MzIxMC8uLSwrKikoJyYlJCMi%0A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',
  'data:image/gif;base64,R0lGODlhawBdAPf/AApa/+fr9ABG4wA8w9vazgBB1Vdsw+PgzWqEvgAntXuO%0Asujk08TK0X2Txu7s4TtcqQA6vvDv5u/s3QAtku7r3BRm/wo0kujl1HmQxURk%0ArQAqonmHw4mdy/Lw6enm1e7q27nF4LPA3gAlxbi7wQBN/P///gBD3ABI7AA+%0AywA1qwBN+oWayqStvevu9vDz+AAypN3j8OPfzNbd7cvLvUVjpJmr09zYxBM7%0AlRxDmmN+u4qat7a7vMDCwQcyku7x+MnMyRBh/wAwnSVLn/z9/gZW/661xAAv%0Al87W6fr59rK/3TVWm4ydv9TRwV12rPn6/BU+mPb07+jm2AA3swEtj2yDtCtP%0AnABK8aOz1wA/zl56uC5So+ro3JGkzzRWpYSVtN3axQBL9fT2+vL0+VVwqsnS%0A55mmu+Heyu3r4Ofk1jxcoVBsqElmooygzf38+8fKyYqWwu/t5Ozq3q+93OPo%0A81l2tlx0pKetwpep0SNInWF5q3SMwihMn2eBvABH5wQoxQ84lgArlwA5uAQv%0AkBZo//Hu4N7axfLv4NzYwvDt3u3q2urn1urm1ezp2eLfy+fk0vHu3+bi0O3q%0A2+TgzdzYw+rm1uDcyOHdydvXwufj0d3ZxN/cx+Xhz/Dt3+Ley+zo2Ovo2Ovn%0A1+Ddyd/bxuThzuHdyuXiz+Leyt7axoGXx9zZw+Dcx9/bx/Ht3/v7+Vd0td7b%0AxmuBru3p2RA3lMLN5MrT6OXhzgFQ/x9GndzYwf39/P7+/ufj0ho6xeDcydDY%0A6tTb7PDs3s7QytbY19nb2ff4+9rWwW+GwPHv54+ZwvDt4OXk34STrQBA0eXm%0A4nuPudzaw5upwwBM+IGUu4mWsp6v1S9LxCBFlk1ppdXVzKe22AArk+jl13uM%0Aq+Ti0OXi0zdap4OPw+DdyPv8/fz7+ZegwnOHsHiLsd7YxFx4ty9Snxc/l+zp%0A2FdwowszoObj1FBmxN/ez+Pfzq+5ytjX0JSm0N7bx6ezyNvaw6GptNjVwebj%0A0dvXwQAsj/Lv4QBO/////yH/C1hNUCBEYXRhWE1QPD94cGFja2V0IGJlZ2lu%0APSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1w%0AbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUg%0AWE1QIENvcmUgNy4xLWMwMDAgNzkuZGFiYWNiYiwgMjAyMS8wNC8xNC0wMDoz%0AOTo0NCAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3%0ALnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNj%0AcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRv%0AYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2Jl%0ALmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9i%0AZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9y%0AVG9vbD0iQWRvYmUgUGhvdG9zaG9wIDIzLjAgKFdpbmRvd3MpIiB4bXBNTTpJ%0AbnN0YW5jZUlEPSJ4bXAuaWlkOkIyRUVEOUFEMzY3NzExRUQ4ODE1RTgwMzRE%0AMzZGOTlFIiB4bXBNTTpEb2N1bWVudElEPSJ4bXAuZGlkOkIyRUVEOUFFMzY3%0ANzExRUQ4ODE1RTgwMzREMzZGOTlFIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0%0AUmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6QjJFRUQ5QUIzNjc3MTFFRDg4MTVF%0AODAzNEQzNkY5OUUiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6QjJFRUQ5%0AQUMzNjc3MTFFRDg4MTVFODAzNEQzNkY5OUUiLz4gPC9yZGY6RGVzY3JpcHRp%0Ab24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIi%0APz4B//79/Pv6+fj39vX08/Lx8O/u7ezr6uno5+bl5OPi4eDf3t3c29rZ2NfW%0A1dTT0tHQz87NzMvKycjHxsXEw8LBwL++vby7urm4t7a1tLOysbCvrq2sq6qp%0AqKempaSjoqGgn56dnJuamZiXlpWUk5KRkI+OjYyLiomIh4aFhIOCgYB/fn18%0Ae3p5eHd2dXRzcnFwb25tbGtqaWhnZmVkY2JhYF9eXVxbWllYV1ZVVFNSUVBP%0ATk1MS0pJSEdGRURDQkFAPz49PDs6OTg3NjU0MzIxMC8uLSwrKikoJyYlJCMi%0A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',
  'data:image/gif;base64,R0lGODlhagBfAPf/APssLIq5MlFVVSbz4nPq+vduAGppafuwKZdaZP/WLa5S%0AARAx57Krq/eyTf/XBc6MBdBmBWwqAO520rVwCv/Y+vmNKP0odv3UTvTuMaCz%0A25UKUZKVY/n/T48tBI5NBQELXlySqfDd0P6Kz9dyHhVjq9r39/6NBlF18S1Q%0AAAa1nv9OEP/6AE8iAf+r7vhFRci6sLIJCPEPDv5mrNWPJf+JiS0tMImutmaU%0AEHeqEQB3bqlcjf+ujPK6Av4zmFILAP/ZaxLMntPMuf/vZfhEMyoJAf1xKM/d%0AIgEjiNgQEKyZl68vhXZECkdvBvTZtO+p3iOI/6vc/7jL2awzIpjMIgOOee5V%0Au/9EmVJrpP1raxGrdtC5l6xSLooIB9AvJNTc9VSX/vuWT63QVtyvMOFVRxEO%0ADUWZ3PytrIgIM2L+pturSP+L7nYAAACZ/xFV/0a82tFQBAGJRPb0tUcyIGdm%0AVbNwLZiZrBB1zQFFxwBASNlUtwF27e5nm3WNUxGq///0krOwlW+V+CYuBFWu%0AAt+qAyNU792K1YfK/pFLL5dqMlWKAURVAMzdiNZFMCpv+bfNqsqURPm4aCPc%0AeM6QkGlTTbcjFm9ILZjMzP1xTj9ERNa3yHEdTJZnCM0kAf2QcdCuedlstmcA%0AIlUBM22QJM6Us5EqMslIm2yv/GqtyraJrRBUZP2Eu5mqhOzRkwEyMzWLwFVE%0AMsZwcMSQadbu32wyGACJ33mSzlBvI1BqbDPL/8VumbD+//gsBACqIiBufk9R%0AcnbFCWZ3Lyp1wsdyTJD5+idEgt3uMjlN449azW9tFANmWIdDdwEBQsvur/9L%0AfHDRbUKqcSumzXeqIi+Wg+rMeOTFULxMbC27RJmZmf//////zO7u7t3d3f//%0A7v//3YeHht3dy8zMzP/u3e7uzO7//8vL3svd3f/u7oaZmu7u3e7u/+LMzMnu%0AZhE0AP/uzKCBe//u/4KGopmZiOnM38bdxu7/7rvu4F93QlWq/zNjYMsRU4iZ%0AdxGcVd3ume7/wgAAAP///yH/C1hNUCBEYXRhWE1QPD94cGFja2V0IGJlZ2lu%0APSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1w%0AbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUg%0AWE1QIENvcmUgNy4xLWMwMDAgNzkuZGFiYWNiYiwgMjAyMS8wNC8xNC0wMDoz%0AOTo0NCAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3%0ALnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNj%0AcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRv%0AYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2Jl%0ALmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9i%0AZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9y%0AVG9vbD0iQWRvYmUgUGhvdG9zaG9wIDIzLjAgKFdpbmRvd3MpIiB4bXBNTTpJ%0AbnN0YW5jZUlEPSJ4bXAuaWlkOkIyRUZBMDA2MzY3NzExRUQ4ODE1RTgwMzRE%0AMzZGOTlFIiB4bXBNTTpEb2N1bWVudElEPSJ4bXAuZGlkOkIyRUZBMDA3MzY3%0ANzExRUQ4ODE1RTgwMzREMzZGOTlFIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0%0AUmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6QjJFRUQ5QUYzNjc3MTFFRDg4MTVF%0AODAzNEQzNkY5OUUiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6QjJFRUQ5%0AQjAzNjc3MTFFRDg4MTVFODAzNEQzNkY5OUUiLz4gPC9yZGY6RGVzY3JpcHRp%0Ab24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIi%0APz4B//79/Pv6+fj39vX08/Lx8O/u7ezr6uno5+bl5OPi4eDf3t3c29rZ2NfW%0A1dTT0tHQz87NzMvKycjHxsXEw8LBwL++vby7urm4t7a1tLOysbCvrq2sq6qp%0AqKempaSjoqGgn56dnJuamZiXlpWUk5KRkI+OjYyLiomIh4aFhIOCgYB/fn18%0Ae3p5eHd2dXRzcnFwb25tbGtqaWhnZmVkY2JhYF9eXVxbWllYV1ZVVFNSUVBP%0ATk1MS0pJSEdGRURDQkFAPz49PDs6OTg3NjU0MzIxMC8uLSwrKikoJyYlJCMi%0A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',
  '../assets/image/1.png',
  '../assets/image/2.png',
  '../assets/image/3.png',
  '../assets/image/4.png',
  '../assets/image/5.png',
  '../assets/image/6.png',
  '../assets/image/7.png',
  '../assets/image/8.png'
]
// 设置最大随机关卡
const maxLevel = 2;
// 以下感谢大佬们提供的算法
const makeScene = (level) => {
  // 获取当前关卡
  const curLevel = Math.min(maxLevel, level);
  // 获取当前关卡应该拥有的icon数量
  const iconPool = icons.slice(0, 3 * curLevel);
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
    // console.log(offset, row, column);
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
    const x2 = x1 + 100, y2 = y1 + 100;
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
    // 隐藏icon,dom
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
<style scoped>
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
  background: url('../assets/bgImg.jpeg');
  background-size: 550px 770px;
  background-repeat: no-repeat;
  left: 0;
  right: 0;
  bottom: 0;
  top: 30px;
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
  padding-bottom: 80%;
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
  width: 90%;
  padding-bottom: 15%;
  border: 2px solid gray;
  margin-bottom: 16px;
  margin: 0 auto;
  box-sizing: border-box;
}

.flex-container {
  width: 90%;
  margin: 10px auto;
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