const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.PORT || 3000;
const DATA_FILE = path.join(__dirname, 'submissions.json');

function ensureDataFile() {
  if (!fs.existsSync(DATA_FILE)) {
    fs.writeFileSync(DATA_FILE, '[]', 'utf8');
  }
}

function readSubmissions() {
  ensureDataFile();
  try {
    return JSON.parse(fs.readFileSync(DATA_FILE, 'utf8'));
  } catch {
    return [];
  }
}

function writeSubmissions(data) {
  fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2), 'utf8');
}

function sendJson(res, statusCode, data) {
  res.writeHead(statusCode, {
    'Content-Type': 'application/json; charset=utf-8',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET,POST,OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type'
  });
  res.end(JSON.stringify(data));
}

function sendHtml(res, html) {
  res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
  res.end(html);
}

function collectBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', chunk => {
      body += chunk.toString();
      if (body.length > 1_000_000) reject(new Error('Solicitud demasiado grande'));
    });
    req.on('end', () => resolve(body));
    req.on('error', reject);
  });
}

function createHtml() {
  return `<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Nébula Motion</title>
  <meta name="description" content="Web fullstack en un solo archivo con diseño 3D, animaciones realistas y backend funcional." />
  <style>
    :root{
      --bg:#040814;
      --bg2:#0b1022;
      --panel:rgba(255,255,255,.08);
      --line:rgba(255,255,255,.12);
      --text:#f5f7ff;
      --muted:rgba(235,240,255,.72);
      --cyan:#4ef2ff;
      --violet:#8c5bff;
      --pink:#ff4fd8;
      --gold:#ffca6b;
      --success:#79ffbf;
      --shadow:0 30px 80px rgba(0,0,0,.45);
      --radius:26px;
      --radius2:20px;
      --max:1180px;
    }

    *{box-sizing:border-box}
    html{scroll-behavior:smooth}
    body{
      margin:0;
      font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,sans-serif;
      color:var(--text);
      background:
        radial-gradient(circle at 15% 20%, rgba(140,91,255,.22), transparent 30%),
        radial-gradient(circle at 85% 20%, rgba(78,242,255,.14), transparent 26%),
        radial-gradient(circle at 50% 90%, rgba(255,79,216,.15), transparent 30%),
        linear-gradient(135deg, #02040b 0%, #07101d 45%, #070b1d 100%);
      overflow-x:hidden;
      min-height:100vh;
    }

    #bgCanvas{
      position:fixed;
      inset:0;
      z-index:0;
      opacity:.95;
    }

    .aurora,
    .grid-overlay{
      position:fixed;
      inset:0;
      pointer-events:none;
      z-index:0;
    }

    .grid-overlay{
      background-image:
        linear-gradient(rgba(255,255,255,.025) 1px, transparent 1px),
        linear-gradient(90deg, rgba(255,255,255,.025) 1px, transparent 1px);
      background-size:56px 56px;
      mask-image:radial-gradient(circle at center, black 42%, transparent 92%);
      opacity:.35;
    }

    .aurora span{
      position:absolute;
      width:42vw;
      height:42vw;
      border-radius:50%;
      filter:blur(40px);
      mix-blend-mode:screen;
      opacity:.18;
      animation:drift 18s ease-in-out infinite;
    }

    .aurora span:nth-child(1){
      left:-8vw; top:0;
      background:radial-gradient(circle, rgba(255,79,216,.95), transparent 65%);
    }

    .aurora span:nth-child(2){
      right:-8vw; top:8vh;
      background:radial-gradient(circle, rgba(78,242,255,.95), transparent 65%);
      animation-delay:-6s;
    }

    .aurora span:nth-child(3){
      left:25vw; bottom:-10vh;
      background:radial-gradient(circle, rgba(140,91,255,.95), transparent 65%);
      animation-delay:-10s;
    }

    .page{
      position:relative;
      z-index:1;
    }

    .wrap{
      width:min(var(--max), calc(100% - 24px));
      margin:0 auto;
    }

    .topbar{
      position:sticky;
      top:0;
      z-index:20;
      backdrop-filter:blur(18px);
      background:rgba(5,8,22,.45);
      border-bottom:1px solid rgba(255,255,255,.06);
    }

    .topbar-inner{
      width:min(var(--max), calc(100% - 24px));
      margin:0 auto;
      min-height:72px;
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:16px;
    }

    .brand{
      display:flex;
      align-items:center;
      gap:12px;
      font-weight:800;
      letter-spacing:.04em;
    }

    .brand-badge{
      width:42px;
      height:42px;
      border-radius:15px;
      position:relative;
      overflow:hidden;
      background:
        linear-gradient(145deg, rgba(255,255,255,.16), rgba(255,255,255,.03)),
        linear-gradient(135deg, rgba(78,242,255,.45), rgba(140,91,255,.55), rgba(255,79,216,.5));
      box-shadow:inset 0 1px 0 rgba(255,255,255,.22), 0 14px 30px rgba(78,242,255,.18);
    }

    .brand-badge:after{
      content:"";
      position:absolute;
      inset:10px 9px;
      border-radius:12px;
      border:1px solid rgba(255,255,255,.28);
      transform:rotate(18deg);
    }

    .nav{
      display:flex;
      align-items:center;
      gap:16px;
      flex-wrap:wrap;
    }

    .nav a{
      color:var(--muted);
      text-decoration:none;
      transition:.2s ease;
      font-size:.97rem;
    }

    .nav a:hover{
      color:#fff;
      transform:translateY(-2px);
    }

    .hero{
      padding:70px 0 40px;
    }

    .hero-grid{
      display:grid;
      grid-template-columns:1.08fr .92fr;
      gap:28px;
      align-items:center;
    }

    .eyebrow{
      display:inline-flex;
      align-items:center;
      gap:10px;
      padding:11px 16px;
      border-radius:999px;
      background:rgba(255,255,255,.06);
      border:1px solid rgba(255,255,255,.08);
      color:var(--muted);
      margin-bottom:18px;
      backdrop-filter:blur(12px);
    }

    .eyebrow-dot{
      width:10px;
      height:10px;
      border-radius:50%;
      background:radial-gradient(circle, #fff 5%, var(--success) 44%, rgba(121,255,191,.2) 74%);
      box-shadow:0 0 16px rgba(121,255,191,.8);
      animation:blink 2.2s ease-in-out infinite;
      flex:0 0 auto;
    }

    h1{
      margin:0;
      font-size:clamp(2.8rem, 7vw, 6rem);
      line-height:.95;
      letter-spacing:-.06em;
    }

    .gradient{
      background:linear-gradient(90deg, #fff 0%, #b4f8ff 22%, #ad99ff 52%, #ff92ea 80%, #ffe5ab 100%);
      background-size:220% auto;
      -webkit-background-clip:text;
      background-clip:text;
      color:transparent;
      animation:shimmer 6s linear infinite;
    }

    .hero p,
    .section-intro p,
    .card p,
    .showcase-copy p,
    .contact-copy p,
    .stat-label,
    .footer p,
    .status{
      color:var(--muted);
      line-height:1.7;
    }

    .hero p{
      font-size:1.08rem;
      margin:18px 0 24px;
      max-width:760px;
    }

    .hero-actions{
      display:flex;
      gap:12px;
      flex-wrap:wrap;
      margin-bottom:24px;
    }

    .btn{
      appearance:none;
      border:0;
      text-decoration:none;
      color:#fff;
      padding:15px 22px;
      border-radius:999px;
      font-weight:800;
      letter-spacing:.02em;
      display:inline-flex;
      align-items:center;
      justify-content:center;
      gap:8px;
      cursor:pointer;
      transition:.2s ease;
    }

    .btn:hover{transform:translateY(-2px)}
    .btn-primary{
      background:linear-gradient(135deg, var(--cyan), var(--violet) 52%, var(--pink));
      box-shadow:0 18px 42px rgba(127,96,255,.28);
    }

    .btn-secondary{
      background:rgba(255,255,255,.08);
      border:1px solid rgba(255,255,255,.08);
      backdrop-filter:blur(16px);
    }

    .hero-meta{
      display:flex;
      gap:12px;
      flex-wrap:wrap;
    }

    .pill{
      min-width:150px;
      padding:14px 16px;
      border-radius:18px;
      background:rgba(255,255,255,.05);
      border:1px solid rgba(255,255,255,.08);
      backdrop-filter:blur(14px);
    }

    .pill strong{
      display:block;
      margin-bottom:4px;
      color:#fff;
      font-size:1.06rem;
    }

    .hero-visual{
      position:relative;
      min-height:560px;
      perspective:1600px;
    }

    .orbital-stage{
      position:absolute;
      inset:0;
      display:grid;
      place-items:center;
      transform-style:preserve-3d;
    }

    .holo-card{
      width:min(100%, 520px);
      transform-style:preserve-3d;
      animation:floatCard 7s ease-in-out infinite;
      will-change:transform;
    }

    .holo-shell{
      position:relative;
      padding:16px;
      border-radius:32px;
      background:
        linear-gradient(145deg, rgba(255,255,255,.16), rgba(255,255,255,.04)),
        rgba(8,12,28,.55);
      border:1px solid rgba(255,255,255,.14);
      box-shadow:var(--shadow);
      backdrop-filter:blur(20px);
      overflow:hidden;
    }

    .holo-shell:before{
      content:"";
      position:absolute;
      inset:-25%;
      background:conic-gradient(from 0deg, rgba(78,242,255,.18), rgba(255,79,216,.2), rgba(140,91,255,.24), rgba(78,242,255,.18));
      filter:blur(24px);
      animation:spin 12s linear infinite;
    }

    .holo-screen{
      position:relative;
      z-index:1;
      min-height:430px;
      padding:18px;
      border-radius:24px;
      background:
        linear-gradient(180deg, rgba(255,255,255,.1), rgba(255,255,255,.03)),
        linear-gradient(145deg, rgba(8,11,28,.95), rgba(12,18,39,.76));
      border:1px solid rgba(255,255,255,.08);
      transform:translateZ(55px);
    }

    .screen-header{
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:10px;
      margin-bottom:14px;
    }

    .dots{display:flex; gap:7px}
    .dots span{
      width:10px;
      height:10px;
      border-radius:50%;
      display:block;
    }
    .dots span:nth-child(1){background:#ff7f7f}
    .dots span:nth-child(2){background:#ffd96d}
    .dots span:nth-child(3){background:#7effbc}

    .metric-bar,
    .mini-panel,
    .wave-panel,
    .card,
    .stat-card,
    .showcase-panel,
    .showcase-copy,
    .contact-box,
    .footer-box{
      position:relative;
      overflow:hidden;
      border-radius:var(--radius);
      background:
        linear-gradient(180deg, rgba(255,255,255,.1), rgba(255,255,255,.03)),
        rgba(8,12,28,.58);
      border:1px solid rgba(255,255,255,.08);
      box-shadow:var(--shadow);
      backdrop-filter:blur(16px);
    }

    .metric-bar,
    .mini-panel,
    .wave-panel{
      padding:16px;
      border-radius:22px;
      background:rgba(255,255,255,.05);
      box-shadow:inset 0 1px 0 rgba(255,255,255,.08);
    }

    .metric-bar{
      min-height:118px;
      display:grid;
      align-content:space-between;
      margin-bottom:14px;
    }

    .metric-track{
      height:13px;
      border-radius:999px;
      background:rgba(255,255,255,.08);
      overflow:hidden;
      position:relative;
    }

    .metric-fill{
      position:absolute;
      inset:0 auto 0 0;
      width:78%;
      border-radius:inherit;
      background:linear-gradient(90deg, var(--cyan), var(--violet), var(--pink));
      box-shadow:0 0 24px rgba(140,91,255,.4);
      animation:progressPulse 4s ease-in-out infinite;
    }

    .mini-grid{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:14px;
      margin-bottom:14px;
    }

    .mini-panel strong,
    .metric-bar strong,
    .wave-panel strong{
      display:block;
      margin-bottom:8px;
    }

    .sparkline{
      height:74px;
      border-radius:16px;
      position:relative;
      overflow:hidden;
      background:
        linear-gradient(180deg, rgba(255,255,255,.06), transparent),
        radial-gradient(circle at 20% 40%, rgba(78,242,255,.22), transparent 55%);
    }

    .sparkline:before,
    .sparkline:after{
      content:"";
      position:absolute;
      inset:auto auto 0 0;
      width:140%;
      height:130%;
      border-radius:46% 54% 58% 42% / 40% 36% 64% 60%;
      border:2px solid rgba(255,255,255,.22);
      opacity:.35;
      transform:translateX(-10%) translateY(26%);
      animation:waveSlide 5s linear infinite;
    }

    .sparkline:after{
      border-color:rgba(78,242,255,.6);
      animation-duration:7s;
      animation-direction:reverse;
    }

    .wave-panel{
      min-height:138px;
    }

    .wave-panel:before,
    .wave-panel:after{
      content:"";
      position:absolute;
      left:-10%;
      width:120%;
      height:88px;
      border-radius:50%;
      border:1px solid rgba(255,255,255,.15);
      animation:scanFloat 6s ease-in-out infinite;
    }

    .wave-panel:before{bottom:12px}
    .wave-panel:after{
      bottom:34px;
      animation-delay:-3s;
      border-color:rgba(78,242,255,.35);
    }

    .floating-tag{
      position:absolute;
      padding:12px 16px;
      border-radius:18px;
      background:rgba(255,255,255,.08);
      border:1px solid rgba(255,255,255,.12);
      backdrop-filter:blur(15px);
      box-shadow:var(--shadow);
      animation:floatTag 6s ease-in-out infinite;
    }

    .tag-a{top:10%; right:8%}
    .tag-b{bottom:13%; left:2%; animation-delay:-2s}
    .tag-c{bottom:3%; right:0; animation-delay:-4s}

    .section{
      padding:34px 0;
    }

    .section-intro{
      max-width:760px;
      margin-bottom:18px;
    }

    .section-intro h2{
      margin:0 0 8px;
      font-size:clamp(2rem,4vw,3.4rem);
      line-height:1.05;
      letter-spacing:-.04em;
    }

    .feature-grid{
      display:grid;
      grid-template-columns:repeat(3, 1fr);
      gap:16px;
    }

    .card{
      padding:22px;
    }

    .card:before,
    .stat-card:before,
    .showcase-panel:before,
    .showcase-copy:before,
    .contact-box:before,
    .footer-box:before{
      content:"";
      position:absolute;
      inset:-40% auto auto -30%;
      width:220px;
      height:220px;
      background:radial-gradient(circle, rgba(78,242,255,.12), transparent 60%);
      transform:rotate(25deg);
    }

    .chip{
      width:48px;
      height:48px;
      display:grid;
      place-items:center;
      border-radius:18px;
      margin-bottom:14px;
      background:linear-gradient(135deg, rgba(78,242,255,.25), rgba(140,91,255,.28), rgba(255,79,216,.2));
      border:1px solid rgba(255,255,255,.1);
      box-shadow:inset 0 1px 0 rgba(255,255,255,.18);
      font-size:1.2rem;
    }

    .card h3,
    .contact-copy h3{
      margin:0 0 10px;
      font-size:1.25rem;
    }

    .stats-grid{
      display:grid;
      grid-template-columns:repeat(4,1fr);
      gap:16px;
    }

    .stat-card{
      padding:22px;
    }

    .stat-value{
      font-size:clamp(2rem,4vw,3.1rem);
      font-weight:900;
      line-height:1;
      margin-bottom:6px;
    }

    .showcase-grid{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:16px;
      align-items:stretch;
    }

    .showcase-panel{
      min-height:470px;
      display:grid;
      place-items:center;
      perspective:1300px;
      padding:20px;
    }

    .cube-wrap{
      position:relative;
      width:260px;
      height:260px;
      transform-style:preserve-3d;
      animation:cubeRotate 16s linear infinite;
    }

    .cube-face{
      position:absolute;
      inset:0;
      border-radius:28px;
      border:1px solid rgba(255,255,255,.12);
      background:
        linear-gradient(145deg, rgba(255,255,255,.13), rgba(255,255,255,.02)),
        rgba(11,15,36,.52);
      box-shadow:inset 0 1px 0 rgba(255,255,255,.15);
      overflow:hidden;
      backdrop-filter:blur(20px);
    }

    .cube-face:before{
      content:"";
      position:absolute;
      inset:12%;
      border-radius:22px;
      background:linear-gradient(135deg, rgba(78,242,255,.2), rgba(140,91,255,.1), rgba(255,79,216,.2));
      border:1px solid rgba(255,255,255,.08);
    }

    .cube-face span{
      position:absolute;
      left:50%;
      top:50%;
      transform:translate(-50%, -50%);
      font-size:1.1rem;
      font-weight:800;
      letter-spacing:.08em;
      text-transform:uppercase;
    }

    .front{transform:translateZ(130px)}
    .back{transform:rotateY(180deg) translateZ(130px)}
    .right{transform:rotateY(90deg) translateZ(130px)}
    .left{transform:rotateY(-90deg) translateZ(130px)}
    .top{transform:rotateX(90deg) translateZ(130px)}
    .bottom{transform:rotateX(-90deg) translateZ(130px)}

    .cube-ring,
    .cube-ring:before,
    .cube-ring:after{
      content:"";
      position:absolute;
      inset:-46px;
      border-radius:50%;
      border:1px solid rgba(255,255,255,.16);
    }

    .cube-ring{
      transform:rotateX(70deg);
      animation:ringSpin 8s linear infinite;
    }

    .cube-ring:before{
      inset:28px;
      border-color:rgba(78,242,255,.34);
      animation:ringSpinReverse 10s linear infinite;
    }

    .cube-ring:after{
      inset:66px;
      border-color:rgba(255,79,216,.32);
      animation:ringSpin 6s linear infinite;
    }

    .showcase-copy{
      padding:24px;
      display:grid;
      align-content:center;
      gap:14px;
    }

    .check{
      display:flex;
      gap:12px;
      align-items:flex-start;
      padding:14px 16px;
      border-radius:18px;
      background:rgba(255,255,255,.04);
      border:1px solid rgba(255,255,255,.06);
      color:var(--muted);
      line-height:1.6;
    }

    .check-mark{
      width:24px;
      height:24px;
      display:grid;
      place-items:center;
      border-radius:50%;
      background:linear-gradient(135deg, var(--success), var(--cyan));
      color:#061018;
      font-weight:900;
      flex:0 0 auto;
      margin-top:2px;
    }

    .check strong{
      display:block;
      color:#fff;
      margin-bottom:3px;
    }

    .contact-grid{
      display:grid;
      grid-template-columns:.95fr 1.05fr;
      gap:16px;
    }

    .contact-box{
      padding:24px;
    }

    .contact-copy ul{
      list-style:none;
      padding:0;
      margin:16px 0 0;
      display:grid;
      gap:10px;
    }

    .contact-copy li{
      padding:14px 16px;
      border-radius:18px;
      background:rgba(255,255,255,.04);
      border:1px solid rgba(255,255,255,.06);
      color:var(--muted);
    }

    form{
      display:grid;
      gap:14px;
    }

    .field-row{
      display:grid;
      grid-template-columns:1fr 1fr;
      gap:14px;
    }

    label{
      display:grid;
      gap:8px;
      font-size:.95rem;
      color:rgba(255,255,255,.9);
    }

    input, textarea, select{
      width:100%;
      border-radius:18px;
      border:1px solid rgba(255,255,255,.08);
      background:rgba(255,255,255,.05);
      padding:15px 16px;
      color:#fff;
      font:inherit;
      outline:none;
      transition:.2s ease;
      box-shadow:inset 0 1px 0 rgba(255,255,255,.06);
    }

    input::placeholder,
    textarea::placeholder{
      color:rgba(230,236,255,.45);
    }

    input:focus,
    textarea:focus,
    select:focus{
      border-color:rgba(78,242,255,.54);
      transform:translateY(-1px);
      box-shadow:0 0 0 4px rgba(78,242,255,.12);
    }

    textarea{
      min-height:170px;
      resize:vertical;
    }

    .submit-row{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:14px;
      flex-wrap:wrap;
    }

    .footer{
      padding:20px 0 48px;
    }

    .footer-grid{
      display:grid;
      grid-template-columns:1.2fr .8fr;
      gap:16px;
    }

    .footer-box{
      padding:22px;
    }

    .footer-box h3{
      margin:0 0 10px;
    }

    .footer-links{
      display:grid;
      gap:8px;
    }

    .footer-links a{
      color:var(--muted);
      text-decoration:none;
    }

    .footer-links a:hover{color:#fff}

    .toast{
      position:fixed;
      right:16px;
      bottom:16px;
      z-index:100;
      max-width:340px;
      padding:16px 18px;
      border-radius:18px;
      background:rgba(12,18,40,.92);
      border:1px solid rgba(255,255,255,.1);
      backdrop-filter:blur(18px);
      box-shadow:var(--shadow);
      transform:translateY(120%);
      transition:transform .28s ease;
    }

    .toast.show{
      transform:translateY(0);
    }

    @keyframes shimmer{
      0%{background-position:0% 50%}
      100%{background-position:200% 50%}
    }

    @keyframes drift{
      0%,100%{transform:translate3d(0,0,0) scale(1)}
      33%{transform:translate3d(4vw,-3vh,0) scale(1.08)}
      66%{transform:translate3d(-3vw,4vh,0) scale(.94)}
    }

    @keyframes blink{
      0%,100%{transform:scale(.88);opacity:.75}
      50%{transform:scale(1.1);opacity:1}
    }

    @keyframes floatCard{
      0%,100%{transform:rotateX(6deg) rotateY(-9deg) translateY(0)}
      50%{transform:rotateX(-2deg) rotateY(9deg) translateY(-16px)}
    }

    @keyframes spin{
      from{transform:rotate(0deg)}
      to{transform:rotate(360deg)}
    }

    @keyframes progressPulse{
      0%,100%{width:72%}
      50%{width:86%}
    }

    @keyframes waveSlide{
      from{transform:translateX(-14%) translateY(28%) rotate(0deg)}
      to{transform:translateX(-2%) translateY(26%) rotate(360deg)}
    }

    @keyframes scanFloat{
      0%,100%{transform:translateY(0) scaleX(1); opacity:.45}
      50%{transform:translateY(-10px) scaleX(1.04); opacity:.9}
    }

    @keyframes floatTag{
      0%,100%{transform:translate3d(0,0,0)}
      50%{transform:translate3d(0,-10px,0)}
    }

    @keyframes cubeRotate{
      0%{transform:rotateX(-18deg) rotateY(0deg)}
      100%{transform:rotateX(-18deg) rotateY(360deg)}
    }

    @keyframes ringSpin{
      from{transform:rotateX(75deg) rotateZ(0deg)}
      to{transform:rotateX(75deg) rotateZ(360deg)}
    }

    @keyframes ringSpinReverse{
      from{transform:rotateY(70deg) rotateZ(360deg)}
      to{transform:rotateY(70deg) rotateZ(0deg)}
    }

    @media (max-width: 980px){
      .hero-grid,
      .feature-grid,
      .stats-grid,
      .showcase-grid,
      .contact-grid,
      .footer-grid{
        grid-template-columns:1fr;
      }

      .hero-visual{min-height:430px}
      .field-row,
      .mini-grid{grid-template-columns:1fr}
    }

    @media (max-width: 640px){
      .topbar-inner{
        padding:12px 0;
        align-items:flex-start;
        flex-direction:column;
      }

      .nav{width:100%; justify-content:space-between}
      .hero{padding-top:46px}
      h1{font-size:clamp(2.4rem, 14vw, 4.6rem)}
      .hero-actions,
      .submit-row{width:100%}
      .btn{width:100%}
      .floating-tag{display:none}
      .hero-visual{min-height:360px}
    }
  </style>
</head>
<body>
  <canvas id="bgCanvas"></canvas>
  <div class="aurora" aria-hidden="true">
    <span></span>
    <span></span>
    <span></span>
  </div>
  <div class="grid-overlay" aria-hidden="true"></div>

  <div class="page">
    <header class="topbar">
      <div class="topbar-inner">
        <div class="brand">
          <div class="brand-badge"></div>
          <span>Nébula Motion</span>
        </div>
        <nav class="nav">
          <a href="#inicio">Inicio</a>
          <a href="#efectos">Efectos</a>
          <a href="#fullstack">Fullstack</a>
          <a href="#contacto">Contacto</a>
        </nav>
      </div>
    </header>

    <main>
      <section class="hero wrap" id="inicio">
        <div class="hero-grid">
          <div>
            <div class="eyebrow">
              <span class="eyebrow-dot"></span>
              Experiencia fullstack en un solo archivo
            </div>

            <h1>
              Web 3D con
              <span class="gradient">movimiento realista</span>
            </h1>

            <p>
              Esta página combina estética cinematográfica, profundidad visual, brillo dinámico,
              partículas en vivo, microanimaciones y un backend funcional para recibir mensajes.
              Todo queda listo para que lo subas a GitHub sin montar una estructura complicada.
            </p>

            <div class="hero-actions">
              <a class="btn btn-primary" href="#contacto">Quiero esta web</a>
              <a class="btn btn-secondary" href="#efectos">Ver efectos</a>
            </div>

            <div class="hero-meta">
              <div class="pill">
                <strong>1 archivo</strong>
                frontend y backend juntos
              </div>
              <div class="pill">
                <strong>3D live</strong>
                profundidad, glow y partículas
              </div>
              <div class="pill">
                <strong>API real</strong>
                formulario conectado
              </div>
            </div>
          </div>

          <div class="hero-visual">
            <div class="orbital-stage">
              <div class="holo-card" id="tiltCard">
                <div class="holo-shell">
                  <div class="holo-screen">
                    <div class="screen-header">
                      <strong>Panel inmersivo</strong>
                      <div class="dots">
                        <span></span><span></span><span></span>
                      </div>
                    </div>

                    <div class="metric-bar">
                      <div>
                        <strong>Intensidad visual</strong>
                        <div class="stat-label">Capas con brillo, cristal, profundidad y sensación de video activo.</div>
                      </div>
                      <div class="metric-track">
                        <div class="metric-fill"></div>
                      </div>
                    </div>

                    <div class="mini-grid">
                      <div class="mini-panel">
                        <strong>Color Flow</strong>
                        <div class="sparkline"></div>
                      </div>
                      <div class="mini-panel">
                        <strong>Depth Motion</strong>
                        <div class="sparkline"></div>
                      </div>
                    </div>

                    <div class="wave-panel">
                      <strong>Escaneo dinámico</strong>
                      <div class="stat-label">Las ondas internas dan sensación de superficie viva y movimiento constante.</div>
                    </div>
                  </div>
                </div>
              </div>

              <div class="floating-tag tag-a">Animación realista</div>
              <div class="floating-tag tag-b">Responsive premium</div>
              <div class="floating-tag tag-c">Backend integrado</div>
            </div>
          </div>
        </div>
      </section>

      <section class="section wrap" id="efectos">
        <div class="section-intro">
          <h2>Una escena web con vida propia</h2>
          <p>
            No es una landing plana. Es una interfaz pensada para impactar desde móvil, con atmósfera,
            movimiento continuo, volumen visual y una estética moderna de alto nivel.
          </p>
        </div>

        <div class="feature-grid">
          <article class="card">
            <div class="chip">✦</div>
            <h3>Visual 3D potente</h3>
            <p>
              Se usa perspectiva, glassmorphism, rotación suave y sombras profundas para crear presencia real.
            </p>
          </article>

          <article class="card">
            <div class="chip">◉</div>
            <h3>Movimiento tipo video</h3>
            <p>
              El fondo y las capas animadas generan una sensación continua de escena viva sin depender de librerías.
            </p>
          </article>

          <article class="card">
            <div class="chip">▣</div>
            <h3>Lista para personalizar</h3>
            <p>
              Puedes cambiar textos, colores, secciones o convertirla en web de marca, negocio, portfolio o startup.
            </p>
          </article>
        </div>
      </section>

      <section class="section wrap">
        <div class="stats-grid">
          <article class="stat-card">
            <div class="stat-value" data-counter="100">0</div>
            <div class="stat-label">Diseño adaptable a móvil</div>
          </article>
          <article class="stat-card">
            <div class="stat-value" data-counter="60">0</div>
            <div class="stat-label">Sensación de fluidez visual</div>
          </article>
          <article class="stat-card">
            <div class="stat-value" id="submissionCount">0</div>
            <div class="stat-label">Mensajes recibidos por la API</div>
          </article>
          <article class="stat-card">
            <div class="stat-value" data-counter="1">0</div>
            <div class="stat-label">Archivo principal para GitHub</div>
          </article>
        </div>
      </section>

      <section class="section wrap" id="fullstack">
        <div class="showcase-grid">
          <div class="showcase-panel" aria-hidden="true">
            <div class="cube-wrap">
              <div class="cube-ring"></div>
              <div class="cube-face front"><span>Live</span></div>
              <div class="cube-face back"><span>Glow</span></div>
              <div class="cube-face right"><span>Depth</span></div>
              <div class="cube-face left"><span>Motion</span></div>
              <div class="cube-face top"><span>3D</span></div>
              <div class="cube-face bottom"><span>Flux</span></div>
            </div>
          </div>

          <div class="showcase-copy">
            <div class="section-intro" style="margin-bottom:0">
              <h2>Bonita por fuera, funcional por dentro</h2>
              <p>
                La página incluye backend real con Node.js nativo. El formulario envía datos a la API
                y los guarda en un archivo local, así que no es solo diseño: responde de verdad.
              </p>
            </div>

            <div class="check">
              <div class="check-mark">✓</div>
              <div>
                <strong>Servidor incluido</strong>
                No necesitas Express ni frameworks para arrancarla.
              </div>
            </div>

            <div class="check">
              <div class="check-mark">✓</div>
              <div>
                <strong>Formulario conectado</strong>
                Usa la ruta <code>/api/contact</code> para guardar mensajes.
              </div>
            </div>

            <div class="check">
              <div class="check-mark">✓</div>
              <div>
                <strong>Escalable</strong>
                Luego puedes meter login, base de datos, panel admin o pagos.
              </div>
            </div>
          </div>
        </div>
      </section>

      <section class="section wrap" id="contacto">
        <div class="contact-grid">
          <div class="contact-box contact-copy">
            <h3>Capta clientes o leads reales</h3>
            <p>
              Esta base ya sirve para una web de marca personal, agencia, estudio creativo,
              artista, influencer o servicio premium.
            </p>
            <ul>
              <li>Fondo cinemático renderizado con canvas en tiempo real.</li>
              <li>Componentes glassmorphism y profundidad visual.</li>
              <li>Formulario enlazado al backend con guardado local.</li>
              <li>Diseño pensado para verse fuerte desde móvil.</li>
            </ul>
          </div>

          <div class="contact-box">
            <form id="contactForm">
              <div class="field-row">
                <label>
                  Nombre
                  <input type="text" name="name" placeholder="Tu nombre o tu marca" required />
                </label>
                <label>
                  Email
                  <input type="email" name="email" placeholder="tu@email.com" required />
                </label>
              </div>

              <div class="field-row">
                <label>
                  Tipo de proyecto
                  <select name="project">
                    <option>Marca personal</option>
                    <option>Tienda online</option>
                    <option>Landing de servicios</option>
                    <option>Portafolio premium</option>
                    <option>Startup digital</option>
                  </select>
                </label>
                <label>
                  Presupuesto ideal
                  <select name="budget">
                    <option>En construcción</option>
                    <option>500 - 1.000</option>
                    <option>1.000 - 3.000</option>
                    <option>3.000+</option>
                  </select>
                </label>
              </div>

              <label>
                Mensaje
                <textarea name="message" placeholder="Describe la web que quieres, colores, estilo y qué debe transmitir..." required></textarea>
              </label>

              <div class="submit-row">
                <button class="btn btn-primary" type="submit">Enviar mensaje</button>
                <div class="status" id="formStatus">La API está esperando tu primer mensaje.</div>
              </div>
            </form>
          </div>
        </div>
      </section>
    </main>

    <footer class="footer wrap">
      <div class="footer-grid">
        <div class="footer-box">
          <h3>Subida simple a GitHub</h3>
          <p>
            Guarda este archivo como <code>server.js</code>, súbelo a tu repositorio y arráncalo con Node.
            Si quieres, después te hago una versión 2 con tu nombre, tus redes, tus textos, WhatsApp, tienda o panel admin.
          </p>
        </div>
        <div class="footer-box">
          <h3>Rutas incluidas</h3>
          <div class="footer-links">
            <a href="/"><code>/</code> interfaz principal</a>
            <a href="/api/stats"><code>/api/stats</code> contador de mensajes</a>
            <a href="#contacto"><code>/api/contact</code> formulario POST</a>
          </div>
        </div>
      </div>
    </footer>
  </div>

  <div class="toast" id="toast"></div>

  <script>
    const canvas = document.getElementById('bgCanvas');
    const ctx = canvas.getContext('2d');
    const tiltCard = document.getElementById('tiltCard');
    const counters = document.querySelectorAll('[data-counter]');
    const submissionCount = document.getElementById('submissionCount');
    const contactForm = document.getElementById('contactForm');
    const formStatus = document.getElementById('formStatus');
    const toast = document.getElementById('toast');

    let width = 0;
    let height = 0;
    let particles = [];

    function resizeCanvas() {
      const ratio = Math.min(window.devicePixelRatio || 1, 2);
      width = window.innerWidth;
      height = window.innerHeight;
      canvas.width = width * ratio;
      canvas.height = height * ratio;
      canvas.style.width = width + 'px';
      canvas.style.height = height + 'px';
      ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
      buildParticles();
    }

    function buildParticles() {
      const total = Math.max(34, Math.floor(width / 34));
      particles = Array.from({ length: total }, (_, index) => ({
        x: Math.random() * width,
        y: Math.random() * height,
        size: Math.random() * 2.5 + 0.8,
        speedX: (Math.random() - 0.5) * 0.6,
        speedY: (Math.random() - 0.5) * 0.6,
        hue: 180 + Math.sin(index) * 90,
        drift: Math.random() * Math.PI * 2
      }));
    }

    function drawScene(time) {
      ctx.clearRect(0, 0, width, height);

      const bg = ctx.createLinearGradient(0, 0, width, height);
      bg.addColorStop(0, 'rgba(78,242,255,.05)');
      bg.addColorStop(.5, 'rgba(140,91,255,.05)');
      bg.addColorStop(1, 'rgba(255,79,216,.04)');
      ctx.fillStyle = bg;
      ctx.fillRect(0, 0, width, height);

      particles.forEach((p, i) => {
        p.x += p.speedX + Math.sin(time * 0.0005 + p.drift) * 0.12;
        p.y += p.speedY + Math.cos(time * 0.0006 + p.drift) * 0.12;

        if (p.x < -40) p.x = width + 40;
        if (p.x > width + 40) p.x = -40;
        if (p.y < -40) p.y = height + 40;
        if (p.y > height + 40) p.y = -40;

        const alpha = 0.36 + Math.sin(time * 0.002 + i) * 0.2;
        ctx.beginPath();
        ctx.fillStyle = 'hsla(' + p.hue + ', 95%, 72%, ' + alpha + ')';
        ctx.shadowBlur = 20;
        ctx.shadowColor = 'hsla(' + p.hue + ', 95%, 70%, .28)';
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fill();

        for (let j = i + 1; j < particles.length; j++) {
          const o = particles[j];
          const dx = p.x - o.x;
          const dy = p.y - o.y;
          const dist = Math.sqrt(dx * dx + dy * dy);
          if (dist < 120) {
            ctx.beginPath();
            ctx.strokeStyle = 'rgba(140,190,255,' + (0.12 - dist / 1200) + ')';
            ctx.lineWidth = 1;
            ctx.moveTo(p.x, p.y);
            ctx.lineTo(o.x, o.y);
            ctx.stroke();
          }
        }
      });

      ctx.shadowBlur = 0;
      requestAnimationFrame(drawScene);
    }

    function animateCounters() {
      counters.forEach(counter => {
        const target = Number(counter.dataset.counter || '0');
        const start = performance.now();
        const duration = 1600;

        function tick(now) {
          const progress = Math.min((now - start) / duration, 1);
          const eased = 1 - Math.pow(1 - progress, 3);
          counter.textContent = String(Math.round(target * eased));
          if (progress < 1) requestAnimationFrame(tick);
        }

        requestAnimationFrame(tick);
      });
    }

    function showToast(message) {
      toast.textContent = message;
      toast.classList.add('show');
      clearTimeout(showToast.timer);
      showToast.timer = setTimeout(() => {
        toast.classList.remove('show');
      }, 2800);
    }

    async function loadStats() {
      try {
        const res = await fetch('/api/stats');
        const data = await res.json();
        submissionCount.textContent = String(data.submissions || 0);
      } catch {
        submissionCount.textContent = '0';
      }
    }

    contactForm.addEventListener('submit', async function (e) {
      e.preventDefault();

      const fd = new FormData(contactForm);
      const payload = {
        name: fd.get('name'),
        email: fd.get('email'),
        project: fd.get('project'),
        budget: fd.get('budget'),
        message: fd.get('message')
      };

      formStatus.textContent = 'Enviando al backend...';

      try {
        const res = await fetch('/api/contact', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        const result = await res.json();

        if (!res.ok) {
          throw new Error(result.error || 'No se pudo enviar el mensaje');
        }

        contactForm.reset();
        formStatus.textContent = 'Mensaje guardado correctamente.';
        submissionCount.textContent = String(result.total || 0);
        showToast('Tu mensaje quedó guardado en el backend.');
      } catch (err) {
        formStatus.textContent = err.message;
        showToast(err.message);
      }
    });

    document.addEventListener('pointermove', function (e) {
      if (!tiltCard) return;
      const rect = tiltCard.getBoundingClientRect();
      const centerX = rect.left + rect.width / 2;
      const centerY = rect.top + rect.height / 2;
      const rotateY = ((e.clientX - centerX) / rect.width) * 18;
      const rotateX = ((centerY - e.clientY) / rect.height) * 14;
      tiltCard.style.transform = 'rotateX(' + rotateX + 'deg) rotateY(' + rotateY + 'deg) translateY(-4px)';
    });

    document.addEventListener('pointerleave', function () {
      if (tiltCard) tiltCard.style.transform = '';
    });

    window.addEventListener('resize', resizeCanvas);

    resizeCanvas();
    animateCounters();
    loadStats();
    requestAnimationFrame(drawScene);
  </script>
</body>
</html>`;
}

const server = http.createServer(async (req, res) => {
  if (req.method === 'OPTIONS') {
    res.writeHead(204, {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET,POST,OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type'
    });
    return res.end();
  }

  if (req.url === '/' && req.method === 'GET') {
    return sendHtml(res, createHtml());
  }

  if (req.url === '/api/stats' && req.method === 'GET') {
    const submissions = readSubmissions();
    return sendJson(res, 200, { submissions: submissions.length });
  }

  if (req.url === '/api/contact' && req.method === 'POST') {
    try {
      const rawBody = await collectBody(req);
      const payload = JSON.parse(rawBody || '{}');

      const name = String(payload.name || '').trim();
      const email = String(payload.email || '').trim();
      const project = String(payload.project || '').trim();
      const budget = String(payload.budget || '').trim();
      const message = String(payload.message || '').trim();

      if (!name || !email || !message) {
        return sendJson(res, 400, {
          error: 'Nombre, email y mensaje son obligatorios.'
        });
      }

      const submissions = readSubmissions();
      submissions.unshift({
        id: Date.now(),
        name,
        email,
        project,
        budget,
        message,
        createdAt: new Date().toISOString()
      });

      writeSubmissions(submissions);

      return sendJson(res, 201, {
        ok: true,
        message: 'Mensaje guardado correctamente.',
        total: submissions.length
      });
    } catch {
      return sendJson(res, 500, {
        error: 'No se pudo procesar la solicitud.'
      });
    }
  }

  return sendJson(res, 404, { error: 'Ruta no encontrada.' });
});

server.listen(PORT, () => {
  console.log('Servidor listo en http://localhost:' + PORT);
});q
