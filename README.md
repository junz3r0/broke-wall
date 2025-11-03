<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Breakout - Jun's Mini Games</title>
  <style>
    :root{--bg:#0b1220;--panel:#0f1724;--accent:#f6c85f}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, Roboto, "Noto Sans KR", sans-serif;background:var(--bg);color:#e6eef8}
    .wrap{display:flex;align-items:center;justify-content:center;height:100%;padding:20px;box-sizing:border-box}
    .card{width:900px;max-width:100%;background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);border-radius:12px;padding:18px;box-shadow:0 8px 30px rgba(2,6,23,0.7)}
    header{display:flex;align-items:center;justify-content:space-between;gap:12px;margin-bottom:10px}
    h1{font-size:18px;margin:0}
    .controls{display:flex;gap:8px;align-items:center}
    button{background:var(--panel);color:inherit;border:1px solid rgba(255,255,255,0.04);padding:8px 10px;border-radius:8px;cursor:pointer}
    button:active{transform:translateY(1px)}
    canvas{display:block;background:#071022;border-radius:8px;width:100%;height:520px}
    .meta{display:flex;gap:14px;margin-top:10px;align-items:center}
    .badge{background:rgba(255,255,255,0.02);padding:6px 10px;border-radius:8px;font-size:13px}
    /* modal */
    .modal{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(2,6,23,0.7);visibility:hidden;opacity:0;transition:opacity .15s}
    .modal.show{visibility:visible;opacity:1}
    .modal .panel{background:white;color:#001; padding:18px;border-radius:10px;max-width:90%;max-height:90%;overflow:auto;box-shadow:0 16px 40px rgba(2,6,23,0.6)}
    .modal img{max-width:100%;height:auto;display:block;border-radius:8px}
    .easter{font-size:13px;color:rgba(255,255,255,0.7)}
    footer{margin-top:12px;font-size:13px;color:rgba(255,255,255,0.6)}
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
          <div class="badge">이스터에그: 오른쪽 맨砖을 4번 맞추면 특별한 이미지가 뜹니다.</div>
        </div>
      </header>

      <canvas id="game" width="860" height="520"></canvas>
      <div class="meta">
        <div class="badge">Lives: <span id="lives">3</span></div>
        <div class="badge">Score: <span id="score">0</span></div>
        <div class="badge">Right-brick hits left: <span id="rbhits">4</span></div>
        <div class="easter">힌트: 패들을 마우스로 움직이세요. 모바일도 지원됩니다.</div>
      </div>
      <footer>원하시면 이 파일을 그대로 GitHub에 올리면 홈페이지로 동작합니다.</footer>
    </div>
  </div>

  <!-- modal for easter egg -->
  <div id="modal" class="modal" role="dialog" aria-hidden="true">
    <div class="panel">
      <button id="closeModal" style="float:right">닫기</button>
      <h3>이스터에그!</h3>
      <img id="eggImg" src="data:image/png;base64,REPLACE_BASE64" alt="easter cat"/>
    </div>
  </div>

  <script>
  // --------- Game setup ---------
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const W = canvas.width; const H = canvas.height;
  let running = false;

  // paddle
  const paddle = {w:120,h:14,x:(W-120)/2,y:H-40,speed:0};

  // ball
  const ball = {r:8,x:W/2,y:H/2,vx:3,vy:-3};

  // bricks configuration
  const rows = 5; const cols = 10; const brickW = Math.floor((W-60)/cols); const brickH = 20; const brickPadding=6; const offsetTop=50; const offsetLeft=30;
  let bricks = [];

  // rightmost brick column index
  const rightCol = cols-1;
  // We will make the rightmost brick have 4 hits to reveal the easter egg.

  function initBricks(){
    bricks = [];
    for(let r=0;r<rows;r++){
      bricks[r]=[];
      for(let c=0;c<cols;c++){
        const hits = (c===rightCol)?4:1; // rightmost column needs 4 hits
        bricks[r][c] = {x:offsetLeft + c*(brickW+brickPadding), y:offsetTop + r*(brickH+brickPadding), w:brickW, h:brickH, alive:true, hits:hits};
      }
    }
    updateRBDisplay();
  }

  initBricks();

  let lives = 3; let score = 0;
  document.getElementById('lives').innerText = lives;
  document.getElementById('score').innerText = score;

  // mouse control
  canvas.addEventListener('mousemove', (e)=>{
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    paddle.x = Math.max(0, Math.min(W - paddle.w, mx - paddle.w/2));
  });
  // touch
  canvas.addEventListener('touchmove', (e)=>{e.preventDefault(); const rect=canvas.getBoundingClientRect(); const t=e.touches[0]; const mx=t.clientX-rect.left; paddle.x=Math.max(0,Math.min(W-paddle.w,mx-paddle.w/2));});

  // controls
  document.getElementById('reset').addEventListener('click', resetGame);
  document.getElementById('toggle').addEventListener('click', ()=>{running=!running; document.getElementById('toggle').innerText = running? '일시정지':'시작'; if(running) loop();});
  window.addEventListener('keydown',(e)=>{ if(e.code==='Space'){ e.preventDefault(); running=!running; document.getElementById('toggle').innerText = running? '일시정지':'시작'; if(running) loop(); }});

  // modal
  const modal = document.getElementById('modal');
  document.getElementById('closeModal').addEventListener('click', ()=>{modal.classList.remove('show'); modal.setAttribute('aria-hidden','true');});

  function showEaster(){ modal.classList.add('show'); modal.setAttribute('aria-hidden','false'); }

  function resetBall(){ ball.x = W/2; ball.y = H/2; ball.vx = 3*(Math.random()>0.5?1:-1); ball.vy = -3; }

  function resetGame(){ lives=3; score=0; document.getElementById('lives').innerText = lives; document.getElementById('score').innerText = score; initBricks(); resetBall(); running=false; document.getElementById('toggle').innerText='시작'; }

  // collision helpers
  function rectsIntersect(ax,ay,aw,ah,bx,by,bw,bh){ return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by; }

  function update(){
    if(!running) return;
    ball.x += ball.vx; ball.y += ball.vy;

    // wall collisions
    if(ball.x - ball.r < 0){ ball.x = ball.r; ball.vx *= -1; }
    if(ball.x + ball.r > W){ ball.x = W - ball.r; ball.vx *= -1; }
    if(ball.y - ball.r < 0){ ball.y = ball.r; ball.vy *= -1; }

    // paddle collision
    if(ball.y + ball.r >= paddle.y && ball.y + ball.r <= paddle.y + paddle.h && ball.x >= paddle.x && ball.x <= paddle.x + paddle.w){
      // reflect with angle based on where it hit the paddle
      const hitPos = (ball.x - (paddle.x + paddle.w/2)) / (paddle.w/2);
      ball.vx = 5 * hitPos;
      ball.vy = -Math.abs(ball.vy);
    }

    // bottom (lose life)
    if(ball.y - ball.r > H){ lives--; document.getElementById('lives').innerText = lives; if(lives<=0){ alert('게임 오버. 리셋합니다.'); resetGame(); } else { resetBall(); running=false; document.getElementById('toggle').innerText='시작'; }}

    // brick collisions
    outer: for(let r=0;r<rows;r++){
      for(let c=0;c<cols;c++){
        const b = bricks[r][c];
        if(!b.alive) continue;
        if(rectsIntersect(ball.x-ball.r, ball.y-ball.r, ball.r*2, ball.r*2, b.x, b.y, b.w, b.h)){
          // simple reflection
          ball.vy *= -1;
          // register hit
          b.hits -= 1;
          // if rightmost column and was hit, update tracker
          if(c===rightCol){ updateRBDisplay(); }
          if(b.hits <= 0){ b.alive = false; score += 100; document.getElementById('score').innerText = score; }
          // check easter egg condition: if the specific rightmost brick (we'll choose bottom-most one in right column) becomes destroyed, show image
          // We'll check if any brick in right column has hits >0; when the special one reaches 0, trigger.
          checkEasterTrigger();
          break outer; // avoid multiple collisions in single update
        }
      }
    }
  }

  // We'll designate the brick at the bottom row, rightmost column as the "special" one.
  function getSpecialBrick(){ return bricks[rows-1][rightCol]; }

  function updateRBDisplay(){ const sb = getSpecialBrick(); const left = sb.hits>0?sb.hits:0; document.getElementById('rbhits').innerText = left; }

  function checkEasterTrigger(){ const sb = getSpecialBrick(); if(!sb.alive){ // triggered when special brick destroyed
      // show modal
      showEaster();
    }}

  function draw(){
    // bg
    ctx.clearRect(0,0,W,H);
    // draw bricks
    for(let r=0;r<rows;r++){
      for(let c=0;c<cols;c++){
        const b = bricks[r][c];
        if(!b.alive) continue;
        // color by column to highlight rightmost
        ctx.save();
        if(c===rightCol){ ctx.fillStyle = 'rgba(246,200,95,0.95)'; }
        else { ctx.fillStyle = 'rgba(100,150,240,0.9)'; }
        ctx.fillRect(b.x, b.y, b.w, b.h);
        // show hits left if >1
        if(b.hits>1){ ctx.fillStyle='rgba(0,0,0,0.6)'; ctx.font='12px sans-serif'; ctx.fillText(b.hits, b.x + b.w - 18, b.y + 14); }
        ctx.restore();
      }
    }

    // paddle
    ctx.fillStyle='rgba(200,200,220,0.95)'; ctx.fillRect(paddle.x, paddle.y, paddle.w, paddle.h);
    // ball
    ctx.beginPath(); ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); ctx.fill();
  }

  function loop(){ if(!running) return; update(); draw(); requestAnimationFrame(loop); }

  // start paused
  resetBall(); draw();

  // initial draw loop to show things even when not running
  draw();

  // expose convenience: click canvas to toggle start
  canvas.addEventListener('click', ()=>{ running = !running; document.getElementById('toggle').innerText = running? '일시정지':'시작'; if(running) loop(); });

  // small accessibility: show image also if user presses the 'E' key
  window.addEventListener('keydown', (e)=>{ if(e.key.toLowerCase()==='e'){ showEaster(); } });

  </script>
</body>
</html>
