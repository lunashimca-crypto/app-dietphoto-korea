# app-dietphoto-korea
This is a link for app that can add your daily meal photos and brief feelings. In the end of the journey (1 month), you can check your own dinosaur based on your responses. Also, it will display photos you uploaded.
import { useState, useMemo, useEffect } from "react";

// ===== 이미지 압축 =====
function compressImage(dataUrl, maxWidth=1000, quality=0.8){
  return new Promise(resolve=>{
    const img=new Image();
    img.onload=()=>{
      const canvas=document.createElement("canvas");
      const r=Math.min(maxWidth/img.width,1);
      canvas.width=img.width*r; canvas.height=img.height*r;
      canvas.getContext("2d").drawImage(img,0,0,canvas.width,canvas.height);
      resolve(canvas.toDataURL("image/jpeg",quality));
    };
    img.onerror=()=>resolve(dataUrl);
    img.src=dataUrl;
  });
}

// ===== 스토리지 (닉네임별 분리) =====
function nkey(n){ return encodeURIComponent((n||"").trim().toLowerCase()); }
async function saveDay(name,day,d){ try{ await window.storage.set(`u:${nkey(name)}:day:${day}`,JSON.stringify(d)); }catch(e){} }
async function loadDay(name,day){ try{ const r=await window.storage.get(`u:${nkey(name)}:day:${day}`); if(r?.value)return JSON.parse(r.value); }catch(e){} return null; }
async function loadRoster(){ try{ const r=await window.storage.get("roster"); if(r?.value)return JSON.parse(r.value); }catch(e){} return []; }
async function saveRoster(l){ try{ await window.storage.set("roster",JSON.stringify(l)); }catch(e){} }
async function addToRoster(name){ const l=await loadRoster(); if(!l.some(n=>nkey(n)===nkey(name))){ l.push(name.trim()); await saveRoster(l); } return l; }

// ===== 상수 =====
const MEAL_SLOTS=[
  {key:"breakfast",label:"Breakfast",ko:"아침",emoji:"🌅"},
  {key:"lunch",label:"Lunch",ko:"점심",emoji:"☀️"},
  {key:"dinner",label:"Dinner",ko:"저녁",emoji:"🌙"},
  {key:"snack",label:"Snack",ko:"간식",emoji:"🍩"},
];
const CHIPS={
  category:["Street food","Restaurant","Cafe","Convenience store","Home-cooked","Skipped"],
  physical:["Satisfied","Full","Light","Heavy","Energized","Nauseous","Still hungry"],
  emotional:["Happy","Comforted","Adventurous","Nostalgic","Proud","Meh","Regret"],
};
const POSITIVE=["Happy","Comforted","Adventurous","Nostalgic","Proud"];
const NEGATIVE=["Meh","Regret"];
const PROGRAM_START=new Date(2025,5,16);

function dayDate(i){ const d=new Date(PROGRAM_START); d.setDate(d.getDate()+i); return d; }
function dayLabel(i){ return dayDate(i).toLocaleDateString("en-US",{month:"numeric",day:"numeric"}); }
function dayWeekday(i){ return dayDate(i).toLocaleDateString("en-US",{weekday:"short"}); }
function programDayIndex(){ const now=new Date(),s=new Date(PROGRAM_START); s.setHours(0,0,0,0); const t=new Date(now); t.setHours(0,0,0,0); return Math.floor((t-s)/(864e5)); }
function isDayUnlocked(i){ const ti=programDayIndex(); if(i<ti)return true; if(i>ti)return false; return new Date().getHours()>=5; }
function dinoRevealed(){ return new Date()>=new Date(2025,6,5,5,0,0); }

// ===== 색상 (은하수 다크) =====
const C={
  bg:"#13111F",bgAlt:"#1E1B2E",mint:"#7FB7C4",mintL:"#2A3340",mintD:"#6FA3D9",
  blush:"#C77DBB",blushL:"#2E2440",blushD:"#E0A4D8",gold:"#E8C879",goldL:"#3A3320",
  ink:"#EDE8F5",inkL:"#A89FC0",border:"#332E47",white:"#211D33",
  stampHappy:"#E06A8C",stampNeutral:"#B09BD0",stampLow:"#6E8FC4",
  night:"#0C0A16",nightL:"#1A1726",
};

// ===== 데이터 모델 =====
const emptyMeal=()=>({category:null,physical:null,emotional:null,location:"",photo:null,loggedAt:null});
const emptyDay=()=>Object.fromEntries(MEAL_SLOTS.map(s=>[s.key,emptyMeal()]));
const mealComplete=m=>m.category==="Skipped"||(!!m.category&&!!m.physical&&!!m.emotional);
const mealLogged=m=>!!m.category;
const dayDone=d=>MEAL_SLOTS.some(s=>mealComplete(d[s.key]));

// ===== 성향 =====
const PERSONAS={
  steady:{key:"steady",name:"꾸준형",en:"The Steady One",dino:"white",creature:"새벽 별빛 도마뱀",
    letter:(top)=>`안녕, 나는 이번 달 네 식사 기억을 먹고 자란 새벽 별빛 도마뱀이야.\n\n넌 하루도 거르지 않고 꾸준히 기록해줬어. 바쁜 날에도, 피곤한 날에도 너의 식탁을 빠짐없이 남겨준 그 성실함이 나를 단단하게 만들었어. 특히 ${top}을(를) 자주 찾았더라.\n\n작은 기록이 모여 내가 태어났어. 너의 꾸준함을 닮은 내가, 앞으로도 너를 응원할게. 🤍`},
  explorer:{key:"explorer",name:"모험형",en:"The Explorer",dino:"gray",creature:"떠돌이 미식 드래곤",
    letter:(top)=>`안녕, 나는 이번 달 네 식사 기억을 먹고 자란 떠돌이 미식 드래곤이야.\n\n넌 늘 새로운 맛을 향해 발걸음을 옮겼어. 길거리 음식부터 낯선 메뉴까지, 용감하게 도전하는 너를 보며 나도 호기심 많은 아이로 자랐어. 그중에서도 ${top}이(가) 네 모험의 단골이었지.\n\n네 용기 덕분에 내가 태어났어. 다음 여정에서도 두근거리는 첫 입을 함께하자. 🧭`},
  nocturnal:{key:"nocturnal",name:"야행성",en:"The Night Owl",dino:"black",creature:"달빛 수달",
    letter:(top)=>`안녕, 나는 이번 달 네 식사 기억을 먹고 자란 달빛 수달이야.\n\n넌 하루를 마무리하는 저녁과 늦은 밤의 한 끼를 소중히 여겼어. 고요한 밤, 따뜻한 음식 앞에서 너는 가장 너다웠지. 특히 ${top}이(가) 네 밤을 채워줬더라.\n\n네 밤의 기록들이 모여 내가 태어났어. 어두운 밤에도 너의 곁을 비출게. 🌙`},
  balanced:{key:"balanced",name:"균형형",en:"The Balanced",dino:"yellow",creature:"숲지기 새싹 공룡",
    letter:(top)=>`안녕, 나는 이번 달 네 식사 기억을 먹고 자란 숲지기 새싹 공룡이야.\n\n넌 아침부터 간식까지 균형 있게 너를 돌봤어. 무리하지 않고, 너의 몸과 마음에 귀 기울인 그 다정함이 나를 건강하게 키웠어. ${top}처럼 든든한 한 끼들이 특히 고마웠어.\n\n네 균형 잡힌 하루들이 모여 내가 태어났어. 앞으로도 너를 다정하게 챙길게. ⚖️`},
};

// ===== 분석 함수 =====
function analyzePersona(data){
  let loggedDays=0,totalMeals=0,categories=new Set(),adventurous=0,streetFood=0;
  let dinnerSnack=0,allFourDays=0,lateLogs=0,posCount=0,emoTotal=0;
  for(let i=0;i<21;i++){
    const d=data[i]; if(dayDone(d))loggedDays++;
    let dm=0;
    for(const s of MEAL_SLOTS){
      const m=d[s.key]; if(!mealLogged(m))continue;
      totalMeals++;dm++;
      if(m.category)categories.add(m.category);
      if(m.category==="Street food")streetFood++;
      if(m.emotional==="Adventurous")adventurous++;
      if(m.emotional){emoTotal++;if(POSITIVE.includes(m.emotional))posCount++;}
      if(s.key==="dinner"||s.key==="snack")dinnerSnack++;
      if(m.loggedAt&&(new Date(m.loggedAt).getHours()>=20))lateLogs++;
    }
    if(dm===4)allFourDays++;
  }
  const scores={
    steady:loggedDays/21*2+allFourDays/21,
    explorer:categories.size/6+adventurous/Math.max(totalMeals,1)*3+streetFood/Math.max(totalMeals,1)*2,
    nocturnal:dinnerSnack/Math.max(totalMeals,1)*2+lateLogs/Math.max(totalMeals,1)*3,
    balanced:allFourDays/21*2+posCount/Math.max(emoTotal,1)*1.5,
  };
  let best="balanced",bs=-1;
  for(const k in scores)if(scores[k]>bs){bs=scores[k];best=k;}
  return PERSONAS[best];
}

function dayMood(d){
  let pos=0,neg=0;
  for(const s of MEAL_SLOTS){const e=d[s.key].emotional;if(POSITIVE.includes(e))pos++;else if(NEGATIVE.includes(e))neg++;}
  const t=pos+neg; if(!t)return{mood:"neutral",color:C.stampNeutral};
  const b=(pos-neg)/t;
  if(b>0.3)return{mood:"happy",color:C.stampHappy};
  if(b<-0.3)return{mood:"low",color:C.stampLow};
  return{mood:"neutral",color:C.stampNeutral};
}

function growthStage(data,adminMode){
  if(adminMode)return{stage:"dino",pct:1,logged:18};
  const logged=Array.from({length:21},(_,i)=>dayDone(data[i])).filter(Boolean).length;
  const pct=logged/21;
  if(dinoRevealed()&&logged>=15)return{stage:"dino",pct,logged};
  if(pct>=0.7)return{stage:"chick",pct,logged};
  return{stage:"egg",pct,logged};
}

function foodRatios(data){
  const cc={};let total=0;
  for(let i=0;i<21;i++)for(const s of MEAL_SLOTS){
    const m=data[i][s.key];
    if(m.category&&m.category!=="Skipped"){cc[m.category]=(cc[m.category]||0)+1;total++;}
  }
  if(!total)return[];
  return Object.entries(cc).sort((a,b)=>b[1]-a[1]).map(([k,v])=>({name:k,pct:Math.round(v/total*100),count:v}));
}

function collectPhotos(data){
  const out=[];
  for(let i=0;i<21;i++)for(const s of MEAL_SLOTS){
    const m=data[i][s.key];
    if(m.photo)out.push({photo:m.photo,day:i,slot:s,location:m.location});
  }
  return out;
}

function makeSampleData(persona){
  const d=Object.fromEntries(Array.from({length:21},(_,i)=>[i,emptyDay()]));
  const cats=["Street food","Restaurant","Cafe","Convenience store","Home-cooked"];
  const pos=["Happy","Comforted","Adventurous","Nostalgic","Proud"];
  const phy=["Satisfied","Full","Light","Energized"];
  const locs=["홍대 포차","광장시장","성수 카페","GS25","이태원 식당","명동 길거리","연남동 브런치"];
  const rnd=a=>a[Math.floor(Math.random()*a.length)];
  for(let i=0;i<21;i++){
    const slots=MEAL_SLOTS.filter(s=>{
      if(persona==="nocturnal")return(s.key==="dinner"||s.key==="snack")?Math.random()<0.95:Math.random()<0.4;
      if(persona==="steady")return Math.random()<0.92;
      if(persona==="balanced")return true;
      return Math.random()<0.7;
    });
    for(const s of slots){
      const m=d[i][s.key];
      if(persona==="explorer"){m.category=rnd(["Street food","Street food","Restaurant","Cafe"]);m.emotional=rnd(["Adventurous","Adventurous","Happy","Proud"]);}
      else if(persona==="nocturnal"){m.category=rnd(["Restaurant","Convenience store","Street food"]);m.emotional=rnd(["Comforted","Happy","Nostalgic"]);}
      else if(persona==="balanced"){m.category=rnd(cats);m.emotional=rnd(pos);}
      else{m.category=rnd(["Home-cooked","Restaurant","Cafe","Convenience store"]);m.emotional=rnd(pos);}
      m.physical=rnd(phy);
      if(Math.random()<0.3)m.location=rnd(locs);
      const hr=persona==="nocturnal"?(20+Math.floor(Math.random()*5)):(8+Math.floor(Math.random()*12));
      m.loggedAt=new Date(2025,5,16+i,hr%24).getTime();
    }
  }
  return d;
}

// ===== SVG 캐릭터 =====
function EggSVG({pct,size=130}){
  const cl=Math.min(pct/0.7,1);
  return(<svg viewBox="0 0 130 130" width={size} height={size}>
    <ellipse cx="65" cy="120" rx="34" ry="6" fill="#00000010"/>
    <defs><radialGradient id="eg" cx="42%" cy="35%"><stop offset="0%" stopColor="#FFFBF7"/><stop offset="70%" stopColor="#F3E2DB"/><stop offset="100%" stopColor="#E2C9BF"/></radialGradient></defs>
    <path d="M65 18 C92 18 105 58 105 82 C105 104 88 116 65 116 C42 116 25 104 25 82 C25 58 38 18 65 18 Z" fill="url(#eg)" stroke="#D8BEB2" strokeWidth="1.5"/>
    <ellipse cx="50" cy="55" rx="6" ry="8" fill="#E8CFC4" opacity=".5"/>
    <ellipse cx="80" cy="70" rx="5" ry="7" fill="#E8CFC4" opacity=".5"/>
    {cl>0.15&&<path d="M65 40 L60 52 L68 60 L62 70" fill="none" stroke="#B89384" strokeWidth="1.5" strokeLinecap="round"/>}
    {cl>0.4&&<path d="M68 60 L78 64 L74 74 L82 80" fill="none" stroke="#B89384" strokeWidth="1.5" strokeLinecap="round"/>}
    {cl>0.6&&<path d="M62 70 L52 76 L58 86" fill="none" stroke="#B89384" strokeWidth="1.5" strokeLinecap="round"/>}
    {cl>0.85&&<path d="M45 60 L40 70 M88 55 L94 64" fill="none" stroke="#B89384" strokeWidth="1.3" strokeLinecap="round"/>}
    <path d="M95 35 l2 5 l5 2 l-5 2 l-2 5 l-2 -5 l-5 -2 l5 -2 z" fill={C.gold} opacity=".7"/>
  </svg>);
}

function ChickSVG({size=130}){
  return(<svg viewBox="0 0 130 130" width={size} height={size}>
    <ellipse cx="65" cy="122" rx="32" ry="5" fill="#00000012"/>
    <defs>
      <radialGradient id="cb" cx="45%" cy="40%"><stop offset="0%" stopColor="#4A5160"/><stop offset="60%" stopColor="#2E3440"/><stop offset="100%" stopColor="#1C2029"/></radialGradient>
      <radialGradient id="ce" cx="40%" cy="35%"><stop offset="0%" stopColor="#BFE3FF"/><stop offset="60%" stopColor="#4A7BC8"/><stop offset="100%" stopColor="#1E3A6E"/></radialGradient>
    </defs>
    <path d="M65 28 C40 28 30 52 32 78 C34 102 48 116 65 116 C82 116 96 102 98 78 C100 52 90 28 65 28 Z" fill="url(#cb)"/>
    <path d="M50 30 l-4 -10 l8 6 z M65 24 l0 -12 l5 10 z M80 30 l5 -9 l-3 9 z" fill="#3A4150"/>
    <path d="M92 70 C104 66 108 80 100 88 C96 84 94 78 92 70 Z" fill="#3A4150" opacity=".9"/>
    <circle cx="55" cy="62" r="9" fill="url(#ce)"/>
    <circle cx="76" cy="62" r="9" fill="url(#ce)"/>
    <circle cx="51" cy="59" r="3" fill="#fff" opacity=".9"/>
    <circle cx="73" cy="59" r="3" fill="#fff" opacity=".9"/>
    <path d="M61 72 L69 72 L65 78 Z" fill="#C8A24A"/>
    <path d="M95 35 l1.5 4 l4 1.5 l-4 1.5 l-1.5 4 l-1.5 -4 l-4 -1.5 l4 -1.5 z" fill="#9FC5E8" opacity=".8"/>
  </svg>);
}

function DinoSVG({persona,size=240}){
  const skin={white:{base:"#EDF1F5",belly:"#FBFCFD",shade:"#CDD6DF",horn:"#B8C2CC"},gray:{base:"#B5BAC1",belly:"#D6DADF",shade:"#8E949C",horn:"#7A8088"},black:{base:"#3C4350",belly:"#535B6A",shade:"#262B34",horn:"#1F232B"},yellow:{base:"#E6CE86",belly:"#F2E4B8",shade:"#C9AC5E",horn:"#B0913F"}}[PERSONAS[persona].dino];
  const hanbok={white:"#3A5BA0",gray:"#D8D2C0",black:"#4A5AA8",yellow:"#E0C060"}[PERSONAS[persona].dino];
  return(<svg viewBox="0 0 240 240" width={size} height={size}>
    <ellipse cx="120" cy="222" rx="62" ry="9" fill="#00000010"/>
    <defs><radialGradient id={`db${persona}`} cx="42%" cy="35%"><stop offset="0%" stopColor={skin.base}/><stop offset="100%" stopColor={skin.shade}/></radialGradient><radialGradient id={`de${persona}`} cx="40%" cy="35%"><stop offset="0%" stopColor="#BFE3FF"/><stop offset="60%" stopColor="#3E6BB8"/><stop offset="100%" stopColor="#1E3A6E"/></radialGradient></defs>
    <path d="M168 170 C200 165 215 140 210 120 C206 134 192 150 170 156 Z" fill={`url(#db${persona})`}/>
    <path d="M205 122 l8 -4 M210 132 l9 -2 M208 145 l8 1" stroke={skin.shade} strokeWidth="2.5" strokeLinecap="round"/>
    <path d="M95 175 l-3 28 l16 0 l-2 -26 Z" fill={`url(#db${persona})`}/>
    <path d="M128 176 l1 28 l16 0 l-3 -27 Z" fill={`url(#db${persona})`}/>
    <path d="M92 203 l-4 5 l8 0 M100 203 l0 6 M108 203 l3 5" stroke={skin.horn} strokeWidth="2" strokeLinecap="round"/>
    <path d="M129 204 l-3 5 l8 0 M137 204 l0 6 M145 204 l3 5" stroke={skin.horn} strokeWidth="2" strokeLinecap="round"/>
    <path d="M70 150 C60 110 80 80 115 80 C150 80 168 112 160 152 C155 175 130 185 112 184 C92 183 76 175 70 150 Z" fill={`url(#db${persona})`}/>
    <ellipse cx="112" cy="150" rx="28" ry="30" fill={skin.belly} opacity=".7"/>
    <path d="M80 130 C82 160 95 180 112 182 C130 182 145 162 148 132 C140 150 95 150 80 130 Z" fill={hanbok} opacity=".92"/>
    <path d="M112 122 L112 182" stroke="#fff" strokeWidth="2" opacity=".5"/>
    <path d="M88 134 L112 124 L136 134" fill="none" stroke="#fff" strokeWidth="2.5"/>
    <rect x="92" y="148" width="40" height="6" rx="3" fill={C.blushD} opacity=".8"/>
    <circle cx="112" cy="158" r="3" fill={C.mintD}/>
    <path d="M112 161 l0 12" stroke={C.blushD} strokeWidth="1.5"/>
    <path d="M115 95 C112 75 116 58 130 52 C148 45 165 55 166 74 C167 90 152 100 138 100 C128 100 120 100 115 95 Z" fill={`url(#db${persona})`}/>
    <path d="M138 50 l-4 -16 l8 12 Z" fill={skin.horn}/>
    <path d="M152 52 l3 -15 l4 14 Z" fill={skin.horn}/>
    <circle cx="142" cy="72" r="9" fill={`url(#de${persona})`}/>
    <circle cx="139" cy="69" r="3" fill="#fff" opacity=".9"/>
    <ellipse cx="160" cy="82" rx="10" ry="7" fill={`url(#db${persona})`}/>
    {persona==="steady"&&<g transform="translate(64,120)"><rect x="-6" y="14" width="34" height="6" rx="2" fill="#7A4A28"/><ellipse cx="6" cy="12" rx="9" ry="5" fill="#C9A84C"/><ellipse cx="6" cy="10" rx="8" ry="4" fill="#fff"/></g>}
    {persona==="explorer"&&<g transform="translate(58,118)"><rect x="0" y="6" width="14" height="20" rx="3" fill="#fff" opacity=".85" stroke={C.border}/><rect x="2" y="12" width="10" height="12" fill={C.mint} opacity=".5"/><path d="M-8 8 c-4 0 -6 4 -3 7 c2 2 8 1 9 -3 z" fill="#D9A24A"/></g>}
    {persona==="nocturnal"&&<g transform="translate(56,108)" opacity=".9"><path d="M10 6 a8 8 0 1 0 6 13 a9 9 0 0 1 -6 -13 z" fill="#E8D49A"/><path d="M-2 20 l1 3 l3 1 l-3 1 l-1 3 l-1 -3 l-3 -1 l3 -1 z" fill="#9FC5E8"/></g>}
    {persona==="balanced"&&<g transform="translate(60,122)"><path d="M0 8 a12 7 0 0 0 24 0 z" fill="#7A4A28"/><ellipse cx="12" cy="8" rx="12" ry="4" fill="#8FBF6F"/><circle cx="7" cy="7" r="2" fill="#E36B5A"/><circle cx="15" cy="6" r="2" fill="#E8A23A"/></g>}
    <path d="M60 70 l2 5 l5 2 l-5 2 l-2 5 l-2 -5 l-5 -2 l5 -2 z" fill={C.gold} opacity=".75"/>
  </svg>);
}

function MoodStamp({day,idx=0,size=58}){
  const m=dayMood(day); const done=dayDone(day);
  if(!done)return(<svg viewBox="0 0 58 58" width={size} height={size}><circle cx="29" cy="29" r="11" fill="none" stroke={C.border} strokeWidth="1.5"/></svg>);
  const rot=((idx*23)%15)-7;
  return(<svg viewBox="0 0 58 58" width={size} height={size}><g transform={`rotate(${rot},29,29)`} opacity="0.92">
    {m.mood==="happy"?<rect x="9" y="9" width="40" height="40" rx="8" fill="none" stroke={m.color} strokeWidth="2.5"/>:<circle cx="29" cy="29" r="20" fill="none" stroke={m.color} strokeWidth="2.5" strokeDasharray={m.mood==="low"?"4,3":"0"}/>}
    <text x="29" y="27" textAnchor="middle" fontSize="14" fill={m.color} fontFamily="Georgia,serif">食</text>
    <text x="29" y="43" textAnchor="middle" fontSize="11" fill={m.color} fontFamily="system-ui">{m.mood==="happy"?"☺":m.mood==="low"?"·":"–"}</text>
  </g></svg>);
}

// ===== CSS =====
const F="'Gaegu',system-ui,sans-serif";
const FT="'Jua','Gaegu',system-ui,sans-serif";
const css=`
@import url('https://fonts.googleapis.com/css2?family=Gaegu:wght@300;400;700&family=Jua&display=swap');
*{box-sizing:border-box;margin:0;padding:0;}
body{background:${C.bg};}
.app{max-width:390px;margin:0 auto;min-height:100vh;font-family:${F};display:flex;flex-direction:column;position:relative;
  background:radial-gradient(ellipse 60% 40% at 25% 20%,rgba(199,125,187,0.18),transparent 70%),radial-gradient(ellipse 50% 35% at 75% 60%,rgba(111,163,217,0.15),transparent 70%),${C.bg};}
.app::before{content:"";position:fixed;inset:0;max-width:390px;margin:0 auto;pointer-events:none;z-index:0;opacity:.5;
  background-image:radial-gradient(1px 1px at 12% 18%,rgba(255,255,255,.9),transparent),radial-gradient(1px 1px at 28% 42%,rgba(255,255,255,.7),transparent),radial-gradient(1.5px 1.5px at 45% 12%,rgba(255,255,255,.8),transparent),radial-gradient(1px 1px at 62% 35%,rgba(224,164,216,.8),transparent),radial-gradient(1px 1px at 78% 22%,rgba(255,255,255,.7),transparent),radial-gradient(1.5px 1.5px at 88% 55%,rgba(255,255,255,.6),transparent),radial-gradient(1px 1px at 18% 70%,rgba(255,255,255,.7),transparent),radial-gradient(1px 1px at 52% 78%,rgba(184,200,255,.7),transparent),radial-gradient(1px 1px at 35% 92%,rgba(255,255,255,.5),transparent);
  background-repeat:no-repeat;}
.app>*{position:relative;z-index:1;}
.hdr{background:linear-gradient(135deg,${C.nightL},${C.bgAlt});color:${C.ink};padding:.875rem 1.25rem .75rem;position:sticky;top:0;z-index:20;border-bottom:1px solid ${C.border};}
.hdr h1{font-size:18px;font-weight:700;font-family:${FT};}
.hdr p{font-size:11px;opacity:.8;margin-top:3px;font-family:${F};}
.back{background:none;border:none;color:${C.blushD};font-size:14px;cursor:pointer;padding:0 0 .4rem;font-family:${F};font-weight:700;}
.nav{display:flex;background:${C.night};border-top:1px solid ${C.border};}
.nt{flex:1;padding:.5rem;font-size:12px;color:${C.inkL};background:none;border:none;cursor:pointer;display:flex;flex-direction:column;align-items:center;gap:2px;font-family:${F};font-weight:700;}
.nt.on{color:${C.gold};}
.nt i{font-size:17px;}
/* INTRO */
.intro{flex:1;overflow-y:auto;padding:2rem 1.5rem;text-align:center;}
.intro-seal{width:100px;height:100px;margin:0 auto 1.25rem;display:block;}
.intro-title{font-size:27px;color:${C.ink};margin-bottom:.5rem;font-family:${FT};}
.intro-story{font-size:15px;color:${C.blushD};margin-bottom:1.5rem;font-family:${F};line-height:1.7;}
.story-box{background:${C.white};border:1px solid ${C.border};border-radius:16px;padding:1rem 1.1rem;margin-bottom:1.25rem;text-align:left;box-shadow:0 2px 16px rgba(0,0,0,.25);}
.story-box h3{font-size:14px;color:${C.gold};margin-bottom:.6rem;font-family:${FT};}
.growth-row{display:flex;align-items:center;gap:10px;margin-bottom:.6rem;}
.growth-emoji{font-size:22px;width:28px;text-align:center;flex-shrink:0;}
.growth-txt{font-size:14px;color:${C.ink};line-height:1.5;}
.growth-txt b{color:${C.blushD};font-weight:700;}
.persona-row{display:flex;gap:6px;flex-wrap:wrap;margin-top:.4rem;}
.persona-tag{font-size:12px;padding:4px 10px;border-radius:12px;background:${C.bgAlt};color:${C.inkL};font-family:${F};font-weight:700;border:1px solid ${C.border};}
.roster-list{display:flex;flex-direction:column;gap:6px;}
.roster-btn{display:flex;align-items:center;justify-content:space-between;width:100%;background:${C.bgAlt};border:1px solid ${C.border};border-radius:12px;padding:11px 14px;font-size:15px;color:${C.ink};cursor:pointer;font-family:${F};font-weight:700;}
.roster-btn:hover{border-color:${C.blush};background:${C.blushL};}
.intro-inp{width:100%;max-width:300px;border:1px solid ${C.border};border-radius:12px;padding:11px 14px;font-size:17px;text-align:center;background:${C.bgAlt};color:${C.ink};outline:none;font-family:${F};margin-bottom:1rem;}
.intro-inp::placeholder{color:${C.inkL};}
.intro-inp:focus{border-color:${C.blush};}
.intro-btn{width:100%;max-width:300px;background:linear-gradient(135deg,${C.blush},${C.mintD});color:#fff;border:none;border-radius:12px;padding:13px;font-size:17px;cursor:pointer;font-family:${FT};box-shadow:0 4px 16px rgba(199,125,187,.3);}
.intro-btn:disabled{opacity:.4;cursor:default;box-shadow:none;}
/* COMPANION */
.companion{background:linear-gradient(160deg,${C.bgAlt},${C.white});border:1px solid ${C.border};border-radius:18px;margin:.5rem .25rem 1rem;padding:1rem;text-align:center;box-shadow:0 2px 20px rgba(0,0,0,.3);cursor:pointer;}
.companion-svg{width:130px;height:130px;margin:0 auto;display:block;}
.companion-title{font-size:17px;color:${C.ink};margin-top:.4rem;font-family:${FT};}
.companion-sub{font-size:13px;color:${C.inkL};margin-top:3px;font-family:${F};line-height:1.5;}
.crack-pct{font-size:12px;color:${C.blushD};font-family:${F};font-weight:700;margin-top:5px;}
/* ROADMAP */
.rmap{flex:1;overflow-y:auto;padding:1rem .75rem 2rem;}
.progress-wrap{margin:.25rem .25rem 1.25rem;}
.progress-top{display:flex;justify-content:space-between;font-size:13px;color:${C.inkL};font-family:${F};font-weight:700;margin-bottom:5px;}
.progress-bar{height:9px;background:${C.bgAlt};border-radius:5px;overflow:hidden;border:1px solid ${C.border};}
.progress-fill{height:100%;background:linear-gradient(90deg,${C.blush},${C.mintD});border-radius:5px;transition:width .4s;}
.wk-hdr{display:flex;align-items:center;gap:8px;margin:1.25rem 0 .5rem;}
.wk-line{flex:1;height:1px;background:${C.gold};opacity:.35;}
.wk-txt{font-size:12px;letter-spacing:.1em;text-transform:uppercase;color:${C.gold};font-family:${FT};}
.nodes-row{display:grid;grid-template-columns:repeat(3,1fr);gap:8px;margin-bottom:14px;}
.node-wrap{display:flex;flex-direction:column;align-items:center;gap:4px;}
.node{width:74px;height:74px;border-radius:50%;cursor:pointer;border:2px solid ${C.border};background:${C.white};display:flex;flex-direction:column;align-items:center;justify-content:center;position:relative;transition:transform .15s;}
.node:hover{transform:scale(1.05);}
.node.cur{border-color:${C.gold};box-shadow:0 0 14px rgba(232,200,121,.5);}
.node.locked{background:${C.night};border-color:${C.border};cursor:default;opacity:.55;}
.node.locked:hover{transform:none;}
.node-num{font-size:12px;color:${C.inkL};font-family:${F};font-weight:700;}
.node.cur .node-num{color:${C.gold};}
.node-dot{width:9px;height:9px;border-radius:50%;background:${C.border};}
.node.cur .node-dot{background:${C.gold};}
.node-date{font-size:11px;color:${C.inkL};font-family:${F};font-weight:700;text-align:center;line-height:1.3;}
.node-wd{color:${C.blushD};}
.dep-badge{font-size:9px;background:${C.goldL};color:${C.gold};padding:1px 5px;border-radius:6px;font-family:${F};font-weight:700;margin-top:1px;}
/* DAY */
.day-scroll{flex:1;overflow-y:auto;padding:1rem .75rem 2rem;}
.meal-hint{font-size:13px;color:${C.inkL};font-family:${F};text-align:center;margin-bottom:.75rem;line-height:1.5;}
.meal-hint b{color:${C.blushD};font-weight:700;}
.meal-tabs-grid{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:1rem;}
.mtab{background:${C.white};border:1px solid ${C.border};border-radius:16px;padding:1.25rem .75rem;cursor:pointer;display:flex;flex-direction:column;align-items:center;gap:6px;transition:all .15s;position:relative;}
.mtab:hover{border-color:${C.blush};transform:translateY(-2px);}
.mtab.logged{background:${C.blushL};border-color:${C.blush};}
.mtab-emoji{font-size:30px;}
.mtab-label{font-size:16px;color:${C.ink};font-family:${FT};}
.mtab-ko{font-size:13px;color:${C.inkL};font-family:${F};}
.mtab-check{position:absolute;top:8px;right:10px;color:${C.blushD};font-size:15px;}
.mtab-partial{position:absolute;top:6px;right:10px;color:${C.gold};font-size:16px;font-weight:700;}
.meal-detail{background:${C.white};border:1px solid ${C.border};border-radius:16px;overflow:hidden;}
.md-hdr{background:${C.bgAlt};padding:.75rem 1rem;display:flex;align-items:center;justify-content:space-between;border-bottom:1px solid ${C.border};}
.md-title{font-size:17px;color:${C.ink};display:flex;align-items:center;gap:7px;font-family:${FT};}
.copy-btn{font-size:12px;color:${C.mintD};background:none;border:1px solid ${C.mintD};border-radius:8px;padding:3px 8px;cursor:pointer;font-family:${F};font-weight:700;}
.md-body{padding:.875rem 1rem;}
.cl{font-size:12px;color:${C.inkL};text-transform:uppercase;letter-spacing:.05em;margin-bottom:5px;font-family:${F};font-weight:700;}
.chips{display:flex;flex-wrap:wrap;gap:5px;margin-bottom:.75rem;}
.chip{font-size:14px;padding:5px 11px;border-radius:20px;cursor:pointer;border:1px solid ${C.border};background:${C.bgAlt};color:${C.ink};font-family:${F};font-weight:700;transition:all .12s;}
.chip.s-cat{background:${C.mintD};color:#fff;border-color:${C.mintD};}
.chip.s-phy{background:${C.blush};color:#fff;border-color:${C.blush};}
.chip.s-emo{background:${C.gold};color:#2A2410;border-color:${C.gold};}
.divider{height:1px;background:${C.border};margin:.625rem 0;}
.loc-inp{width:100%;border:1px solid ${C.border};border-radius:8px;padding:8px 10px;font-size:15px;color:${C.ink};background:${C.bgAlt};outline:none;font-family:${F};}
.loc-inp::placeholder{color:${C.inkL};}
.loc-inp:focus{border-color:${C.blush};}
.photo-area{border:1.5px dashed ${C.border};border-radius:12px;padding:1rem;text-align:center;cursor:pointer;color:${C.inkL};font-size:14px;background:${C.bgAlt};font-family:${F};font-weight:700;}
.photo-preview{width:100%;border-radius:12px;display:block;}
.rm-photo-btn{font-size:12px;color:${C.stampHappy};background:none;border:none;cursor:pointer;margin-top:5px;font-family:${F};font-weight:700;}
.meal-status{margin-top:.875rem;padding:9px 10px;border-radius:10px;font-size:13px;font-family:${F};font-weight:700;text-align:center;background:${C.bgAlt};color:${C.inkL};}
.meal-status.done{background:${C.blushL};color:${C.blushD};}
/* RECAP */
.rec-scroll{flex:1;overflow-y:auto;padding-bottom:2rem;}
.stats{display:grid;grid-template-columns:1fr 1fr;gap:8px;padding:.875rem;}
.sc{background:${C.white};border:1px solid ${C.border};border-radius:12px;padding:.75rem;}
.sc.w{grid-column:1/-1;}
.sv{font-size:16px;color:${C.blushD};font-family:${FT};}
.sk{font-size:12px;color:${C.inkL};margin-top:2px;font-family:${F};}
.rwk{margin:0 .75rem .5rem;border:1px solid ${C.border};border-radius:14px;overflow:hidden;background:${C.white};}
.wtog{width:100%;background:${C.bgAlt};border:none;padding:.75rem 1rem;font-size:15px;color:${C.ink};cursor:pointer;display:flex;justify-content:space-between;align-items:center;font-family:${FT};}
.rday{padding:.75rem 1rem;border-top:1px solid ${C.border};display:flex;gap:10px;align-items:flex-start;}
.rday-stamp{flex-shrink:0;width:46px;height:46px;}
.rday-body{flex:1;}
.rdt{font-size:13px;color:${C.mintD};margin-bottom:6px;font-family:${F};font-weight:700;}
.rmr{display:flex;align-items:flex-start;gap:7px;margin-bottom:5px;}
.rce{font-size:13px;flex-shrink:0;}
.rcc{display:flex;flex-wrap:wrap;gap:3px;align-items:center;}
.rc{font-size:12px;padding:2px 8px;border-radius:10px;font-family:${F};font-weight:700;}
.rc-c{background:${C.mintL};color:${C.mint};}
.rc-p{background:${C.blushL};color:${C.blushD};}
.rc-e{background:${C.goldL};color:${C.gold};}
.rc-l{background:${C.bgAlt};color:${C.inkL};}
.rth{width:34px;height:34px;border-radius:6px;object-fit:cover;flex-shrink:0;}
.emp{color:${C.inkL};font-style:italic;font-size:13px;font-family:${F};}
/* HATCH */
.hatch{flex:1;overflow-y:auto;padding:1.5rem 1.25rem 2rem;text-align:center;}
.hatch-label{font-size:13px;letter-spacing:.16em;text-transform:uppercase;color:${C.blushD};font-family:${FT};margin-bottom:.5rem;}
.hatch-dino{width:240px;height:240px;margin:0 auto;display:block;}
.hatch-name{font-size:26px;color:${C.ink};margin-top:.75rem;font-family:${FT};}
.hatch-en{font-size:14px;color:${C.inkL};font-family:${F};margin-top:2px;letter-spacing:.05em;}
.hatch-desc{font-size:15px;color:${C.ink};font-family:${F};margin-top:1rem;line-height:1.7;background:${C.white};border:1px solid ${C.border};border-radius:16px;padding:1rem;}
.hatch-stats{margin-top:1rem;text-align:left;background:${C.white};border:1px solid ${C.border};border-radius:16px;padding:1rem;}
.hsrow{display:flex;justify-content:space-between;font-size:14px;color:${C.ink};font-family:${F};font-weight:700;padding:6px 0;border-bottom:1px solid ${C.border};}
.hsrow:last-child{border-bottom:none;}
.cheer{margin-top:1.25rem;font-size:16px;color:${C.blushD};font-family:${F};line-height:1.7;}
/* 관리자 */
.admin-bar{background:${C.night};border:1px solid ${C.gold};border-radius:14px;padding:.75rem;margin-bottom:1rem;}
.admin-label{font-size:12px;color:${C.gold};font-family:${F};font-weight:700;margin-bottom:6px;}
.admin-tabs{display:grid;grid-template-columns:repeat(4,1fr);gap:5px;}
.admin-tab{font-size:12px;padding:7px 4px;border-radius:8px;border:1px solid ${C.border};background:${C.bgAlt};color:${C.inkL};cursor:pointer;font-family:${F};font-weight:700;}
.admin-tab.on{background:${C.gold};color:#2A2410;border-color:${C.gold};}
/* 섹션 카드 */
.section-card{background:${C.white};border:1px solid ${C.border};border-radius:16px;padding:1rem;margin-top:1rem;text-align:left;}
.section-title{font-size:15px;color:${C.ink};font-family:${FT};margin-bottom:.75rem;}
/* 식사 비율 */
.ratio-row{margin-bottom:.6rem;}
.ratio-head{display:flex;justify-content:space-between;font-size:14px;color:${C.ink};font-family:${F};font-weight:700;margin-bottom:3px;}
.ratio-pct{color:${C.blushD};}
.ratio-bar{height:10px;background:${C.bgAlt};border-radius:5px;overflow:hidden;}
.ratio-fill{height:100%;background:linear-gradient(90deg,${C.blush},${C.gold});border-radius:5px;}
.ratio-caption{font-size:13px;color:${C.inkL};font-family:${F};text-align:right;margin-top:4px;}
/* 편지 */
.letter-card{background:linear-gradient(160deg,${C.blushL},${C.white});border:1px solid ${C.blush};border-radius:16px;padding:1.1rem;margin-top:1rem;text-align:left;box-shadow:0 2px 16px rgba(199,125,187,.2);}
.letter-head{font-size:16px;color:${C.blushD};font-family:${FT};margin-bottom:.6rem;}
.letter-body{font-size:14.5px;color:${C.ink};font-family:${F};line-height:1.85;white-space:pre-line;}
/* 사진 갤러리 */
.photo-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:6px;}
.photo-thumb{position:relative;aspect-ratio:1;border-radius:10px;overflow:hidden;cursor:pointer;border:1px solid ${C.border};}
.photo-thumb img{width:100%;height:100%;object-fit:cover;display:block;}
.photo-day{position:absolute;bottom:3px;left:4px;font-size:10px;color:#fff;background:rgba(0,0,0,.55);padding:1px 5px;border-radius:5px;font-family:${F};font-weight:700;}
.photo-modal{position:fixed;inset:0;background:rgba(8,6,18,.94);z-index:100;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:1.5rem;cursor:pointer;}
.photo-modal img{max-width:100%;max-height:75vh;border-radius:14px;}
.photo-modal-cap{color:${C.ink};font-family:${F};font-weight:700;font-size:14px;margin-top:1rem;text-align:center;}
.photo-modal-close{color:${C.inkL};font-family:${F};font-size:12px;margin-top:.5rem;}
/* 자가평가 */
.eval-q{margin-bottom:.875rem;}
.eval-qtext{font-size:14px;color:${C.ink};font-family:${F};font-weight:700;margin-bottom:6px;}
.eval-opts{display:flex;flex-wrap:wrap;gap:5px;}
.eval-opt{font-size:13px;padding:6px 11px;border-radius:18px;border:1px solid ${C.border};background:${C.bgAlt};color:${C.ink};cursor:pointer;font-family:${F};font-weight:700;}
.eval-opt.on{background:${C.mintD};color:#fff;border-color:${C.mintD};}
/* 목표 */
.goal-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px;}
.goal-btn{display:flex;flex-direction:column;align-items:center;gap:5px;padding:.9rem;border-radius:14px;border:1px solid ${C.border};background:${C.bgAlt};color:${C.ink};cursor:pointer;font-family:${F};font-weight:700;font-size:14px;}
.goal-btn.on{background:${C.blushL};border-color:${C.blush};color:${C.blushD};}
`;

// ===== 앱 =====
export default function App(){
  const [loading,setLoading]=useState(true);
  const [profile,setProfile]=useState(null);
  const [roster,setRoster]=useState([]);
  const [nameInput,setNameInput]=useState("");
  const [introSeen,setIntroSeen]=useState(false);
  const [screen,setScreen]=useState("map");
  const [tab,setTab]=useState("map");
  const [activeDay,setActiveDay]=useState(0);
  const [activeMeal,setActiveMeal]=useState(null);
  const [openWks,setOpenWks]=useState({0:true,1:true,2:true});
  const [data,setData]=useState(()=>Object.fromEntries(Array.from({length:21},(_,i)=>[i,emptyDay()])));
  const [adminMode,setAdminMode]=useState(false);
  const [adminPersona,setAdminPersona]=useState("steady");
  const [evalAnswers,setEvalAnswers]=useState({});
  const [goalSel,setGoalSel]=useState(null);
  const [photoView,setPhotoView]=useState(null);

  useEffect(()=>{(async()=>{
    const r=await loadRoster(); setRoster(r); setLoading(false);
  })();},[]);

  async function loadUserData(name){
    const loaded={};
    for(let i=0;i<21;i++){const d=await loadDay(name,i); loaded[i]=d||emptyDay();}
    setData(loaded);
  }
  async function selectUser(name){
    await loadUserData(name);
    setProfile({name}); setScreen("map"); setTab("map");
  }
  async function createUser(){
    const name=nameInput.trim(); if(!name)return;
    if(name.toLowerCase()==="admin03"){
      const sd=makeSampleData("steady");
      setData(sd); setAdminMode(true); setAdminPersona("steady");
      setProfile({name:"관리자 미리보기"});
      setScreen("hatch"); setTab("hatch"); return;
    }
    const list=await addToRoster(name); setRoster(list);
    setNameInput(""); await selectUser(name);
  }
  function logout(){ setProfile(null); setIntroSeen(false); setAdminMode(false); setData(Object.fromEntries(Array.from({length:21},(_,i)=>[i,emptyDay()]))); }
  function switchAdminPersona(p){ setAdminPersona(p); setData(makeSampleData(p)); }
  function setMealField(day,meal,field,val){ setData(prev=>{const d=JSON.parse(JSON.stringify(prev));d[day][meal][field]=val;if(field==="category"&&val&&!d[day][meal].loggedAt)d[day][meal].loggedAt=Date.now();saveDay(profile.name,day,d[day]);return d;}); }
  function toggleChip(day,meal,field,val){ setData(prev=>{const d=JSON.parse(JSON.stringify(prev));d[day][meal][field]=d[day][meal][field]===val?null:val;if(field==="category"&&d[day][meal][field])d[day][meal].loggedAt=Date.now();saveDay(profile.name,day,d[day]);return d;}); }
  function handlePhoto(day,meal,e){ const f=e.target.files[0];if(!f)return;const r=new FileReader();r.onload=async ev=>{const c=await compressImage(ev.target.result);setMealField(day,meal,"photo",c);};r.readAsDataURL(f); }
  function copyYesterday(day,meal){ setData(prev=>{const d=JSON.parse(JSON.stringify(prev));d[day][meal]={...prev[day-1][meal],loggedAt:Date.now()};saveDay(profile.name,day,d[day]);return d;}); }
  function saveEval(){ try{ window.storage.set(`u:${nkey(profile.name)}:eval`,JSON.stringify({answers:evalAnswers,goal:goalSel,at:Date.now()})); }catch(e){} }

  const stampsEarned=useMemo(()=>Array.from({length:21},(_,i)=>dayDone(data[i])).filter(Boolean).length,[data]);
  const growth=useMemo(()=>growthStage(data,adminMode),[data,adminMode]);
  const persona=useMemo(()=>adminMode?PERSONAS[adminPersona]:analyzePersona(data),[data,adminMode,adminPersona]);
  const ratios=useMemo(()=>foodRatios(data),[data]);
  const photos=useMemo(()=>collectPhotos(data),[data]);
  const stats=useMemo(()=>{
    const cc={},pc={},ec={};
    for(let i=0;i<21;i++)for(const s of MEAL_SLOTS){const m=data[i][s.key];if(m.category)cc[m.category]=(cc[m.category]||0)+1;if(m.physical)pc[m.physical]=(pc[m.physical]||0)+1;if(m.emotional)ec[m.emotional]=(ec[m.emotional]||0)+1;}
    return{topCat:Object.entries(cc).sort((a,b)=>b[1]-a[1])[0],topPhy:Object.entries(pc).sort((a,b)=>b[1]-a[1])[0],topEmo:Object.entries(ec).sort((a,b)=>b[1]-a[1])[0]};
  },[data]);
  const todayIdx=programDayIndex();
  const rows=[];
  for(let i=0;i<21;i+=3){const row=[i,i+1,i+2].filter(x=>x<21);rows.push((Math.floor(i/3)%2===1)?[...row].reverse():row);}

  // ===== INTRO: 닉네임 선택 =====
  if(!loading&&!profile&&!introSeen) return(<><style>{css}</style><div className="app">
    <div className="intro">
      <svg className="intro-seal" viewBox="0 0 100 100">
        <ellipse cx="50" cy="92" rx="22" ry="4" fill="#00000010"/>
        <path d="M50 12 C70 12 80 42 80 60 C80 78 67 86 50 86 C33 86 20 78 20 60 C20 42 30 12 50 12 Z" fill="#F3E2DB" stroke="#D8BEB2" strokeWidth="1.5"/>
        <path d="M50 30 L45 42 L53 50 L47 60" fill="none" stroke="#B89384" strokeWidth="1.5" strokeLinecap="round"/>
        <path d="M76 28 l1.5 4 l4 1.5 l-4 1.5 l-1.5 4 l-1.5 -4 l-4 -1.5 l4 -1.5 z" fill={C.gold}/>
      </svg>
      <div className="intro-title">한국 미식 여정</div>
      <div className="intro-story">"오늘의 식사를 기록하면<br/>알이 너의 기억을 먹고 자란다."</div>
      {roster.length>0&&<div className="story-box"><h3>👤 이어서 기록하기</h3><div className="roster-list">{roster.map(n=>(<button key={n} className="roster-btn" onClick={()=>selectUser(n)}><span>🌸 {n}</span><span style={{color:C.inkL,fontSize:11}}>›</span></button>))}</div></div>}
      <div className="story-box"><h3>✨ 처음 시작하기</h3>
        <input className="intro-inp" style={{maxWidth:"100%",marginBottom:10}} placeholder="닉네임을 입력하세요" value={nameInput} onChange={e=>setNameInput(e.target.value)} maxLength={20} onKeyDown={e=>{if(e.key==="Enter"&&nameInput.trim())setIntroSeen(true);}}/>
        <button className="intro-btn" style={{maxWidth:"100%"}} disabled={!nameInput.trim()} onClick={()=>setIntroSeen(true)}>다음 →</button>
      </div>
      <div style={{fontSize:11.5,color:C.inkL,fontFamily:F,lineHeight:1.6,marginTop:4}}>※ 기록은 현재 기기에 저장됩니다.</div>
    </div>
  </div></>);

  // ===== INTRO: 스토리 설명 =====
  if(!loading&&!profile&&introSeen) return(<><style>{css}</style><div className="app">
    <div className="intro">
      <div className="intro-title" style={{marginTop:"1rem"}}>{nameInput.trim()}님, 환영해요!</div>
      <div className="intro-story">이제 21일간의 미식 여정이 시작돼요 🌸</div>
      <div className="story-box"><h3>🥚 너의 알을 키우는 법</h3>
        <div className="growth-row"><span className="growth-emoji">🥚</span><span className="growth-txt"><b>알</b> — 매일 식사를 기록할 때마다 알에 금이 가요.</span></div>
        <div className="growth-row"><span className="growth-emoji">🐤</span><span className="growth-txt"><b>병아리</b> — 여정의 70%를 넘기면 검은 털 아기새로 부화해요.</span></div>
        <div className="growth-row"><span className="growth-emoji">🦖</span><span className="growth-txt"><b>공룡</b> — <b>7월 5일 오전 5시</b>, 나만의 공룡이 태어나요!</span></div>
      </div>
      <div className="story-box"><h3>🎭 어떤 공룡이 태어날까?</h3>
        <div className="growth-txt" style={{marginBottom:8,fontSize:12}}>식습관과 기록 습관에 따라 4가지 성향 중 하나가 탄생해요.</div>
        <div className="persona-row"><span className="persona-tag">🤍 꾸준형</span><span className="persona-tag">🧭 모험형</span><span className="persona-tag">🌙 야행성</span><span className="persona-tag">⚖️ 균형형</span></div>
      </div>
      <div className="story-box"><h3>🏮 도장 획득 조건</h3>
        <div className="growth-txt" style={{fontSize:12}}>하루 중 <b>한 끼라도</b> 식사종류·신체느낌·감정 <b>세 가지를 모두</b> 입력하면 도장이 찍혀요.</div>
      </div>
      <button className="intro-btn" onClick={createUser}>여정 시작하기 →</button>
      <button className="roster-btn" style={{maxWidth:300,margin:"10px auto 0",justifyContent:"center"}} onClick={()=>setIntroSeen(false)}>← 닉네임 다시 정하기</button>
    </div>
  </div></>);

  return(<><style>{css}</style><div className="app">
    {loading&&<div style={{padding:"3rem",textAlign:"center",color:C.inkL,fontFamily:F}}>Loading…</div>}

    {/* MAP */}
    {!loading&&screen==="map"&&<>
      <div className="hdr">
        <button className="back" onClick={logout}>‹ 닉네임 변경</button>
        <h1>🌸 {profile?.name}님의 여정</h1>
        <p>{stampsEarned}/21 도장 수집</p>
      </div>
      <div className="nav">
        {[["map","ti-map-2","Map"],["recap","ti-book","Diary"],["hatch","ti-egg","공룡"]].map(([t,ic,lb])=>(
          <button key={t} className={`nt${tab===t?" on":""}`} onClick={()=>{setTab(t);setScreen(t);}}>
            <i className={`ti ${ic}`}/><span>{lb}</span>
          </button>
        ))}
      </div>
      <div className="rmap">
        <div className="companion" onClick={()=>{setScreen("hatch");setTab("hatch");}}>
          {growth.stage==="egg"&&<><EggSVG pct={growth.pct}/><div className="companion-title">🥚 기억의 알</div><div className="companion-sub">매일 기록하면 알이 자라요</div><div className="crack-pct">부화까지 {Math.max(0,Math.ceil((0.7-growth.pct)*21))}일 · {Math.round(growth.pct/0.7*100)}%</div></>}
          {growth.stage==="chick"&&<><ChickSVG/><div className="companion-title">🐤 아기새 부화!</div><div className="companion-sub">마지막 날, 공룡으로 성장해요</div></>}
          {growth.stage==="dino"&&<><DinoSVG persona={persona.key} size={130}/><div className="companion-title">🦖 {persona.creature}</div><div className="companion-sub">탭하여 결과 확인</div></>}
        </div>
        <div className="progress-wrap">
          <div className="progress-top"><span>여정 진행률</span><span>{Math.round(stampsEarned/21*100)}%</span></div>
          <div className="progress-bar"><div className="progress-fill" style={{width:`${stampsEarned/21*100}%`}}/></div>
        </div>
        {rows.map((row,ri)=>{
          const wn=ri<3?1:ri<6?2:3;
          return(<div key={ri}>
            {(ri===0||ri===3||ri===6)&&<div className="wk-hdr"><div className="wk-line"/><div className="wk-txt">Week {wn}</div><div className="wk-line"/></div>}
            <div className="nodes-row">{row.map(i=>{
              const done=dayDone(data[i]),ul=isDayUnlocked(i),isCur=i===todayIdx,isDep=i===20;
              return(<div key={i} className="node-wrap">
                <div className={`node${isCur&&!done?" cur":""}${!ul?" locked":""}`} onClick={()=>{if(!ul)return;setActiveDay(i);setActiveMeal(null);setScreen("day");}}>
                  {done?<MoodStamp day={data[i]} idx={i} size={62}/>:!ul?<><span style={{fontSize:18,opacity:.5}}>🔒</span><span className="node-num">{i+1}</span></>:isCur?<><span style={{fontSize:22}}>📍</span><span className="node-num">{i+1}</span></>:<><div className="node-dot"/><span className="node-num">{i+1}</span></>}
                </div>
                <div className="node-date">{dayLabel(i)}<br/><span className="node-wd">{dayWeekday(i)}</span>{isDep&&<div className="dep-badge">DEPARTURE</div>}</div>
              </div>);
            })}</div>
          </div>);
        })}
      </div>
    </>}

    {/* DAY */}
    {!loading&&screen==="day"&&<>
      <div className="hdr">
        <button className="back" onClick={()=>{if(activeMeal)setActiveMeal(null);else setScreen("map");}}>‹ {activeMeal?"끼니 선택으로":"지도로"}</button>
        <h1>Day {activeDay+1} — {dayLabel(activeDay)} ({dayWeekday(activeDay)})</h1>
        <p>{activeDay===20?"✈️ Departure day":dayDone(data[activeDay])?"🏮 오늘의 도장 획득!":"한 끼 완성하면 도장이 찍혀요"}</p>
      </div>
      <div className="day-scroll">
        {!activeMeal&&<>
          <div className="meal-hint">한 끼라도 <b>종류·신체·감정</b>을 모두 채우면 🏮 도장!</div>
          <div className="meal-tabs-grid">{MEAL_SLOTS.map(slot=>{const m=data[activeDay][slot.key];const complete=mealComplete(m);const started=mealLogged(m);return(<div key={slot.key} className={`mtab${complete?" logged":""}`} onClick={()=>setActiveMeal(slot.key)}>{complete?<span className="mtab-check">✓</span>:started?<span className="mtab-partial">…</span>:null}<span className="mtab-emoji">{slot.emoji}</span><span className="mtab-label">{slot.label}</span><span className="mtab-ko">{slot.ko}</span></div>);})}</div>
        </>}
        {activeMeal&&(()=>{
          const slot=MEAL_SLOTS.find(s=>s.key===activeMeal); const m=data[activeDay][activeMeal];
          return(<div className="meal-detail">
            <div className="md-hdr"><span className="md-title"><span style={{fontSize:18}}>{slot.emoji}</span>{slot.label} · {slot.ko}</span>{activeDay>0&&<button className="copy-btn" onClick={()=>copyYesterday(activeDay,activeMeal)}>어제 복사</button>}</div>
            <div className="md-body">
              <div className="cl">식사 종류</div>
              <div className="chips">{CHIPS.category.map(v=>(<button key={v} className={`chip${m.category===v?" s-cat":""}`} onClick={()=>toggleChip(activeDay,activeMeal,"category",v)}>{v}</button>))}</div>
              {m.category&&m.category!=="Skipped"&&<>
                <div className="divider"/>
                <div className="cl">식사 후 신체 느낌</div>
                <div className="chips">{CHIPS.physical.map(v=>(<button key={v} className={`chip${m.physical===v?" s-phy":""}`} onClick={()=>toggleChip(activeDay,activeMeal,"physical",v)}>{v}</button>))}</div>
                <div className="cl">식사 후 감정</div>
                <div className="chips">{CHIPS.emotional.map(v=>(<button key={v} className={`chip${m.emotional===v?" s-emo":""}`} onClick={()=>toggleChip(activeDay,activeMeal,"emotional",v)}>{v}</button>))}</div>
              </>}
              <div className="divider"/>
              <div className="cl">장소</div>
              <input className="loc-inp" placeholder="예: 홍대 포차, GS25…" value={m.location} onChange={e=>setMealField(activeDay,activeMeal,"location",e.target.value)}/>
              <div className="divider"/>
              <div className="cl">사진</div>
              {m.photo?<div><img src={m.photo} alt="meal" className="photo-preview"/><button className="rm-photo-btn" onClick={()=>setMealField(activeDay,activeMeal,"photo",null)}>✕ 사진 삭제</button></div>:<div className="photo-area" onClick={()=>{const inp=document.createElement("input");inp.type="file";inp.accept="image/*";inp.onchange=e=>handlePhoto(activeDay,activeMeal,e);inp.click();}}><div style={{fontSize:24,marginBottom:4}}>🌸</div><div>사진 추가하기</div></div>}
              <div className={`meal-status${mealComplete(m)?" done":""}`}>{mealComplete(m)?"✓ 이 끼니로 도장 조건 완성!":"종류·신체·감정을 모두 채우면 도장 조건 완성"}</div>
            </div>
          </div>);
        })()}
      </div>
    </>}

    {/* HATCH */}
    {!loading&&screen==="hatch"&&<>
      <div className="hdr">
        <button className="back" onClick={()=>{if(adminMode){setAdminMode(false);logout();}else{setScreen("map");setTab("map");}}}>‹ {adminMode?"미리보기 종료":"지도로"}</button>
        <h1>나의 동반자</h1>
        <p>{growth.stage==="dino"?"공룡이 탄생했어요!":growth.stage==="chick"?"부화 완료":"알을 키우는 중"}</p>
      </div>
      <div className="hatch">
        {adminMode&&<div className="admin-bar">
          <div className="admin-label">🔧 성향별 결과 미리보기</div>
          <div className="admin-tabs">{Object.values(PERSONAS).map(p=>(<button key={p.key} className={`admin-tab${adminPersona===p.key?" on":""}`} onClick={()=>switchAdminPersona(p.key)}>{p.name}</button>))}</div>
        </div>}

        {growth.stage==="egg"&&<>
          <div className="hatch-label">Stage 1 · The Egg</div>
          <EggSVG pct={growth.pct} size={200}/>
          <div className="hatch-name">🥚 기억의 알</div>
          <div className="hatch-desc">매일의 식사를 기록할수록 알에 금이 가요.<br/>여정의 <b>70%</b>를 넘기면 아기새로 부화합니다.<br/><br/>지금까지 <b>{growth.logged}일</b> 기록 · 부화까지 <b>{Math.max(0,Math.ceil((0.7-growth.pct)*21))}일</b> 남음</div>
        </>}
        {growth.stage==="chick"&&<>
          <div className="hatch-label">Stage 2 · The Hatchling</div>
          <ChickSVG size={200}/>
          <div className="hatch-name">🐤 별빛 아기새</div>
          <div className="hatch-desc">알을 깨고 검은 털의 아기새가 태어났어요!<br/>마지막 날까지 기록을 이어가면 나만의 공룡으로 성장합니다.</div>
        </>}
        {growth.stage==="dino"&&(()=>{
          const p=persona; const topFood=ratios[0]?ratios[0].name:"맛있는 음식";
          return(<>
            <div className="hatch-label">Final · {p.en}</div>
            <DinoSVG persona={p.key} size={230}/>
            <div className="hatch-name">🦖 {p.creature}</div>
            <div className="hatch-en">{p.name} · {p.en}</div>

            {/* 식사 비율 */}
            <div className="section-card">
              <div className="section-title">🍽️ 이번 달 당신은</div>
              {ratios.length>0?<>{ratios.slice(0,5).map(r=>(<div key={r.name} className="ratio-row">
                <div className="ratio-head"><span>{r.name}</span><span className="ratio-pct">{r.pct}%</span></div>
                <div className="ratio-bar"><div className="ratio-fill" style={{width:`${r.pct}%`}}/></div>
              </div>))}<div className="ratio-caption">…을(를) 먹었어요!</div></>:<div style={{fontSize:13,color:C.inkL,fontFamily:F}}>기록된 식사가 없어요</div>}
            </div>

            {/* 편지 */}
            <div className="letter-card">
              <div className="letter-head">💌 {p.creature}의 편지</div>
              <div className="letter-body">{p.letter(topFood)}</div>
            </div>

            {/* 사진 갤러리 */}
            {photos.length>0&&<div className="section-card">
              <div className="section-title">📸 한 달의 기록 ({photos.length}장)</div>
              <div className="photo-grid">{photos.map((ph,idx)=>(<div key={idx} className="photo-thumb" onClick={()=>setPhotoView(ph)}><img src={ph.photo} alt=""/><span className="photo-day">D{ph.day+1}</span></div>))}</div>
            </div>}

            {/* 통계 */}
            <div className="hatch-stats">
              <div className="hsrow"><span>총 기록한 날</span><span>{growth.logged}/21일</span></div>
              <div className="hsrow"><span>가장 많이 먹은</span><span>{stats.topCat?stats.topCat[0]:"—"}</span></div>
              <div className="hsrow"><span>대표 감정</span><span>{stats.topEmo?stats.topEmo[0]:"—"}</span></div>
              <div className="hsrow"><span>나의 성향</span><span>{p.name}</span></div>
            </div>

            {/* 자가 평가 */}
            <div className="section-card">
              <div className="section-title">📝 스스로 평가하기</div>
              {[
                {q:"이번 달 식습관에 만족하나요?",opts:["매우 만족","대체로 만족","보통","아쉬움"]},
                {q:"가장 개선된 점은?",opts:["다양한 음식 시도","꾸준한 기록","감정 인식","균형 잡힌 식사"]},
                {q:"식사가 기분에 영향을 줬나요?",opts:["많이","조금","잘 모르겠음"]},
                {q:"다음 달에도 기록할 의향이 있나요?",opts:["꼭 할래요","아마도","글쎄요"]},
              ].map((item,qi)=>(<div key={qi} className="eval-q">
                <div className="eval-qtext">{item.q}</div>
                <div className="eval-opts">{item.opts.map(o=>(<button key={o} className={`eval-opt${evalAnswers[qi]===o?" on":""}`} onClick={()=>setEvalAnswers(p=>({...p,[qi]:o}))}>{o}</button>))}</div>
              </div>))}
            </div>

            {/* 다음 달 목표 */}
            <div className="section-card">
              <div className="section-title">🎯 다음 달 챌린지를 시작하시겠습니까?</div>
              <div className="goal-grid">
                {[["🥗","채소 늘리기"],["🌅","아침 챙겨 먹기"],["💧","물 많이 마시기"],["🌙","야식 줄이기"]].map(([ic,g])=>(
                  <button key={g} className={`goal-btn${goalSel===g?" on":""}`} onClick={()=>setGoalSel(g)}><span style={{fontSize:22}}>{ic}</span><span>{g}</span></button>
                ))}
              </div>
              <button className="intro-btn" style={{maxWidth:"100%",marginTop:14}} disabled={!goalSel} onClick={()=>{saveEval();setScreen("nextcycle");}}>
                {goalSel?`'${goalSel}' 챌린지 시작하기 →`:"챌린지를 선택해주세요"}
              </button>
            </div>

            <div className="cheer">"한 달 동안 정말 잘 해냈어요, {profile?.name}님.<br/>이 작은 생명체는 온전히 당신만의 것이에요. 🌸"</div>
          </>);
        })()}
      </div>
    </>}

    {/* NEXT CYCLE */}
    {!loading&&screen==="nextcycle"&&<>
      <div className="hdr"><button className="back" onClick={()=>setScreen("hatch")}>‹ 결과로</button><h1>다음 여정</h1><p>새로운 사이클</p></div>
      <div className="hatch">
        <div style={{fontSize:64,margin:"1rem 0"}}>🥚</div>
        <div className="hatch-name">새로운 알이 준비됐어요</div>
        <div className="hatch-desc" style={{marginTop:"1rem"}}>이번 달 목표: <b>{goalSel}</b><br/><br/>다음 사이클에서 이 챌린지를 마음에 품고<br/>또 한 번 너만의 생명체를 키워보세요.<br/><br/>너의 기록은 언제나 새로운 시작이 될 거예요. 🌙</div>
        <button className="intro-btn" style={{maxWidth:"100%",marginTop:18}} onClick={()=>{setScreen("map");setTab("map");}}>지도로 돌아가기</button>
      </div>
    </>}

    {/* RECAP */}
    {!loading&&screen==="recap"&&<>
      <div className="hdr">
        <button className="back" onClick={()=>{setScreen("map");setTab("map");}}>‹ 지도로</button>
        <h1>여행 다이어리</h1>
        <p>🏮 {stampsEarned}/21 도장 · {profile?.name}님</p>
      </div>
      <div className="rec-scroll">
        <div className="stats">
          <div className="sc"><div className="sv">{stats.topCat?stats.topCat[0]:"—"}</div><div className="sk">가장 많이 먹은 ({stats.topCat?stats.topCat[1]:0}회)</div></div>
          <div className="sc"><div className="sv">{stats.topEmo?stats.topEmo[0]:"—"}</div><div className="sk">대표 감정</div></div>
          <div className="sc w"><div className="sv">{stats.topPhy?stats.topPhy[0]:"—"}</div><div className="sk">가장 흔한 신체 느낌</div></div>
        </div>
        {[0,1,2].map(w=>(<div key={w} className="rwk">
          <button className="wtog" onClick={()=>setOpenWks(p=>({...p,[w]:!p[w]}))}><span>Week {w+1}</span><span>{openWks[w]?"▲":"▼"}</span></button>
          {openWks[w]&&Array.from({length:7},(_,d)=>w*7+d).filter(i=>i<21).map(i=>(<div key={i} className="rday">
            <div className="rday-stamp"><MoodStamp day={data[i]} idx={i} size={46}/></div>
            <div className="rday-body">
              <div className="rdt">Day {i+1} · {dayLabel(i)} ({dayWeekday(i)}){i===20?" · ✈️ Departure":""}</div>
              {MEAL_SLOTS.map(slot=>{const m=data[i][slot.key];return(<div key={slot.key} className="rmr"><span className="rce">{slot.emoji}</span>{mealLogged(m)?<div className="rcc">{m.category&&<span className="rc rc-c">{m.category}</span>}{m.physical&&<span className="rc rc-p">{m.physical}</span>}{m.emotional&&<span className="rc rc-e">{m.emotional}</span>}{m.location&&<span className="rc rc-l">📍{m.location}</span>}{m.photo&&<img src={m.photo} className="rth" alt=""/>}</div>:<span className="emp">미기록</span>}</div>);})}
            </div>
          </div>))}
        </div>))}
      </div>
    </>}

    {/* 사진 확대 뷰어 */}
    {photoView&&<div className="photo-modal" onClick={()=>setPhotoView(null)}>
      <img src={photoView.photo} alt="full"/>
      <div className="photo-modal-cap">Day {photoView.day+1} · {photoView.slot.ko} {photoView.location?`· 📍${photoView.location}`:""}</div>
      <div className="photo-modal-close">탭하여 닫기</div>
    </div>}
  </div></>);
}
