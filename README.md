<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Breakout - Jun's Mini Games</title>
  <style>
    :root { --bg:#0b1220; --panel:#0f1724; --accent:#f6c85f; }
    html, body {
      height: 100%; margin: 0;
      font-family: Inter, system-ui, -apple-system, Roboto, "Noto Sans KR", sans-serif;
      background: var(--bg); color: #e6eef8;
    }
    .wrap { display:flex; align-items:center; justify-content:center; height:100%; padding:20px; box-sizing:border-box; }
    .card {
      width:900px; max-width:100%;
      background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);
      border-radius:12px; padding:18px;
      box-shadow:0 8px 30px rgba(2,6,23,0.7);
    }
    header { display:flex; align-items:center; justify-content:space-between; gap:12px; margin-bottom:10px; }
    h1 { font-size:18px; margin:0; }
    .controls { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
    button {
      background:var(--panel); color:inherit;
      border:1px solid rgba(255,255,255,0.04);
      padding:8px 10px; border-radius:8px; cursor:pointer;
    }
    button:active { transform:translateY(1px); }
    canvas { display:block; background:#071022; border-radius:8px; width:100%; height:520px; }
    .meta { display:flex; gap:14px; margin-top:10px; align-items:center; flex-wrap:wrap; }
    .badge { background:rgba(255,255,255,0.02); padding:6px 10px; border-radius:8px; font-size:13px; }
    footer { margin-top:12px; font-size:13px; color:rgba(255,255,255,0.6); }
    .easter { font-size:13px; color:rgba(255,255,255,0.7); }

    /* Modal */
    .modal {
      position:fixed; inset:0;
      display:flex; align-items:center; justify-content:center;
      background:rgba(2,6,23,0.7);
      visibility:hidden; opacity:0; transition:opacity .15s;
    }
    .modal.show { visibility:visible; opacity:1; }
    .modal .panel {
      background:white; color:#001; padding:18px;
      border-radius:10px; max-width:90%; max-height:90%;
      overflow:auto; box-shadow:0 16px 40px rgba(2,6,23,0.6);
    }
    .modal img { max-width:100%; height:auto; display:block; border-radius:8px; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <header>
        <h1>벽돌깨기 (Breakout) — 마우스로 패들 이동, 스페이스로 시작/일시정지</h1>
        <div class="controls">
          <button id="reset">리셋</button>
          <button id="toggle">시작</button>
          <div class="badge">이스터에그: 오른쪽 맨 벽돌을 4번 맞추면 이미지가 뜹니다.</div>
        </div>
      </header>

      <canvas id="game" width="860" height="520"></canvas>
      <div class="meta">
        <div class="badge">Lives: <span id="lives">3</span></div>
        <div class="badge">Score: <span id="score">0</span></div>
        <div class="badge">Right-brick hits left: <span id="rbhits">4</span></div>
        <div class="easter">힌트: 마우스로 패들을 움직이세요. 모바일도 지원됩니다.</div>
      </div>
      <footer>이 파일을 GitHub에 올리면 자동으로 웹페이지로 작동합니다.</footer>
    </div>
  </div>

  <!-- 이스터에그 모달 -->
  <div id="modal" class="modal" role="dialog" aria-hidden="true">
    <div class="panel">
      <button id="closeModal" style="float:right">닫기</button>
      <h3>이스터에그!</h3>
      <img id="eggImg" src="cat.png" alt="easter cat"/>
    </div>
  </div>

  <script>
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  let running = false;

  const paddle = { w:120, h:14, x:(W-120)/2, y:H-40 };
  const ball = { r:8, x:W/2, y:H/2, vx:3, vy:-3 };
  const rows=5, cols=10, brickW=Math.floor((W-60)/cols), brickH=20, brickPadding=6, offsetTop=50, offsetLeft=30;
  const rightCol = cols-1;
  let bricks=[], lives=3, score=0;

  function initBricks(){
    bricks=[];
    for(let r=0;r<rows;r++){
      bricks[r]=[];
      for(let c=0;c<cols;c++){
        const hits=(c===rightCol)?4:1;
        bricks[r][c]={x:offsetLeft+c*(brickW+brickPadding),y:offsetTop+r*(brickH+brickPadding),w:brickW,h:brickH,alive:true,hits:hits};
      }
    }
    updateRBDisplay();
  }

  function resetBall(){
    ball.x=W/2; ball.y=H/2;
    ball.vx=3*(Math.random()>0.5?1:-1);
    ball.vy=-3;
  }

  function resetGame(){
    lives=3; score=0;
    document.getElementById('lives').innerText=lives;
    document.getElementById('score').innerText=score;
    initBricks(); resetBall();
    running=false;
    document.getElementById('toggle').innerText='시작';
  }

  function rectsIntersect(ax,ay,aw,ah,bx,by,bw,bh){
    return ax<bx+bw && ax+aw>bx && ay<by+bh && ay+ah>by;
  }

  function update(){
    if(!running) return;
    ball.x+=ball.vx; ball.y+=ball.vy;

    if(ball.x-ball.r<0){ball.x=ball.r;ball.vx*=-1;}
    if(ball.x+ball.r>W){ball.x=W-ball.r;ball.vx*=-1;}
    if(ball.y-ball.r<0){ball.y=ball.r;ball.vy*=-1;}

    if(ball.y+ball.r>=paddle.y && ball.y+ball.r<=paddle.y+paddle.h && ball.x>=paddle.x && ball.x<=paddle.x+paddle.w){
      const hitPos=(ball.x-(paddle.x+paddle.w/2))/(paddle.w/2);
      ball.vx=5*hitPos;
      ball.vy=-Math.abs(ball.vy);
    }

    if(ball.y-ball.r>H){
      lives--; document.getElementById('lives').innerText=lives;
      if(lives<=0){ alert('게임 오버!'); resetGame(); }
      else{ resetBall(); running=false; document.getElementById('toggle').innerText='시작'; }
    }

    outer: for(let r=0;r<rows;r++){
      for(let c=0;c<cols;c++){
        const b=bricks[r][c];
        if(!b.alive) continue;
        if(rectsIntersect(ball.x-ball.r,ball.y-ball.r,ball.r*2,ball.r*2,b.x,b.y,b.w,b.h)){
          ball.vy*=-1;
          b.hits--;
          if(c===rightCol) updateRBDisplay();
          if(b.hits<=0){
            b.alive=false;
            score+=100;
            document.getElementById('score').innerText=score;
          }
          checkEasterTrigger();
          break outer;
        }
      }
    }
  }

  function getSpecialBrick(){ return bricks[rows-1][rightCol]; }
  function updateRBDisplay(){
    const sb=getSpecialBrick();
    const left=sb.hits>0?sb.hits:0;
    document.getElementById('rbhits').innerText=left;
  }
  function checkEasterTrigger(){
    const sb=getSpecialBrick();
    if(!sb.alive) showEaster();
  }

  function draw(){
    ctx.clearRect(0,0,W,H);
    for(let r=0;r<rows;r++){
      for(let c=0;c<cols;c++){
        const b=bricks[r][c];
        if(!b.alive) continue;
        ctx.save();
        ctx.fillStyle=(c===rightCol)?'rgba(246,200,95,0.95)':'rgba(100,150,240,0.9)';
        ctx.fillRect(b.x,b.y,b.w,b.h);
        if(b.hits>1){
          ctx.fillStyle='rgba(0,0,0,0.6)';
          ctx.font='12px sans-serif';
          ctx.fillText(b.hits,b.x+b.w-18,b.y+14);
        }
        ctx.restore();
      }
    }
    ctx.fillStyle='rgba(200,200,220,0.95)';
    ctx.fillRect(paddle.x,paddle.y,paddle.w,paddle.h);
    ctx.beginPath();
    ctx.arc(ball.x,ball.y,ball.r,0,Math.PI*2);
    ctx.fill();
  }

  function loop(){ if(!running) return; update(); draw(); requestAnimationFrame(loop); }

  canvas.addEventListener('mousemove',e=>{
    const rect=canvas.getBoundingClientRect();
    const mx=e.clientX-rect.left;
    paddle.x=Math.max(0,Math.min(W-paddle.w,mx-paddle.w/2));
  });
  canvas.addEventListener('touchmove',e=>{
    e.preventDefault();
    const rect=canvas.getBoundingClientRect();
    const t=e.touches[0];
    const mx=t.clientX-rect.left;
    paddle.x=Math.max(0,Math.min(W-paddle.w,mx-paddle.w/2));
  });

  document.getElementById('reset').addEventListener('click',resetGame);
  document.getElementById('toggle').addEventListener('click',()=>{
    running=!running;
    document.getElementById('toggle').innerText=running?'일시정지':'시작';
    if(running) loop();
  });

  const modal=document.getElementById('modal');
  document.getElementById('closeModal').addEventListener('click',()=>{
    modal.classList.remove('show');
    modal.setAttribute('aria-hidden','true');
  });

  function showEaster(){
    modal.classList.add('show');
    modal.setAttribute('aria-hidden','false');
  }

  resetGame(); draw();
  </script>
</body>
</html>
