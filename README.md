<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport"
      content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">

<title>孤岛臭臭鼠快跑</title>

<style>
*{
  margin:0;
  padding:0;
  box-sizing:border-box;
  -webkit-tap-highlight-color:transparent;
}

html,body{
  width:100%;
  height:100%;
  overflow:hidden;
  background:#071820;
  font-family:
    -apple-system,
    BlinkMacSystemFont,
    "PingFang SC",
    "Microsoft YaHei",
    sans-serif;
}

canvas{
  position:fixed;
  inset:0;
  width:100vw;
  height:100vh;
  touch-action:none;
}

#hud{
  position:fixed;
  top:calc(12px + env(safe-area-inset-top));
  left:12px;
  right:12px;
  display:flex;
  justify-content:space-between;
  z-index:20;
  pointer-events:none;
  color:white;
}

.hudBox{
  min-width:90px;
  padding:8px 12px;
  border-radius:15px;
  background:rgba(0,0,0,.30);
  border:1px solid rgba(255,255,255,.18);
  backdrop-filter:blur(10px);
  box-shadow:0 5px 20px rgba(0,0,0,.18);
}

.hudLabel{
  font-size:11px;
  opacity:.7;
}

.hudValue{
  margin-top:2px;
  font-size:20px;
  font-weight:900;
}

#introText{
  position:fixed;
  left:50%;
  top:35%;
  transform:translate(-50%,-50%);
  z-index:30;
  color:white;
  font-size:25px;
  font-weight:900;
  white-space:nowrap;
  text-shadow:
    0 3px 12px rgba(0,0,0,.9),
    0 0 20px rgba(255,255,255,.15);
  opacity:0;
  pointer-events:none;
}

.screen{
  position:fixed;
  inset:0;
  z-index:50;
  display:flex;
  justify-content:center;
  align-items:center;
  background:
    linear-gradient(
      rgba(3,18,24,.08),
      rgba(3,18,24,.68)
    );
}

.card{
  width:min(88vw,420px);
  padding:30px 22px;
  border-radius:28px;
  color:white;
  text-align:center;
  background:rgba(5,23,29,.78);
  border:1px solid rgba(255,255,255,.15);
  backdrop-filter:blur(16px);
  box-shadow:0 25px 80px rgba(0,0,0,.45);
}

.title{
  font-size:34px;
  font-weight:1000;
  letter-spacing:2px;
}

.subtitle{
  margin-top:8px;
  opacity:.68;
  font-size:15px;
}

button{
  width:100%;
  margin-top:24px;
  padding:15px;
  border:0;
  border-radius:18px;
  background:#ffe16a;
  color:#132027;
  font-size:18px;
  font-weight:900;
}

.tip{
  margin-top:14px;
  font-size:13px;
  line-height:1.8;
  opacity:.58;
}

#gameOver{
  display:none;
}

#power{
  position:fixed;
  left:50%;
  bottom:25px;
  transform:translateX(-50%);
  z-index:20;
  color:white;
  padding:7px 13px;
  border-radius:20px;
  background:rgba(0,0,0,.28);
  font-size:13px;
  opacity:0;
}
</style>
</head>

<body>

<canvas id="game"></canvas>

<div id="hud">

  <div class="hudBox">
    <div class="hudLabel">距离</div>
    <div class="hudValue">
      <span id="distance">0</span>m
    </div>
  </div>

  <div class="hudBox">
    <div class="hudLabel">金币</div>
    <div class="hudValue">
      🪙 <span id="coins">0</span>
    </div>
  </div>

</div>

<div id="power"></div>

<div id="introText">
  你们好，我是臭臭鼠
</div>

<div id="startScreen" class="screen">
  <div class="card">

    <div class="title">
      孤岛臭臭鼠
    </div>

    <div class="subtitle">
      海岛极速跑酷
    </div>

    <button id="startButton">
      开始游戏
    </button>

    <div class="tip">
      左右滑动：移动<br>
      上滑：跳跃
    </div>

  </div>
</div>

<div id="gameOver" class="screen">

  <div class="card">

    <div class="title">
      游戏结束
    </div>

    <div class="subtitle">
      你跑了
      <b id="finalDistance">0</b>
      米
      <br>
      获得 🪙
      <b id="finalCoins">0</b>
    </div>

    <button id="restartButton">
      再跑一次
    </button>

  </div>

</div>

<script>

const canvas =
  document.getElementById("game");

const ctx =
  canvas.getContext("2d");

let W=innerWidth;
let H=innerHeight;
let DPR=1;

function resize(){

  DPR=Math.min(
    window.devicePixelRatio || 1,
    2
  );

  W=innerWidth;
  H=innerHeight;

  canvas.width=W*DPR;
  canvas.height=H*DPR;

  canvas.style.width=W+"px";
  canvas.style.height=H+"px";

  ctx.setTransform(
    DPR,0,0,DPR,0,0
  );
}

window.addEventListener(
  "resize",
  resize
);

resize();

/* =====================================================
   游戏数据
===================================================== */

const G={

  state:"menu",

  time:0,

  distance:0,

  coins:0,

  lane:1,

  targetLane:1,

  playerZ:0,

  playerX:0,

  jumpY:0,

  jumpV:0,

  speed:8,

  objects:[],

  particles:[],

  spawnDistance:20,

  shake:0,

  introTime:0,

  magnet:0,

  shield:0,

  boost:0

};

let best=
  Number(
    localStorage.getItem(
      "chouchoushu_best"
    ) || 0
  );

/* =====================================================
   工具
===================================================== */

function clamp(v,a,b){
  return Math.max(
    a,
    Math.min(b,v)
  );
}

function lerp(a,b,t){
  return a+(b-a)*t;
}

function rand(a,b){
  return a+Math.random()*(b-a);
}

function choose(a){
  return a[
    Math.floor(
      Math.random()*a.length
    )
  ];
}

/* =====================================================
   第三人称摄像机
===================================================== */

/*
   世界坐标：

   z 越大 = 越远

   玩家一直真正向 z 正方向移动。
   摄像机跟随玩家。
*/

const CAMERA_HEIGHT=2.2;
const CAMERA_DISTANCE=7;

function playerWorldZ(){
  return G.playerZ;
}

/*
   将世界位置转换成屏幕透视。
*/

function project(z,height=0){

  const relative =
    z-G.playerZ;

  const distance =
    Math.max(
      .4,
      relative+CAMERA_DISTANCE
    );

  const scale =
    1.55/distance;

  const horizon=
    H*.37;

  const y=
    horizon+
    CAMERA_HEIGHT/distance*H*.72-
    height*scale*H*.72;

  return {
    scale,
    y
  };
}

/*
   道路宽度随距离变化。
*/

function roadWidth(z){

  const relative =
    z-G.playerZ;

  const d=
    clamp(
      relative,
      -CAMERA_DISTANCE,
      80
    );

  const t=
    clamp(
      1-d/80,
      0,
      1
    );

  return lerp(
    W*.82,
    W*.11,
    t
  );
}

/*
   三条道路位置。
*/

function roadX(
  lane,
  z
){

  const relative=
    z-G.playerZ;

  const t=
    clamp(
      1-relative/80,
      0,
      1
    );

  const width=
    lerp(
      W*.78,
      W*.10,
      t
    );

  return W/2+
    (lane-1)*width*.29;
}

/*
   世界 z → 屏幕 y
*/

function worldY(z){

  const relative=
    z-G.playerZ;

  const t=
    clamp(
      relative/80,
      -0.05,
      1
    );

  return lerp(
    H*.90,
    H*.38,
    t
  );
}

/* =====================================================
   天空
===================================================== */

function drawSky(){

  const sky=
    ctx.createLinearGradient(
      0,0,0,H
    );

  sky.addColorStop(
    0,
    "#5ec4dc"
  );

  sky.addColorStop(
    .45,
    "#a9e4e4"
  );

  sky.addColorStop(
    1,
    "#5d9d91"
  );

  ctx.fillStyle=sky;

  ctx.fillRect(
    0,0,W,H
  );

  /* 太阳 */

  ctx.beginPath();

  ctx.arc(
    W*.78,
    H*.16,
    45,
    0,
    Math.PI*2
  );

  ctx.fillStyle=
    "rgba(255,239,170,.78)";

  ctx.fill();

  /* 云 */

  drawCloud(
    W*.18,
    H*.15,
    1
  );

  drawCloud(
    W*.56,
    H*.10,
    .7
  );

  drawCloud(
    W*.88,
    H*.26,
    .65
  );
}

function drawCloud(
  x,y,s
){

  ctx.save();

  ctx.globalAlpha=.35;

  ctx.fillStyle="#fff";

  ctx.beginPath();

  ctx.arc(
    x,
    y,
    25*s,
    0,
    Math.PI*2
  );

  ctx.arc(
    x+27*s,
    y+5*s,
    18*s,
    0,
    Math.PI*2
  );

  ctx.arc(
    x-25*s,
    y+8*s,
    18*s,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.restore();
}

/* =====================================================
   海和远处岛屿
===================================================== */

function drawSea(){

  const hy=H*.37;

  ctx.fillStyle="#2d989d";

  ctx.fillRect(
    0,
    hy,
    W,
    H-hy
  );

  ctx.globalAlpha=.25;

  for(
    let i=0;
    i<12;
    i++
  ){

    const y=
      hy+20+i*17;

    ctx.beginPath();

    for(
      let x=0;
      x<W;
      x+=16
    ){

      const wave=
        Math.sin(
          x*.035+
          G.time*.001+
          i
        )*3;

      if(x===0)
        ctx.moveTo(x,y+wave);
      else
        ctx.lineTo(x,y+wave);
    }

    ctx.strokeStyle="#d9ffff";
    ctx.lineWidth=1;

    ctx.stroke();
  }

  ctx.globalAlpha=1;
}

function drawIsland(){

  const y=H*.38;

  ctx.fillStyle="#315f52";

  ctx.beginPath();

  ctx.moveTo(
    0,
    y+8
  );

  ctx.quadraticCurveTo(
    W*.18,
    y-35,
    W*.32,
    y+3
  );

  ctx.quadraticCurveTo(
    W*.48,
    y-45,
    W*.65,
    y+4
  );

  ctx.quadraticCurveTo(
    W*.84,
    y-30,
    W,
    y+7
  );

  ctx.lineTo(
    W,
    y+70
  );

  ctx.lineTo(
    0,
    y+70
  );

  ctx.closePath();

  ctx.fill();
}

/* =====================================================
   道路
===================================================== */

function drawRoad(){

  const horizon=
    H*.37;

  /* 草地 */

  ctx.fillStyle="#5d9b61";

  ctx.fillRect(
    0,
    horizon,
    W,
    H-horizon
  );

  /*
    道路真正向远处延伸
  */

  const farW=W*.10;
  const nearW=W*.88;

  ctx.beginPath();

  ctx.moveTo(
    W/2-farW/2,
    horizon
  );

  ctx.lineTo(
    W/2+farW/2,
    horizon
  );

  ctx.lineTo(
    W/2+nearW/2,
    H
  );

  ctx.lineTo(
    W/2-nearW/2,
    H
  );

  ctx.closePath();

  const roadGradient=
    ctx.createLinearGradient(
      0,
      horizon,
      0,
      H
    );

  roadGradient.addColorStop(
    0,
    "#71817e"
  );

  roadGradient.addColorStop(
    .5,
    "#53635f"
  );

  roadGradient.addColorStop(
    1,
    "#374541"
  );

  ctx.fillStyle=
    roadGradient;

  ctx.fill();

  /* 道路边线 */

  ctx.strokeStyle="#eadc9d";
  ctx.lineWidth=5;

  ctx.beginPath();

  ctx.moveTo(
    W/2-farW/2,
    horizon
  );

  ctx.lineTo(
    W/2-nearW/2,
    H
  );

  ctx.stroke();

  ctx.beginPath();

  ctx.moveTo(
    W/2+farW/2,
    horizon
  );

  ctx.lineTo(
    W/2+nearW/2,
    H
  );

  ctx.stroke();

  /*
     路面虚线
  */

  for(
    let z=3;
    z<90;
    z+=5
  ){

    const y1=
      worldY(z);

    const y2=
      worldY(z+2);

    if(
      y1<H &&
      y2>horizon
    ){

      for(
        let lane=0;
        lane<2;
        lane++
      ){

        const x1=
          roadX(lane,z);

        const x2=
          roadX(lane,z+2);

        ctx.strokeStyle=
          "rgba(244,240,205,.7)";

        ctx.lineWidth=
          Math.max(
            1,
            4/(z*.08+1)
          );

        ctx.beginPath();

        ctx.moveTo(
          x1,
          y1
        );

        ctx.lineTo(
          x2,
          y2
        );

        ctx.stroke();
      }
    }
  }

  /*
    路面横向纹理
  */

  for(
    let z=4;
    z<85;
    z+=7
  ){

    const y=
      worldY(z);

    if(
      y<horizon ||
      y>H
    ) continue;

    const width=
      roadWidth(z);

    ctx.strokeStyle=
      "rgba(255,255,255,.045)";

    ctx.lineWidth=1;

    ctx.beginPath();

    ctx.moveTo(
      W/2-width/2,
      y
    );

    ctx.lineTo(
      W/2+width/2,
      y
    );

    ctx.stroke();
  }
}

/* =====================================================
   路边棕榈树
===================================================== */

function drawPalm(
  x,
  y,
  s
){

  ctx.save();

  ctx.translate(x,y);

  ctx.scale(s,s);

  ctx.strokeStyle="#67452e";
  ctx.lineWidth=6;
  ctx.lineCap="round";

  ctx.beginPath();

  ctx.moveTo(
    0,
    0
  );

  ctx.quadraticCurveTo(
    -5,
    -35,
    3,
    -72
  );

  ctx.stroke();

  ctx.fillStyle="#27744a";

  for(
    let i=0;
    i<7;
    i++
  ){

    ctx.save();

    ctx.rotate(
      -1.4+i*.45
    );

    ctx.beginPath();

    ctx.ellipse(
      0,
      -72,
      8,
      33,
      0,
      0,
      Math.PI*2
    );

    ctx.fill();

    ctx.restore();
  }

  ctx.restore();
}

function drawEnvironment(){

  for(
    let z=7;
    z<75;
    z+=9
  ){

    const y=
      worldY(z);

    const width=
      roadWidth(z);

    const scale=
      clamp(
        1.2/(z*.08+1),
        .12,
        1
      );

    if(
      y>H ||
      y<H*.36
    ) continue;

    drawPalm(
      W/2-width/2-28*scale,
      y,
      scale
    );

    drawPalm(
      W/2+width/2+28*scale,
      y,
      scale
    );
  }
}

/* =====================================================
   物体
===================================================== */

function addObject(
  type,
  lane,
  z
){

  G.objects.push({

    type,
    lane,
    z,

    collected:false,

    spin:
      Math.random()*Math.PI*2

  });
}

/* =====================================================
   生成路线
===================================================== */

function spawnPattern(){

  const safe=
    Math.floor(
      Math.random()*3
    );

  const r=Math.random();

  /*
     金币长龙
  */

  if(r<.24){

    const lane=
      Math.floor(
        Math.random()*3
      );

    for(
      let i=0;
      i<6;
      i++
    ){

      addObject(
        "coin",
        lane,
        G.playerZ+
        22+
        i*4
      );
    }

    return;
  }

  /*
     单障碍
  */

  if(r<.43){

    const lane=
      Math.floor(
        Math.random()*3
      );

    addObject(
      choose([
        "rock",
        "log",
        "barrier"
      ]),
      lane,
      G.playerZ+25
    );

    return;
  }

  /*
     双障碍
  */

  if(r<.62){

    const lanes=[
      0,1,2
    ];

    lanes.splice(
      safe,
      1
    );

    addObject(
      choose([
        "rock",
        "log"
      ]),
      lanes[0],
      G.playerZ+27
    );

    addObject(
      choose([
        "rock",
        "barrier"
      ]),
      lanes[1],
      G.playerZ+27
    );

    /*
       安全路线给金币
    */

    for(
      let i=0;
      i<3;
      i++
    ){

      addObject(
        "coin",
        safe,
        G.playerZ+
        25+
        i*4
      );
    }

    return;
  }

  /*
     道具
  */

  if(r<.82){

    addObject(
      choose([
        "boost",
        "magnet",
        "shield"
      ]),
      safe,
      G.playerZ+25
    );

    for(
      let i=0;
      i<5;
      i++
    ){

      addObject(
        "coin",
        safe,
        G.playerZ+
        29+
        i*3
      );
    }

    return;
  }

  /*
     宝箱
  */

  addObject(
    "chest",
    safe,
    G.playerZ+27
  );

  for(
    let i=0;
    i<5;
    i++
  ){

    addObject(
      "coin",
      safe,
      G.playerZ+
      31+
      i*3
    );
  }
}

/* =====================================================
   绘制物体
===================================================== */

function drawObject(o){

  const relative=
    o.z-G.playerZ;

  if(
    relative<-.5 ||
    relative>90
  ) return;

  const p=
    project(o.z);

  const x=
    roadX(
      o.lane,
      o.z
    );

  const y=
    worldY(o.z);

  const s=
    clamp(
      1.45/(relative*.08+1),
      .12,
      1.55
    );

  ctx.save();

  ctx.translate(
    x,
    y
  );

  if(o.type==="coin"){

    ctx.rotate(
      Math.sin(
        G.time*.006+
        o.spin
      )*.35
    );

    ctx.fillStyle="#ffd84d";

    ctx.beginPath();

    ctx.ellipse(
      0,
      -30*s,
      12*s,
      16*s,
      0,
      0,
      Math.PI*2
    );

    ctx.fill();

    ctx.strokeStyle="#fff4a5";
    ctx.lineWidth=3*s;

    ctx.stroke();

  }

  else if(o.type==="rock"){

    ctx.fillStyle="#655d55";

    ctx.beginPath();

    ctx.moveTo(
      -25*s,
      0
    );

    ctx.lineTo(
      -20*s,
      -35*s
    );

    ctx.lineTo(
      7*s,
      -44*s
    );

    ctx.lineTo(
      28*s,
      -17*s
    );

    ctx.lineTo(
      22*s,
      0
    );

    ctx.closePath();

    ctx.fill();

    ctx.fillStyle="#837a6f";

    ctx.beginPath();

    ctx.moveTo(
      -20*s,
      -35*s
    );

    ctx.lineTo(
      7*s,
      -44*s
    );

    ctx.lineTo(
      1*s,
      -28*s
    );

    ctx.closePath();

    ctx.fill();
  }

  else if(o.type==="log"){

    ctx.fillStyle="#75492f";

    ctx.rotate(.05);

    ctx.fillRect(
      -34*s,
      -27*s,
      68*s,
      25*s
    );

    ctx.fillStyle="#b47b4e";

    ctx.beginPath();

    ctx.arc(
      34*s,
      -14*s,
      12*s,
      0,
      Math.PI*2
    );

    ctx.fill();
  }

  else if(o.type==="barrier"){

    ctx.fillStyle="#d45642";

    ctx.fillRect(
      -34*s,
      -36*s,
      68*s,
      29*s
    );

    ctx.fillStyle="#f2e4bd";

    for(
      let i=-2;
      i<=2;
      i++
    ){

      ctx.save();

      ctx.translate(
        i*17*s,
        -21*s
      );

      ctx.rotate(-.5);

      ctx.fillRect(
        -4*s,
        -13*s,
        8*s,
        27*s
      );

      ctx.restore();
    }
  }

  else if(
    o.type==="boost" ||
    o.type==="magnet" ||
    o.type==="shield"
  ){

    let icon="⚡";

    if(o.type==="magnet")
      icon="🧲";

    if(o.type==="shield")
      icon="🛡️";

    ctx.shadowBlur=22*s;
    ctx.shadowColor="#fff";

    ctx.fillStyle=
      "rgba(255,255,255,.20)";

    ctx.beginPath();

    ctx.arc(
      0,
      -30*s,
      29*s,
      0,
      Math.PI*2
    );

    ctx.fill();

    ctx.shadowBlur=0;

    ctx.textAlign="center";
    ctx.textBaseline="middle";

    ctx.font=
      `${34*s}px sans-serif`;

    ctx.fillText(
      icon,
      0,
      -30*s
    );
  }

  else if(o.type==="chest"){

    ctx.fillStyle="#9c5f30";

    ctx.fillRect(
      -32*s,
      -42*s,
      64*s,
      42*s
    );

    ctx.fillStyle="#f3c342";

    ctx.fillRect(
      -6*s,
      -42*s,
      12*s,
      42*s
    );

    ctx.beginPath();

    ctx.arc(
      0,
      -42*s,
      31*s,
      Math.PI,
      Math.PI*2
    );

    ctx.fillStyle="#bd793b";

    ctx.fill();
  }

  ctx.restore();
}

/* =====================================================
   臭臭鼠
===================================================== */

function drawMouse(){

  /*
      玩家位置：

      真正跟随 playerZ。
      摄像机也跟随 playerZ。

      视觉上人物处于第三人称
      下方中央。
  */

  const x=
    lerp(
      roadX(
        1,
        G.playerZ
      ),
      roadX(
        G.targetLane,
        G.playerZ
      ),
      G.lane
    );

  const groundY=
    H*.84;

  const y=
    groundY-
    G.jumpY;

  /*
     跑步频率跟速度相关
  */

  const runSpeed=
    G.speed*.65;

  const step=
    Math.sin(
      G.time*.012+
      G.distance*
      runSpeed
    );

  const bounce=
    Math.abs(step)*4;

  ctx.save();

  ctx.translate(
    x,
    y-bounce
  );

  /*
     人物稍微向前倾
  */

  ctx.rotate(
    -0.035+
    Math.sin(
      G.time*.004
    )*.012
  );

  /*
     阴影
  */

  ctx.save();

  ctx.translate(
    0,
    G.jumpY*.25
  );

  ctx.scale(
    1,
    .25
  );

  ctx.fillStyle=
    "rgba(0,0,0,.30)";

  ctx.beginPath();

  ctx.ellipse(
    0,
    10,
    38,
    13,
    0,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.restore();

  /*
     尾巴
  */

  ctx.strokeStyle="#995f4b";
  ctx.lineWidth=7;
  ctx.lineCap="round";

  ctx.beginPath();

  ctx.moveTo(
    25,
    5
  );

  ctx.bezierCurveTo(
    55,
    -10,
    70+
      Math.sin(
        G.time*.01
      )*8,
    -38,
    91,
    -17
  );

  ctx.stroke();

  /*
     后腿交替
  */

  ctx.strokeStyle="#704334";
  ctx.lineWidth=9;

  ctx.beginPath();

  ctx.moveTo(
    -10,
    23
  );

  ctx.lineTo(
    -18-step*10,
    47
  );

  ctx.lineTo(
    -5-step*12,
    57
  );

  ctx.stroke();

  ctx.beginPath();

  ctx.moveTo(
    10,
    23
  );

  ctx.lineTo(
    18+step*10,
    47
  );

  ctx.lineTo(
    30+step*12,
    57
  );

  ctx.stroke();

  /*
     身体
  */

  ctx.fillStyle="#a86c4e";

  ctx.beginPath();

  ctx.ellipse(
    0,
    4,
    34,
    40,
    0,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     肚子
  */

  ctx.fillStyle="#d89a74";

  ctx.beginPath();

  ctx.ellipse(
    5,
    13,
    20,
    25,
    0,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     头
  */

  ctx.fillStyle="#b97857";

  ctx.beginPath();

  ctx.ellipse(
    -4,
    -35,
    33,
    30,
    0,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     耳朵
  */

  ctx.fillStyle="#8d5845";

  ctx.beginPath();

  ctx.arc(
    -29,
    -58,
    16,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.beginPath();

  ctx.arc(
    20,
    -59,
    16,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.fillStyle="#e5a18b";

  ctx.beginPath();

  ctx.arc(
    -29,
    -58,
    8,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.beginPath();

  ctx.arc(
    20,
    -59,
    8,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     眼睛
  */

  ctx.fillStyle="#171313";

  ctx.beginPath();

  ctx.arc(
    -14,
    -39,
    4.2,
    0,
    Math.PI*2
  );

  ctx.fill();

  ctx.beginPath();

  ctx.arc(
    9,
    -39,
    4.2,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     鼻子
  */

  ctx.fillStyle="#301a18";

  ctx.beginPath();

  ctx.arc(
    -2,
    -25,
    5,
    0,
    Math.PI*2
  );

  ctx.fill();

  /*
     嘴
  */

  ctx.strokeStyle="#4e2926";
  ctx.lineWidth=2;

  ctx.beginPath();

  ctx.arc(
    -2,
    -23,
    9,
    .15,
    1.1
  );

  ctx.stroke();

  /*
     前腿
  */

  ctx.strokeStyle="#704334";
  ctx.lineWidth=8;

  ctx.beginPath();

  ctx.moveTo(
    -19,
    18
  );

  ctx.lineTo(
    -29-step*9,
    42
  );

  ctx.lineTo(
    -17-step*11,
    51
  );

  ctx.stroke();

  ctx.beginPath();

  ctx.moveTo(
    17,
    18
  );

  ctx.lineTo(
    28+step*9,
    42
  );

  ctx.lineTo(
    39+step*11,
    51
  );

  ctx.stroke();

  ctx.restore();
}

/* =====================================================
   粒子
===================================================== */

function particle(
  x,
  y,
  color
){

  G.particles.push({

    x,
    y,

    vx:rand(-2,2),

    vy:rand(-4,-1),

    life:1,

    color

  });
}

function updateParticles(dt){

  for(
    let i=G.particles.length-1;
    i>=0;
    i--
  ){

    const p=
      G.particles[i];

    p.x+=
      p.vx;

    p.y+=
      p.vy;

    p.vy+=.15;

    p.life-=
      dt*2.2;

    if(p.life<=0){

      G.particles.splice(
        i,
        1
      );
    }
  }
}

function drawParticles(){

  for(
    const p of G.particles
  ){

    ctx.globalAlpha=
      p.life;

    ctx.fillStyle=
      p.color;

    ctx.beginPath();

    ctx.arc(
      p.x,
      p.y,
      3,
      0,
      Math.PI*2
    );

    ctx.fill();
  }

  ctx.globalAlpha=1;
}

/* =====================================================
   开始
===================================================== */

function resetGame(){

  G.distance=0;

  G.coins=0;

  G.lane=1;

  G.targetLane=1;

  G.playerZ=0;

  G.jumpY=0;

  G.jumpV=0;

  G.speed=7;

  G.objects=[];

  G.particles=[];

  G.spawnDistance=18;

  G.shake=0;

  G.magnet=0;

  G.shield=0;

  G.boost=0;

  G.introTime=0;

  G.state="intro";

  document
    .getElementById(
      "startScreen"
    )
    .style.display="none";

  document
    .getElementById(
      "gameOver"
    )
    .style.display="none";

  document
    .getElementById(
      "introText"
    )
    .style.opacity=0;
}

function startRun(){

  G.state="run";

  document
    .getElementById(
      "introText"
    )
    .style.opacity=0;
}

/* =====================================================
   开场
===================================================== */

function updateIntro(dt){

  G.introTime+=dt;

  const t=
    G.introTime;

  const text=
    document.getElementById(
      "introText"
    );

  /*
     老鼠向玩家方向跑来
  */

  if(t<1.2){

    text.style.opacity=0;

  }else if(t<1.5){

    text.style.opacity=
      (t-1.2)/.3;

  }else if(t<3){

    text.style.opacity=1;

  }else if(t<3.5){

    text.style.opacity=
      1-(t-3)/.5;

  }

  /*
     3.7秒正式开始
  */

  if(t>=3.7){

    startRun();
  }
}

/* =====================================================
   更新游戏
===================================================== */

function update(dt){

  if(G.state==="intro"){

    updateIntro(dt);

    return;
  }

  if(G.state!=="run"){

    return;
  }

  /*
     速度曲线
  */

  let targetSpeed;

  if(G.distance<100){

    targetSpeed=6.5;

  }else if(G.distance<300){

    targetSpeed=7.5;

  }else if(G.distance<600){

    targetSpeed=8.5;

  }else if(G.distance<1000){

    targetSpeed=9.7;

  }else{

    targetSpeed=11;
  }

  if(G.boost>0){

    targetSpeed*=1.35;

    G.boost-=dt;
  }

  G.speed=
    lerp(
      G.speed,
      targetSpeed,
      dt*2
    );

  /*
     真正的玩家前进
  */

  G.playerZ+=
    G.speed*dt;

  G.distance=
    G.playerZ;

  /*
     平滑换道
  */

  G.lane=
    lerp(
      G.lane,
      G.targetLane,
      dt*10
    );

  /*
     跳跃
  */

  if(
    G.jumpY>0 ||
    G.jumpV>0
  ){

    G.jumpV-=
      24*dt;

    G.jumpY+=
      G.jumpV*dt;

    if(G.jumpY<=0){

      G.jumpY=0;
      G.jumpV=0;
    }
  }

  /*
     按距离生成地图
  */

  if(
    G.playerZ>
    G.spawnDistance
  ){

    spawnPattern();

    G.spawnDistance+=
      Math.max(
        17,
        25-G.speed*.7
      );
  }

  /*
     磁铁
  */

  if(G.magnet>0){

    G.magnet-=dt;
  }

  /*
     护盾
  */

  if(G.shield>0){

    G.shield-=dt;
  }

  /*
     检查所有物体
  */

  for(
    let i=G.objects.length-1;
    i>=0;
    i--
  ){

    const o=
      G.objects[i];

    const relative=
      o.z-G.playerZ;

    /*
       太远的删除
    */

    if(relative<-8){

      G.objects.splice(
        i,
        1
      );

      continue;
    }

    /*
       碰撞区域
    */

    if(
      !o.collected &&
      Math.abs(relative)<1.4
    ){

      const laneDiff=
        Math.abs(
          o.lane-G.lane
        );

      /*
         磁铁自动吸金币
      */

      if(
        o.type==="coin" &&
        G.magnet>0 &&
        laneDiff<1.3
      ){

        collectCoin(o);

      }

      /*
         正常碰撞
      */

      else if(
        laneDiff<.34
      ){

        if(
          o.type==="coin"
        ){

          collectCoin(o);

        }

        else if(
          o.type==="boost"
        ){

          o.collected=true;

          G.boost=4;

          showPower(
            "⚡ 加速"
          );
        }

        else if(
          o.type==="magnet"
        ){

          o.collected=true;

          G.magnet=7;

          showPower(
            "🧲 磁铁"
          );
        }

        else if(
          o.type==="shield"
        ){

          o.collected=true;

          G.shield=8;

          showPower(
            "🛡️ 护盾"
          );
        }

        else if(
          o.type==="chest"
        ){

          o.collected=true;

          const amount=
            5+
            Math.floor(
              Math.random()*6
            );

          G.coins+=amount;

          for(
            let k=0;
            k<15;
            k++
          ){

            particle(
              roadX(
                o.lane,
                o.z
              ),
              worldY(o.z)-20,
              "#ffe05a"
            );
          }

        }

        else if(
          [
            "rock",
            "log",
            "barrier"
          ].includes(
            o.type
          )
        ){

          /*
             跳起来可以躲
          */

          if(
            G.jumpY<28
          ){

            if(G.shield>0){

              G.shield=0;

              o.collected=true;

              showPower(
                "护盾挡住了!"
              );

            }else{

              gameOver();

              return;
            }

          }
        }
      }
    }
  }

  updateParticles(dt);

  /*
     距离显示
  */

  document
    .getElementById(
      "distance"
    )
    .textContent=
    Math.floor(
      G.distance
    );

  document
    .getElementById(
      "coins"
    )
    .textContent=
    G.coins;
}

/* =====================================================
   收金币
===================================================== */

function collectCoin(o){

  if(o.collected)
    return;

  o.collected=true;

  G.coins++;

  const x=
    roadX(
      o.lane,
      o.z
    );

  const y=
    worldY(o.z)-20;

  for(
    let i=0;
    i<7;
    i++
  ){

    particle(
      x,
      y,
      "#ffe05b"
    );
  }
}

/* =====================================================
   道具提示
===================================================== */

let powerTimer=0;

function showPower(text){

  const el=
    document.getElementById(
      "power"
    );

  el.textContent=text;
  el.style.opacity=1;

  powerTimer=1.5;
}

/* =====================================================
   游戏结束
===================================================== */

function gameOver(){

  if(
    G.state==="over"
  ) return;

  G.state="over";

  best=Math.max(
    best,
    Math.floor(
      G.distance
    )
  );

  localStorage.setItem(
    "chouchoushu_best",
    best
  );

  document
    .getElementById(
      "finalDistance"
    )
    .textContent=
    Math.floor(
      G.distance
    );

  document
    .getElementById(
      "finalCoins"
    )
    .textContent=
    G.coins;

  document
    .getElementById(
      "gameOver"
    )
    .style.display="flex";
}

/* =====================================================
   绘制
===================================================== */

function draw(){

  ctx.clearRect(
    0,
    0,
    W,
    H
  );

  ctx.save();

  /*
     镜头轻微晃动
  */

  const runShake=
    G.state==="run"
      ? Math.sin(
          G.time*.012
        )*.8
      : 0;

  ctx.translate(
    runShake,
    Math.abs(runShake)*.3
  );

  drawSky();

  drawSea();

  drawIsland();

  drawRoad();

  drawEnvironment();

  /*
     远处物体先画
  */

  const visible=
    G.objects
      .filter(
        o=>{
          const d=
            o.z-G.playerZ;

          return (
            d>-2 &&
            d<90 &&
            !o.collected
          );
        }
      )
      .sort(
        (a,b)=>b.z-a.z
      );

  for(
    const o of visible
  ){

    drawObject(o);
  }

  /*
     玩家最后绘制
     保证臭臭鼠是主体
  */

  if(
    G.state==="run" ||
    G.state==="intro"
  ){

    drawMouse();
  }

  drawParticles();

  ctx.restore();

  /*
     道具提示计时
  */

  if(powerTimer>0){

    powerTimer-=.016;

    if(powerTimer<=0){

      document
        .getElementById(
          "power"
        )
        .style.opacity=0;
    }
  }
}

/* =====================================================
   主循环
===================================================== */

let lastTime=
  performance.now();

function loop(now){

  const dt=
    Math.min(
      .035,
      (now-lastTime)/1000
    );

  lastTime=now;

  G.time+=
    dt*1000;

  draw();

  update(dt);

  requestAnimationFrame(
    loop
  );
}

requestAnimationFrame(
  loop
);

/* =====================================================
   触摸操作
===================================================== */

let touchStartX=0;
let touchStartY=0;

canvas.addEventListener(
  "touchstart",
  e=>{

    const t=
      e.touches[0];

    touchStartX=
      t.clientX;

    touchStartY=
      t.clientY;

  },
  {passive:true}
);

canvas.addEventListener(
  "touchend",
  e=>{

    if(
      G.state!=="run"
    )
      return;

    const t=
      e.changedTouches[0];

    const dx=
      t.clientX-
      touchStartX;

    const dy=
      t.clientY-
      touchStartY;

    /*
       左右
    */

    if(
      Math.abs(dx)>
      Math.abs(dy)
    ){

      if(
        Math.abs(dx)>25
      ){

        if(dx>0){

          G.targetLane=
            Math.min(
              2,
              G.targetLane+1
            );

        }else{

          G.targetLane=
            Math.max(
              0,
              G.targetLane-1
            );
        }
      }

    }

    /*
       上滑
    */

    else if(
      dy<-30
    ){

      if(
        G.jumpY<=0 &&
        G.jumpV<=0
      ){

        G.jumpV=11.5;

        G.jumpY=1;
      }
    }

  },
  {passive:true}
);

/* =====================================================
   键盘
===================================================== */

window.addEventListener(
  "keydown",
  e=>{

    if(
      G.state!=="run"
    )
      return;

    if(
      e.key==="ArrowLeft" ||
      e.key==="a"
    ){

      G.targetLane=
        Math.max(
          0,
          G.targetLane-1
        );
    }

    if(
      e.key==="ArrowRight" ||
      e.key==="d"
    ){

      G.targetLane=
        Math.min(
          2,
          G.targetLane+1
        );
    }

    if(
      e.key==="ArrowUp" ||
      e.key==="w" ||
      e.key===" "
    ){

      if(
        G.jumpY<=0 &&
        G.jumpV<=0
      ){

        G.jumpV=11.5;
        G.jumpY=1;
      }
    }

  }
);

/* =====================================================
   按钮
===================================================== */

document
  .getElementById(
    "startButton"
  )
  .addEventListener(
    "click",
    resetGame
  );

document
  .getElementById(
    "restartButton"
  )
  .addEventListener(
    "click",
    resetGame
  );

</script>

</body>
</html>
