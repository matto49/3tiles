<template>
  <div class="chooseDifficulty" v-if="showChooseDifficulty">
    <div>横向卡片最大平铺排数
      <input v-model="colNum" type="range" min="3" max="10">{{colNum}}
    </div>
    <div>纵向卡片最大平铺排数
      <input v-model="rowNum" type="range" min="3" max="10">{{rowNum}}
    </div>
    <div>卡片最大堆叠层数
      <input v-model="pilesNum" type="range" min="3" max="15">{{pilesNum}}
    </div>
    <div>卡片密度
      <input v-model="Density" type="range" max="10">{{Density}}
    </div>
    <div>最大卡片种类
      <input v-model="cardTypeNum" type="range" min="3" max="15">{{cardTypeNum}}
    </div>
    <button @click="restart">开始游戏</button>
  </div>
  <div class="game" v-show="gameStart">
    <div class="card-container" :style="{'height':rowNum*2*pointHeight+'px','width':colNum*2*pointWidth +'px'}">
      <div v-for="(card,index) in cardList" class="card"
        :class="{'is-covered':lists[card[0]][card[1]][card[2]].isCovered}" :key="card"
        :style="[cardPosition(card),{'background-color':backgroundColorList[lists[card[0]][card[1]][card[2]].cardType]}]"
        @click="removeCard(card,index)"><img :src="imgList[lists[card[0]][card[1]][card[2]].cardType]">
      </div>
    </div>
    <div class="remove-prop-container">
      <div v-for="(card,index) in removePropCards" :key="card" class="card"
        :style="{'left':index*2*pointWidth+'px','background-color':backgroundColorList[card.cardType]}"
        v-show="!card.isRemoved">
        <img :src="imgList[card.cardType]" @click="removePropCard(card,index)">
      </div>
    </div>
    <div class="remove-container" :style="{'width':colNum*2*pointWidth +'px'}">
      <div v-for="(cardType,index) in removeList" :key="cardType" class="card"
        :style="[removeCardPostion(index),{'background-color':backgroundColorList[cardType]}]"><img
          :src="imgList[cardType]"></div>
    </div>
    <div class="props">道具：
      <button @click="removeThreeCards" :disabled="removeHasUsed">取出三个卡片</button>
      <button @click="resort" :disabled="resortHasUsed">重新排序</button>
      <button @click="restart">重新开始</button>
    </div>
  </div>
  <div class="gameJudge" v-if="isDefeated||isVictory">
    <div class="words" v-if="isVictory">You Win! 🎉</div>
    <div class="words" v-if="isDefeated">You Lose! 😢</div>
    <div class="next">
      <button @click="restart">再来一轮</button>
      <button @click="chooseDifficulty">难度调节</button>
    </div>
  </div>
</template>

<script>
// import func from 'vue-editor-bridge'
let removeFlag = false
let removingType = -1
let removingIndex = 0
export default {
  name: 'App',
  components: {
  },
  data() {
    return {
      lists: [],
      cardList: [],
      removeList: [],
      showChooseDifficulty: true,
      gameStart: false,
      isDefeated: false,
      isVictory: false,
      rowNum: 4,
      colNum: 6,
      pilesNum: 10,
      Density: 2,
      cardTypeNum: 9,
      imgList: ['https://s3.bmp.ovh/imgs/2022/09/25/21d3b210ea6cfef8.gif', 'https://s3.bmp.ovh/imgs/2022/09/25/6b11c181a07963d1.gif', 'https://s3.bmp.ovh/imgs/2022/09/25/1cc041fead7d8c94.gif', "https://s3.bmp.ovh/imgs/2022/09/25/735d8e5bfcfbe752.gif", "https://s3.bmp.ovh/imgs/2022/09/25/c65d7342c4edf8fb.gif", 'https://s3.bmp.ovh/imgs/2022/09/25/c62dbe7712fd7e0c.gif', "https://s3.bmp.ovh/imgs/2022/09/25/bf55ab82711e8281.gif", "https://s3.bmp.ovh/imgs/2022/09/25/6b396331f2e17a1f.gif", "https://s3.bmp.ovh/imgs/2022/09/25/0ce0a527b14a741a.gif", 'https://s3.bmp.ovh/imgs/2022/09/25/f3f08b0e56d7a7df.gif', "https://s3.bmp.ovh/imgs/2022/09/24/5d1ac6dad1470a83.webp", "https://s3.bmp.ovh/imgs/2022/09/24/12bd3acaac8e72fe.webp", "https://s3.bmp.ovh/imgs/2022/09/24/7ed7dcc3140b7f61.webp", 'https://s3.bmp.ovh/imgs/2022/09/25/7ae04c8997a23b35.png', 'https://s3.bmp.ovh/imgs/2022/09/25/21c3e555fd5e9c7a.png'],
      backgroundColorList: ['rgb(153, 255, 153)', 'rgb(255, 200, 180)', 'rgb(255, 221, 170)', 'rgb(170, 255, 238)', 'rgb(204, 255, 153)', 'rgb(255, 255, 187)', 'rgb(255, 183, 221)', 'rgb(153, 255, 255)', 'rgb(204, 238, 255)', 'rgb(204, 221, 255)', 'rgb(153, 255, 255)', 'rgb(238, 255, 187)', 'rgb(255, 238, 153)'],
      pointHeight: 23,
      pointWidth: 22,
      removeHasUsed: false,
      removePropCards: [],
      resortHasUsed: false
    }
  },
  computed: {
  },
  methods: {
    //初始化
    init() {
      //使用行数×2 * 列数×2 * 层数的三维网格来实现效果，每个卡片占四个网格，以实现叠卡片时偏1/2卡片的效果
      //pilesNum在range的input里切换后会变成string类型
      //使用*2可以把它变回数字类型
      this.lists = new Array(Number(this.pilesNum)).fill(1).map((item, z) => {
        return new Array(this.colNum * 2).fill(0).map((item, x) => {
          return new Array(this.rowNum * 2).fill(0).map((item, y) => {
            return {
              hasCard: false,
              cardType: -1,
              x, y, z,
              isCovered: false,
              isRemoved: false
            }
          })
        })
      })
      //通过随机数来判断该位置是否放卡片，同时还要判断左、下、左下有卡片时不能重复放卡片
      this.lists.forEach((item, z) => {
        for (let x = 0; x < this.colNum * 2; x++) {
          for (let y = 0; y < this.rowNum * 2; y++) {
            if (10 * Math.random() < this.Density && (x == 0 || !this.lists[z][x - 1][y].hasCard) && (y == 0 || !this.lists[z][x][y - 1].hasCard) && (x == 0 || y == 0 || !this.lists[z][x - 1][y - 1].hasCard) && (y == this.rowNum * 2 - 1 || x == 0 || !this.lists[z][x - 1][y + 1].hasCard)) {
              //使用cardList来便于管理卡片
              // if (y != this.rowNum * 2 - 1) console.log(x, y, this.lists[z][x][y + 1].hasCard)
              this.cardList.push([z, x, y])
              this.lists[z][x][y].hasCard = true
            }
          }
        }
      })
      const cardListLength = this.cardList.length
      //使卡片数量变成3的倍数
      for (let i = 0; i < cardListLength % 3; i++) {
        const card = this.cardList.shift()
        //cardList存的值是先x坐标再y轴坐标
        this.lists[card[0]][card[1]][card[2]].hasCard = false
      }
      const tempCardList = [...this.cardList]
      //给卡片加上cardType
      for (let i = (tempCardList.length / 3); i > 0; i--) {
        const cardType = Math.floor(Math.random() * this.cardTypeNum)
        //3的倍数来给与卡片类型值
        for (let j = 0; j < 3; j++) {
          //随机一个tempCardList上的一个元素
          const index = Math.floor(Math.random() * tempCardList.length)
          //切除tempCardList里的该元素并获取到
          const card = tempCardList.splice(index, 1)[0]
          //给该卡片赋予cardType的值
          this.lists[card[0]][card[1]][card[2]].cardType = cardType
        }
      }
      //给卡片加上isCovered
      this.putCovered()
      console.log(this.lists)
    },
    putCovered() {
      this.cardList.forEach((item) => {
        this.lists[item[0]][item[1]][item[2]].isCovered = this.cardList.some((card) => {
          if (card[0] > item[0] && card[1] - item[1] >= -1 && card[1] - item[1] <= 1 && card[2] - item[2] >= -1 && card[2] - item[2] <= 1) {
            return true
          }
        }
        )
      })
    },
    //移卡片
    removeCard(card, index) {
      if (removeFlag) return
      removeFlag = true
      this.lists[card[0]][card[1]][card[2]].hasCard = false
      removingType = this.lists[card[0]][card[1]][card[2]].cardType
      removingIndex = this.removeList.lastIndexOf(removingType) == -1 ? this.removeList.length : this.removeList.lastIndexOf(removingType) + 1
      setTimeout(() => {
        this.cardList.splice(index, 1)[0]
        this.removeList.splice(removingIndex, 0, removingType)
        this.lists[card[0]][card[1]][card[2]].hasCard = false
        //加遮罩
        this.putCovered()
        //3消
        this.threeRemove()
        //判断胜负
        this.judgeGame()
        removeFlag = false
      }, 300)
    },
    //上面堆着的卡片的位置
    cardPosition(card) {
      if (this.lists[card[0]][card[1]][card[2]].hasCard)
        return { 'left': card[1] * this.pointWidth + 'px', 'top': card[2] * this.pointHeight + 'px', 'z-index': card[0] }
      else {
        return { 'left': removingIndex * 2 * this.pointWidth + 'px', 'top': this.rowNum * 2 * this.pointHeight + 150 + 'px', 'z-index': card[0] }
      }
    },
    //removelist里面的卡片的位置
    removeCardPostion(index) {
      if (!removeFlag || removingIndex > index)
        return { 'left': index * 2 * this.pointWidth + 'px' }
      else
        return { 'left': (index + 1) * 2 * this.pointWidth + 'px' }
    },
    //3个消除
    threeRemove() {
      const removeMap = new Map()
      this.removeList.forEach((item, index) => {
        if (!removeMap.has(item)) removeMap.set(item, [index])
        else removeMap.get(item).push(index)
      })
      for (let key of removeMap.keys()) {
        if (removeMap.get(key).length == 3) {
          this.removeList.splice(removeMap.get(key)[0], 3)
          removeMap.set(key, [])
        }
      }
    },
    //判断胜负
    judgeGame() {
      if (this.removeList.length === 7) {
        this.isDefeated = true
        this.gameStart = false
      }
      if (this.cardList.length == 0 && this.removePropCards.every((item) => item.isRemoved)) {
        this.isVictory = true
        this.gameStart = false
      }
    },
    //重开游戏
    restart() {
      this.showChooseDifficulty = false
      this.isDefeated = false
      this.isVictory = false
      this.gameStart = true
      this.removeHasUsed = false
      this.resortHasUsed = false
      this.lists = []
      this.cardList = []
      this.removeList = []
      this.removePropCards = []
      this.init()
    },
    //调节难度
    chooseDifficulty() {
      this.isDefeated = false
      this.isVictory = false
      this.showChooseDifficulty = true
    },
    //道具
    removeThreeCards() {
      this.removePropCards = this.removeList.splice(0, 3).map((item) => {
        return {
          cardType: item,
          isRemoved: false
        }
      })
      this.removeHasUsed = true
    },
    removePropCard(card, index) {
      this.removePropCards[index].isRemoved = true
      removingIndex = this.removeList.lastIndexOf(card.cardType) == -1 ? this.removeList.length : this.removeList.lastIndexOf(card.cardType) + 1
      this.removeList.splice(removingIndex, 0, card.cardType)
      //3消
      this.threeRemove()
      //判断胜负
      this.judgeGame()
    },
    resort() {
      //新的坐标点
      for (let i = 0; i < this.cardList.length; i++) {
        const card = this.cardList[i]
        let newCard = [Math.floor(Math.random() * this.pilesNum), Math.floor(Math.random() * this.colNum * 2), Math.floor(Math.random() * this.rowNum * 2)]
        //不断随机出有效的新位置，有效即同层九宫格里面不重复
        while (this.cardList.some((item) => item[0] == newCard[0] && newCard[1] - item[1] >= -1 && newCard[1] - item[1] <= 1 && newCard[2] - item[2] >= -1 && newCard[2] - item[2] <= 1)) {
          newCard = [Math.floor(Math.random() * this.pilesNum), Math.floor(Math.random() * this.colNum * 2), Math.floor(Math.random() * this.rowNum * 2)]
        }
        this.lists[card[0]][card[1]][card[2]].hasCard = false
        this.lists[newCard[0]][newCard[1]][newCard[2]].hasCard = true
        this.lists[newCard[0]][newCard[1]][newCard[2]].cardType = this.lists[card[0]][card[1]][card[2]].cardType
        this.lists[card[0]][card[1]][card[2]].cardType = -1
        this.cardList[i] = newCard
      }
      //给卡片加上isCovered
      this.putCovered()
      this.resortHasUsed = true
    },
  }
}
</script>

<style>
.chooseDifficulty {
  margin: 50px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.chooseDifficulty button {
  margin-top: 30px;
}

.card-container {
  margin: 80px auto 20px;
  position: relative;
}

.card {
  position: absolute;
  height: 38px;
  width: 42px;
  box-shadow: 0px 1px 0 0 #fff, 0 8px 0 0 #ddd, 0 8px 0 2px #333, 0 0 0 2px #333;
  transition: all .3s ease;
}

img {
  height: 38px;
  width: 42px;
}

.is-covered {
  pointer-events: none;
}

.is-covered::after {
  content: "";
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
  opacity: .55;
  background: #000;
}

.remove-prop-container {
  position: relative;
  margin: 70px auto 30px;
  height: 50px;
  width: 250px
}

.remove-container {
  position: relative;
  margin: 0 auto;
  height: 50px;
  width: 250px
}

.remove-container::after {
  border: 2px solid #000;
  position: absolute;
  left: -2px;
  top: -2px;
  content: "";
  height: 46px;
  width: 306px;
}

.props {
  display: flex;
  justify-content: center;
}

.words {
  font-size: 36px;
  font-weight: bold;
  text-align: center;
  margin-top: 150px;
}

.next {
  display: flex;
  justify-content: center;
  margin: 30px auto;
}
</style>
