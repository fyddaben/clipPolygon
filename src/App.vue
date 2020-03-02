<template>
  <div class='container'>
    <!-- <canvas width='512' height='800' class="app" id='app'></canvas> -->
    
    <div class="img-block">
      <canvas  class="app" id='app' :style="canvasStyle"></canvas>
      <div class="operate-block">
        <div class="add btn" @click='changeStatusToAdd'>增加</div>
        <div class="del btn" @click='changeStatusToDel'>删除</div>
        <div class="del btn" @click='changeStatusToBoth'>增加/删除</div>
        <div class="del btn" @click='changeStatusToPen'>画笔</div>
        <div class="del btn" @click='changeStatusToWipe'>橡皮</div>
      </div>
    </div>
    
  </div>
</template>

<script>
const polygonClipping = require('polygon-clipping')
var polygonInside = require('point-in-polygon');

export default {
  name: 'App',
  data() {
    return {
      isDrawing: false,
      canvasStyle: 'cursor:crosshair',
      points: [],
      polyindex: 0,
      // add 为相互交, subtract 为删除, correction 为增加/删除，paint为画笔 ,wipe表示橡皮
      polyStatus: 'add',
      // 画笔的中心点数组
      penpoints: [],
      // 画笔的路径
      penintersection:[],
      // 橡皮的中心点数组
      wipepoints: [],
      // 橡皮的路径
      wipeintersection: [],
      // 画笔半径
      circleR: 10,
    }
  },
 
  mounted(){
    var canvas;
    var context;

    // 初始化
    
    // 获取画布已经绘图上下文
    canvas = document.getElementById("app");
    canvas.width = document.body.clientHeight
    canvas.height = document.body.clientHeight
    context = canvas.getContext("2d");
    // 设置线条颜色
    context.strokeStyle = 'red';
    context.lineWidth = 1;
    context.lineJoin = 'round';
    context.lineCap = 'round';
    context.fillStyle= "rgba(100, 40, 40, 0.3)";
    
    // 画布添加鼠标事件
    canvas.addEventListener('mousedown', this.startDrawing, false);
    canvas.addEventListener('mousemove', this.draw, false);
    canvas.addEventListener('mouseup', this.stopDrawing, false);
    canvas.addEventListener('mouseout', this.stopDrawing, false);
    this.canvas = canvas
    this.context = context
    requestAnimationFrame(this.render);
    this.points[this.polyindex] = []
  },
  methods: {
    changeStatusToAdd() {
      this.polyStatus = 'add'
      this.canvasStyle = 'cursor:crosshair'
    },
    changeStatusToDel() {
      this.polyStatus = 'subtract'
      this.canvasStyle = 'cursor:e-resize'
    },
    changeStatusToBoth() {
      this.polyStatus = 'correction'
      this.canvasStyle = 'cursor:pointer'
    },
    changeStatusToPen() {
      this.polyStatus = 'paint'
      this.canvasStyle = 'cursor:pointer'
    },
    changeStatusToWipe() {
      this.polyStatus = 'wipe'
      this.canvasStyle = 'cursor:pointer'
    },
    getPos(evt) {
      var top = document.getElementById('app').offsetTop
      var left = document.getElementById('app').offsetLeft
      return {
        x: evt.clientX - left,
        y: evt.clientY - top
      }
    },
    startDrawing(evt) {
      this.isDrawing = true;
      const { x, y } = this.getPos(evt);
      if (this.polyStatus == 'wipe') {
        this.wipepoints.push([x,y])
      } else if (this.polyStatus == 'paint') {
        this.penpoints.push([x,y])
      } else {
        this.points[this.polyindex].push([x, y]);
      }
    },
    stopDrawing(evt) {
      if (!this.isDrawing) return;
      const { x, y } = this.getPos(evt);
      
      if (this.polyStatus == 'wipe') {
        this.wipepoints.push([x,y])
      } else if (this.polyStatus == 'paint') {
        this.penpoints.push([x,y])
      } else {
        this.points[this.polyindex].push([x, y]);
      }
      this.context.fill()
      
      this.isDrawing = false;
      if (['add', 'correction', 'paint'].indexOf(this.polyStatus)!=-1) {
        this.turfCompare()
      }
      // 删除和橡皮，必须在前置有数据下，再进行操作
      if (['subtract', 'wipe'].indexOf(this.polyStatus)!=-1 && this.points.length == 1) {
        this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)
        this.points = []
        this.poins[this.polyindex] = []
      }
      if (['subtract', 'wipe'].indexOf(this.polyStatus)!=-1 && this.points.length > 1) {
        this.turfCompare()
      }
    },
    
    turfCompare() {
      var polyArr = []
      var polyArr_a = []
      
      polyArr_a = JSON.parse(JSON.stringify(this.points))
      var polyStatus = this.polyStatus
      var relation = ''
      if (polyStatus == 'correction') {
        // 判断起始点是否在polygon之内
        var lastchild = polyArr_a[polyArr_a.length - 1]
        var isIn = false
        polyArr_a.map((v, i)=> {
          if (i < polyArr_a.length - 1) {
            if (polygonInside(lastchild[0][0], v[0])) {
              isIn = true
            }
          }
        })
        if (isIn) {
          relation = 'union'
        } else {
          relation = 'difference'
        }
      }
      if (polyStatus == 'add' || polyStatus == 'paint') {
        relation = 'union'
      }
      if (polyStatus == 'subtract' || polyStatus == 'wipe') {
        relation = 'difference'
      }
    
      
      if (polyStatus == 'wipe') {
        //因为最后一个为空数组，忽略最后一个
        polyArr.push(polyArr_a.slice(0, polyArr_a.length - 1))
        polyArr.push(JSON.parse(JSON.stringify(this.wipeintersection)))
        
        // 然后可以清空橡皮的数组了
        this.wipepoints = []
      } else if (polyStatus == 'paint') {
        //因为最后一个为空数组，忽略最后一个
        polyArr.push(polyArr_a.slice(0, polyArr_a.length - 1))
        polyArr.push(JSON.parse(JSON.stringify(this.penintersection)))
        
        // 然后可以清空画笔的数组了
        this.penpoints = []
      } else {
        if (polyArr_a.length == 1) {
          polyArr.push(polyArr_a)
          polyArr.push([])
        } else {
          var lastchild = polyArr_a.slice(polyArr_a.length - 1)
          var child = polyArr_a.slice(0, polyArr_a.length - 1)
          polyArr.push(child)
          polyArr.push(lastchild)
        }
      }
      console.log(relation, polyArr)
      var intersection = polygonClipping[relation](...polyArr)
      
      if (intersection.length > 0) {
        
        this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)
        this.points = []
        this.polyindex = 0
        
        intersection.map((polys)=> {
          this.context.beginPath()
          polys.map((poly)=> {
            
            poly.map((point, b)=> {
              if (b == 0) {
                this.context.moveTo(point[0], point[1])
              } else {
                this.context.lineTo(point[0], point[1])
              }
              this.context.strokeStyle = 'green';
              this.context.stroke();
            })
            
          })
          this.context.fill();
        })
        this.points = intersection
        
      }
      
      this.polyindex = this.points.length
      this.points[this.polyindex] = []
    },
    draw(evt) {
      if (!this.isDrawing) return;
      evt.preventDefault();
      evt.stopPropagation();
      const { x, y } = this.getPos(evt);
      
      if (this.polyStatus == 'wipe') {
        this.wipepoints.push([x,y])
      } else if (this.polyStatus == 'paint') {
        this.penpoints.push([x,y])
      } else {
        this.points[this.polyindex].push([x, y]);
      }
      
    },
    //根据圆心,获取圆形坐标数组
    getCircleAxios(v) {
      
      var center = v; //圆心坐标
      var radius = this.circleR; //半径
      var newarr = []
      for (var i=0;i<60;i++) {
        var hudu = (2*Math.PI / 360) * i * 10; //90度角的弧度

        var X = center[0] + Math.sin(hudu) * radius; //求出90度角的x坐标
        var Y = center[1] - Math.cos(hudu) * radius; //求出90度角的y坐标
        newarr.push([X,Y])
      }
      return newarr
    },
    drawLine(){
      
      for (var i=0;i<=this.polyindex;i++) {
        this.context.beginPath()
        this.points[i].map((v, b)=> {
          if (Object.prototype.toString.call(v[0]) === '[object Array]') {
            v.map((vchild, vi)=> {
              if (vi == 0) {
                this.context.moveTo(vchild[0], vchild[1])
              } else {
                this.context.lineTo(vchild[0], vchild[1])
              } 
            })
          } else {
            if (b == 0) {
              this.context.moveTo(v[0], v[1])
            } else {
              this.context.lineTo(v[0], v[1])
            }
          }
        })
        if (this.points[i].length > 0) {
          this.context.lineTo(this.points[i][0][0], this.points[i][0][1])
          this.context.stroke();
        }
        if (i<this.polyindex) {
          this.context.fill()
        }
      }
    },
    // 画笔的痕迹
    drawPen() {
      if (this.polyStatus != 'paint') {return}
      var circleArr = []
      this.penpoints.map((v)=> {
        var circle = this.getCircleAxios(v)
        circleArr.push([circle])
      })
      var intersection = polygonClipping.union(...circleArr)
      
      this.context.beginPath()
      intersection.map((v)=> {
        v.map((vchild)=> {
          vchild.map((vc, vi)=> {
            if (vi == 0) {
              this.context.moveTo(vc[0], vc[1])
            } else {
              this.context.lineTo(vc[0], vc[1])
            } 
          })
        })
      })
      
      this.context.stroke();
      this.context.fill()
      this.penintersection = intersection
      
    },
    // 画橡皮的痕迹
    drawWipe() {
      if (this.polyStatus != 'wipe') {return}
      var circleArr = []
      this.wipepoints.map((v)=> {
        var circle = this.getCircleAxios(v)
        circleArr.push([circle])
      })
      var intersection = polygonClipping.union(...circleArr)
      
      this.context.beginPath()
      intersection.map((v)=> {
        v.map((vchild)=> {
          vchild.map((vc, vi)=> {
            if (vi == 0) {
              this.context.moveTo(vc[0], vc[1])
            } else {
              this.context.lineTo(vc[0], vc[1])
            } 
          })
        })
      })
      
      this.context.stroke();
      this.context.fill()
      this.wipeintersection = intersection
      
    },
    render() {
      if (!this.isDrawing) {
        requestAnimationFrame(this.render)
        return;
      }
      this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)
      
      this.drawLine();
      this.drawPen();
      this.drawWipe();
      // 继续渲染
      requestAnimationFrame(this.render)
    }
  }
}
</script>

<style>
html,body{
  margin: 0;
  padding: 0;
}
.operate-block{
  position: absolute;
  right: 40px;
  top: 50%;
  
}
.operate-block .btn{
  width: 100px;
  height: 40px;
  line-height: 40px;
  background: #ccc;
  color: #000;
  margin-bottom: 10px;
  cursor: pointer;
}
.container {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  background: #000;
}
canvas{
  background: rgba(0, 0, 0, 0.6);
}
.img-block{
  margin-left:auto;
  margin-right: auto;
  height: 100vh;
  background-color: #000;
  background-size: auto 100vh;
  background-image: url('/img.jpg');
  background-repeat: no-repeat;
  background-position: center center;
}
</style>
