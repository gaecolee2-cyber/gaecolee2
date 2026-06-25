<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<!-- [#7] zoom 허용: user-scalable=no / maximum-scale 제거 (WCAG) -->
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<!-- [#17] CSP 하드닝 기준선. MediaPipe는 blob worker + wasm(eval) + jsdelivr fetch를 사용하므로
     아래 소스가 필요합니다. 카메라/모델 로드가 실패하면 이 메타를 제거하세요(환경 민감). -->
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' blob: https://cdn.jsdelivr.net https://unpkg.com; worker-src 'self' blob:; child-src blob:; style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net https://unpkg.com https://fonts.googleapis.com; font-src 'self' data: https://fonts.gstatic.com https://cdn.jsdelivr.net https://unpkg.com; img-src 'self' data: blob:; media-src 'self' blob:; connect-src 'self' https://cdn.jsdelivr.net https://unpkg.com; manifest-src 'self' blob:; frame-src blob:;" />
<meta name="theme-color" content="#0a0a0b" />
<meta name="mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<title>협응성 청기백기 — MOTION-SENSING GAME</title>
<!-- [#6] CDN 버전 고정. (SRI integrity는 배포 시 실제 해시를 산출해 추가하세요:
     openssl dgst -sha384 -binary file.js | openssl base64 -A) -->
<!-- [수정#1,#20] CDN 단일 의존 제거: jsdelivr 실패 시 unpkg 자동 폴백.
     로드 완료를 window.__mpReady(Promise)로 알려 초기화 타이밍 보장.
     locateFile(wasm/model)도 실제로 성공한 CDN 기준으로 자동 정렬. -->
<script>
window.__MP=(function(){
  var CDNS=["https://cdn.jsdelivr.net/npm","https://unpkg.com"];
  var HANDS="@mediapipe/hands@0.4.1675469240";
  var CAMU="@mediapipe/camera_utils@0.3.1675466862";
  var handsBase=CDNS[0]+"/"+HANDS+"/";
  function loadOne(path, ok){
    var i=0;
    (function next(){
      if(i>=CDNS.length){ ok(false); return; }
      var base=CDNS[i++], s=document.createElement("script");
      s.src=base+"/"+path; s.crossOrigin="anonymous";
      s.onload=function(){ ok(true, base); };
      s.onerror=function(){ s.remove(); next(); };
      document.head.appendChild(s);
    })();
  }
  var ready=new Promise(function(resolve){
    loadOne(CAMU+"/camera_utils.js", function(okC){
      loadOne(HANDS+"/hands.js", function(okH, base){
        if(okH && base){ handsBase=base+"/"+HANDS+"/"; }
        window.__MP_HANDS_BASE=handsBase;
        resolve(!!(okC && okH && typeof window.Hands!=="undefined" && typeof window.Camera!=="undefined"));
      });
    });
  });
  window.__MP_HANDS_BASE=handsBase;
  return { ready:ready };
})();
window.__mpReady=window.__MP.ready;
</script>
<style>
  @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css');
  @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@500;900&family=Black+Han+Sans&display=swap');

  :root{
    --bg:#0a0a0b; --panel:#161618; --border:#2a2a2e; --muted:#9b9ba3;
    --primary:#facc15; --blue:#2563eb; --white:#f4f4f5; --ok:#10b981; --bad:#ef4444;
    --combo:#fb923c; --xp:#a78bfa; --fever:#f43f5e;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  html,body{height:100%;overflow:hidden}
  body{background:var(--bg);color:#fff;font-family:'Pretendard',system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;display:flex;justify-content:center;overflow:hidden;overscroll-behavior:none;-webkit-text-size-adjust:100%}
  #app{width:100%;max-width:960px;height:100vh;height:100dvh;padding:calc(6px + env(safe-area-inset-top)) calc(8px + env(safe-area-inset-right)) calc(6px + env(safe-area-inset-bottom)) calc(8px + env(safe-area-inset-left));display:flex;flex-direction:column;gap:6px}
  h1{text-align:center;font-size:clamp(21px,4.8vw,30px);font-weight:900;flex:0 0 auto}

  /* [#9] 스크린리더 전용 라이브 리전 */
  .sr-only{position:absolute;width:1px;height:1px;padding:0;margin:-1px;overflow:hidden;clip:rect(0,0,0,0);white-space:nowrap;border:0}

  /* ---- top bar ---- */
  #topbar{display:flex;align-items:center;justify-content:space-between;gap:6px;background:var(--panel);border:1px solid var(--border);border-radius:12px;padding:6px 12px;flex:0 0 auto}
  #fsBtn{background:transparent;border:1px solid var(--border);color:#fff;border-radius:8px;padding:5px 9px;font-size:21px;font-weight:700;cursor:pointer;line-height:1}
  #topbar .rnd{font-weight:800;font-size:18px;white-space:nowrap}
  #topbar .lives{font-size:21px;letter-spacing:1px;white-space:nowrap}
  #topbar .time{font-weight:900;font-size:clamp(27px,7.5vw,36px);font-variant-numeric:tabular-nums}
  #topbar .time.warn{color:var(--bad)}
  #topbar .scr{font-weight:900;font-size:19.5px;color:var(--primary);white-space:nowrap}
  #roundbar{height:5px;width:100%;background:#27272a;border-radius:99px;overflow:hidden;flex:0 0 auto}
  #roundfill{height:100%;width:100%;background:var(--primary);border-radius:99px;transition:width .1s linear}

  /* top control bar */
  #ctrlbar{display:flex;align-items:center;justify-content:space-between;gap:6px;flex-wrap:nowrap;flex:0 0 auto}
  .stylewrap{display:flex;align-items:center;gap:6px;flex-wrap:nowrap}
  .stylelabel{font-size:18px;font-weight:800;color:var(--muted);white-space:nowrap}
  .stylebar{display:flex;gap:5px;flex-wrap:nowrap}
  .stylebar button{background:var(--panel);color:#cbd5e1;border:1px solid var(--border);border-radius:99px;padding:6px 12px;font-size:18px;font-weight:800;line-height:1;box-shadow:none;cursor:pointer;transition:background .15s,color .15s,border-color .15s,transform .1s;white-space:nowrap}
  .stylebar button:hover{transform:translateY(-1px)}
  .stylebar button.sel{background:linear-gradient(135deg,#facc15,#f59e0b);color:#111;border-color:transparent}
  .gamectrls{display:flex;gap:5px;flex-wrap:nowrap;justify-content:flex-end}
  .gamectrls button{border-radius:99px;padding:6px 12px;font-size:18px;font-weight:800;line-height:1;cursor:pointer;box-shadow:none;white-space:nowrap}
  .gamectrls button:disabled{opacity:0.4;cursor:not-allowed;transform:none !important}
  #pauseToggle{background:linear-gradient(135deg,#facc15,#f59e0b);color:#111;border:0}
  .ctrl-sec{background:var(--panel);color:#fff;border:1px solid var(--border)}
  .ctrl-sec:hover,#pauseToggle:hover{transform:translateY(-1px)}
  @media (max-width:640px){
    .stylelabel{display:none}
    .stylebar button{padding:6px 9px;font-size:16.5px}
    .gamectrls button{padding:6px 9px;font-size:16.5px}
    /* 좁은 화면: 컨트롤이 넘치면 줄바꿈 허용 */
    #ctrlbar{gap:4px;flex-wrap:wrap;justify-content:center}
    .stylewrap,.stylebar,.gamectrls{flex-wrap:wrap;justify-content:center}
  }
  /* 초소형 폭: 폰트/패딩 추가 축소 */
  @media (max-width:420px){
    .stylebar button,.gamectrls button{padding:5px 8px;font-size:14px}
    #topbar{padding:5px 9px;gap:4px}
    #topbar .rnd{font-size:14px}
    #topbar .lives{font-size:17px}
    #topbar .scr{font-size:15px}
    #fsBtn{padding:4px 7px;font-size:17px}
  }
  /* 가로(landscape)·낮은 높이: 제목 숨겨 스테이지 공간 확보 */
  @media (max-height:520px){
    h1{display:none}
    #topbar{padding:4px 10px}
  }

  /* 인트로 룰셋 구조 */
  .rules2{max-width:540px;width:100%;text-align:left;display:flex;flex-direction:column;gap:5px;font-size:clamp(16.5px, 3.75vw, 19.5px);color:#d4d4d8;line-height:1.4;margin:2px auto;padding:0 10px}
  .rules2 .rsec{font-weight:900;color:#fff;margin-top:4px;font-size:clamp(18px, 4.2vw, 21px);border-left:3px solid var(--primary);padding-left:6px}
  .rules2 ul{display:grid;gap:2px;padding:0;margin:0}
  .rules2 li{list-style:none;padding-left:14px;position:relative}
  .rules2 li::before{content:"▸";position:absolute;left:2px;color:var(--primary)}

  /* ---- stage ---- */
  #stage{position:relative;width:100%;flex:1 1 auto;min-height:0;border:1px solid var(--border);border-radius:14px;overflow:hidden;background:#000;touch-action:none}
  #vid{position:absolute;inset:0;width:100%;height:100%;object-fit:cover;opacity:0;pointer-events:none}
  #cv{position:absolute;inset:0;width:100%;height:100%;display:block;z-index:1;transition:transform .4s cubic-bezier(.175,.885,.32,1.275)}
  #cv.flipX{transform:scaleX(-1)}
  #cv.flipY{transform:scaleY(-1)}
  #cv.flipX.flipY{transform:scaleX(-1) scaleY(-1)}

  #badge{position:absolute;top:46px;left:50%;transform:translateX(-50%);z-index:3;font-weight:500;font-size:clamp(15px,4vw,19px);padding:5px 13px;border-radius:99px;display:none;text-shadow:0 1px 3px rgba(0,0,0,.8);max-width:86%;white-space:nowrap;text-align:center}
  #badge.bonus{background:rgba(16,185,129,.85);color:#03130d}
  #badge.boss{background:rgba(239,68,68,.9);color:#fff}
  #badge.rev{background:rgba(250,204,21,.95);color:#111;animation:pop .25s ease}
  /* [요청1] 중앙 마스코트 이모지 제거 */

  /* 다이내믹 콤보 인터랙티브 HUD 레이아웃 */
  #comboHud{position:absolute;top:90px;left:50%;transform:translateX(-50%);z-index:2;font-weight:500;color:var(--combo);text-shadow:0 2px 10px rgba(0,0,0,1);font-size:clamp(22px,6.5vw,34px);display:none;white-space:nowrap}
  #comboHud.pop-active{animation:comboPop 0.35s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards}
  @keyframes comboPop{
    0%{transform:translateX(-50%) scale(0.6);opacity:0.2}
    50%{transform:translateX(-50%) scale(1.25);filter:brightness(1.2)}
    100%{transform:translateX(-50%) scale(1);opacity:1}
  }

  /* 고가시성 상단 좌우 대칭형 지표 */
  .stage-hud-item{position:absolute;top:12px;z-index:2;font-weight:900;font-size:clamp(18px, 4.2vw, 22.5px);padding:6px 14px;border-radius:8px;background:rgba(22,22,24,0.85);border:1px solid rgba(42,42,46,0.7);text-shadow:0 1px 4px rgba(0,0,0,0.9);display:none}
  #hudTaskCount{left:12px;color:#fff}
  #hudAccuracy{right:12px}
  .hud-pass{color:var(--ok) !important}
  .hud-fail{color:var(--bad) !important}

  #centerText{position:absolute;inset:0;z-index:2;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:4px;pointer-events:none;text-align:center;padding:12px}
  #centerText.hidden{display:none}
  #centerSub{font-size:clamp(19.5px,4.5vw,27px);font-weight:700;color:#e4e4e7;text-shadow:0 2px 8px rgba(0,0,0,.9)}
  #centerBig{font-size:clamp(42px,13.5vw,108px);font-weight:900;line-height:1.05;text-shadow:0 3px 14px rgba(0,0,0,.95);white-space:nowrap;max-width:100%}
  #centerBig.ok{color:var(--ok)} #centerBig.bad{color:var(--bad)} #centerBig.rev{color:var(--primary)}

  /* 스크롤 제거형 컴팩트 오버레이 */
  /* [수정#3] 콘텐츠가 스테이지 높이를 넘으면 시작 버튼이 잘려 보이지 않고, #stage{touch-action:none}로
     손가락 스크롤도 막혔다. → 오버레이 자체를 세로 스크롤 컨테이너로 전환:
     - overflow-y:auto + touch-action:pan-y 로 부모(none)를 덮어써 터치 스크롤 복구
     - justify-content:safe center 로 넘칠 때 상단이 잘리지 않게(닿을 수 있게) 폴백
     - overscroll-behavior:contain 으로 바운스 전파 차단 */
  .overlay{position:absolute;inset:0;z-index:3;display:flex;flex-direction:column;align-items:center;justify-content:safe center;gap:8px;background:rgba(6,6,8,.94);text-align:center;padding:16px 24px;overflow-y:auto;overflow-x:hidden;touch-action:pan-y;-webkit-overflow-scrolling:touch;overscroll-behavior:contain}
  /* 구형 브라우저(safe 미지원) 폴백: 첫/끝 요소 auto 마진으로 '맞으면 중앙, 넘치면 스크롤' */
  .overlay>*{flex-shrink:0}
  .overlay>:first-child{margin-top:auto}
  .overlay>:last-child{margin-bottom:auto}
  .overlay.hidden{display:none}
  .overlay h2{font-size:clamp(24px,6.75vw,33px);font-weight:900;margin-bottom:1px;color:var(--primary)}
  button{background:var(--primary);color:#111;border:0;border-radius:10px;padding:10px 24px;font-size:21px;font-weight:800;cursor:pointer;transition:transform 0.1s;flex-shrink:0}
  button:active{transform:scale(0.98)}
  button:disabled{opacity:.5;cursor:not-allowed}
  .btnrow{display:flex;gap:8px;flex-wrap:wrap;justify-content:center;flex-shrink:0}
  .btn-secondary{background:var(--panel);color:#fff;border:1px solid var(--border)}
  .field{display:flex;gap:6px;align-items:center;justify-content:center;flex-shrink:0}
  .field input{background:#0e0e10;border:1px solid var(--border);color:#fff;border-radius:8px;padding:8px 12px;font-size:21px;font-weight:700;max-width:150px;text-align:center}

  /* 10단위 정렬 전용 그리드 레이아웃 */
  .roundgrid{display:grid;grid-template-columns:repeat(10, minmax(0, 1fr));gap:5px;width:100%;max-width:540px;padding:2px;flex-shrink:1;overflow:hidden}
  .roundgrid button{padding:6px 0;font-size:18px;font-weight:700;border-radius:6px;width:100%}
  .roundgrid button.r5{background:var(--blue);color:#fff}
  .skinrow{display:flex;gap:6px;flex-wrap:wrap;justify-content:center;max-width:540px;flex-shrink:0}
  .skinrow .skin{padding:6px 12px;font-size:17.25px;border-radius:99px;border:1px solid var(--border);background:var(--panel);color:#fff;display:flex;gap:5px;align-items:center}
  .skinrow .skin.sel{outline:2px solid var(--primary)}
  .skinrow .skin .dot{width:8px;height:8px;border-radius:99px;display:inline-block}
  .metaline{font-size:17.25px;color:var(--muted);flex-shrink:0}
  .metaline b{color:var(--xp)}
  .xpbar{width:200px;max-width:50vw;height:6px;background:#27272a;border-radius:99px;overflow:hidden;flex-shrink:0}
  .xpbar>i{display:block;height:100%;background:var(--xp);border-radius:99px}

  /* ---- end screen ---- */
  .statgrid{display:flex;flex-wrap:wrap;gap:8px;justify-content:center;margin:2px 0;flex-shrink:0}
  .stat{background:var(--panel);border:1px solid var(--border);border-radius:8px;padding:8px 14px;min-width:86px}
  .stat .k{font-size:15px;color:var(--muted)}
  .stat .v{font-size:27px;font-weight:900}
  .stars{font-size:36px;letter-spacing:1px;line-height:1}
  table{border-collapse:collapse;width:100%;max-width:500px;font-size:17.25px;flex-shrink:1;overflow:hidden}
  th,td{padding:5px 6px;border-bottom:1px solid var(--border);text-align:center}
  th{color:var(--muted);font-weight:700}
  tr.me td{color:var(--primary);font-weight:800}

  /* ---- panel ---- */
  #panel{background:var(--panel);border:1px solid var(--border);border-radius:12px;padding:10px;display:grid;gap:6px;flex:0 0 auto}
  #panel.hidden{display:none}
  #cmdtimer{height:8px;width:100%;background:#27272a;border-radius:99px;overflow:hidden}
  #cmdfill{height:100%;width:100%;background:#3b82f6;border-radius:99px}
  #fatal{display:none;color:var(--bad);text-align:center;padding:12px;font-size:18.75px;line-height:1.5;background:var(--panel);border:1px solid var(--bad);border-radius:12px}

  @media (max-width:640px) {
    #app{padding:4px;gap:4px}
    .overlay{padding:8px 12px;gap:5px}
    .rules2{padding:0 4px;gap:3px}
    .roundgrid{gap:3px}
    .roundgrid button{padding:4px 0;font-size:16.5px}
    #panel{padding:8px;gap:4px}
    table{font-size:15.75px}
    th,td{padding:4px 4px}
    .stage-hud-item{padding:4px 10px;font-size:18px}
  }

  /* ===== neon polish + fever/perfect juice ===== */
  h1{font-family:'Orbitron','Black Han Sans',sans-serif;letter-spacing:1px;background:linear-gradient(90deg,#facc15,#f59e0b);-webkit-background-clip:text;background-clip:text;-webkit-text-fill-color:transparent}
  #topbar .time{font-family:'Orbitron',sans-serif}
  #topbar .scr{font-family:'Orbitron',sans-serif}
  #topbar.fever-active{border-color:var(--fever);box-shadow:0 0 18px rgba(244,63,94,.55)}
  #centerBig{transition:transform .1s ease}
  #centerBig.ok{animation:pop .22s ease-out} #centerBig.bad{animation:shk .3s ease-in-out}
  #judgeText{font-family:'Orbitron',sans-serif;font-size:clamp(27px,6.75vw,36px);font-weight:900;color:#fbbf24;text-shadow:0 0 10px #fbbf24;opacity:0;transition:opacity .12s}
  #feverText{font-family:'Orbitron',sans-serif;font-size:clamp(27px,7.5vw,51px);font-weight:900;color:var(--fever);text-shadow:0 0 14px var(--fever);animation:fpulse .45s infinite;min-height:1px}
  @keyframes pop{0%{transform:scale(.8)}50%{transform:scale(1.15)}100%{transform:scale(1)}}
  @keyframes fpulse{0%,100%{transform:scale(1);filter:brightness(1)}50%{transform:scale(1.06);filter:brightness(1.3)}}
  @keyframes shk{0%,100%{transform:translateX(0)}25%{transform:translateX(-6px)}75%{transform:translateX(6px)}}
  #stage.glitch{animation:glitch .28s steps(2) 2}
  @keyframes glitch{0%{filter:none}50%{filter:hue-rotate(90deg) saturate(1.6)}100%{filter:none}}

  /* ===== [요청4·8] 모바일 전용 최적화 ===== */
  /* 사운드 안내 배너 */
  .sound-notice{
    max-width:540px;width:100%;margin:2px auto;padding:9px 12px;border-radius:10px;
    background:linear-gradient(135deg,rgba(250,204,21,.16),rgba(245,158,11,.10));
    border:1px solid rgba(250,204,21,.45);color:#fde68a;
    font-size:clamp(13px,3.4vw,15.5px);line-height:1.45;text-align:center;flex-shrink:0
  }
  .sound-notice b{color:#fff}
  /* 터치 타겟 ≥44px(애플 HIG) — 손가락 조작 정확도 */
  @media (pointer:coarse){
    .stylebar button,.gamectrls button{min-height:44px;padding-top:10px;padding-bottom:10px}
    #fsBtn{min-width:44px;min-height:44px}
    .roundgrid button{min-height:40px}
    .skinrow .skin{min-height:40px}
    button{min-height:46px}
    .field input{min-height:46px}
    /* iOS 더블탭 줌·롱프레스 메뉴 억제(게임 컨트롤) */
    .stylebar button,.gamectrls button,#fsBtn,.roundgrid button{touch-action:manipulation;-webkit-touch-callout:none;user-select:none}
  }
  /* 세로(폼팩터) 우선: 카메라 스테이지가 화면 대부분을 차지 */
  @media (max-width:768px) and (orientation:portrait){
    #app{max-width:100%}
    #stage{flex:1 1 auto}
    h1{font-size:clamp(18px,5.4vw,26px)}
  }
  /* 가로모드 회전 권장 오버레이(셀피 카메라는 세로가 유리) */
  #rotateHint{
    position:fixed;inset:0;z-index:50;display:none;flex-direction:column;gap:14px;
    align-items:center;justify-content:center;background:rgba(6,6,8,.97);text-align:center;padding:24px
  }
  #rotateHint .ico{font-size:64px;animation:rotw 1.6s ease-in-out infinite}
  #rotateHint .t{font-size:20px;font-weight:900;color:var(--primary)}
  #rotateHint .s{font-size:15px;color:#d4d4d8;max-width:300px;line-height:1.5}
  @keyframes rotw{0%,100%{transform:rotate(0)}50%{transform:rotate(-90deg)}}
  /* 폰만(coarse pointer) 가로 + 낮은 높이일 때 회전 권장 */
  @media (pointer:coarse) and (orientation:landscape) and (max-height:480px){
    #rotateHint{display:flex}
  }

  /* ===== [요청6·7] 전역 BOLD 제거 + 고가시성 서체 통일 =====
     장식·볼드 서체(Orbitron/Black Han Sans) 대신 가독성 높은 Pretendard로 통일.
     모든 굵기를 medium(500) 이하로 낮춰 BOLD를 제거. 강조는 크기·색·그림자로. */
  html, body, button, input, table, th, td, h1, h2, h3,
  span, div, li, ul, p, a, b, strong{
    font-family:'Pretendard', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif !important;
    letter-spacing:0 !important;
  }
  *{ font-weight:500 !important; }
  b, strong{ font-weight:500 !important; }      /* <b> 태그도 볼드 해제 */
  /* h1 그라데이션 텍스트는 유지하되 굵기만 해제 */
  h1{ background:linear-gradient(90deg,#facc15,#f59e0b);-webkit-background-clip:text;background-clip:text;-webkit-text-fill-color:transparent }
  /* 큰 명령 텍스트는 카메라 위 가독성을 위해 그림자만 강화(굵기는 500 유지) */
  #centerBig{ text-shadow:0 3px 16px rgba(0,0,0,1), 0 0 4px rgba(0,0,0,.9) }

  /* [수정#5] dvh 미지원 구형 iOS Safari 높이 폴백 */
  @supports (-webkit-touch-callout: none){
    #app{ min-height:-webkit-fill-available }
  }
  /* [수정#6] 라운드 선택 버튼 터치 타겟 높이 ≥44px(HIG) — 10칸 정렬은 유지 */
  @media (pointer:coarse){
    .roundgrid button{ min-height:44px }
  }
  /* [수정#15] 좁은 폭에서 표가 넘쳐도 페이지가 깨지지 않게 가로 스크롤 허용 */
  .overlay table{ display:block; max-width:100%; overflow-x:auto; -webkit-overflow-scrolling:touch }
  /* [수정#17] 모션 민감 사용자: 화면 흔들림/글리치/펄스 등 강한 애니메이션 비활성(가독성·멀미 완화 + 부하 감소) */
  @media (prefers-reduced-motion: reduce){
    *{ animation-duration:.001ms !important; animation-iteration-count:1 !important; transition-duration:.001ms !important; scroll-behavior:auto !important }
    #stage.glitch{ animation:none !important }
    #feverText{ animation:none !important }
  }
</style>
</head>
<body>
<div id="app">
  <h1>협응성 청기백기 - MOTION-SENSING GAME</h1>

  <!-- [#9] 결과/명령을 스크린리더에 전달하는 라이브 리전 -->
  <div id="srLive" class="sr-only" aria-live="assertive" role="status"></div>

  <div id="topbar">
    <span class="rnd" id="roundLabel">ROUND 0</span>
    <span class="lives" id="livesLabel">❤️❤️❤️</span>
    <span class="time" id="roundTime">20.0s</span>
    <span class="scr" id="scoreLabel">SCORE 0</span>
    <button id="fsBtn" title="전체화면" aria-label="전체화면">⛶</button>
  </div>
  <div id="roundbar"><div id="roundfill"></div></div>

  <div id="ctrlbar">
    <div class="stylewrap">
      <span class="stylelabel">GAME MODE:</span>
      <div id="styleRow" class="stylebar">
        <button class="stbtn" data-style="original" title="기본 청기백기 - 들리는 그대로 수행">BASIC</button>
        <button class="stbtn" data-style="textReverse" title="반대로! 명령 - 들리는 말과 반대로 수행">REVERSE</button>
        <button class="stbtn" data-style="screenFlip" title="라운드마다 화면이 좌우/상하로 뒤집힘">ROTATION</button>
      </div>
    </div>
    <div id="gameCtrls" class="gamectrls">
      <button id="pauseToggle" disabled>⏸ 일시정지</button>
      <button id="quitCtrlBtn" class="ctrl-sec" disabled>⏹ 게임 종료</button>
    </div>
  </div>

  <div id="stage">
    <!-- [요청2] WAVE 진행 표기: n/10 (10 도달 시 클램프) -->
    <div id="hudTaskCount" class="stage-hud-item">WAVE 0/10</div>
    <div id="hudAccuracy" class="stage-hud-item hud-fail">SUCCESS 0%</div>

    <canvas id="cv" width="1280" height="720"></canvas>
    <video id="vid" playsinline muted></video>

    <div id="badge" class="badge"></div>
    <div id="comboHud"></div>

    <div id="centerText" class="hidden">
      <div id="centerSub"></div>
      <div id="centerBig"></div>
      <div id="judgeText"></div>
      <div id="feverText"></div>
    </div>

    <div id="introOverlay" class="overlay">
      <h2>청기백기 게임 방법</h2>
      <div class="rules2">
        <div class="rsec">1. 기본 조작</div>
        <ul>
          <li><b>기본 조작:</b> 지시문에 해당하는 위치에 손을 올립니다.<br>좌상(청기▲) · 좌하(청기▼) · 우상(백기▲) · 우하(백기▼)</li>
          <li><b>진행 방식:</b> 중앙 지시문과 음성에 따라 손을 움직입니다.</li>
        </ul>
        <div class="rsec">2. 게임 방식(GAME MODE)</div>
        <ul>
          <li><b>BASIC:</b> 지시문에 따라 움직입니다.</li>
          <li><b>REVERSE:</b> 지시문과 <b>정반대로</b> 움직여야 합니다.</li>
          <li><b>ROTATION:</b> 화면이 <b>상하 또는 좌우</b>로 반전됩니다.</li>
        </ul>
        <div class="rsec">3. 점수 및 콤보</div>
        <ul>
          <li><b>콤보 배수:</b> 콤보가 쌓일수록 점수 배수가 상승합니다.</li>
          <li><b>FEVER / PERFECT:</b> 7콤보 시 점수 2배</li>
        </ul>
        <div class="rsec">4. 라이프 및 라운드 구성</div>
        <ul>
          <li><b>❤️ 라이프:</b> 하트 3개 제공. SUCCESS 55% 미만 시 라운드 재도전.</li>
          <li><b>😈 10 라운드:</b> 속도 증가 및 보스 특수 패턴 강화.</li>
        </ul>
      </div>
      <div id="soundNotice" class="sound-notice">🔊 음성으로 명령이 나옵니다. 볼륨을 올리고 시작하세요.<br> 본 서비스는 현재 iOS 운영 체제만을 지원하는 상태입니다.</div>
      <div class="btnrow">
        <button id="startBtn" disabled>모델 로딩 중…</button>
      </div>
    </div>

    <div id="endOverlay" class="overlay hidden"></div>

    <div id="selectOverlay" class="overlay hidden">
      <h2>🎯 난이도 선택</h2>
      <div class="metaline" id="metaInfo"></div>
      <div class="xpbar"><i id="xpFill"></i></div>
      <div class="metaline"> 10라운드 마다 보스 라운드가 진행됩니다.</div>
      <div id="roundGrid" class="roundgrid"></div>
    </div>

    <div id="resultOverlay" class="overlay hidden"></div>
  </div>

  <div id="panel" class="hidden">
    <div id="cmdtimer"><div id="cmdfill"></div></div>
  </div>

  <div id="fatal"></div>
</div>

<div id="rotateHint" aria-hidden="true">
  <div class="ico">📱</div>
  <div class="t">세로로 돌려주세요</div>
  <div class="s">전면 카메라로 양손을 인식하는 게임이라 세로 화면에서 가장 잘 작동합니다.</div>
</div>

<script>
/* Hand recognition engine: @mediapipe/hands + camera_utils (global Hands / Camera) */
/* [#10] 전역 오염 제거: 전체를 IIFE로 캡슐화. 순수 함수는 window.__cbg 로 테스트 노출. */
(function(){
"use strict";

/* ===================== i18n / 문자열 테이블 ([#20]) ===================== */
const STR={
  hudTask:(n)=>`WAVE ${Math.min(n,10)}/10`,
  hudAcc:(p)=>`SUCCESS ${p}%`,
  confirmQuit:"진행 중인 게임을 종료하고 난이도 설정으로 이동할까요?",
  confirmEnd:"진행 중인 게임을 종료할까요?",
  cam:{
    NO_API:"카메라 API 가용성 확보 실패. 최신 브라우저를 권장합니다.",
    INSECURE:"보안 컨텍스트 오류. HTTPS 환경 실행이 강제됩니다.",
    DENIED:"카메라 접근 권한 획득 거부.",
    NOT_FOUND:"카메라 장치를 찾을 수 없습니다. 연결 상태를 확인하세요.",
    IN_USE:"다른 앱이 카메라를 사용 중입니다. 종료 후 다시 시도하세요.",
    CONSTRAINT:"요청한 카메라 설정을 지원하지 않습니다. 다른 카메라로 시도하세요.",
    DEFAULT:"하드웨어 장치 마운트 실패.",
  },
  mpFail:"MediaPipe 스크립트 로드 실패.",
  modelFail:(m)=>"손 인식 모델 초기화 실패: "+m,
  saveFail:(m)=>"이미지 저장 실패: "+m,
  honor:{ god:"🌌 공간 인지의 파괴신", react:"⚡ 초전도체급 반응 속도", champ:"🔥 동네 청기백기 챔피언", arcader:"🎯 제법 하는 아케이더", rookie:"🌱 평범한 일반인 유저" },
};

/* ===================== DOM ===================== */
const $ = (id)=>document.getElementById(id);
const cv=$("cv"), ctx=cv.getContext("2d"), vid=$("vid");
const roundLabel=$("roundLabel"), roundTime=$("roundTime"), scoreLabel=$("scoreLabel"), livesLabel=$("livesLabel");
const roundfill=$("roundfill");
const centerText=$("centerText"), centerSub=$("centerSub"), centerBig=$("centerBig");
const badge=$("badge"), comboHud=$("comboHud");
const judgeText=$("judgeText"), feverText=$("feverText"), topbar=$("topbar"), stage=$("stage");
const introOverlay=$("introOverlay"), endOverlay=$("endOverlay"), startBtn=$("startBtn");
const selectOverlay=$("selectOverlay"), resultOverlay=$("resultOverlay"), roundGrid=$("roundGrid");
const metaInfo=$("metaInfo"), xpFill=$("xpFill");
const styleRow=$("styleRow");
const gameCtrls=$("gameCtrls"), pauseToggle=$("pauseToggle"), quitCtrlBtn=$("quitCtrlBtn");
const panel=$("panel"), cmdfill=$("cmdfill");
const hudTaskCount=$("hudTaskCount"), hudAccuracy=$("hudAccuracy");
const fatalEl=$("fatal");
const app=$("app"), fsBtn=$("fsBtn");
const srLive=$("srLive");
let W=cv.width, H=cv.height;

/* ===================== Config / Tuning ([#15] 매직넘버 중앙화) ===================== */
const INITIAL_POSE={ blue:"down", white:"down" };
const ROUNDS=100;
const ROUND_MS=20000;
const PASS_RATE=0.55;
const GAP_MS=250;
const LIVES_START=3;
const COMBO_MAX=20;
const REVERSE_START=6;

const MS_PER_SYLLABLE=290;
const TTS_STARTUP_MS=250;
/* [요청7] 발음 정확도 우선: 고라운드 속도 상한을 하향(1.55 일반 / 1.65 보스).
   너무 빠른 합성은 한국어 받침·이중모음이 뭉개져 알아듣기 어려움. */
const TTS_RATE_BASE=1.0, TTS_RATE_STEP=0.025, TTS_RATE_MAX=1.55;
const TTS_BOSS_MUL=1.08, TTS_BOSS_MAX=1.65;
const ttsRate=(round)=> Math.min(TTS_RATE_BASE + (round-1)*TTS_RATE_STEP, TTS_RATE_MAX);
const bossRate=(rate)=> Math.min(rate*TTS_BOSS_MUL, TTS_BOSS_MAX);
const reactionBufferMs=(round)=> Math.max(650, 1900-(round-1)*70);

/* 별점 / 점수 튜닝 상수 */
const STAR3_RATE=0.90, STAR2_RATE=0.78, CLOSE_RATE=0.50;
const BASE_SCORE=100, ROUND_SCORE_STEP=10, SPEED_BONUS_MAX=50, COMBO_SCORE_STEP=4, PERFECT_BONUS=50, COMBO_MUL_STEP=0.1;
const XP_PER_CORRECT=5, XP_PER_STAR=30, XP_BOSS_BONUS=100, XP_BONUS_BONUS=40;

const reverseP=(round,type)=>{
  if(round<REVERSE_START) return 0;
  let p=Math.min(0.12+(round-REVERSE_START)*0.02, 0.40);
  if(type==="boss") p=Math.min(p+0.25, 0.62);
  return p;
};
const roundTypeOf=(round)=> round%10===0 ? "boss" : (round%5===0 ? "bonus" : "normal");
const roundMulOf=(type)=> type==="bonus" ? 2 : (type==="boss" ? 1.5 : 1);
function estSpeechMs(text, rate){
  const syl=(text.match(/[가-힣0-9]/g)||[]).length;
  return Math.round(syl*MS_PER_SYLLABLE/rate)+TTS_STARTUP_MS;
}
const commandWindow=(text, round, rate)=> estSpeechMs(text, rate)+reactionBufferMs(round);

/* ===================== Game styles ===================== */
const STYLES={
  original:   { name:"🎌 오리지널",     desc:"정통 청기백기 — 들리는 그대로 수행. 함정(올리지마/내리지마)만 등장." },
  textReverse:{ name:"🔄 텍스트 반대로", desc:"🔄 반대로! 명령이 섞여 나옵니다. 들리는 말과 반대로 움직이세요." },
  screenFlip: { name:"🔃 화면 뒤집기",   desc:"라운드마다 카메라가 좌우/상하로 뒤집힙니다. 감각이 뒤집힌 채 정답을 맞추세요." },
};
const isStyle=(s)=> Object.prototype.hasOwnProperty.call(STYLES,s);
/* ===================== Skins ===================== */
const SKINS=[
  { id:"classic", name:"클래식", lvl:1, blue:"#2563eb", white:"#f4f4f5" },
  { id:"neon",    name:"네온",   lvl:3, blue:"#06b6d4", white:"#f472b6" },
  { id:"sunset",  name:"선셋",   lvl:5, blue:"#f59e0b", white:"#ef4444" },
  { id:"matrix",  name:"매트릭스", lvl:8, blue:"#22c55e", white:"#a3e635" },
];
let SKIN={ blue:SKINS[0].blue, white:SKINS[0].white };

/* ===================== RNG =====================
   [#3] 미사용 seeded RNG(mulberry32) 제거. 기본 Math.random 사용. */
const rnd=()=>Math.random();
const pick=(a)=>a[Math.floor(rnd()*a.length)];

/* ===================== State ===================== */
const state={
  phase:"intro",
  mode:"normal",
  round:0, count:0,
  score:0,
  target:{...INITIAL_POSE}, command:null, detected:{...INITIAL_POSE},
  lastResult:null, _hands:[],
  roundDeadline:0, roundTotal:0, roundCorrect:0,
  cmdStartTs:0, cmdDeadline:0, firstCorrectTs:0,
  /* [#13] 무한 누적 배열 제거 → 러닝 집계 */
  reactionSum:0, reactionN:0, totalCmds:0, totalCorrect:0,
  bestScore:0, startedAt:1,
  lives:LIVES_START, combo:0, maxCombo:0,
  roundType:"normal", roundMul:1,
  xp:0, nick:"Player",
  paused:false, pausedAt:0,
  feverMode:false,
  style:"original", flipX:false, flipY:false,
  userPaused:false,
};
let hands=null, camera=null, sending=false;
/* [수정#8/끊김] 적응형 프레임 스킵 상태 */
let _sendAvg=0, _skipCtr=0, _skipMod=1;
let loopRunning=false, cameraReady=false, connecting=false;
let cmdTimer=0, cdTimer=0;

/* [FIX#1/#7] pause-aware 커맨드 스케줄러: 어떤 콜백(issue/evaluate)이 대기 중인지와
   '절대 발화 시각(_pendAt)'을 보존한다. 일시정지/탭전환 후에도 올바른 콜백을 정확한
   잔여시간으로 재개해 더블 카운트·점수 오염을 차단한다. cmdSeq는 커맨드별 1회 평가 토큰. */
let _pendFn=null, _pendAt=0, cmdSeq=0, _evaluatedSeq=-1;
function schedule(fn, delay){
  clearTimeout(cmdTimer);
  delay=Math.max(0, delay|0);
  _pendFn=fn; _pendAt=performance.now()+delay;
  cmdTimer=setTimeout(()=>{ const f=_pendFn; _pendFn=null; if(f) f(); }, delay);
}
function clearSchedule(){ clearTimeout(cmdTimer); _pendFn=null; _pendAt=0; }

/* [FIX#15] 동일 ms 종료 시 _id 충돌 → 'me' 하이라이트 오인 방지용 단조 증가 ID */
let _recSeq=0;
const RECID=()=> (Date.now()*1000 + (++_recSeq % 1000));

/* [FIX#9] 파티클 상한 — 롱콤보에서 배열 폭증/GC 압박 차단 */
const PARTICLE_CAP=220;

/* ===================== Web Audio SFX ===================== */
const FEVER_COMBO=7;
const PERFECT_MS=380;
let audioCtx=null, sfxOn=true;
function initAudio(){
  try{
    if(!audioCtx) audioCtx=new (window.AudioContext||window.webkitAudioContext)();
    if(audioCtx.state==="suspended") audioCtx.resume();
  }catch{ audioCtx=null; }
  unlockTTS(); /* [요청7] 동일 제스처 내에서 음성 합성도 해제 → iOS 첫 명령 무음 방지 */
}
function playTone(freq, type, dur, vol=0.12){
  if(!sfxOn) return;
  try{
    initAudio(); if(!audioCtx) return;
    const osc=audioCtx.createOscillator(), g=audioCtx.createGain();
    osc.type=type; osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
    g.gain.setValueAtTime(vol, audioCtx.currentTime);
    g.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime+dur);
    osc.connect(g); g.connect(audioCtx.destination);
    osc.start(); osc.stop(audioCtx.currentTime+dur);
  }catch{}
}
const sfxCorrect =()=>playTone(523.25,"sine",0.08,0.10);
const sfxMilestone=()=>{ playTone(659.25,"sine",0.10,0.14); setTimeout(()=>playTone(987.77,"sine",0.15,0.14),60); };
const sfxFever   =()=>playTone(1046.50,"sawtooth",0.25,0.10);
const sfxWrong   =()=>playTone(185.00,"sawtooth",0.22,0.18);
const sfxTick    =()=>playTone(523.25,"sine",0.07,0.09);
const sfxGo      =()=>playTone(783.99,"sine",0.18,0.13);
const sfxPerfect =()=>{ playTone(1318.51,"triangle",0.10,0.12); setTimeout(()=>playTone(1760.00,"triangle",0.12,0.12),70); };

/* ===================== Engine ===================== */
const FLAG_LABEL={ blue:"청기", white:"백기" };
const ACTION_TEXT={ raise:"올려", lower:"내려", dontRaise:"올리지 마", dontLower:"내리지 마" };
const ARROW={ up:"▲", down:"▼" };
const INVERT={ raise:"lower", lower:"raise", dontRaise:"dontLower", dontLower:"dontRaise" };
const effAction=(a, reverse)=> reverse ? INVERT[a] : a;

function applyInstruction(pose, ins, reverse){
  const n={...pose}; const a=effAction(ins.action, reverse);
  if(a==="raise") n[ins.flag]="up";
  else if(a==="lower") n[ins.flag]="down";
  return n;
}
const resolveTarget=(prev,list,reverse)=>list.reduce((p,i)=>applyInstruction(p,i,reverse),prev);
const buildText=(list)=>list.map(i=>`${FLAG_LABEL[i.flag]} ${ACTION_TEXT[i.action]}`).join(", ");

function generateCommand(prev, round, avoidDontLower, type, style){
  const crazyP=Math.min(0.05+round*0.014, 0.34);
  if(style==="textReverse" && round>3 && rnd()<crazyP){
    const f=pick(["blue","white"]), o=f==="blue"?"white":"blue";
    const cur=prev[f], curO=prev[o];
    const moveAct=cur==="down"?"raise":"lower";
    const dontAct=curO==="down"?"dontRaise":"dontLower";
    const t={...prev}; if(moveAct==="raise") t[f]="up"; else t[f]="down";
    const text=`${FLAG_LABEL[o]} ${ACTION_TEXT[dontAct]}, ${FLAG_LABEL[f]} ${ACTION_TEXT[moveAct]}`;
    return {
      instructions:[{flag:o,action:dontAct},{flag:f,action:moveAct}],
      text, speakText:text, target:t,
      hasDontLower:dontAct==="dontLower", reverse:false, crazy:true
    };
  }

  const dual=rnd() < Math.min(0.45+round*0.03, 0.85);
  const flags=dual?["blue","white"]:[pick(["blue","white"])];
  const trapP=Math.min(0.10+round*0.02, 0.35);
  const reverse = style==="textReverse" && rnd() < reverseP(round, type);

  let instructions=flags.map(flag=>{
    const cur=prev[flag];
    let trap=rnd()<trapP;
    if(trap && cur==="up" && avoidDontLower) trap=false;
    const action=trap?(cur==="down"?"dontRaise":"dontLower"):(cur==="down"?"raise":"lower");
    return { flag, action };
  });

  let t=resolveTarget(prev,instructions,reverse);
  if(t.blue===prev.blue && t.white===prev.white){
    const f=pick(flags), cur=prev[f];
    const effDesired=(cur==="down")?"raise":"lower";
    const spoken=reverse?INVERT[effDesired]:effDesired;
    instructions=instructions.map(ins=> ins.flag===f ? { flag:f, action:spoken } : ins);
    t=resolveTarget(prev,instructions,reverse);
  }
  const hasDontLower=instructions.some(i=> effAction(i.action,reverse)==="dontLower");
  const text=buildText(instructions);
  const speakText=(reverse?"반대로, ":"")+text;
  return { instructions, text, speakText, target:t, hasDontLower, reverse, crazy:false };
}
const isPoseCorrect=(d,t)=> d.blue===t.blue && d.white===t.white;

/* ===================== Persistence ([#1] 스토리지 키 버그 수정) ===================== */
const BEST_KEY="cbg.best";
const LB_KEY="cbg.board.v3";
const XP_KEY="cbg.xp.v1";
const NICK_KEY="cbg.nick";
const SKIN_KEY="cbg.skin";
const STYLE_KEY="cbg.style";
function safeGet(k,fb){ try{ const v=JSON.parse(localStorage.getItem(k)); return v==null?fb:v; }catch{ return fb; } }
function safeSet(k,v){ try{ localStorage.setItem(k,JSON.stringify(v)); }catch{} }
function loadBest(){ const n=safeGet(BEST_KEY,0); return (typeof n==="number"&&n>=0)?n:0; }
function saveBest(n){ safeSet(BEST_KEY,n); }
function loadXP(){ const n=safeGet(XP_KEY,0); return (typeof n==="number"&&n>=0)?n:0; }
function saveXP(n){ safeSet(XP_KEY,n); }
/* 닉네임은 NICK_KEY 에만, 스킨은 SKIN_KEY 에만 저장(이전: 함수명 혼동으로 스킨이 닉네임을 덮어씀) */
function loadNick(){ const s=safeGet(NICK_KEY,"Player"); const v=(typeof s==="string"&&s.trim())?s.trim().slice(0,10):"Player"; return v.replace(/[<>"'&]/g,""); }
function saveNick(s){ safeSet(NICK_KEY, String(s).replace(/[<>"'&]/g,"").trim().slice(0,10)); }
function loadSkinId(){ const s=safeGet(SKIN_KEY,"classic"); return SKINS.some(x=>x.id===s)?s:"classic"; }
function saveSkin(id){ if(SKINS.some(x=>x.id===id)) safeSet(SKIN_KEY, id); }
function saveStyle(s){ safeSet(STYLE_KEY,s); }
function loadStyle(){ const s=safeGet(STYLE_KEY,"original"); return isStyle(s)?s:"original"; }
function validRec(r){ return r && typeof r.score==="number" && typeof r.accuracy==="number" && typeof r.avgReaction==="number"; }
function loadBoard(){ const a=safeGet(LB_KEY,[]); return Array.isArray(a)?a.filter(validRec):[]; }
function saveBoard(a){ safeSet(LB_KEY,a.slice(0,10)); }
function addRecord(rec){
  const b=loadBoard(); rec._id=RECID(); b.push(rec);
  b.sort((x,y)=> y.score-x.score || x.avgReaction-y.avgReaction);
  const top=b.slice(0,10); saveBoard(top); return top;
}

const levelOf=(xp)=> Math.max(1, Math.floor(Math.sqrt(Math.max(0,xp)/100))+1);
const xpForLevel=(lv)=> Math.pow(lv-1,2)*100;
function applySkin(id){
  const s=SKINS.find(x=>x.id===id)||SKINS[0];
  SKIN={ blue:s.blue, white:s.white };
  document.documentElement.style.setProperty("--blue", s.blue);
  document.documentElement.style.setProperty("--white", s.white);
  saveSkin(s.id);
}

/* ===================== TTS (강건화: 무음 방지 + 발음 정확도) ===================== */
const TTS_SUPPORTED = ("speechSynthesis" in window) && (typeof SpeechSynthesisUtterance!=="undefined");
const IS_IOS = /iPad|iPhone|iPod/.test(navigator.userAgent) || (navigator.platform==="MacIntel" && navigator.maxTouchPoints>1);
/* [수정#8/끊김] 모바일 공용 플래그 — DPR 상한·파티클·그림자·모델복잡도 적응에 사용 */
const IS_MOBILE = /Mobi|Android|iPhone|iPad|iPod/i.test(navigator.userAgent) || (("ontouchstart" in window) && Math.min(screen.width,screen.height)<820);
let koVoice=null, ttsReady=false, ttsUnlocked=false, _voicePollT=0, _voicePolls=0;
let _ttsWatchdog=0, _ttsRetryT=0;

/* [요청6,7] 한국어 보이스 선별: 로컬(오프라인) 우선 → 발음 안정·지연 최소. */
function pickKoVoice(vs){
  const ko = vs.filter(v=>/^ko/i.test(v.lang||""));
  if(!ko.length) return null;
  const score=(v)=>{
    let s=0;
    if(/ko(-|_)?KR/i.test(v.lang)) s+=4;
    if(v.localService) s+=3;                         // 로컬 보이스 = 끊김/지연 적음
    if(/yuna|sora|google|미정|heami|nuri/i.test(v.name||"")) s+=2; // 품질 좋은 한국어 보이스
    if(v.default) s+=1;
    return s;
  };
  return ko.slice().sort((a,b)=>score(b)-score(a))[0];
}
function refreshVoices(){
  if(!TTS_SUPPORTED) return;
  const vs=window.speechSynthesis.getVoices()||[];
  if(vs.length){ koVoice=pickKoVoice(vs); ttsReady=true; }
}
/* getVoices()는 비동기(Chrome/Edge/일부 모바일) — 폴링으로 확실히 로드 */
function startVoicePolling(){
  if(!TTS_SUPPORTED) return;
  refreshVoices();
  clearInterval(_voicePollT); _voicePolls=0;
  _voicePollT=setInterval(()=>{
    refreshVoices(); _voicePolls++;
    if(ttsReady || _voicePolls>=16){ clearInterval(_voicePollT); } // 최대 ~4s
  }, 250);
}

/* [요청7] iOS/모바일 무음 차단 해제 — 반드시 사용자 제스처(탭) 안에서 1회 호출.
   빈 발화로 합성 엔진을 깨운다. */
function unlockTTS(){
  if(!TTS_SUPPORTED || ttsUnlocked) return;
  try{
    window.speechSynthesis.cancel();
    const u=new SpeechSynthesisUtterance("\u200b"); // zero-width: 들리지 않음
    u.volume=0; u.lang="ko-KR"; if(koVoice) u.voice=koVoice;
    u.onend=()=>{ ttsUnlocked=true; };
    u.onerror=()=>{ ttsUnlocked=true; };
    window.speechSynthesis.speak(u);
    ttsUnlocked=true;
  }catch{}
}

/* [요청6,7] 핵심 발화 함수 — 무조건 소리가 나도록 다중 안전장치:
   1) cancel→resume(끊김 버그 회피)  2) onerror/무발화 워치독 자동 재시도
   3) pitch/volume 고정으로 발음 명료도 확보 */
function speak(text, rate){
  if(!TTS_SUPPORTED || !text) return;
  const synth=window.speechSynthesis;
  clearTimeout(_ttsRetryT); clearTimeout(_ttsWatchdog);

  const fire=(attempt)=>{
    try{
      synth.cancel();                       // 이전 발화 제거
      if(synth.paused) synth.resume();       // 일부 브라우저 paused 고착 해제
      const u=new SpeechSynthesisUtterance(String(text));
      u.lang="ko-KR";
      if(koVoice) u.voice=koVoice;
      u.rate=Math.max(0.6, Math.min(rate||1.05, 2.0));
      u.pitch=1.0;                           // 발음 명료도(과한 피치 변형 금지)
      u.volume=1.0;                          // 항상 최대 볼륨
      let started=false;
      u.onstart=()=>{ started=true; clearTimeout(_ttsWatchdog); };
      u.onerror=()=>{
        if(attempt<2){ _ttsRetryT=setTimeout(()=>fire(attempt+1), 120); }
      };
      synth.speak(u);
      /* 워치독: 500ms 내 발화가 시작되지 않으면(무음 의심) 1회 재시도.
         synth.speaking 교차 확인으로 정상 발화 중 오재시도 방지. */
      _ttsWatchdog=setTimeout(()=>{
        if(!started && !synth.speaking && attempt<2){ fire(attempt+1); }
      }, 500);
    }catch{
      if(attempt<2) _ttsRetryT=setTimeout(()=>fire(attempt+1), 120);
    }
  };
  fire(0);
}
/* [FIX#2/#3] 발화 완전 정지: 합성 취소 + 재시도/워치독 타이머 제거.
   일시정지·탭전환·라운드 종료·게임오버 시 호출해 '스테일 음성'이 결과 화면에서
   뒤늦게 재생되는 문제를 차단한다. */
function stopSpeak(){
  clearTimeout(_ttsRetryT); clearTimeout(_ttsWatchdog);
  if(TTS_SUPPORTED){ try{ window.speechSynthesis.cancel(); }catch{} }
}
/* [#9] 스크린리더 알림 (중복 억제) */
let _lastAnnounce="";
function announce(t){ if(t && t!==_lastAnnounce){ _lastAnnounce=t; srLive.textContent=t; } }

/* ===================== Juice ===================== */
let particles=[], shake=0;
let _ox=0,_oy=0,_dw=0,_dh=0;
function spawnBurst(cx,cy,color){
  const N = IS_MOBILE ? 12 : 22;            /* [끊김] 모바일 파티클 수 축소 */
  for(let i=0;i<N;i++){
    const a=Math.random()*Math.PI*2, sp=2+Math.random()*5;
    particles.push({ x:cx, y:cy, vx:Math.cos(a)*sp, vy:Math.sin(a)*sp-1.5, life:1, color, r:3+Math.random()*4 });
  }
  /* [FIX#9] 상한 초과분은 가장 오래된 것부터 제거 → 롱콤보에서도 배열 폭증 방지 */
  if(particles.length>PARTICLE_CAP){ particles.splice(0, particles.length-PARTICLE_CAP); }
}
function buzz(ms){ try{ if(navigator.vibrate) navigator.vibrate(ms); }catch{} }
let _judgeTO=0;
function showJudge(txt){
  judgeText.textContent=txt; judgeText.style.opacity="1";
  clearTimeout(_judgeTO); _judgeTO=setTimeout(()=>{ judgeText.style.opacity="0"; }, 420);
}

/* ===================== Game flow ===================== */
/* [수정#9] 플레이 중 화면 꺼짐 방지(Screen Wake Lock). 미지원 기기는 무시. */
let _wakeLock=null;
async function acquireWake(){
  try{ if("wakeLock" in navigator && !_wakeLock){ _wakeLock=await navigator.wakeLock.request("screen"); _wakeLock.addEventListener("release",()=>{ _wakeLock=null; }); } }catch{}
}
async function releaseWake(){
  try{ if(_wakeLock){ const w=_wakeLock; _wakeLock=null; await w.release(); } }catch{}
}
function connectCamera(next){
  if(connecting) return Promise.resolve();
  connecting=true; fatalEl.style.display="none";
  startBtn.disabled=true;
  startBtn.textContent="카메라 연결 중…";
  return startCamera().then(()=>{
    connecting=false;
    if(!loopRunning){ loopRunning=true; requestAnimationFrame(uiLoop); }
    resizeCanvas();
    introOverlay.classList.add("hidden");
    if(typeof next==="function") next();
  }).catch((e)=>{
    connecting=false;
    showFatal(cameraErrorMessage(e));
    startBtn.disabled=false;
    startBtn.textContent="다시 시도";
  });
}

function showRoundSelect(){
  state.phase="select"; state.mode="normal";
  clearSchedule(); clearInterval(cdTimer); stopSpeak();
  clearFlip(); stage.classList.remove("glitch"); topbar.classList.remove("fever-active");
  state.paused=false; state.pausedAt=0; state.userPaused=false;
  endOverlay.classList.add("hidden"); resultOverlay.classList.add("hidden");
  hudTaskCount.style.display = "none";
  hudAccuracy.style.display = "none";
  renderMeta();
  if(!roundGrid.childElementCount){
    for(let r=1;r<=ROUNDS;r++){
      const b=document.createElement("button");
      b.textContent=r; if(r%10===0) b.classList.add("r5");
      b.addEventListener("click",()=>beginGameAt(r,"normal"));
      roundGrid.appendChild(b);
    }
  }
  selectOverlay.classList.remove("hidden");
}

function renderMeta(){
  const lv=levelOf(state.xp), cur=xpForLevel(lv), next=xpForLevel(lv+1);
  const pct=next>cur ? Math.round((state.xp-cur)/(next-cur)*100) : 100;
  metaInfo.innerHTML=`Lv.${lv} · XP ${state.xp} · 최고점 ${state.bestScore}`;
  xpFill.style.width=Math.max(0,Math.min(100,pct))+"%";
}

function setStyle(id){
  if(!isStyle(id)) id="original";
  const changed = state.style!==id;
  state.style=id; saveStyle(id); renderStyles();
  if(id!=="screenFlip"){ clearFlip(); }
  else if(changed && (state.phase==="playing"||state.phase==="countdown")){ applyFlip(state.round); }
}
function renderStyles(){
  /* [#4] 존재하지 않던 styleDesc 참조 제거 */
  for(const b of document.querySelectorAll(".stbtn")){
    b.classList.toggle("sel", b.dataset.style===state.style);
  }
}

/* ===================== Pause & Resume ===================== */
function pauseGame(){
  if(state.phase==="playing" && !state.paused){
    state.paused=true; state.pausedAt=performance.now();
    clearTimeout(cmdTimer);        /* 대기 콜백(_pendFn/_pendAt)은 보존 — 재개 때 그대로 사용 */
    stopSpeak();                   /* [FIX#2/#3] 발화·재시도·워치독 모두 정지 */
  }
}
function resumeGame(){
  if(state.paused && state.phase==="playing"){
    const dt=performance.now()-state.pausedAt;
    /* 게임 시간축 + 대기 콜백 발화시각을 동일하게 이동 → 라운드/명령 타이밍 일관성 유지 */
    state.roundDeadline+=dt; state.cmdDeadline+=dt; state.cmdStartTs+=dt;
    if(state.firstCorrectTs) state.firstCorrectTs+=dt;
    _pendAt+=dt;
    state.paused=false; state.pausedAt=0;
    if(_pendFn){                   /* [FIX#1] issue/evaluate 중 '실제로 대기 중이던' 콜백을 정확히 재개 */
      const rem=Math.max(0, _pendAt-performance.now());
      schedule(_pendFn, rem);
    }
  } else { state.paused=false; state.pausedAt=0; }
}
function togglePause(){
  if(state.phase!=="playing") return;
  if(state.paused){ state.userPaused=false; resumeGame(); }
  else { state.userPaused=true; pauseGame(); }
}

function beginGameAt(r, mode){
  selectOverlay.classList.add("hidden"); resultOverlay.classList.add("hidden"); endOverlay.classList.add("hidden");
  introOverlay.classList.add("hidden");
  clearSchedule(); clearInterval(cdTimer); stopSpeak();
  state.mode=mode||"normal";
  Object.assign(state,{
    score:0, round:0, command:null, lastResult:null, target:{...INITIAL_POSE},
    reactionSum:0, reactionN:0, totalCmds:0, totalCorrect:0, roundTotal:0, roundCorrect:0, startedAt:r,
    lives:LIVES_START, combo:0, maxCombo:0, paused:false, pausedAt:0, userPaused:false,
  });

  hudTaskCount.style.display = "block";
  hudAccuracy.style.display = "block";
  startRound(r);
}

function startRound(r){
  state.round=r;
  state.target={...INITIAL_POSE};
  state.roundType=roundTypeOf(r); state.roundMul=roundMulOf(state.roundType);
  state.roundTotal=0; state.roundCorrect=0; state.lastResult=null; state.command=null;
  state.feverMode=false; topbar.classList.remove("fever-active");
  applyFlip(r);
  if(state.roundType==="boss"){ stage.classList.remove("glitch"); void stage.offsetWidth; stage.classList.add("glitch"); }
  runCountdown(beginRoundPlay);
}

function applyFlip(r){
  /* [#18] 주의: screenFlip 의 시각 반전(CSS)은 의도적으로 손-사분면 매핑과 분리됩니다.
     (감각 교란이 핵심 난이도) 랜드마크 오버레이는 화면과 함께 뒤집혀 보이지만,
     정오 판정은 항상 원본 좌표 기준입니다. */
  let fx=false, fy=false;
  if(state.style==="screenFlip" && r>=2){
    const boss=state.roundType==="boss";
    const opts = boss ? ["both","both","flipX","flipY"] : ["flipX","flipY","flipX","flipY","both"];
    const mode = pick(opts);
    fx = (mode==="flipX"||mode==="both");
    fy = (mode==="flipY"||mode==="both");
  }
  state.flipX=fx; state.flipY=fy;
  cv.classList.toggle("flipX", fx);
  cv.classList.toggle("flipY", fy);
}
function clearFlip(){
  state.flipX=false; state.flipY=false;
  cv.classList.remove("flipX","flipY");
}

function runCountdown(done){
  state.phase="countdown"; state.count=3;
  clearInterval(cdTimer); clearTimeout(cdTimer);
  sfxTick();
  /* [FIX#11] 절대 시각 기준 드리프트 보정 + 페이즈 이탈 시 자동 중단.
     기존 setInterval은 백그라운드 스로틀에서 누적 지연·중복 틱이 발생했다. */
  const t0=performance.now();
  const tick=()=>{
    if(state.phase!=="countdown"){ return; }
    const n=3-Math.floor((performance.now()-t0)/1000);
    state.count=Math.max(0,n);
    if(state.count<=0){ sfxGo(); if(state.phase==="countdown") done(); return; }
    sfxTick();
    cdTimer=setTimeout(tick, Math.max(0, t0+(4-state.count)*1000-performance.now()));
  };
  cdTimer=setTimeout(tick, 1000);
}

function beginRoundPlay(){
  state.phase="playing";
  acquireWake();   /* [수정#9] 플레이 중 화면 유지 */
  state.roundDeadline=performance.now()+ROUND_MS;
  issueCommand();
}

function issueCommand(){
  const now=performance.now();
  const remain=state.roundDeadline-now;
  /* 남은 라운드 시간이 사실상 없으면 새 명령을 띄우지 않고 라운드 종료 */
  if(remain<=900){ endRound(); return; }
  const prevDontLower=!!(state.command && state.command.hasDontLower);
  const command=generateCommand(state.target, state.round, prevDontLower, state.roundType, state.style);
  state.command=command; state.target=command.target;
  state.lastResult=null; state.firstCorrectTs=0; state.cmdStartTs=now;
  let rate=ttsRate(state.round); if(state.roundType==="boss") rate=bossRate(rate);
  /* 명령 평가 시한이 라운드 종료 시각을 넘지 않도록 제한 → 시간이 다되면 풀던 도중이라도 종료 */
  state.cmdDeadline=Math.min(now+commandWindow(command.speakText, state.round, rate), state.roundDeadline);
  speak(command.speakText, rate);
  announce((command.reverse?"반대로! ":"")+command.text);
  cmdSeq++;                                   /* [FIX#7] 새 커맨드 → 새 평가 토큰 */
  schedule(evaluateCommand, state.cmdDeadline-now);
}

function evaluateCommand(){
  if(_evaluatedSeq===cmdSeq) return;          /* [FIX#7] 동일 커맨드 중복 평가 차단 */
  _evaluatedSeq=cmdSeq;
  const now=performance.now();
  /* [FIX#17] 명령 윈도 내에 한 번이라도 정답 포즈에 도달했으면 정답으로 인정.
     마감 순간의 일시적 손 추적 드롭아웃이 정답을 오답으로 뒤집는 false-negative 제거. */
  const correct = state.firstCorrectTs>0 || isPoseCorrect(state.detected, state.target);
  state.roundTotal++; state.totalCmds++;
  if(correct){
    state.roundCorrect++; state.totalCorrect++;
    state.combo++; state.maxCombo=Math.max(state.maxCombo, state.combo);

    if(state.combo>=FEVER_COMBO && !state.feverMode){
      state.feverMode=true; topbar.classList.add("fever-active"); sfxFever();
    }
    if(state.combo%5===0) sfxMilestone(); else sfxCorrect();

    let rate=ttsRate(state.round); if(state.roundType==="boss") rate=bossRate(rate);
    const win=commandWindow(state.command.speakText, state.round, rate);
    const reaction=state.firstCorrectTs ? (state.firstCorrectTs-state.cmdStartTs) : win;
    state.reactionSum+=reaction; state.reactionN++;

    const mult=1+Math.min(state.combo, COMBO_MAX)*COMBO_MUL_STEP;
    let add=BASE_SCORE+(state.round-1)*ROUND_SCORE_STEP+Math.round(Math.max(0,(win-reaction)/win)*SPEED_BONUS_MAX);
    add+=state.combo*COMBO_SCORE_STEP;
    const perfect = state.firstCorrectTs>0 && reaction<PERFECT_MS;
    if(perfect){ add+=PERFECT_BONUS; showJudge("PERFECT!"); sfxPerfect(); }
    let gained=Math.round(add*mult*state.roundMul);
    if(state.feverMode) gained*=2;
    state.score+=gained;
    state.lastResult="correct";
    announce("정답");

    comboHud.classList.remove("pop-active");
    void comboHud.offsetWidth;
    comboHud.classList.add("pop-active");

    const col = state.feverMode ? "#f43f5e" : (state.combo>=8 ? "#fb923c" : "#10b981");
    if(state._hands.length){
      for(const h of state._hands) spawnBurst(_ox+h.sx*_dw, _oy+h.sy*_dh, h.side==="blue"?SKIN.blue:col);
    } else spawnBurst(W/2, H*0.42, col);
    if(state.combo>0 && state.combo%5===0){ shake=Math.max(shake,6); buzz(20); }
  } else {
    state.combo=0;
    if(state.feverMode){ state.feverMode=false; topbar.classList.remove("fever-active"); }
    state.lastResult="wrong";
    announce("오답");
    sfxWrong(); shake=Math.max(shake,12); buzz([40,30,40]);
  }
  if(now>=state.roundDeadline){ schedule(endRound,500); }
  else { schedule(issueCommand, GAP_MS); }
}

/* [#2] deleteRound (endRound 중복 데드코드) 제거 */

function endRound(){
  clearSchedule(); clearInterval(cdTimer); stopSpeak();
  const pct=state.roundTotal ? state.roundCorrect/state.roundTotal : 0;
  if(pct>=PASS_RATE){
    const stars = pct>=STAR3_RATE ? 3 : pct>=STAR2_RATE ? 2 : 1;
    const gain = state.roundCorrect*XP_PER_CORRECT + stars*XP_PER_STAR + (state.roundType==="boss"?XP_BOSS_BONUS:state.roundType==="bonus"?XP_BONUS_BONUS:0);
    state.xp+=gain; saveXP(state.xp);
    if(state.round>=ROUNDS){ finishGame(true); return; }
    showRoundResult(pct, stars, gain);
  } else {
    state.lives--;
    if(state.lives<=0){ finishGame(false); return; }
    showFailRetry(pct);
  }
}

function showRoundResult(pct, stars, gain){
  state.phase="roundresult";
  hudTaskCount.style.display = "none";
  hudAccuracy.style.display = "none";
  const next=state.round+1, nType=roundTypeOf(next);
  const nTag = nType==="boss" ? " · 😈 BOSS" : nType==="bonus" ? " · ✨ BONUS ×2" : "";
  resultOverlay.innerHTML=`
    <h2>✅ ROUND ${state.round} 통과!</h2>
    <div class="stars">${"★".repeat(stars)}${"☆".repeat(3-stars)}</div>
    <div class="statgrid">
      <div class="stat"><div class="k">정답</div><div class="v">${state.roundCorrect}/${state.roundTotal}</div></div>
      <div class="stat"><div class="k">SUCCESS</div><div class="v">${Math.round(pct*100)}%</div></div>
      <div class="stat"><div class="k">최대 콤보</div><div class="v">${state.maxCombo}회</div></div>
      <div class="stat"><div class="k">누적 점수</div><div class="v">${state.score}</div></div>
    </div>
    <div class="metaline">+${gain} XP · 다음: ROUND ${next} / ${ROUNDS}${nTag}</div>
    <div class="btnrow">
      <button id="nextRoundBtn">다음 라운드 가기 ▶</button>
      <button id="resultHomeBtn" class="btn-secondary">🏠 홈으로</button>
    </div>`;
  resultOverlay.classList.remove("hidden");
  $("nextRoundBtn").addEventListener("click",()=>{ resultOverlay.classList.add("hidden"); hudTaskCount.style.display="block"; hudAccuracy.style.display="block"; startRound(next); });
  $("resultHomeBtn").addEventListener("click",()=>{ resultOverlay.classList.add("hidden"); showRoundSelect(); });
}

function showFailRetry(pct){
  state.phase="roundresult";
  hudTaskCount.style.display = "none";
  hudAccuracy.style.display = "none";
  const close = pct>=CLOSE_RATE;
  resultOverlay.innerHTML=`
    <h2>${close?"😣 아깝다!":"💥 라운드 실패"}</h2>
    <div class="metaline">SUCCESS ${Math.round(pct*100)}% · 통과 기준 55%</div>
    <div class="lives" style="font-size:36px">${"❤️".repeat(Math.max(0,state.lives))}${"🖤".repeat(LIVES_START-Math.max(0,state.lives))}</div>
    <div class="metaline">${close?"한 번만 더 하면 됩니다 — 같은 라운드 재도전":"한 번만 더 하면 됩니다"}</div>
    <div class="btnrow">
      <button id="retryRoundBtn">↻ ROUND ${state.round} 다시 도전</button>
      <button id="failHomeBtn" class="btn-secondary">🏠 홈으로</button>
    </div>`;
  resultOverlay.classList.remove("hidden");
  $("retryRoundBtn").addEventListener("click",()=>{ resultOverlay.classList.add("hidden"); hudTaskCount.style.display="block"; hudAccuracy.style.display="block"; startRound(state.round); });
  $("failHomeBtn").addEventListener("click",()=>{ resultOverlay.classList.add("hidden"); showRoundSelect(); });
}

function finishGame(win){
  state.phase="gameover";
  releaseWake();   /* [수정#9] 종료 시 화면 유지 해제 */
  clearSchedule(); clearInterval(cdTimer); stopSpeak();
  clearFlip(); topbar.classList.remove("fever-active");
  state.paused=false; state.userPaused=false;
  resultOverlay.classList.add("hidden");
  hudTaskCount.style.display = "none";
  hudAccuracy.style.display = "none";
  const acc=state.totalCmds ? Math.round(state.totalCorrect/state.totalCmds*100) : 0;
  const avgR=state.reactionN ? Math.round(state.reactionSum/state.reactionN) : 0;
  const rec={
    nick:state.nick, mode:state.mode, score:state.score, rounds: win?ROUNDS:state.round,
    accuracy:acc, avgReaction:avgR, maxCombo:state.maxCombo,
    date:new Date().toISOString().slice(0,10)
  };
  const board=addRecord(rec);
  if(state.score>state.bestScore){ state.bestScore=state.score; saveBest(state.bestScore); }
  showEnd(win, rec, board);
}

function getHonorTitle(score, maxCombo){
  if(score>50000) return STR.honor.god;
  if(maxCombo>=22) return STR.honor.react;
  if(score>18000) return STR.honor.champ;
  if(score>6000)  return STR.honor.arcader;
  return STR.honor.rookie;
}

function showEnd(win, rec, board){
  const cell=(v)=>escapeHtml(String(v));   /* [FIX#8] 변조된 localStorage 값의 HTML 주입 차단 */
  const rows=board.map((r,i)=>{
    const me = r._id===rec._id ? ' class="me"' : '';
    return `<tr${me}><td>${i+1}</td><td>${cell(Number(r.score)||0)}</td><td>${cell(Number(r.rounds)||0)}</td><td>${cell(Number(r.accuracy)||0)}%</td><td>${cell(Number(r.maxCombo)||0)}회</td></tr>`;
  }).join("");
  const honor=getHonorTitle(rec.score, rec.maxCombo);
  endOverlay.innerHTML=`
    <h2>${win?"🎉 클리어!":"게임 종료"}</h2>
    <div class="metaline" style="font-size:21px;color:var(--primary);font-weight:900">RANK · ${honor}</div>
    <div class="statgrid">
      <div class="stat"><div class="k">점수</div><div class="v">${rec.score}</div></div>
      <div class="stat"><div class="k">라운드</div><div class="v">${rec.rounds}</div></div>
      <div class="stat"><div class="k">SUCCESS</div><div class="v">${rec.accuracy}%</div></div>
      <div class="stat"><div class="k">최대 콤보</div><div class="v">${rec.maxCombo}회</div></div>
    </div>
    <div class="metaline">🏆 누적 기록 Top 10</div>
    <table>
      <thead><tr><th>등수</th><th>점수</th><th>라운드</th><th>SUCCESS</th><th>콤보</th></tr></thead>
      <tbody>${rows||'<tr><td colspan="5">기록 없음</td></tr>'}</tbody>
    </table>
    <div class="btnrow">
      <button id="shareBtn">🖼️ 결과 카드 저장</button>
      <button id="retryBtn">↻ ROUND ${state.round}부터</button>
      <button id="homeBtn" class="btn-secondary">🏠 홈으로</button>
    </div>`;
  endOverlay.classList.remove("hidden");
  $("shareBtn").addEventListener("click", ()=>downloadShareCard(rec));
  $("retryBtn").addEventListener("click", ()=>{ endOverlay.classList.add("hidden"); hudTaskCount.style.display="block"; hudAccuracy.style.display="block"; beginGameAt(state.round, state.mode); });
  $("homeBtn").addEventListener("click", ()=>{ endOverlay.classList.add("hidden"); showRoundSelect(); });
}

/* ===================== Share card ([#16] 폰트 로드 대기) ===================== */
async function downloadShareCard(rec){
  try{ if(document.fonts && document.fonts.ready) await document.fonts.ready; }catch{}
  const c=document.createElement("canvas"); c.width=800; c.height=1000;
  const x=c.getContext("2d");
  const g=x.createLinearGradient(0,0,0,1000); g.addColorStop(0,"#0a0a0b"); g.addColorStop(1,"#161618");
  x.fillStyle=g; x.fillRect(0,0,800,1000);
  x.strokeStyle="#2a2a2e"; x.lineWidth=3; x.strokeRect(24,24,752,952);
  x.textAlign="center";
  x.fillStyle="#facc15"; x.font="500 84px 'Pretendard', sans-serif"; x.fillText("협응성 청기백기", 400, 128);
  x.fillStyle="#9b9ba3"; x.font="500 39px 'Pretendard', sans-serif"; x.fillText("MOTION-SENSING GAME", 400, 186);
  x.fillStyle="#f4f4f5"; x.font="500 66px 'Pretendard', sans-serif"; x.fillText(rec.nick||"Player", 400, 270);
  x.fillStyle="#facc15"; x.font="500 220px 'Pretendard', sans-serif"; x.fillText(String(rec.score), 400, 470);
  x.fillStyle="#9b9ba3"; x.font="500 42px 'Pretendard', sans-serif"; x.fillText("SCORE", 400, 510);
  const stats=[
    ["도달 라운드", `${rec.rounds} / ${ROUNDS}`],
    ["SUCCESS", `${rec.accuracy}%`],
    ["최대 콤보", `${rec.maxCombo}회`],
    ["평균 반응", `${(rec.avgReaction/1000).toFixed(2)}s`],
  ];
  let yy=600;
  for(const [k,v] of stats){
    x.fillStyle="#9b9ba3"; x.font="500 45px 'Pretendard', sans-serif"; x.textAlign="left"; x.fillText(k, 150, yy);
    x.fillStyle="#fff"; x.font="500 51px 'Pretendard', sans-serif"; x.textAlign="right"; x.fillText(v, 650, yy);
    yy+=70;
  }
  x.textAlign="center"; x.fillStyle="#71717a"; x.font="500 36px 'Pretendard', sans-serif"; x.fillText(rec.date, 400, 930);
  try{
    const a=document.createElement("a");
    a.href=c.toDataURL("image/png"); a.download=`cheonggi-${rec.score}.png`; a.click();
  }catch(e){ showFatal(STR.saveFail(e&&e.message||e)); }
}

/* ===================== Detection ===================== */
const UP_Y=0.43, DOWN_Y=0.57;
const PRESENCE_FRAMES=4;
const FLIP_FRAMES=2;                 /* [FIX#18] 상태 전환에 필요한 연속 동일신호 프레임 수 */
let poseStable={ blue:"down", white:"down" };
let presence={ blue:0, white:0 };
let _flipCand={ blue:{v:"down",n:0}, white:{v:"down",n:0} };
let _lastFrameTs=0;                  /* [FIX#16] 마지막 추론결과 수신 시각(스톨 감지용) */
const classifyY=(prev,y)=> y<UP_Y ? "up" : (y>DOWN_Y ? "down" : prev);
/* [FIX#18] 랜드마크 노이즈로 인한 단일프레임 튐을 흡수: 같은 신호가 FLIP_FRAMES회
   연속될 때만 상태를 바꾼다(데드존 히스테리시스와 함께 판정 깜빡임 제거). */
function smoothState(side, raw){
  if(raw===poseStable[side]){ _flipCand[side].v=raw; _flipCand[side].n=0; return poseStable[side]; }
  const c=_flipCand[side];
  if(raw===c.v){ c.n++; } else { c.v=raw; c.n=1; }
  if(c.n>=FLIP_FRAMES){ c.n=0; return raw; }
  return poseStable[side];
}

function onHandResults(results){
 try{
  const handsArr=[];
  const seen={ blue:false, white:false };
  for(const lm of (results.multiHandLandmarks||[])){
    const ref=lm[9];
    const sx=1-ref.x, sy=ref.y;
    const side= sx<0.5 ? "blue" : "white";
    seen[side]=true;
    poseStable[side]=smoothState(side, classifyY(poseStable[side], sy));
    handsArr.push({ sx, sy, side, st:poseStable[side], lm });
  }
  for(const s of ["blue","white"]){
    if(seen[s]) presence[s]=PRESENCE_FRAMES;
    else { presence[s]=Math.max(0,presence[s]-1); if(presence[s]===0){ poseStable[s]="down"; _flipCand[s].v="down"; _flipCand[s].n=0; } }
  }
  state.detected={ blue:poseStable.blue, white:poseStable.white };
  state._hands=handsArr;

  if(state.phase==="playing" && !state.paused && state.firstCorrectTs===0 && isPoseCorrect(state.detected,state.target)){
    state.firstCorrectTs=performance.now();
  }

  _lastFrameTs=performance.now();
  drawFrame(results.image, handsArr);
 }catch(err){ /* [FIX#20] 프레임 처리 예외가 카메라 콜백을 죽이지 않도록 격리 */ }
}

function drawFrame(image, handsArr){
  shake=Math.max(0, shake-1);
  const sh = shake>0 ? (Math.random()*2-1)*shake*0.7 : 0;
  ctx.save(); ctx.clearRect(0,0,W,H); ctx.translate(sh, sh);

  const vw=vid.videoWidth||W, vh=vid.videoHeight||H;
  const s=Math.max(W/vw, H/vh), dw=vw*s, dh=vh*s, ox=(W-dw)/2, oy=(H-dh)/2;
  _ox=ox; _oy=oy; _dw=dw; _dh=dh;
  ctx.save(); ctx.translate(W,0); ctx.scale(-1,1); ctx.drawImage(image, ox, oy, dw, dh); ctx.restore();

  drawGrid();

  const S=H/720;
  for(const h of handsArr){
    /* 정답 매칭(체크) 표시는 제거 — 손의 up/down 상태만 표시(정답 미노출) */
    ctx.fillStyle = (h.st==="up" ? "#00ff66" : "#ff3333");
    for(const p of h.lm){ ctx.beginPath(); ctx.arc(ox+(1-p.x)*dw, oy+p.y*dh, 4*S, 0, Math.PI*2); ctx.fill(); }
    const bx=ox+h.sx*dw, by=oy+h.sy*dh;
    ctx.beginPath(); ctx.arc(bx, by, 22*S, 0, Math.PI*2);
    ctx.fillStyle=h.side==="blue"?hexA(SKIN.blue,.55):hexA(SKIN.white,.6);
    ctx.fill();
    /* 색각 보조: up=실선 / down=점선 링 (형태로도 구분) + ▲▼ 글리프 (상태만, 정답 아님) */
    ctx.lineWidth=4*S; ctx.strokeStyle="rgba(0,0,0,.6)";
    ctx.setLineDash(h.st==="up" ? [] : [5*S,4*S]); ctx.stroke(); ctx.setLineDash([]);
    const glyph = (h.st==="up" ? "▲" : "▼");
    ctx.fillStyle="#0b0b0c"; ctx.font=`500 ${Math.round(20*S)}px 'Pretendard', sans-serif`;
    ctx.textAlign="center"; ctx.textBaseline="middle";
    ctx.fillText(glyph, bx, by);
  }

  ctx.save();
  for(let i=particles.length-1;i>=0;i--){
    const p=particles[i]; p.x+=p.vx; p.y+=p.vy; p.vy+=0.35; p.life-=0.03;
    if(p.life<=0){ particles.splice(i,1); continue; }
    ctx.globalAlpha=Math.max(0,p.life); ctx.fillStyle=p.color;
    if(!IS_MOBILE){ ctx.shadowBlur=12*S; ctx.shadowColor=p.color; }  /* [끊김] 모바일 blur 생략 */
    ctx.beginPath(); ctx.arc(p.x, p.y, (p.r||4)*S*p.life+1, 0, Math.PI*2); ctx.fill();
  }
  ctx.restore();
  ctx.globalAlpha=1;
  ctx.restore();
}

function hexA(hex,a){
  const h=hex.replace("#",""); const n=h.length===3 ? h.split("").map(c=>c+c).join("") : h;
  const r=parseInt(n.slice(0,2),16), g=parseInt(n.slice(2,4),16), b=parseInt(n.slice(4,6),16);
  return `rgba(${r},${g},${b},${a})`;
}

function drawGrid(){
  const S=H/720;
  ctx.save();
  ctx.lineWidth=2*S; ctx.strokeStyle="rgba(255,255,255,.45)";
  ctx.beginPath(); ctx.moveTo(W/2,0); ctx.lineTo(W/2,H); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(0,H/2); ctx.lineTo(W,H/2); ctx.stroke();

  const cells=[
    { x:W*.25,y:H*.25,label:"청기 ▲",side:"blue" },
    { x:W*.25,y:H*.75,label:"청기 ▼",side:"blue" },
    { x:W*.75,y:H*.25,label:"백기 ▲",side:"white" },
    { x:W*.75,y:H*.75,label:"백기 ▼",side:"white" },
  ];
  ctx.font=`500 ${Math.round(54*S)}px 'Pretendard', sans-serif`; ctx.textAlign="center"; ctx.textBaseline="middle";

  for(const c of cells){
    ctx.fillStyle=(c.side==="blue"?hexA(SKIN.blue,.92):hexA(SKIN.white,.9));
    ctx.fillText(c.label, c.x, c.y);
  }
  ctx.restore();
}

function uiLoop(){ try{ syncUI(); }catch(e){ /* [FIX#20] syncUI 예외가 rAF 루프를 끊지 않도록 격리 */ } requestAnimationFrame(uiLoop); }

/* [요청5/끊김] 중앙 명령 텍스트를 1줄로 유지하며 폭에 맞춰 폰트 자동 조절.
   기존: fontSize 변경→scrollWidth 읽기를 최대 12회 반복 = 강제 리플로우 12회/명령 전환(끊김 원인).
   변경: 오프스크린 canvas measureText로 1회 측정 후 선형 환산 → 리플로우 0회. */
let _lastBig=null, _measCtx=null;
function fitBig(){
  const el=centerBig, txt=el.textContent;
  if(!txt){ el.style.fontSize=""; return; }
  const stageW=stage.clientWidth||W;
  const maxW=Math.max(60, stageW*0.92);
  const hi=Math.min(112, Math.round(stageW*0.18));
  if(!_measCtx){ try{ _measCtx=document.createElement("canvas").getContext("2d"); }catch{ _measCtx=null; } }
  let size=hi;
  if(_measCtx){
    _measCtx.font="500 100px 'Pretendard', system-ui, -apple-system, sans-serif";
    const w100=_measCtx.measureText(txt).width||1;
    size=Math.floor(100*maxW/w100);
    size=Math.max(18, Math.min(hi, size));
  }
  el.style.whiteSpace="nowrap";
  el.style.fontSize=size+"px";
}

/* ===================== UI sync =====================
   [끊김] syncUI는 매 rAF(~60fps) 실행된다. 기존엔 매 프레임 textContent/innerHTML/style을
   무조건 다시 써서 스타일 재계산·리플로우가 60회/초 누적 → 모바일 끊김. 아래 캐시 세터로
   '값이 바뀐 경우에만' DOM에 반영하여 불필요한 쓰기를 제거한다(동작 동일, 출력 동일). */
const _uic=Object.create(null);
function _txt(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.textContent=v; } }
function _html(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.innerHTML=v; } }
function _w(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.style.width=v; } }
function _disp(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.style.display=v; } }
function _cls(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.className=v; } }
function _op(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.style.opacity=v; } }
function _dis(el,k,v){ if(_uic[k]!==v){ _uic[k]=v; el.disabled=v; } }
function _tog(el,k,cls,on){ const v=!!on; if(_uic[k]!==v){ _uic[k]=v; el.classList.toggle(cls,v); } }   /* [FIX#10] 변경 시에만 class 토글 */

let _idleFrame=0;
function syncUI(){
  const now=performance.now();
  const playing=state.phase==="playing", counting=state.phase==="countdown";

  /* [FIX#6] 유휴(인트로/선택/결과/게임오버 & 미연결) 구간에서는 격프레임으로 갱신해
     불필요한 60fps DOM 비교/연산을 절반으로 줄인다(동작·표시 동일, 부하만 감소). */
  if(!playing && !counting && !connecting){
    if((_idleFrame++ & 1)===1) return;
  } else { _idleFrame=0; }

  _dis(pauseToggle,"pdis",!playing);
  _dis(quitCtrlBtn,"qdis",!playing);
  if(playing) _txt(pauseToggle,"ptxt", state.paused ? "▶ 계속하기" : "⏸ 일시정지");

  _txt(roundLabel,"rl",`ROUND ${state.round}`);
  _txt(scoreLabel,"sl",`SCORE ${state.score}`);
  _txt(livesLabel,"lv",(state.phase==="intro"||state.phase==="select")
    ? "❤️".repeat(LIVES_START)
    : "❤️".repeat(Math.max(0,state.lives))+"🖤".repeat(Math.max(0,LIVES_START-state.lives)));

  const tLeft=(playing && !state.paused) ? Math.max(0,(state.roundDeadline-now)/1000) : (playing?Math.max(0,(state.roundDeadline-state.pausedAt)/1000):ROUND_MS/1000);
  _txt(roundTime,"rt",tLeft.toFixed(1)+"s");
  _tog(roundTime,"rtw","warn", playing && tLeft<=5);
  _w(roundfill,"rf",(playing ? (tLeft/(ROUND_MS/1000))*100 : 100)+"%");

  let big="", sub="", cls="";
  if(connecting){ sub="카메라 연결 중…"; }
  else if(state.paused && playing){ sub="계속하기 버튼을 누르면 게임이 계속됩니다"; big="⏸ 일시정지"; }
  else if(counting){ sub=`ROUND ${state.round}`; big=String(state.count); }
  else if(playing){
    if(state.lastResult==="correct"){ big="정답!"; cls="ok"; }
    else if(state.lastResult==="wrong"){ big="오답!"; cls="bad"; }
    else if(state.command){ big=state.command.text; cls=state.command.reverse?"rev":""; }
  }
  /* [FIX#16] 추론 결과가 2.5초 이상 끊기면(카메라 프리즈/컨텍스트 손실) 사용자에게 알림 */
  if(playing && !state.paused && _lastFrameTs && (now-_lastFrameTs)>2500){ sub="📷 카메라 신호 지연… 잠시만 기다려 주세요"; }

  _txt(centerSub,"csub",sub); _txt(centerBig,"cbig",big); _cls(centerBig,"cbigc",cls);
  if(big!==_lastBig){ _lastBig=big; fitBig(); }
  _tog(centerText,"cth","hidden", !(connecting||counting||playing||(state.paused&&playing)));

  let bt="", bcls="";
  const flipTxt = state.flipX&&state.flipY ? "🔃 상하좌우 반전" : state.flipX ? "↔️ 좌우 반전" : state.flipY ? "↕️ 상하 반전" : "";
  if(playing && state.command && state.command.reverse){ bt="🔄 반대로!"; bcls="rev"; }
  else if((playing||counting) && flipTxt){ bt=flipTxt; bcls="rev"; }
  else if(state.roundType==="boss"){ bt="😈 BOSS ×1.5"; bcls="boss"; }
  else if(state.roundType==="bonus"){ bt="✨ BONUS ×2"; bcls="bonus"; }
  _txt(badge,"bt",bt); _cls(badge,"btc","badge "+bcls);
  _disp(badge,"bd",(bt && (playing||counting)) ? "block" : "none");

  if(playing && state.combo>=1){
    const mult=(1+Math.min(state.combo,COMBO_MAX)*COMBO_MUL_STEP).toFixed(1);
    _txt(comboHud,"ch",`COMBO ${state.combo}  (×${mult})`);
    _disp(comboHud,"chd","block");
  } else {
    _disp(comboHud,"chd","none");
    comboHud.classList.remove("pop-active");
  }

  _txt(feverText,"ft",(playing && state.feverMode) ? "🔥 FEVER ×2 🔥" : "");
  if(!playing){ _op(judgeText,"jo","0"); }

  const showPanel=playing||counting;
  _tog(panel,"pnl","hidden", !showPanel);
  if(playing){
    const total = state.cmdDeadline - state.cmdStartTs;
    const left = state.paused ? (state.cmdDeadline - state.pausedAt) : (state.cmdDeadline - now);
    _w(cmdfill,"cf",(total>0?Math.max(0,Math.min(1,left/total))*100:0)+"%");

    /* [요청2] WAVE n/10 표기 (STR.hudTask에서 10 클램프) */
    _html(hudTaskCount,"htc",STR.hudTask(state.roundTotal));
    const pct = state.roundTotal ? Math.round(state.roundCorrect / state.roundTotal * 100) : 0;
    _html(hudAccuracy,"hac",STR.hudAcc(pct));
    _cls(hudAccuracy,"hacc","stage-hud-item " + (pct >= PASS_RATE*100 ? "hud-pass" : "hud-fail"));
  } else if(counting){
    _w(cmdfill,"cf","100%");
    _html(hudTaskCount,"htc",STR.hudTask(0));
    _html(hudAccuracy,"hac",STR.hudAcc(0));
    _cls(hudAccuracy,"hacc","stage-hud-item hud-fail");
  }
}
function escapeHtml(s){ return String(s).replace(/[&<>"']/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c])); }

/* ===================== Camera ===================== */
async function startCamera(){
  if(cameraReady) return;
  const local=location.hostname==="localhost"||location.hostname==="127.0.0.1";
  if(!window.isSecureContext && !local) throw new Error("INSECURE");
  /* [수정#20] 카메라 API 자체가 없는 구형/임베디드 WebView → 친절한 메시지로 우아하게 실패 */
  if(!(navigator.mediaDevices && navigator.mediaDevices.getUserMedia) && typeof Camera==="undefined") throw new Error("NO_API");
  if(!camera){
    /* 모바일: 전면(셀피) 카메라 + 저해상도로 성능 확보. 데스크톱: 720p */
    camera=new Camera(vid,{
      onFrame: async ()=>{
        if(sending||!hands) return;
        /* [FIX#5] 추론은 필요한 페이즈에서만: 플레이(미정지)·카운트다운.
           인트로/선택/결과/게임오버/일시정지 중 불필요한 손인식 추론을 차단해
           발열·배터리·메인스레드 부하를 줄인다(오버레이가 스테이지를 덮는 구간). */
        const active=(state.phase==="playing"&&!state.paused)||state.phase==="countdown";
        if(!active) return;
        /* [수정#8/끊김] 적응형 프레임 스킵: 직전 추론시간 EMA가 길면 프레임을 건너뛰어
           메인스레드 여유를 확보(끊김 완화). 빠른 기기는 _skipMod=1 → 동작 동일. */
        if((_skipCtr++ % _skipMod) !== 0) return;
        sending=true;
        const t0=performance.now();
        try{ await hands.send({ image: vid }); }
        catch{ /* [FIX#20] WebGL 컨텍스트 손실 등 일시 오류는 삼키고 다음 프레임 재시도 */ }
        finally{
          const dt=performance.now()-t0;
          _sendAvg = _sendAvg ? _sendAvg*0.8 + dt*0.2 : dt;
          _skipMod = _sendAvg>55 ? 3 : _sendAvg>38 ? 2 : 1;
          sending=false;
        }
      },
      width: IS_MOBILE?640:1280, height: IS_MOBILE?480:720,
      facingMode:"user"
    });
  }
  await camera.start();
  cameraReady=true;
}
/* [FIX#4] 카메라 스트림 해제: 디바이스 점유·표시등·배터리 누수 방지 */
function stopCamera(){
  try{ if(camera && typeof camera.stop==="function") camera.stop(); }catch{}
  try{
    const ms=vid.srcObject;
    if(ms && ms.getTracks){ for(const t of ms.getTracks()){ try{ t.stop(); }catch{} } }
    vid.srcObject=null;
  }catch{}
  camera=null; cameraReady=false;
}

/* [FIX#4/#12] 페이지 이탈 정리: 발화·스케줄·웨이크락·카메라 모두 해제 */
function teardownAll(){
  try{ clearSchedule(); }catch{}
  try{ clearInterval(cdTimer); clearTimeout(cdTimer); }catch{}
  stopSpeak(); releaseWake(); stopCamera();
}
window.addEventListener("pagehide", teardownAll);
window.addEventListener("beforeunload", teardownAll);

/* [FIX#20] 전역 소프트 에러 핸들러: 미처리 예외/거부가 콘솔 폭주·루프 중단으로
   이어지지 않게 흡수(게임 루프는 rAF try/catch로 계속 유지). */
window.addEventListener("error", ()=>{ /* swallow to keep game alive */ });
window.addEventListener("unhandledrejection", (e)=>{ try{ e.preventDefault(); }catch{} });

function cameraErrorMessage(e){
  switch(e&&(e.name||e.message)){
    case "NO_API": return STR.cam.NO_API;
    case "INSECURE": return STR.cam.INSECURE;
    case "NotAllowedError": case "PermissionDeniedError": case "SecurityError": return STR.cam.DENIED;
    case "NotFoundError": case "DevicesNotFoundError": return STR.cam.NOT_FOUND;
    case "NotReadableError": case "TrackStartError": case "AbortError": return STR.cam.IN_USE;
    case "OverconstrainedError": case "ConstraintNotSatisfiedError": return STR.cam.CONSTRAINT;
    default: return STR.cam.DEFAULT;
  }
}
function showFatal(msg){ fatalEl.textContent=msg; fatalEl.style.display="block"; }

/* ===================== Layout Responsive ===================== */
function resizeCanvas(){
  const r=cv.getBoundingClientRect();
  if(r.width<2||r.height<2) return;
  const dpr=Math.min(window.devicePixelRatio||1, IS_MOBILE?1.5:2);  /* [끊김] 모바일 픽셀면적 절감 */
  W=Math.round(r.width*dpr); H=Math.round(r.height*dpr);
  cv.width=W; cv.height=H;
  _lastBig=null; /* 리사이즈 시 다음 프레임에 중앙 텍스트 재맞춤 */
}
let _rzT=0;
function onResize(){
  /* [수정#14] 회전/주소창 변화 시 즉시 표시 크기에 맞춰 캔버스를 갱신해 검은 띠·왜곡을 방지하고,
     이어서 디바운스로 최종 안정값을 한 번 더 적용(레이아웃 진동 흡수). */
  resizeCanvas();
  clearTimeout(_rzT); _rzT=setTimeout(resizeCanvas, 120);
}
function fsSupported(){ return !!(app.requestFullscreen||app.webkitRequestFullscreen||app.webkitRequestFullScreen); }
function fsElement(){ return document.fullscreenElement||document.webkitFullscreenElement||null; }
function toggleFullscreen(){
  if(!fsElement()){
    const req=app.requestFullscreen||app.webkitRequestFullscreen||app.webkitRequestFullScreen;
    if(req){ const p=req.call(app); if(p&&p.catch) p.catch(()=>{}); }
  } else {
    const exit=document.exitFullscreen||document.webkitExitFullscreen;
    if(exit) exit.call(document);
  }
}

/* ===================== Visibility System ===================== */
document.addEventListener("visibilitychange", ()=>{
  if(document.hidden){
    /* 탭 숨김: 진행 중이면 일시정지(스케줄러 보존 + 발화 정지) */
    if(state.phase==="playing" && !state.paused){ pauseGame(); }
  } else {
    /* 복귀: 사용자가 명시적으로 누른 일시정지가 아니면 자동 재개 */
    if(state.paused && !state.userPaused && state.phase==="playing"){
      acquireWake();               /* [수정#9] 탭 복귀 시 화면 유지 재획득 */
      resumeGame();
    }
  }
});

/* ===================== PWA ([#12] manifest + service worker) ===================== */
function installPWA(){
  /* [수정#4] 아이콘 없는 빈 manifest → 홈 추가 시 아이콘 미표시. 캔버스로 즉석 PNG 생성해 주입. */
  function makeIcon(size, maskable){
    try{
      const c=document.createElement("canvas"); c.width=c.height=size;
      const x=c.getContext("2d");
      x.fillStyle="#0a0a0b"; x.fillRect(0,0,size,size);
      const pad=maskable?size*0.18:size*0.12, fl=size*0.30;
      x.fillStyle="#2563eb"; x.fillRect(pad, pad, fl, fl);                       /* 청기 */
      x.fillStyle="#f4f4f5"; x.fillRect(size-pad-fl, size-pad-fl, fl, fl);       /* 백기 */
      x.fillStyle="#facc15"; x.font=`900 ${Math.round(size*0.22)}px 'Pretendard',system-ui,sans-serif`;
      x.textAlign="center"; x.textBaseline="middle"; x.fillText("청백", size/2, size/2);
      return c.toDataURL("image/png");
    }catch{ return ""; }
  }
  try{
    const i192=makeIcon(192,false), i512=makeIcon(512,true);
    const icons=[];
    if(i192) icons.push({ src:i192, sizes:"192x192", type:"image/png", purpose:"any" });
    if(i512) icons.push({ src:i512, sizes:"512x512", type:"image/png", purpose:"any maskable" });
    const manifest={
      name:"청기백기 — MOTION-SENSING GAME", short_name:"청기백기",
      start_url:".", display:"standalone", orientation:"any",
      background_color:"#0a0a0b", theme_color:"#0a0a0b",
      icons
    };
    const blob=new Blob([JSON.stringify(manifest)],{type:"application/manifest+json"});
    const link=document.createElement("link");
    link.rel="manifest"; link.href=URL.createObjectURL(blob);
    document.head.appendChild(link);
  }catch{}
  if("serviceWorker" in navigator && location.protocol==="https:"){
    /* [수정#3] sw.js 를 함께 배포하면 오프라인 캐싱 동작. 미배포 시 조용히 무시. */
    navigator.serviceWorker.register("sw.js").catch(()=>{});
  }
}

/* ===================== Self tests ([#14] ?selftest=1) ===================== */
function runSelfTests(){
  const t=[]; const ok=(n,c)=>t.push({test:n, pass:!!c});
  ok("INVERT involutive", INVERT[INVERT.raise]==="raise" && INVERT[INVERT.dontRaise]==="dontRaise");
  ok("applyInstruction raise", applyInstruction({blue:"down",white:"down"},{flag:"blue",action:"raise"},false).blue==="up");
  ok("reverse flips action", applyInstruction({blue:"down",white:"down"},{flag:"blue",action:"raise"},true).blue==="down");
  ok("dontRaise keeps pose", applyInstruction({blue:"down",white:"down"},{flag:"blue",action:"dontRaise"},false).blue==="down");
  ok("isPoseCorrect", isPoseCorrect({blue:"up",white:"down"},{blue:"up",white:"down"})===true && isPoseCorrect({blue:"up",white:"up"},{blue:"up",white:"down"})===false);
  ok("levelOf/xpForLevel inverse", levelOf(xpForLevel(5))===5 && levelOf(0)===1);
  ok("estSpeechMs positive", estSpeechMs("청기 올려",1.2)>0);
  ok("reverseP bounds", reverseP(1,"normal")===0 && reverseP(99,"boss")<=0.62 && reverseP(99,"boss")>=0);
  ok("roundType cadence", roundTypeOf(10)==="boss" && roundTypeOf(5)==="bonus" && roundTypeOf(3)==="normal");
  // generateCommand always changes target (no-op guard)
  let allChanged=true;
  for(let i=0;i<300;i++){
    const prev={blue: i%2?"up":"down", white: i%3?"up":"down"};
    const c=generateCommand(prev, 1+(i%40), false, roundTypeOf(1+(i%40)), i%2?"textReverse":"original");
    if(c.target.blue===prev.blue && c.target.white===prev.white){ allChanged=false; break; }
  }
  ok("generateCommand mutates target (x300)", allChanged);
  const passed=t.filter(x=>x.pass).length;
  console.log(`%c[selftest] ${passed}/${t.length} passed`, `color:${passed===t.length?"#10b981":"#ef4444"};font-weight:bold`);
  console.table(t);
  return t;
}

/* ===================== Initialization ===================== */
function createHands(){
  const base = window.__MP_HANDS_BASE || "https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1675469240/";
  const h=new Hands({ locateFile:(f)=> base + f });
  /* [수정#8/끊김] 모바일은 modelComplexity 0(경량)으로 시작 → 메인스레드 추론 부하·프레임 끊김 대폭 감소.
     데스크톱은 1(정확도) 유지. 추가 적응은 onFrame 프레임 스킵으로 처리. */
  h.setOptions({ maxNumHands:2, modelComplexity:IS_MOBILE?0:1, minDetectionConfidence:0.5, minTrackingConfidence:0.5 });
  h.onResults(onHandResults);
  return h;
}

function init(){
  state.bestScore=loadBest();
  state.xp=loadXP();
  state.nick=loadNick();
  applySkin("classic"); /* [요청5] 스킨 선택 제거 — 클래식 고정 */
  state.style=loadStyle(); renderStyles();
  for(const b of document.querySelectorAll(".stbtn")){
    b.addEventListener("click", ()=>{ initAudio(); setStyle(b.dataset.style); });
  }

  pauseToggle.addEventListener("click", togglePause);
  /* [#11] 진행 중 이탈은 확인 후 진행 */
  quitCtrlBtn.addEventListener("click", ()=>{
    if(state.phase==="playing" && !window.confirm(STR.confirmEnd)) return;
    state.userPaused=false; finishGame(false);
  });

  if(TTS_SUPPORTED){
    startVoicePolling();
    window.speechSynthesis.onvoiceschanged=refreshVoices;
  }

  resizeCanvas();
  window.addEventListener("resize", onResize);
  window.addEventListener("orientationchange", onResize);
  /* [수정#5/#14] 모바일 주소창 노출/키보드 등 visualViewport 변화에도 캔버스 재맞춤 */
  try{ if(window.visualViewport){ window.visualViewport.addEventListener("resize", onResize); } }catch{}
  document.addEventListener("fullscreenchange", onResize);
  document.addEventListener("webkitfullscreenchange", onResize);
  /* [요청5] 폰트 로드 후 중앙 텍스트 재맞춤(웹폰트는 늦게 로드되어 폭이 바뀜) */
  try{ if(document.fonts && document.fonts.ready) document.fonts.ready.then(()=>{ _lastBig=null; fitBig(); }); }catch{}
  /* [요청5,8] 스테이지 크기 변화(회전·전체화면·키보드)에 폰트 재맞춤 */
  try{ if("ResizeObserver" in window){ new ResizeObserver(()=>{ _lastBig=null; }).observe(stage); } }catch{}
  if(fsSupported()){ fsBtn.addEventListener("click", toggleFullscreen); }
  else { fsBtn.style.display="none"; }

  startBtn.addEventListener("click", ()=>{ initAudio(); connectCamera(showRoundSelect); });

  installPWA();

  if(typeof Hands==="undefined" || typeof Camera==="undefined"){
    showFatal(STR.mpFail);
    startBtn.textContent="로드 실패"; return;
  }
  try{ hands=createHands(); }
  catch(e){ showFatal(STR.modelFail(e.message)); startBtn.textContent="로드 실패"; return; }
  startBtn.disabled=false;
  startBtn.textContent="게임 시작 (카메라 연결)";

  if(/[?&]selftest=1/.test(location.search)) runSelfTests();
}

/* [수정#1] MediaPipe 로드 완료 후 초기화(로드 실패해도 init 진입 → 기존 실패 메시지 유지) */
if(window.__mpReady && typeof window.__mpReady.then==="function"){ window.__mpReady.then(init); }
else { init(); }

/* [#10][#14] 순수 로직 테스트 노출 */
window.__cbg={ generateCommand, resolveTarget, applyInstruction, isPoseCorrect, levelOf, xpForLevel, INVERT, reverseP, estSpeechMs, roundTypeOf, runSelfTests };

})();
</script>
</body>
</html>
