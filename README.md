// ═══════════════════════════════════════
// STATE
// ═══════════════════════════════════════
var S={
  files:[],clips:{},nid:1,
  activeId:null,ar:'16:9',
  vol:1,spd:1,mute:false,
  trimIn:0,trimOut:0,trimOutSet:false,trimVisible:false,
  zoom:100,pxSec:10,
  expRes:'1280x720',
  cropActive:false,cropAR:'free',
  cropX:0,cropY:0,cropW:0,cropH:0,
};
var vid=document.getElementById('prev-vid');
var ffmpeg=null;

// ─── BACKGROUND AUDIO PLAYER ───
// เล่น audio track ซิงค์กับ vid
var bgAudio = new Audio();
bgAudio.preload = 'auto';
var bgAudioCid = null; // cid ของ audio clip ที่โหลดอยู่

function getBgAudioClip(){
  // หา audio clip ที่ควรเล่นตาม globalTime
  var ps = pxSec();
  var globalTime = (window.playQueueOffset||0) + (vid.currentTime||0);
  var keys = Object.keys(S.clips);
  for(var i=0;i<keys.length;i++){
    var c = S.clips[keys[i]];
    if(c.type !== 'audio') continue;
    var startSec = (c.startSec !== undefined) ? c.startSec : (c.left/ps);
    var endSec   = startSec + c.dur;
    if(globalTime >= startSec - 0.1 && globalTime < endSec){
      return {c:c, cid:keys[i], startSec:startSec};
    }
  }
  return null;
}

function syncBgAudio(){
  var ps = pxSec();
  var globalTime = (window.playQueueOffset||0) + (vid.currentTime||0);
  var found = getBgAudioClip();

  if(!found){
    if(!bgAudio.paused){ bgAudio.pause(); }
    bgAudioCid = null;
    return;
  }

  var c = found.c;
  var entry = S.files && S.files.find(function(f){ return f.id === c.fid; });
  if(!entry){ return; }

  // โหลด audio ถ้ายังไม่ได้โหลด
  if(bgAudioCid !== found.cid){
    bgAudio.src = entry.url;
    bgAudioCid = found.cid;
  }

  // คำนวณตำแหน่งใน audio file
  var localTime = globalTime - found.startSec;
  localTime = Math.max(0, Math.min(entry.dur - 0.05, localTime));

  // sync ถ้าห่างเกิน 0.3 วินาที
  if(Math.abs(bgAudio.currentTime - localTime) > 0.3){
    bgAudio.currentTime = localTime;
  }

  bgAudio.volume = S.vol !== undefined ? S.vol : 1;

  if(window.isPlaying && bgAudio.paused){
    var pb = bgAudio.play();
    if(pb) pb.catch(function(){});
  } else if(!window.isPlaying && !bgAudio.paused){
    bgAudio.pause();
  }
}

// ═══════════════════════════════════════
// ICON BAR
// ═══════════════════════════════════════
document.querySelectorAll('.ib').forEach(function(b){
  b.addEventListener('click',function(){
    var p=b.dataset.p;
    if(p==='exp-modal'){openExp();return;}
    document.querySelectorAll('.ib').forEach(function(x){x.classList.remove('on');});
    b.classList.add('on');
    ['p-media','p-cut','p-text','p-fx'].forEach(function(id){
      document.getElementById(id).style.display=(id==='p-'+p)?'flex':'none';
    });
  });
});

// ═══════════════════════════════════════
// LEFT PANEL SUB-PANELS
// ═══════════════════════════════════════
var tools=['crop','trim','speed','vol'];
tools.forEach(function(t){
  var row=document.getElementById('t-'+t);
  var sub=document.getElementById('s-'+t);
  var close=document.getElementById('sc-'+t);
  if(!row||!sub) return;
  row.addEventListener('click',function(){
    tools.forEach(function(x){var s=document.getElementById('s-'+x);if(s)s.style.display='none';});
    sub.style.display='block';
  });
  if(close) close.addEventListener('click',function(){sub.style.display='none';});
});

// Crop AR in left panel
document.querySelectorAll('.ar-card').forEach(function(c){
  c.addEventListener('click',function(){
    document.querySelectorAll('.ar-card').forEach(function(x){x.classList.remove('on');});
    c.classList.add('on');
    S.cropAR=c.dataset.ar;
    if(S.cropActive) applyCropAR(c.dataset.ar);
  });
});
document.getElementById('btn-apply-crop').addEventListener('click',function(){
  if(!document.getElementById('prev-wrap').style.display||document.getElementById('prev-wrap').style.display==='none'){
    showToast('⚠️ นำเข้าวิดีโอก่อน');return;
  }
  S.cropActive=!S.cropActive;
  document.getElementById('crop-ov').classList.toggle('active',S.cropActive);
  if(S.cropActive){
    initCropBox();
    showToast('🔲 ลากกรอบเพื่อครอบตัด');
    this.textContent='✅ ปิดการครอบตัด';
  } else {
    this.textContent='✅ ใช้การครอบตัดนี้';
    showToast('✂ บันทึกการครอบตัดแล้ว');
    // reset vid style
    vid.style.position='';
    vid.style.left=''; vid.style.top='';
    vid.style.width=''; vid.style.height='';
    applyARToPreview();
  }
});

// Trim sliders left
document.getElementById('sl-in').addEventListener('input',function(){
  S.trimIn=Math.min(parseFloat(this.value),S.trimOut-0.3);
  this.value=S.trimIn;
  document.getElementById('in-v').textContent=S.trimIn.toFixed(1)+'s';
  syncTrimRP();updateTrimMarkers();
  vid.currentTime=S.trimIn;
});
document.getElementById('sl-out').addEventListener('input',function(){
  S.trimOut=Math.max(parseFloat(this.value),S.trimIn+0.3);
  S.trimOutSet=true;
  this.value=S.trimOut;
  document.getElementById('out-v').textContent=S.trimOut.toFixed(1)+'s';
  syncTrimRP();updateTrimMarkers();
  vid.currentTime=S.trimOut;
});
document.getElementById('btn-set-in').addEventListener('click',function(){
  S.trimIn=vid.currentTime;
  setTrimUI();updateTrimMarkers();
  showToast('⬅ IN='+S.trimIn.toFixed(1)+'s');
});
document.getElementById('btn-set-out').addEventListener('click',function(){
  S.trimOut=vid.currentTime; S.trimOutSet=true;
  setTrimUI();updateTrimMarkers();
  showToast('➡ OUT='+S.trimOut.toFixed(1)+'s');
});
document.getElementById('btn-clear-trim').addEventListener('click',function(){
  S.trimIn=0;S.trimOut=vid.duration||0; S.trimOutSet=false;
  setTrimUI();updateTrimMarkers();
  showToast('🔄 รีเซ็ต Trim แล้ว');
});

// Speed
document.getElementById('sl-spd').addEventListener('input',function(){
  S.spd=parseInt(this.value)/100;vid.playbackRate=S.spd;
  document.getElementById('spd-v').textContent=S.spd.toFixed(2)+'×';
  document.getElementById('rp-spd').value=this.value;
  document.getElementById('rp-spd-v').textContent=S.spd.toFixed(2)+'×';
});
document.querySelectorAll('[data-spd]').forEach(function(b){
  b.addEventListener('click',function(){
    var v=parseInt(b.dataset.spd);
    document.getElementById('sl-spd').value=v;
    S.spd=v/100;vid.playbackRate=S.spd;
    document.getElementById('spd-v').textContent=S.spd.toFixed(2)+'×';
    showToast('⚡ ความเร็ว '+S.spd.toFixed(2)+'×');
  });
});
// Vol
document.getElementById('sl-vol').addEventListener('input',function(){
  S.vol=parseInt(this.value)/100;
  vid.volume=Math.min(1,S.vol);
  if(typeof bgAudio!=='undefined') bgAudio.volume=Math.min(1,S.vol);
  document.getElementById('vol-v').textContent=this.value+'%';
  document.getElementById('rp-vol').value=this.value;
  document.getElementById('rp-vol-v').textContent=this.value+'%';
  // sync pb-vol slider
  var pbv=document.getElementById('pb-vol');
  if(pbv) pbv.value=Math.min(1,S.vol);
});
document.getElementById('cb-mute').addEventListener('change',function(){
  S.mute=this.checked;
  vid.muted=S.mute;
  if(typeof bgAudio!=='undefined') bgAudio.muted=S.mute;
});

// ═══════════════════════════════════════
// RIGHT PANEL
// ═══════════════════════════════════════
// AR
document.querySelectorAll('.ar-opt').forEach(function(el){
  el.addEventListener('click',function(){
    document.querySelectorAll('.ar-opt').forEach(function(o){o.classList.remove('on');});
    el.classList.add('on');
    S.ar=el.dataset.ar;
    applyARToPreview();
    showToast('📐 '+S.ar+' – '+el.dataset.lbl);
  });
});
// Res
document.querySelectorAll('.res-btn').forEach(function(b){
  b.addEventListener('click',function(){
    document.querySelectorAll('.res-btn').forEach(function(x){x.classList.remove('on');});
    b.classList.add('on');S.expRes=b.dataset.res;
    showToast('📐 '+b.textContent);
  });
});
// Trim RP
document.getElementById('rp-in').addEventListener('input',function(){
  S.trimIn=Math.min(parseFloat(this.value),S.trimOut-0.3);
  this.value=S.trimIn;
  document.getElementById('rp-in-v').textContent=S.trimIn.toFixed(1)+'s';
  syncTrimLeft();updateTrimMarkers();vid.currentTime=S.trimIn;
});
document.getElementById('rp-out').addEventListener('input',function(){
  S.trimOut=Math.max(parseFloat(this.value),S.trimIn+0.3);
  S.trimOutSet=true;
  this.value=S.trimOut;
  document.getElementById('rp-out-v').textContent=S.trimOut.toFixed(1)+'s';
  syncTrimLeft();updateTrimMarkers();vid.currentTime=S.trimOut;
});
document.getElementById('rp-set-in').addEventListener('click',function(){
  S.trimIn=vid.currentTime;setTrimUI();updateTrimMarkers();showToast('⬅ IN='+S.trimIn.toFixed(1)+'s');
});
document.getElementById('rp-set-out').addEventListener('click',function(){
  S.trimOut=vid.currentTime; S.trimOutSet=true;
  setTrimUI();updateTrimMarkers();showToast('➡ OUT='+S.trimOut.toFixed(1)+'s');
});
document.getElementById('rp-vol').addEventListener('input',function(){
  S.vol=parseInt(this.value)/100;
  vid.volume=Math.min(1,S.vol);
  if(typeof bgAudio!=='undefined') bgAudio.volume=Math.min(1,S.vol);
  document.getElementById('rp-vol-v').textContent=this.value+'%';
  var slv=document.getElementById('sl-vol');
  if(slv) slv.value=this.value;
  document.getElementById('vol-v').textContent=this.value+'%';
  var pbv=document.getElementById('pb-vol');
  if(pbv) pbv.value=Math.min(1,S.vol);
});
document.getElementById('rp-spd').addEventListener('input',function(){
  S.spd=parseInt(this.value)/100;vid.playbackRate=S.spd;document.getElementById('rp-spd-v').textContent=S.spd.toFixed(2)+'×';
});
document.getElementById('rp-mute').addEventListener('change',function(){S.mute=this.checked;});

function syncTrimRP(){
  document.getElementById('rp-in').value=S.trimIn;
  document.getElementById('rp-out').value=S.trimOut;
  document.getElementById('rp-in-v').textContent=S.trimIn.toFixed(1)+'s';
  document.getElementById('rp-out-v').textContent=S.trimOut.toFixed(1)+'s';
}
function syncTrimLeft(){
  document.getElementById('sl-in').value=S.trimIn;
  document.getElementById('sl-out').value=S.trimOut;
  document.getElementById('in-v').textContent=S.trimIn.toFixed(1)+'s';
  document.getElementById('out-v').textContent=S.trimOut.toFixed(1)+'s';
}
function setTrimUI(){
  syncTrimLeft();syncTrimRP();
}

// ═══════════════════════════════════════
// AR → PREVIEW
// ═══════════════════════════════════════
var AR_R={'16:9':16/9,'9:16':9/16,'1:1':1,'4:3':4/3,'4:5':4/5,'21:9':21/9};
// applyARToPreview — resize preview ตาม AR ปกติ
function applyARToPreview(){
  var r = AR_R[S.ar]||16/9;
  var a = document.getElementById('prev-area');
  var aW = a.offsetWidth - 20, aH = a.offsetHeight - 20;
  var w, h;
  if(aW/aH > r){ h = aH; w = h * r; } else { w = aW; h = w / r; }
  w = Math.floor(w); h = Math.floor(h);
  var wr = document.getElementById('prev-wrap');
  wr.style.width  = w + 'px'; wr.style.height = h + 'px';
  vid.style.width  = w + 'px'; vid.style.height = h + 'px';
}
window.addEventListener('resize', applyARToPreview);
// CROP BOX (draggable + handles)
// ═══════════════════════════════════════
function initCropBox(){
  var wr=document.getElementById('prev-wrap');
  var w=wr.offsetWidth,h=wr.offsetHeight;
  var pad=20;
  S.cropX=pad;S.cropY=pad;S.cropW=w-pad*2;S.cropH=h-pad*2;
  updateCropBox();
  if(S.cropAR!=='free') applyCropAR(S.cropAR);
}
function updateCropBox(){
  var box=document.getElementById('crop-box');
  box.style.left=S.cropX+'px';box.style.top=S.cropY+'px';
  box.style.width=S.cropW+'px';box.style.height=S.cropH+'px';
}
function applyCropAR(ar){
  var wr=document.getElementById('prev-wrap');
  var vw=wr.offsetWidth,vh=wr.offsetHeight;
  var r=AR_R[ar];
  if(!r) return;
  var w,h;
  if(vw/vh>r){h=vh;w=h*r;}else{w=vw;h=w/r;}
  S.cropX=Math.floor((vw-w)/2);S.cropY=Math.floor((vh-h)/2);
  S.cropW=Math.floor(w);S.cropH=Math.floor(h);
  updateCropBox();
}
// Drag crop box
(function(){
  var box=document.getElementById('crop-box');
  // Move
  box.addEventListener('mousedown',function(e){
    if(e.target.classList.contains('ch')) return;
    var sx=e.clientX,sy=e.clientY,ox=S.cropX,oy=S.cropY;
    var mm=function(e2){
      var wr=document.getElementById('prev-wrap');
      S.cropX=Math.max(0,Math.min(wr.offsetWidth-S.cropW,ox+e2.clientX-sx));
      S.cropY=Math.max(0,Math.min(wr.offsetHeight-S.cropH,oy+e2.clientY-sy));
      updateCropBox();
    };
    var mu=function(){document.removeEventListener('mousemove',mm);document.removeEventListener('mouseup',mu);};
    document.addEventListener('mousemove',mm);document.addEventListener('mouseup',mu);
    e.stopPropagation();
  });
  // Handles
  var handles=[
    {cls:'tl',dx:-1,dy:-1,dw:1,dh:1},
    {cls:'tc',dx:0,dy:-1,dw:0,dh:1},
    {cls:'tr',dx:0,dy:-1,dw:1,dh:1},
    {cls:'ml',dx:-1,dy:0,dw:1,dh:0},
    {cls:'mr',dx:0,dy:0,dw:1,dh:0},
    {cls:'bl',dx:-1,dy:0,dw:1,dh:-1},
    {cls:'bc',dx:0,dy:0,dw:0,dh:1},  // fixed: bc should grow down
    {cls:'br',dx:0,dy:0,dw:1,dh:1},
  ];
  // simpler resize via mousedown on each handle
  box.querySelectorAll('.ch').forEach(function(h){
    h.addEventListener('mousedown',function(e){
      e.stopPropagation();
      var sx=e.clientX,sy=e.clientY;
      var ox=S.cropX,oy=S.cropY,ow=S.cropW,oh=S.cropH;
      var cls=h.className.split(' ')[1];
      var mm=function(e2){
        var dx=e2.clientX-sx,dy=e2.clientY-sy;
        var wr=document.getElementById('prev-wrap');
        var vw=wr.offsetWidth,vh=wr.offsetHeight;
        if(cls==='tl'){S.cropX=Math.max(0,ox+dx);S.cropY=Math.max(0,oy+dy);S.cropW=Math.max(40,ow-dx);S.cropH=Math.max(30,oh-dy);}
        else if(cls==='tc'){S.cropY=Math.max(0,oy+dy);S.cropH=Math.max(30,oh-dy);}
        else if(cls==='tr'){S.cropY=Math.max(0,oy+dy);S.cropW=Math.max(40,ow+dx);S.cropH=Math.max(30,oh-dy);}
        else if(cls==='ml'){S.cropX=Math.max(0,ox+dx);S.cropW=Math.max(40,ow-dx);}
        else if(cls==='mr'){S.cropW=Math.max(40,ow+dx);}
        else if(cls==='bl'){S.cropX=Math.max(0,ox+dx);S.cropW=Math.max(40,ow-dx);S.cropH=Math.max(30,oh+dy);}
        else if(cls==='bc'){S.cropH=Math.max(30,oh+dy);}
        else if(cls==='br'){S.cropW=Math.max(40,ow+dx);S.cropH=Math.max(30,oh+dy);}
        updateCropBox();
      };
      var mu=function(){document.removeEventListener('mousemove',mm);document.removeEventListener('mouseup',mu);};
      document.addEventListener('mousemove',mm);document.addEventListener('mouseup',mu);
    });
  });
})();

// ═══════════════════════════════════════
// FILE INPUT
// ═══════════════════════════════════════
var dz=document.getElementById('dz');
var fi=document.getElementById('fi');
dz.addEventListener('click',function(){fi.click();});
dz.addEventListener('dragover',function(e){e.preventDefault();dz.classList.add('ov');});
dz.addEventListener('dragleave',function(){dz.classList.remove('ov');});
dz.addEventListener('drop',function(e){
  e.preventDefault();dz.classList.remove('ov');
  addFiles(Array.from(e.dataTransfer.files).filter(function(f){return f.type.startsWith('video/');}));
});
fi.addEventListener('change',function(){addFiles(Array.from(this.files));this.value='';});

function addFiles(files){
  if(!files.length) return;
  var done=0;
  files.forEach(function(f){
    var isAudio=f.type.startsWith('audio/');
    var url=URL.createObjectURL(f);
    var tmp=isAudio ? new Audio(url) : document.createElement('video');
    if(!isAudio){tmp.src=url;}
    tmp.preload='metadata';
    tmp.onloadedmetadata=function(){
      var e={id:'f'+(S.nid++),file:f,url:url,dur:tmp.duration,name:f.name,type:isAudio?'audio':'video'};
      S.files.push(e);
      if(!isAudio && S.files.filter(function(x){return x.type!=='audio';}).length===1) loadPreview(e);
      if(isAudio) addAudioClipTL(e);
      else addClipTL(e);
      done++;
      if(done===files.length){renderML();showToast('✅ นำเข้า '+files.length+' ไฟล์');drawRuler();}
    };
    tmp.onerror=function(){done++;showToast('❌ โหลดไม่ได้: '+f.name);};
    if(isAudio) tmp.load();
  });
}

// เพิ่มคลิปเสียงในแทร็ก audio
function addAudioClipTL(entry){
  var track=document.getElementById('tr-a');
  var maxRight=0;
  track.querySelectorAll('.clip').forEach(function(c){
    var r=parseFloat(c.style.left||0)+parseFloat(c.style.width||0);
    if(r>maxRight) maxRight=r;
  });
  var cid='c'+(S.nid++);
  var ps=pxSec();
  var w=entry.dur*ps;
  S.clips[cid]={id:cid,fid:entry.id,dur:entry.dur,w:w,left:maxRight,type:'audio'};
  buildClip(cid,track,entry);
  drawRuler();
}

// อัปเดต dropzone ของ fi ให้รับทั้งวิดีโอและเสียง
document.getElementById('fi').accept='video/*,audio/*';

// ═══════════════════════════════════════
// MEDIA LIST
// ═══════════════════════════════════════

// ── MEDIA TAB: วิดีโอ / เสียง ──
(function(){
  var tabs = document.querySelectorAll('.lp-tab');
  if(!tabs.length) return;
  tabs.forEach(function(tab, idx){
    tab.addEventListener('click', function(){
      tabs.forEach(function(t){ t.classList.remove('on'); });
      tab.classList.add('on');
      // filter ml-items
      var items = document.querySelectorAll('.ml-item');
      items.forEach(function(item){
        var fid = item.dataset.fid;
        var entry = S.files.find(function(f){ return f.id===fid; });
        if(!entry){ item.style.display=''; return; }
        if(idx===0){
          // วิดีโอ tab
          item.style.display = (entry.type==='audio') ? 'none' : '';
        } else {
          // เสียง tab
          item.style.display = (entry.type==='audio') ? '' : 'none';
        }
      });
    });
  });
})();
function renderML(){
  var ml=document.getElementById('ml');
  ml.innerHTML='';
  document.getElementById('ml-cnt').textContent=S.files.length;
  S.files.forEach(function(e){
    var d=document.createElement('div');
    d.className='ml-item'+(S.activeId&&S.clips[S.activeId]&&S.clips[S.activeId].fid===e.id?' active':'');
    d.dataset.fid=e.id;
    // draggable=true สำหรับลากไปวางในไทม์ไลน์
    d.draggable=true;
    d.dataset.fid=e.id;
    d.innerHTML=
      '<div class="ml-thumb" title="ลากมาวางในไทม์ไลน์ได้"><video src="'+e.url+'" muted preload="metadata" style="width:100%;height:100%;object-fit:cover;"></video></div>'+
      '<div class="ml-info"><div class="ml-name">'+e.name+'</div><div class="ml-dur">'+fmt(e.dur)+'</div></div>'+
      '<div class="ml-acts">'+
        '<button class="ml-act" data-a="add" title="เพิ่มในไทม์ไลน์">+</button>'+
        '<button class="ml-act del" data-a="del" title="ลบ">✕</button>'+
      '</div>';

    // คลิกเพื่อ preview
    d.addEventListener('click',function(ev){
      if(ev.target.dataset.a) return;
      document.querySelectorAll('.ml-item').forEach(function(x){x.classList.remove('active');});
      d.classList.add('active'); loadPreview(e);
    });
    // ปุ่ม + และ ✕
    d.addEventListener('click',function(ev){
      if(ev.target.dataset.a==='add'){ addClipTL(e); showToast('➕ เพิ่ม '+e.name+' ในไทม์ไลน์'); }
      if(ev.target.dataset.a==='del') removeFile(e.id);
    });

    // ลากเพื่อสลับลำดับใน media list
    d.addEventListener('dragstart',function(ev){
      ev.dataTransfer.setData('fid',e.id);
      ev.dataTransfer.setData('type','media-reorder');
      d.style.opacity='0.5';
    });
    d.addEventListener('dragend',function(){ d.style.opacity=''; });
    d.addEventListener('dragover',function(ev){ ev.preventDefault(); d.style.outline='1px solid var(--acc)'; });
    d.addEventListener('dragleave',function(){ d.style.outline=''; });
    d.addEventListener('drop',function(ev){
      ev.preventDefault(); d.style.outline='';
      var fid=ev.dataTransfer.getData('fid');
      var type=ev.dataTransfer.getData('type');
      if(type!=='media-reorder'||fid===e.id) return;
      var fi2=S.files.findIndex(function(x){return x.id===fid;});
      var ti=S.files.findIndex(function(x){return x.id===e.id;});
      var tmp=S.files.splice(fi2,1)[0]; S.files.splice(ti,0,tmp); renderML();
    });
    ml.appendChild(d);
  });
}

// ═══════════════════════════════════════
// DROP FROM MEDIA LIST → TIMELINE TRACKS
// ═══════════════════════════════════════
function setupTrackDrop(trackEl){
  trackEl.addEventListener('dragover',function(e){
    e.preventDefault();
    trackEl.classList.add('drag-over');
  });
  trackEl.addEventListener('dragleave',function(){
    trackEl.classList.remove('drag-over');
  });
  trackEl.addEventListener('drop',function(e){
    e.preventDefault();
    trackEl.classList.remove('drag-over');
    var fid=e.dataTransfer.getData('fid');
    if(!fid) return;
    var entry=S.files.find(function(f){return f.id===fid;});
    if(!entry) return;
    var r=trackEl.getBoundingClientRect();
    var sc=document.getElementById('tl-scroll');
    var xInTrack=Math.max(0, e.clientX-r.left+sc.scrollLeft);
    var ps=pxSec();
    var cid='c'+(S.nid++);
    var w=entry.dur*ps;
    var startSec=xInTrack/ps;
    S.clips[cid]={id:cid,fid:entry.id,dur:entry.dur,w:w,left:xInTrack,startSec:startSec,type:entry.type||'video'};
    buildClip(cid,trackEl,entry);
    drawRuler();
    showToast('🎬 วาง '+entry.name);
    if(entry.type!=='audio') loadPreview(entry);
  });
}
function removeFile(fid){
  S.files=S.files.filter(function(f){return f.id!==fid;});
  Object.keys(S.clips).forEach(function(cid){
    if(S.clips[cid]&&S.clips[cid].fid===fid){
      var el=document.querySelector('[data-cid="'+cid+'"]');
      if(el) el.remove();
      delete S.clips[cid];
    }
  });
  renderML();drawRuler();
  if(!S.files.length){
    document.getElementById('prev-wrap').style.display='none';
    document.getElementById('prev-empty').style.display='block';
  } else {loadPreview(S.files[0]);}
  showToast('🗑 ลบแล้ว');
}

// ═══════════════════════════════════════
// PREVIEW LOAD
// ═══════════════════════════════════════
function loadPreview(e){
  vid.src=e.url;
  vid.onloadedmetadata=function(){
    var d=vid.duration; // actual duration from video
    document.getElementById('prev-empty').style.display='none';
    document.getElementById('prev-wrap').style.display='flex';
    document.getElementById('proj-name').textContent=e.name.replace(/\.[^.]+$/,'');
    S.trimIn=0;
    S.trimOut=d;   // always use real duration, never hardcoded
    initTrimSliders(d);
    applyARToPreview();
    showTrimMarkers();
    // Show real duration in playbar
    document.getElementById('pb-tc').textContent=fmt(0)+' / '+fmt(d);
  };
}
function initTrimSliders(d){
  // d = actual video duration in seconds — set as max so slider reaches full length
  ['sl-in','sl-out','rp-in','rp-out'].forEach(function(id){
    var el=document.getElementById(id);
    el.min=0; el.max=d; el.step=0.1;
  });
  document.getElementById('sl-in').value=0;
  document.getElementById('sl-out').value=d;
  document.getElementById('rp-in').value=0;
  document.getElementById('rp-out').value=d;
  document.getElementById('in-v').textContent='0.0s';
  document.getElementById('out-v').textContent=d.toFixed(1)+'s';
  document.getElementById('rp-in-v').textContent='0.0s';
  document.getElementById('rp-out-v').textContent=d.toFixed(1)+'s';
}

// ═══════════════════════════════════════
// TRIM MARKERS (draggable IN/OUT lines)
// ═══════════════════════════════════════
var trimVisible=false;
// IN/OUT toggle (button ถูกเอาออกแล้ว แต่เก็บ logic ไว้ใช้ผ่าน right panel)
var _trimToggleBtn = document.getElementById('tl-trim-toggle');
if(_trimToggleBtn) _trimToggleBtn.addEventListener('click',function(){
  trimVisible=!trimVisible;
  showTrimMarkers();
  this.classList.toggle('on',trimVisible);
  showToast(trimVisible?'🔴 แสดง IN/OUT markers — ลากเส้นได้':'🔴 ซ่อน IN/OUT markers');
});
function showTrimMarkers(){
  var disp=trimVisible?'block':'none';
  document.getElementById('tl-trim-in').style.display=disp;
  document.getElementById('tl-trim-out').style.display=disp;
  document.getElementById('tl-trim-zone').style.display=disp;
  if(trimVisible) updateTrimMarkers();
}
function updateTrimMarkers(){
  if(!vid.duration) return;
  var ps=pxSec();
  var xi=S.trimIn*ps, xo=S.trimOut*ps;
  document.getElementById('tl-trim-in').style.left=xi+'px';
  document.getElementById('tl-trim-out').style.left=xo+'px';
  document.getElementById('tl-trim-zone').style.left=xi+'px';
  document.getElementById('tl-trim-zone').style.width=(xo-xi)+'px';
}
// Drag trim markers — ใช้ pointer capture เพื่อให้ลากนอก element ได้
(function(){
  function dragMarker(el, cb){
    el.style.pointerEvents = 'all';
    el.style.touchAction = 'none';
    el.addEventListener('mousedown', function(e){
      e.preventDefault();
      e.stopPropagation();
      el.style.cursor = 'grabbing';
      var sc = document.getElementById('tl-scroll');
      function onMove(e2){
        var r = sc.getBoundingClientRect();
        var x = Math.max(0, e2.clientX - r.left + sc.scrollLeft);
        var t = x / pxSec();
        cb(Math.max(0, Math.min(vid.duration||999, t)));
      }
      function onUp(){
        el.style.cursor = 'ew-resize';
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', onUp);
      }
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp);
    });
  }
  dragMarker(document.getElementById('tl-trim-in'), function(t){
    S.trimIn = Math.min(t, S.trimOut - 0.3);
    setTrimUI(); updateTrimMarkers(); vid.currentTime = S.trimIn;
    document.getElementById('ep-stat').textContent = '';
    showToast('🔴 IN → ' + S.trimIn.toFixed(2) + 's');
  });
  dragMarker(document.getElementById('tl-trim-out'), function(t){
    S.trimOut = Math.max(t, S.trimIn + 0.3);
    S.trimOutSet = true;
    setTrimUI(); updateTrimMarkers(); vid.currentTime = S.trimOut;
    showToast('🟠 OUT → ' + S.trimOut.toFixed(2) + 's');
  });
})();

// ═══════════════════════════════════════
// TIMELINE CLIPS
// ═══════════════════════════════════════
function pxSec(){return S.pxSec*S.zoom/100;}

function addClipTL(entry){
  var track=document.getElementById('tr-v1');
  // หาตำแหน่งสิ้นสุดของคลิปสุดท้ายในหน่วยวินาที
  var maxSec=0;
  Object.keys(S.clips).forEach(function(cid){
    var c=S.clips[cid];
    var sec=(c.startSec!==undefined)?c.startSec:(c.left/pxSec());
    var end=sec+c.dur;
    if(end>maxSec) maxSec=end;
  });
  var cid='c'+(S.nid++);
  var ps=pxSec();
  var w=entry.dur*ps;
  S.clips[cid]={id:cid,fid:entry.id,dur:entry.dur,w:w,left:maxSec*ps,startSec:maxSec};
  buildClip(cid,track,entry);
  drawRuler();
  scheduleSnapUpdate();
}
function buildClip(cid, track, entry){
  var c = S.clips[cid];
  var isAudio = entry.type === 'audio';
  var el = document.createElement('div');
  el.className = 'clip ' + (isAudio ? 'ac' : 'vc');
  el.dataset.cid = cid;
  el.style.left  = c.left + 'px';
  el.style.width = c.w   + 'px';

  // สร้าง innerHTML
  var inner =
    '<div class="clip-frames" id="cf-'+cid+'"></div>'+
    '<div class="clip-name">'+entry.name+'</div>'+
    '<div class="clip-dur">'+fmt(entry.dur)+'</div>';

  if(isAudio){
    // Waveform bars สุ่ม
    var bars='<div class="clip-wave">';
    for(var b=0;b<40;b++){
      var h=Math.max(2,Math.floor(Math.random()*9)+1);
      bars+='<div style="width:2px;height:'+h+'px;background:#4ade80;border-radius:1px;flex-shrink:0;"></div>';
    }
    bars+='</div>';
    inner += bars;
  } else {
    // Mute button สำหรับวิดีโอ
    inner += '<div class="clip-mute" title="ปิด/เปิดเสียงต้นฉบับ">🔊</div>';
  }

  inner += '<div class="clip-hdl l"></div><div class="clip-hdl r"></div>';
  el.innerHTML = inner;

  // Mute toggle สำหรับวิดีโอ
  if(!isAudio){
    var muteBtn = el.querySelector('.clip-mute');
    c.muted = false;
    muteBtn.addEventListener('click', function(e){
      e.stopPropagation();
      c.muted = !c.muted;
      muteBtn.classList.toggle('muted', c.muted);
      muteBtn.textContent = c.muted ? '🔇' : '🔊';
      muteBtn.title = c.muted ? 'คลิกเพื่อเปิดเสียง' : 'คลิกเพื่อปิดเสียง';
      // ถ้าคลิปนี้กำลังเล่นอยู่ → ปิด/เปิดเสียง vid ทันที
      if(S.activeId === cid || (playQueueClips[playIdx]&&playQueueClips[playIdx].c&&playQueueClips[playIdx].c.id === cid)){
        vid.muted = c.muted;
      }
      showToast(c.muted ? '🔇 ปิดเสียงต้นฉบับ' : '🔊 เปิดเสียงต้นฉบับ');
    });
  }

  // Thumbnail frames (วิดีโอเท่านั้น)
  if(!isAudio){
    var fr = el.querySelector('.clip-frames');
    var nf = Math.max(1, Math.floor(c.w/50));
    var tv = document.createElement('video');
    tv.src=entry.url; tv.muted=true; tv.preload='metadata';
    tv.onloadedmetadata=function(){
      var drawn=0;
      for(var i=0;i<nf;i++){
        (function(idx){
          var cv=document.createElement('canvas'); cv.width=50; cv.height=36;
          var cx2=cv.getContext('2d');
          var t=(entry.dur/nf)*(idx+0.5);
          tv.currentTime=t;
          tv.onseeked=function(){
            cx2.drawImage(tv,0,0,50,36);
            var fd=document.createElement('div');
            fd.className='clip-frm'; fd.style.width='50px';
            fd.style.backgroundImage='url('+cv.toDataURL()+')';
            fr.appendChild(fd);
            drawn++; if(drawn===nf) tv.src='';
          };
        })(i);
      }
    };
  }

  // Hide audio-drop-hint when clip added
  var hint = track.querySelector('.audio-drop-hint');
  if(hint) hint.style.display='none';

  // Select clip → preview
  el.addEventListener('click', function(e){
    if(e.target.classList.contains('clip-hdl')) return;
    if(e.target.classList.contains('clip-mute')) return;
    document.querySelectorAll('.clip').forEach(function(x){x.classList.remove('sel');});
    el.classList.add('sel'); S.activeId = cid;
    if(!isAudio) loadPreview(entry);
  });

  // Apply mute state when this clip plays
  el.addEventListener('click', function(){
    if(S.activeId===cid && !isAudio) vid.muted = c.muted||false;
  });

  // Drag move
  el.addEventListener('mousedown', function(e){
    if(e.target.classList.contains('clip-hdl')) return;
    if(e.target.classList.contains('clip-mute')) return;
    var sx=e.clientX, sl=parseFloat(el.style.left);
    saveUndo();
    var mm=function(e2){
      c.left=Math.max(0,sl+e2.clientX-sx);
      c.startSec=c.left/pxSec(); // sync startSec
      el.style.left=c.left+'px';
      snapUpdateNow(); // อัปเดต marker ทันที ไม่ delay
    };
    var mu=function(){
      document.removeEventListener('mousemove',mm);
      document.removeEventListener('mouseup',mu);
      c.startSec=c.left/pxSec();
      snapUpdateNow();
    };
    document.addEventListener('mousemove',mm); document.addEventListener('mouseup',mu);
  });

  // Trim handles
  (function(){
    var initLeft = c.left;
    var initW    = c.w;
    var initTIn  = c.tIn || 0;
    el.querySelector('.clip-hdl.l').addEventListener('mousedown', function(e){
      e.stopPropagation(); e.preventDefault();
      saveUndo();
      var startX = e.clientX;
      function onMove(e2){
        var ps = pxSec();
        var totalDrag = e2.clientX - startX;
        var nr = initLeft + initW; // ขอบขวาคงที่เสมอ
        var nl = Math.max(0, Math.min(nr - 28, initLeft + totalDrag));
        var nw = Math.max(28, nr - nl);
        var newTIn = Math.max(0, Math.min((entry.dur||999)-0.1, initTIn + (nl-initLeft)/ps));
        // อัปเดต S.clips โดยตรง
        S.clips[cid].tIn = newTIn;
        S.clips[cid].w   = nw;
        S.clips[cid].left = nl;
        S.clips[cid].startSec = nl/ps;
        S.clips[cid].dur = nw/ps;
        // อัปเดต c reference ด้วย (กัน stale)
        c.tIn = newTIn; c.w = nw; c.left = nl;
        c.startSec = nl/ps; c.dur = nw/ps;
        el.style.left = nl+'px'; el.style.width = nw+'px';
      }
      function onUp(){
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', onUp);
        // อัปเดต initials สำหรับ drag ครั้งต่อไป
        initLeft = c.left; initW = c.w; initTIn = c.tIn||0;
        snapUpdateNow();
      }
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp);
    });
  })();
  mkHdl(el.querySelector('.clip-hdl.r'), function(dx){
    c.w=Math.max(28,c.w+dx); el.style.width=c.w+'px';
  });

  // Right-click context menu บน clip
  el.addEventListener('contextmenu', function(e){
    e.preventDefault();
    e.stopPropagation();
    showClipContextMenu(e.clientX, e.clientY, cid, entry, track, el, isAudio);
  });

  track.appendChild(el);
}

function showClipContextMenu(x, y, cid, entry, track, el, isAudio){
  // ลบ menu เก่า
  var old = document.getElementById('clip-ctx-menu');
  if(old) old.remove();

  var menu = document.createElement('div');
  menu.id = 'clip-ctx-menu';
  menu.style.cssText = [
    'position:fixed','left:'+x+'px','top:'+y+'px',
    'background:#1a1a1a','border:1px solid #444',
    'border-radius:8px','padding:4px 0',
    'z-index:9999','min-width:160px',
    'box-shadow:0 4px 20px rgba(0,0,0,.6)',
    'font-size:13px','color:#fff',
  ].join(';');

  function menuItem(ico, label, fn){
    var item = document.createElement('div');
    item.style.cssText = 'padding:8px 14px;cursor:pointer;display:flex;gap:8px;align-items:center;';
    item.innerHTML = '<span>'+ico+'</span><span>'+label+'</span>';
    item.addEventListener('mouseenter', function(){ item.style.background='rgba(245,197,24,.15)'; });
    item.addEventListener('mouseleave', function(){ item.style.background=''; });
    item.addEventListener('mousedown', function(e){ e.stopPropagation(); fn(); menu.remove(); });
    return item;
  }

  // Duplicate
  menu.appendChild(menuItem('📋', 'สร้างซ้ำ (Duplicate)', function(){
    var c = S.clips[cid];
    var newCid = 'c'+(S.nid++);
    var newC = Object.assign({}, c, {
      id: newCid,
      left: c.left + c.w + 4,
      startSec: (c.startSec !== undefined ? c.startSec : c.left/pxSec()) + c.dur + (4/pxSec()),
    });
    S.clips[newCid] = newC;
    buildClip(newCid, track, entry);
    scheduleSnapUpdate();
    showToast('📋 สร้างซ้ำแล้ว');
  }));

  // Split at playhead
  menu.appendChild(menuItem('✂️', 'ตัด ณ ตำแหน่งนี้', function(){
    document.getElementById('tl-spl').click();
  }));

  // Rename
  menu.appendChild(menuItem('✏️', 'เปลี่ยนชื่อ', function(){
    var newName = prompt('ชื่อใหม่:', entry.name);
    if(newName){ entry.name = newName; }
  }));

  // Separator
  var sep = document.createElement('div');
  sep.style.cssText = 'height:1px;background:#333;margin:4px 0;';
  menu.appendChild(sep);

  // Delete
  menu.appendChild(menuItem('🗑', 'ลบออก', function(){
    el.remove();
    delete S.clips[cid];
    scheduleSnapUpdate();
    showToast('🗑 ลบคลิปแล้ว');
  }));

  document.body.appendChild(menu);

  // ปิด menu เมื่อคลิกที่อื่น
  setTimeout(function(){
    document.addEventListener('mousedown', function close(e){
      if(!menu.contains(e.target)) { menu.remove(); document.removeEventListener('mousedown', close); }
    });
  }, 10);
}

function mkHdl(h, fn){
  h.addEventListener('mousedown', function(e){
    e.stopPropagation(); e.preventDefault();
    var sx = e.clientX;
    saveUndo(); // บันทึก state ก่อนลาก
    var mm = function(e2){ fn(e2.clientX - sx); sx = e2.clientX; snapUpdateNow(); };
    var mu = function(){
      document.removeEventListener('mousemove', mm);
      document.removeEventListener('mouseup', mu);
      scheduleSnapUpdate();
    };
    document.addEventListener('mousemove', mm);
    document.addEventListener('mouseup', mu);
  });
}

// ═══════════════════════════════════════
// UNDO / REDO — Ctrl+Z / Ctrl+Shift+Z
// ═══════════════════════════════════════
var undoStack = [];
var redoStack = [];

function getState(){
  // snapshot S.clips (position + size)
  var snap = {};
  Object.keys(S.clips).forEach(function(cid){
    var c = S.clips[cid];
    snap[cid] = { left: c.left, w: c.w, dur: c.dur, fid: c.fid, type: c.type, muted: c.muted };
  });
  return JSON.stringify(snap);
}

function saveUndo(){
  var st = getState();
  if(undoStack.length && undoStack[undoStack.length-1] === st) return; // ไม่บันทึกซ้ำ
  undoStack.push(st);
  if(undoStack.length > 50) undoStack.shift(); // จำกัด 50 ขั้น
  redoStack = []; // เคลียร์ redo เมื่อมี action ใหม่
}

function applyState(st){
  var snap = JSON.parse(st);
  // ลบ DOM clips ทั้งหมดก่อน
  document.querySelectorAll('.clip').forEach(function(el){ el.remove(); });
  S.clips = {};
  Object.keys(snap).forEach(function(cid){
    var s = snap[cid];
    var entry = S.files.find(function(f){ return f.id === s.fid; });
    if(!entry) return;
    S.clips[cid] = { id:cid, fid:s.fid, dur:s.dur, w:s.w, left:s.left, type:s.type||'video', muted:s.muted||false };
    var track = (s.type==='audio') ? document.getElementById('tr-a') : document.getElementById('tr-v1');
    buildClip(cid, track, entry);
  });
  drawRuler(); scheduleSnapUpdate();
}

function doUndo(){
  if(!undoStack.length){ showToast('↩ ไม่มีอะไรให้ย้อนกลับ'); return; }
  redoStack.push(getState());
  var prev = undoStack.pop();
  applyState(prev);
  showToast('↩ ย้อนกลับแล้ว (เหลือ '+undoStack.length+' ขั้น)');
}
function doRedo(){
  if(!redoStack.length){ showToast('↪ ไม่มีอะไรให้ทำซ้ำ'); return; }
  undoStack.push(getState());
  var next = redoStack.pop();
  applyState(next);
  showToast('↪ ทำซ้ำแล้ว');
}

document.getElementById('btn-undo').addEventListener('click', doUndo);
document.getElementById('btn-redo').addEventListener('click', doRedo);

// ═══════════════════════════════════════
// KEYBOARD SHORTCUTS
// ═══════════════════════════════════════
document.addEventListener('keydown', function(e){
  var tag = document.activeElement.tagName;
  if(tag==='INPUT'||tag==='TEXTAREA'||tag==='SELECT') return;
  var ctrl = e.ctrlKey || e.metaKey;

  // Space — เล่น/หยุด
  if(e.code==='Space'){ e.preventDefault(); togglePlay(); return; }

  // ← → — เดินเฟรม
  if(e.code==='ArrowLeft'){ e.preventDefault(); vid.currentTime=Math.max(0,vid.currentTime-1/30); return; }
  if(e.code==='ArrowRight'){ e.preventDefault(); vid.currentTime=Math.min(vid.duration||0,vid.currentTime+1/30); return; }

  // Ctrl+Z — Undo
  if(ctrl && !e.shiftKey && e.code==='KeyZ'){ e.preventDefault(); doUndo(); return; }

  // Ctrl+Shift+Z หรือ Ctrl+Y — Redo
  if((ctrl && e.shiftKey && e.code==='KeyZ') || (ctrl && e.code==='KeyY')){ e.preventDefault(); doRedo(); return; }

  // Delete / Backspace — ลบคลิปที่เลือก
  if(e.code==='Delete'||e.code==='Backspace'){
    if(S.activeId){
      e.preventDefault();
      saveUndo();
      var el=document.querySelector('[data-cid="'+S.activeId+'"]');
      if(el) el.remove();
      delete S.clips[S.activeId]; S.activeId=null;
      drawRuler(); scheduleSnapUpdate();
      showToast('🗑 ลบคลิปแล้ว (Ctrl+Z เพื่อยกเลิก)');
    }
    return;
  }

  // S — Split ที่ playhead
  if(e.code==='KeyS' && !ctrl){
    e.preventDefault();
    document.getElementById('tl-spl').click();
    return;
  }

  // I — ตั้ง IN
  if(e.code==='KeyI'){ document.getElementById('rp-set-in').click(); return; }
  // O — ตั้ง OUT
  if(e.code==='KeyO'){ document.getElementById('rp-set-out').click(); return; }

  // Ctrl+D — ลบคลิปที่เลือก (alternative)
  if(ctrl && e.code==='KeyD'){
    e.preventDefault();
    if(S.activeId) document.getElementById('tl-del').click();
    return;
  }

  // + / = — ซูมเข้า
  if(e.code==='Equal'||e.code==='NumpadAdd'){ setZoom(S.zoom+25); return; }
  // - — ซูมออก
  if(e.code==='Minus'||e.code==='NumpadSubtract'){ setZoom(S.zoom-25); return; }
});

// ลบคลิปที่เลือก (ปุ่ม 🗑 ใน toolbar)
document.getElementById('tl-del').addEventListener('click', function(){
  if(!S.activeId){ showToast('⚠️ เลือกคลิปก่อน'); return; }
  saveUndo();
  var el = document.querySelector('[data-cid="'+S.activeId+'"]');
  if(el) el.remove();
  delete S.clips[S.activeId]; S.activeId = null;
  drawRuler(); scheduleSnapUpdate();
  showToast('🗑 ลบแล้ว (Ctrl+Z คืนได้)');
});
// ═══════════════════════════════════════
(function(){
  var ph = document.getElementById('tl-ph');
  var sc = document.getElementById('tl-scroll');
  var ruler = document.getElementById('ruler-c');
  var isDragging = false;

  // คำนวณ globalTime จากตำแหน่ง x ใน tl-scroll
  function xToGlobalTime(clientX){
    var r = sc.getBoundingClientRect();
    var x = Math.max(0, clientX - r.left + sc.scrollLeft);
    return x / pxSec();
  }

  // seek วิดีโอ + อัปเดต playhead จาก globalTime
  function seekToGlobal(gt){
    gt = Math.max(0, gt);
    var ps = pxSec();
    ph.style.left = (gt * ps) + 'px';
    // sync audio เมื่อ seek
    window.playQueueOffset = window.playQueueOffset || 0;
    setTimeout(syncBgAudio, 50);

    buildQueue();
    for(var i=0;i<playQueue.length;i++){
      var qc = playQueueClips[i];
      if(!qc) continue;
      var clipStart = (qc.c.startSec!==undefined) ? qc.c.startSec : (qc.c.left/ps);
      var clipEnd   = clipStart + qc.c.dur;
      if(gt >= clipStart && gt < clipEnd){
        var localT = gt - clipStart;
        playQueueOffset = clipStart;
        playIdx = i;
        var qItem = playQueue[i];
        var entry = qItem.entry;

        // เปรียบเทียบ entry ที่โหลดอยู่ด้วย ID (ไม่ใช่ URL string)
        var isSameClip = (currentEntryId === entry.id) && vid.readyState >= 1;

        if(isSameClip && vid.readyState >= 1){
          // คลิปเดิม seek ตรงๆ ไม่ restart
          vid.currentTime = localT;
        } else {
          // คลิปอื่น โหลดใหม่แล้ว seek ไปตำแหน่งนั้น
          var wasPlaying = isPlaying;
          vid.pause();
          currentEntryId = entry.id;
          vid.src = entry.url;
          (function(lt, wp){
            vid.onloadedmetadata = function(){
              vid.currentTime = lt;
              var d=vid.duration;
              S.trimIn=0; S.trimOut=d; S.trimOutSet=false;
              initTrimSliders(d); updateTrimMarkers();
              if(wp){ var pb=vid.play(); if(pb) pb.catch(function(){}); }
            };
          })(localT, wasPlaying);
          vid.load();
        }
        highlightCurrentClip();
        return;
      }
    }
    // นอกช่วงคลิป — seek วิดีโอปัจจุบัน
    if(vid.readyState >= 1){
      var lt2 = gt - playQueueOffset;
      vid.currentTime = Math.max(0, Math.min(vid.duration||0, lt2));
    }
  }

  // ── DRAG PLAYHEAD ──
  ph.addEventListener('mousedown', function(e){
    e.preventDefault();
    e.stopPropagation();
    isDragging = true;
    ph.style.opacity = '0.85';
    ph.style.cursor = 'grabbing';
    function onMove(e2){
      seekToGlobal(xToGlobalTime(e2.clientX));
    }
    function onUp(){
      isDragging = false;
      ph.style.opacity = '';
      ph.style.cursor = 'ew-resize';
      document.removeEventListener('mousemove', onMove);
      document.removeEventListener('mouseup', onUp);
    }
    document.addEventListener('mousemove', onMove);
    document.addEventListener('mouseup', onUp);
  });

  // ── CLICK/DRAG RULER ──
  ruler.addEventListener('mousedown', function(e){
    e.preventDefault();
    seekToGlobal(xToGlobalTime(e.clientX));
    if(typeof syncTextToPlayhead==='function') syncTextToPlayhead();
    function onMove(e2){ seekToGlobal(xToGlobalTime(e2.clientX)); }
    function onUp(){ document.removeEventListener('mousemove', onMove); document.removeEventListener('mouseup', onUp); }
    document.addEventListener('mousemove', onMove);
    document.addEventListener('mouseup', onUp);
  });

  // ── CLICK/DRAG TRACK AREA → seek ──
  // reset isDragging เสมอเมื่อ mouseup
  document.addEventListener('mouseup', function(){ isDragging = false; });

  // bind seek บน tl-tracks และ ruler (ทั้งหมด)
  var tlInner = document.getElementById('tl-inner');
  if(tlInner){
    tlInner.addEventListener('mousedown', function(e){
      // ข้าม clip, handle, trim marker
      if(e.target.closest && (
        e.target.closest('.clip') ||
        e.target.closest('#tl-trim-in') ||
        e.target.closest('#tl-trim-out') ||
        e.target.closest('#tl-ph')
      )) return;
      // ต้องมีไฟล์
      if(!S.files || S.files.length === 0) return;
      isDragging = false;
      seekToGlobal(xToGlobalTime(e.clientX));
      function onMove(e2){ isDragging=true; seekToGlobal(xToGlobalTime(e2.clientX)); }
      function onUp(){ setTimeout(function(){isDragging=false;},50);
        document.removeEventListener('mousemove',onMove);
        document.removeEventListener('mouseup',onUp); }
      document.addEventListener('mousemove',onMove);
      document.addEventListener('mouseup',onUp);
    });
  }
})();

// TL toolbar tools
['tl-sel','tl-cut'].forEach(function(id){
  var el=document.getElementById(id);
  if(!el) return;
  el.addEventListener('click',function(){
    document.querySelectorAll('#tl-bar .tlb').forEach(function(b){b.classList.remove('on');});
    el.classList.add('on');
  });
});

// ช่องพิมพ์ข้อความด่วนใน toolbar
function addQuickText(){
  var inp=document.getElementById('tl-txt-quick');
  var txt=(inp&&inp.value.trim())||'';
  if(!txt){showToast('⚠️ พิมพ์ข้อความก่อน');return;}
  var wr=document.getElementById('prev-wrap');
  if(!wr||wr.style.display==='none'){showToast('⚠️ เปิดวิดีโอก่อน');return;}
  addTextLayer({text:txt,x:50,y:50,align:'center'});
  inp.value='';
  // สลับไปแผง text
  document.querySelector('.ib[data-p="text"]').click();
}
var _tlTxtQ=document.getElementById('tl-txt-quick');
var _tlTxtBtn=document.getElementById('tl-txt-add');
if(_tlTxtQ){ _tlTxtQ.addEventListener('keydown',function(e){ if(e.key==='Enter'){e.preventDefault();addQuickText();} }); }
if(_tlTxtBtn){ _tlTxtBtn.addEventListener('click',addQuickText); }

// ✂ กรรไกร — ตัดคลิปที่ playhead จริง
document.getElementById('tl-spl').addEventListener('click', function(){
  var currentT = vid.currentTime;
  if(!currentT || currentT <= 0){ showToast('⚠️ เลื่อน playhead ไปที่จุดที่ต้องการตัดก่อน'); return; }
  saveUndo();

  var ps  = pxSec();
  var phX = (playQueueOffset + currentT) * ps;

  // ตัดได้ทั้ง video และ audio track
  var allTracks = ['tr-v1','tr-v2','tr-a'].map(function(id){
    return document.getElementById(id);
  }).filter(Boolean);

  var splitDone = false;
  allTracks.forEach(function(track){
   var clips = Array.from(track.querySelectorAll('.clip'));
   clips.forEach(function(clipEl){
    if(splitDone) return;
    var cid    = clipEl.dataset.cid;
    var c      = S.clips[cid];
    if(!c) return;
    var cLeft  = c.left;
    var cRight = c.left + c.w;

    // playhead อยู่ภายในคลิปนี้?
    if(phX > cLeft + 4 && phX < cRight - 4){
      splitDone = true;
      var entry = S.files.find(function(f){ return f.id === c.fid; });
      if(!entry) return;

      // คำนวณเวลาแบ่ง
      var clipStartT = cLeft / ps;
      var splitLocalT= (playQueueOffset + currentT) - clipStartT;

      // คลิปซ้าย — ตัดท้ายที่ splitPoint
      var leftW = phX - cLeft;
      var leftDur = leftW / ps;
      c.w = leftW;
      clipEl.style.width = leftW + 'px';

      // คลิปขวา — ใหม่ เริ่มที่ splitPoint
      var rightCid = 'c' + S.nid++;
      var rightW   = cRight - phX;
      var rightDur = rightW / ps;
      // tIn ของ right clip = tIn ของ left clip + leftDur
      var rightTIn = (c.tIn || 0) + leftDur;
      S.clips[rightCid] = {
        id: rightCid, fid: c.fid,
        dur: rightDur, w: rightW, left: phX,
        tIn: rightTIn,
        startSec: phX / ps,
        type: c.type || 'video', muted: c.muted || false
      };
      // อัปเดต left clip tIn/dur ด้วย
      c.tIn = c.tIn || 0;
      c.dur = leftDur;
      c.startSec = cLeft / ps;
      buildClip(rightCid, track, entry); // track มาจาก allTracks loop

      // highlight flash แสดงว่าตัดแล้ว
      clipEl.style.boxShadow = '0 0 0 2px #22c55e';
      var rightEl = document.querySelector('[data-cid="'+rightCid+'"]');
      if(rightEl) rightEl.style.boxShadow = '0 0 0 2px #22c55e';
      setTimeout(function(){
        clipEl.style.boxShadow='';
        if(rightEl) rightEl.style.boxShadow='';
      }, 800);

      drawRuler();
      scheduleSnapUpdate();
      showToast('✂ ตัดคลิปที่ '+fmt(playQueueOffset+currentT)+' สำเร็จ!');
    }
   }); // clips.forEach
  }); // allTracks.forEach

  if(!splitDone){
    showToast('⚠️ playhead ไม่ได้อยู่บนคลิปใด — เลื่อนเส้นแดงไปบนคลิปก่อน');
  }
});

// ═══════════════════════════════════════
// RULER
// ═══════════════════════════════════════
function drawRuler(){
  var c=document.getElementById('ruler-c');
  var ps=pxSec();
  // คำนวณความยาวรวม inline (calcTotalDur อาจยังไม่ถูก define)
  var td=0;
  Object.values(S.clips).forEach(function(cl){var r=(cl.left/ps)+cl.dur;if(r>td)td=r;});
  td=Math.max(60,td+30);
  var w=td*ps+100;
  c.width=w;c.height=20;
  var cx=c.getContext('2d');
  cx.fillStyle='#1a1a1a';cx.fillRect(0,0,w,20);
  cx.strokeStyle='#3a3a3a';cx.lineWidth=1;
  cx.fillStyle='#666';cx.font='9px monospace';cx.textAlign='left';
  for(var t=0;t*ps<=w;t++){
    var x=t*ps;
    if(t%10===0){cx.beginPath();cx.moveTo(x,0);cx.lineTo(x,20);cx.stroke();cx.fillText(fmt(t),x+2,13);}
    else if(t%5===0){cx.beginPath();cx.moveTo(x,8);cx.lineTo(x,20);cx.stroke();}
    else{cx.beginPath();cx.moveTo(x,14);cx.lineTo(x,20);cx.stroke();}
  }
  document.querySelectorAll('.tl-track').forEach(function(t){t.style.minWidth=w+'px';});
  document.getElementById('tl-inner').style.minWidth=w+'px';
}

// ═══════════════════════════════════════
// PLAYBACK — เล่นต่อเนื่อง + Playhead ซิงค์กับไทม์ไลน์จริง
// ═══════════════════════════════════════
var playQueue=[];var playQueueClips=[];var playIdx=0;var isPlaying=false;
var playQueueOffset=0;
var transEffect='fade';
var currentEntryId=null; // track which entry is loaded in vid

// Transition canvas overlay
var transCanvas=document.createElement('canvas');
transCanvas.style.cssText='position:absolute;inset:0;pointer-events:none;z-index:10;border-radius:4px;';
document.getElementById('prev-wrap').appendChild(transCanvas);
var tctx=transCanvas.getContext('2d');
var transAnim=null;

function resizeTransCanvas(){
  var w=document.getElementById('prev-wrap');
  transCanvas.width=w.offsetWidth||480;
  transCanvas.height=w.offsetHeight||270;
}

function playTransition(cb){
  resizeTransCanvas();
  var W=transCanvas.width,H=transCanvas.height;
  var start=null,dur=500;
  if(transEffect==='none'){cb();return;}
  cancelAnimationFrame(transAnim);
  var called=false;
  function step(ts){
    if(!start)start=ts;
    var p=Math.min(1,(ts-start)/dur);
    tctx.clearRect(0,0,W,H);
    if(transEffect==='fade'){
      if(p<0.5){tctx.fillStyle='rgba(0,0,0,'+(p*2)+')';tctx.fillRect(0,0,W,H);}
      else{
        if(!called){called=true;cb();}
        tctx.fillStyle='rgba(0,0,0,'+(2-p*2)+')';tctx.fillRect(0,0,W,H);
      }
    }else if(transEffect==='wipe'){
      if(p<0.5){var x=(p*2)*W;tctx.fillStyle='#000';tctx.fillRect(0,0,x,H);}
      else{
        if(!called){called=true;cb();}
        var x2=((p-0.5)*2)*W;tctx.fillStyle='#000';tctx.fillRect(x2,0,W-x2,H);
      }
    }
    if(p<1){transAnim=requestAnimationFrame(step);}
    else{tctx.clearRect(0,0,W,H);}
  }
  transAnim=requestAnimationFrame(step);
}

// buildQueue — ใช้คลิปในไทม์ไลน์เท่านั้น ไม่ fallback ไป S.files
function buildQueue(){
  playQueue=[];
  playQueueClips=[];
  var clips=Array.from(document.getElementById('tr-v1').querySelectorAll('.clip'));
  clips.sort(function(a,b){return parseFloat(a.style.left)-parseFloat(b.style.left);});
  clips.forEach(function(el){
    var cid=el.dataset.cid;
    var c=S.clips[cid];if(!c)return;
    var entry=S.files.find(function(f){return f.id===c.fid;});
    if(entry){
      playQueue.push({entry:entry, c:c, el:el});
      playQueueClips.push({el:el,c:c});
    }
  });
  // ถ้าไม่มีคลิปใน timeline เลย ไม่เล่นอะไรทั้งนั้น
}

// คำนวณ left position ของคลิปใน timeline (pixel) → เวลาสะสม (วินาที)
function clipStartTime(idx){
  if(!playQueueClips[idx]) return 0;
  var c=playQueueClips[idx].c;
  return c.left/pxSec();
}

function togglePlay(){
  var btn=document.getElementById('pb-p');
  if(isPlaying){ vid.pause();isPlaying=false;btn.textContent='▶';btn.classList.remove('on');bgAudio.pause();return;}
  buildQueue();
  if(!playQueue.length){showToast('⚠️ ลากวิดีโอมาวางในไทม์ไลน์ก่อน');return;}

  // หาตำแหน่ง playhead ปัจจุบัน (globalTime)
  var ph = document.getElementById('tl-ph');
  var ps = pxSec();
  var currentGT = ph ? (parseFloat(ph.style.left)||0) / ps : 0;

  // หา clip ที่ playhead อยู่
  var startIdx = 0;
  for(var i=0;i<playQueue.length;i++){
    var qc = playQueueClips[i];
    if(!qc) continue;
    var clipStart = (qc.c.startSec!==undefined) ? qc.c.startSec : (qc.c.left/ps);
    var clipEnd   = clipStart + qc.c.dur;
    if(currentGT >= clipStart && currentGT < clipEnd){
      startIdx = i;
      break;
    }
    // ถ้า playhead อยู่ก่อน clip แรก
    if(currentGT < clipStart && i===0){ startIdx=0; break; }
    // ถ้าผ่าน clip นี้แล้ว ลองต่อไป
    startIdx = i;
  }

  playIdx = startIdx;
  playQueueOffset = clipStartTime(startIdx);

  // seek วิดีโอไปตำแหน่งที่ถูกต้องภายใน clip
  var qc2 = playQueueClips[startIdx];
  var seekOffset = 0;
  if(qc2){
    var cs = (qc2.c.startSec!==undefined) ? qc2.c.startSec : (qc2.c.left/ps);
    seekOffset = Math.max(0, currentGT - cs);
  }

  // โหลด clip และ seek ไปตำแหน่งที่ต้องการ
  var qItem = playQueue[startIdx];
  var entry = qItem.entry; var c = qItem.c;
  vid.src = entry.url;
  currentEntryId = entry.id;
  vid.onloadedmetadata = function(){
    var d = vid.duration;
    var filePs = pxSec();
    // tIn = tIn ของ clip (trim หน้า) + seekOffset (ตำแหน่งที่ seek ภายใน clip)
    var clipTIn = (c.tIn !== undefined) ? c.tIn : 0;
    var tIn = clipTIn + seekOffset;
    // tOut = tIn + duration ของ clip (ไม่ใช่ duration ของไฟล์)
    var clipDurSec = c.w / filePs;
    var tOut = Math.min(d, clipTIn + clipDurSec);
    tIn=Math.max(0,tIn); tOut=Math.min(d,tOut);
    S.trimIn=tIn; S.trimOut=tOut; S.trimOutSet=(tOut<d-0.3);
    initTrimSliders(d); updateTrimMarkers();
    vid.currentTime = tIn;
    vid.muted = c.muted||false;
    var pb=vid.play();
    if(pb)pb.catch(function(e){console.warn('play:',e);});
    isPlaying=true;
    document.getElementById('pb-p').textContent='⏸';
    document.getElementById('pb-p').classList.add('on');
    highlightCurrentClip();
  };
  vid.onerror=function(){showToast('❌ โหลดวิดีโอไม่ได้: '+entry.name);};
}

// loadAndPlay — เล่นคลิปตามขนาด pixel ที่ตัดไว้จริง
function loadAndPlay(qItem,immediate){
  var entry=qItem.entry, c=qItem.c;
  var doPlay=function(){
    vid.src = entry.url;
    currentEntryId = entry.id; // track loaded entry
    vid.onloadedmetadata=function(){
      var d=vid.duration;
      var ps=pxSec();
      // คำนวณ tIn, tOut จากสัดส่วน pixel ของคลิปเทียบกับไฟล์จริง
      var filePxW=entry.dur*ps;
      var clipLeft=c.left; // left ของคลิปในไทม์ไลน์
      var clipW=c.w;
      // offset ของคลิปจาก start ของไฟล์ (ถ้าลากขอบซ้าย)
      var fileOffset=0; // TODO: track per-clip trimIn separately
      var tIn=(c.tIn!==undefined) ? c.tIn : fileOffset;
      // tOut = tIn + duration ของ clip (clipW = pixel width)
      var tOut=Math.min(d, tIn+(clipW/ps));
      // clamp
      tIn=Math.max(0,tIn); tOut=Math.min(d,tOut);
      S.trimIn=tIn; S.trimOut=tOut; S.trimOutSet=(tOut<d-0.3);
      initTrimSliders(d);
      updateTrimMarkers();
      vid.currentTime=tIn;
      vid.muted=c.muted||false;
      var pb=vid.play();
      if(pb)pb.catch(function(e){console.warn('play:',e);});
      isPlaying=true;
      document.getElementById('pb-p').textContent='⏸';
      document.getElementById('pb-p').classList.add('on');
      highlightCurrentClip();
    };
    vid.onerror=function(){showToast('❌ โหลดวิดีโอไม่ได้: '+entry.name);};
  };
  if(immediate)doPlay(); else playTransition(doPlay);
}

function highlightCurrentClip(){
  document.querySelectorAll('.clip').forEach(function(el){el.classList.remove('playing');});
  if(playQueueClips[playIdx]) playQueueClips[playIdx].el.classList.add('playing');
}

vid.addEventListener('timeupdate',function(){
  var dur=vid.duration||0, t=vid.currentTime;
  if(S.trimOutSet&&S.trimOut>0&&S.trimOut<(dur-0.3)&&t>=S.trimOut){advanceClip();return;}
  var globalTime=playQueueOffset+t, ps=pxSec();
  document.getElementById('tl-ph').style.left=(globalTime*ps)+'px';
  var totalDur=calcTotalDur();
  document.getElementById('pb-tc').textContent=fmt(globalTime)+' / '+fmt(totalDur);
  document.getElementById('tc-badge').textContent=fmt(globalTime);
  var sc=document.getElementById('tl-scroll'),ph=globalTime*ps,vw=sc.clientWidth;
  if(ph>sc.scrollLeft+vw*0.8) sc.scrollLeft=ph-vw*0.3;
  else if(ph<sc.scrollLeft) sc.scrollLeft=Math.max(0,ph-50);
  // แสดง/ซ่อน text layers ตาม tIn/tOut
  updateTextVisibility(globalTime);
  // sync background audio
  syncBgAudio();
});
vid.addEventListener('ended',function(){
  // ถ้า clip สุดท้าย → หยุดพอดี ไม่ advance
  if(typeof playQueueIdx !== 'undefined' && typeof playQueue !== 'undefined'){
    if(playQueueIdx >= playQueue.length - 1){
      vid.pause(); isPlaying=false;
      var pbp=document.getElementById('pb-p');
      if(pbp){pbp.textContent='▶';pbp.classList.remove('on');}
      // เอาเข็มไปจุดสิ้นสุดจริง
      var ps=pxSec();
      var totalDur=calcTotalDur();
      var ph=document.getElementById('tl-ph');
      if(ph) ph.style.left=(totalDur*ps)+'px';
      document.getElementById('pb-tc').textContent=fmt(totalDur)+' / '+fmt(totalDur);
      return;
    }
  }
  advanceClip();
});


// แสดง/ซ่อน text overlay ตาม globalTime
function updateTextVisibility(globalTime){
  if(typeof TXT === 'undefined') return;
  TXT.layers.forEach(function(layer){
    var el = document.getElementById('tl-'+layer.id);
    if(!el) return;
    var tIn  = (layer.tIn  !== undefined) ? layer.tIn  : 0;
    var tOut = (layer.tOut !== undefined) ? layer.tOut : 9999;
    var visible = (globalTime >= tIn && globalTime < tOut);
    el.style.display = visible ? '' : 'none';
  });
}

// เรียก updateTextVisibility ตอน seek ด้วย (ไม่ใช่แค่ตอนเล่น)
function syncTextToPlayhead(){
  var ph = document.getElementById('tl-ph');
  var ps = pxSec();
  var gt = ph ? (parseFloat(ph.style.left)||0)/ps : 0;
  updateTextVisibility(gt);
}
function calcTotalDur(){
  var max=0;
  Object.values(S.clips).forEach(function(c){var ps=pxSec(),r=(c.left/ps)+c.dur;if(r>max)max=r;});
  return max||0;
}

function advanceClip(){
  if(!isPlaying)return;
  buildQueue();
  if(playQueue.length>1&&playIdx<playQueue.length-1){
    playIdx++; playQueueOffset=clipStartTime(playIdx);
    loadAndPlay(playQueue[playIdx],false); return;
  }
  cancelAnimationFrame(transAnim);
  tctx.clearRect(0,0,transCanvas.width,transCanvas.height);
  vid.pause();isPlaying=false;
  bgAudio.pause();
  document.getElementById('pb-p').textContent='▶';
  document.getElementById('pb-p').classList.remove('on');
  document.querySelectorAll('.clip').forEach(function(el){el.classList.remove('playing');});
  showToast('⏹ เล่นจบแล้ว');
}

document.getElementById('pb-p').addEventListener('click',togglePlay);
document.getElementById('pb-s').addEventListener('click',function(){ vid.currentTime=S.trimIn; updateTrimMarkers(); });
document.getElementById('pb-e').addEventListener('click',function(){ vid.currentTime=S.trimOut||vid.duration; });
document.getElementById('pb-b').addEventListener('click',function(){ vid.currentTime=Math.max(0,vid.currentTime-5); });
document.getElementById('pb-f').addEventListener('click',function(){ vid.currentTime=Math.min(vid.duration||0,vid.currentTime+5); });
document.getElementById('pb-vol').addEventListener('input',function(){
  var v=parseFloat(this.value);
  vid.volume=v;
  if(typeof bgAudio!=='undefined') bgAudio.volume=v;
  S.vol=v;
  // sync sliders
  var pct=Math.round(v*100);
  var slv=document.getElementById('sl-vol'); if(slv) slv.value=pct;
  var rpv=document.getElementById('rp-vol'); if(rpv) rpv.value=pct;
  var vv=document.getElementById('vol-v');   if(vv)  vv.textContent=pct+'%';
  var rpvv=document.getElementById('rp-vol-v'); if(rpvv) rpvv.textContent=pct+'%';
});
document.getElementById('trans-sel').addEventListener('change',function(){ transEffect=this.value; showToast('🎞 Transition: '+this.options[this.selectedIndex].text); });

// ═══════════════════════════════════════
// ZOOM — ซูมไทม์ไลน์ คลิปดันกันไม่ซ้อน
// ตำแหน่งคลิปเก็บเป็น "วินาที" (startSec)
// แปลงเป็น pixel ทุกครั้งที่ zoom เปลี่ยน
// ═══════════════════════════════════════

// migrate clip.left → clip.startSec ครั้งแรกที่โหลด
function ensureClipStartSec(){
  Object.keys(S.clips).forEach(function(cid){
    var c=S.clips[cid];
    if(c.startSec===undefined){
      c.startSec = c.left / pxSec(); // แปลง pixel → วินาที
    }
  });
}

// จัดเรียงคลิปใน track ตาม startSec แล้วดันกันไม่ซ้อน
function packClips(trackId){
  var track=document.getElementById(trackId||'tr-v1');
  if(!track) return;
  // รวบรวม cid จาก track
  var cids=[];
  track.querySelectorAll('.clip').forEach(function(el){
    var cid=el.dataset.cid;
    if(S.clips[cid]) cids.push(cid);
  });
  if(!cids.length) return;
  // เรียงตาม startSec
  cids.sort(function(a,b){ return (S.clips[a].startSec||0)-(S.clips[b].startSec||0); });
  // ดัน — ห้ามซ้อนกัน
  var cursor=0;
  cids.forEach(function(cid){
    var c=S.clips[cid];
    if(c.startSec<cursor) c.startSec=cursor;
    cursor=c.startSec+c.dur;
  });
}

function setZoom(v){
  v=Math.max(20,Math.min(800,v));
  S.zoom=v;
  ['z-lbl','tl-z-lbl'].forEach(function(id){
    var el=document.getElementById(id);if(el) el.textContent=v+'%';
  });
  var zsl=document.getElementById('z-sl'); if(zsl) zsl.value=v;
  var tlz=document.getElementById('tl-z'); if(tlz) tlz.value=v;

  // migrate ก่อน
  ensureClipStartSec();

  // อัปเดต pixel ทุกคลิป จาก startSec × pxSec
  var ps=pxSec();
  Object.keys(S.clips).forEach(function(cid){
    var c=S.clips[cid];
    if(c.startSec===undefined) c.startSec=c.left/ps;
    c.left  = c.startSec * ps;
    c.w     = c.dur * ps;
    var el=document.querySelector('[data-cid="'+cid+'"]');
    if(el){ el.style.left=c.left+'px'; el.style.width=c.w+'px'; }
  });

  drawRuler();
  updateTrimMarkers();
  if(typeof renderTextTrack==='function') renderTextTrack();
  var gt=(playQueueOffset||0)+(vid.currentTime||0);
  var ph=document.getElementById('tl-ph');
  if(ph) ph.style.left=(gt*ps)+'px';
  // อัปเดต snap markers และ thumbnail frames
  snapUpdateNow();
  refreshAllClipFrames();
  if(typeof refreshWaveClips==='function') refreshWaveClips();
}

document.getElementById('z-in').addEventListener('click',    function(){ setZoom(S.zoom+25); });
document.getElementById('z-out').addEventListener('click',   function(){ setZoom(S.zoom-25); });
document.getElementById('z-sl').addEventListener('input',    function(){ setZoom(parseInt(this.value)); });
document.getElementById('tl-z-in').addEventListener('click', function(){ setZoom(S.zoom+25); });
document.getElementById('tl-z-out').addEventListener('click',function(){ setZoom(S.zoom-25); });
document.getElementById('tl-z').addEventListener('input',    function(){ setZoom(parseInt(this.value)); });


// Redraw thumbnail frames เมื่อ zoom เปลี่ยน
function refreshAllClipFrames(){
  var ps = pxSec();
  Object.keys(S.clips).forEach(function(cid){
    var c = S.clips[cid];
    var el = document.querySelector('[data-cid="'+cid+'"]');
    if(!el) return;
    var fr = document.getElementById('cf-'+cid);
    if(!fr) return;
    // คำนวณจำนวน frame ที่ต้องการตาม width ใหม่
    var nf = Math.max(1, Math.floor(c.w / 50));
    var existing = fr.querySelectorAll('.clip-frm').length;
    if(existing === nf) return; // ไม่ต้อง redraw
    // ลบเก่า
    fr.innerHTML = '';
    // หา entry
    var entry = S.files && S.files.find(function(f){ return f.id === c.fid; });
    if(!entry || entry.type === 'audio') return;
    // วาดใหม่
    var tv = document.createElement('video');
    tv.src = entry.url; tv.muted = true; tv.preload = 'metadata';
    tv.onloadedmetadata = function(){
      var drawn = 0;
      for(var i = 0; i < nf; i++){
        (function(idx){
          var cv = document.createElement('canvas');
          cv.width = 50; cv.height = 36;
          var cx2 = cv.getContext('2d');
          var t = (entry.dur / nf) * (idx + 0.5);
          tv.currentTime = t;
          tv.onseeked = function(){
            cx2.drawImage(tv, 0, 0, 50, 36);
            var fd = document.createElement('div');
            fd.className = 'clip-frm';
            fd.style.width = '50px';
            fd.style.backgroundImage = 'url(' + cv.toDataURL() + ')';
            fr.appendChild(fd);
            drawn++;
            if(drawn === nf) tv.src = '';
          };
        })(i);
      }
    };
  });
}
// ═══════════════════════════════════════
// INNER FRAME RESIZE — คลิกภาพ → กรอบ 8 จุด ยืดหดภาพ ในเฟรม 9:16
// เฟรมไม่เปลี่ยน แต่ภาพ video ยืดหด/ขยับภายใน
// ═══════════════════════════════════════
(function(){
  var pifOn = false;
  var pifOverlay = document.getElementById('pif-overlay');
  var pifFrame   = document.getElementById('pif-frame');

  // state ตำแหน่ง/ขนาดภาพในเฟรม
  var F = { x:0, y:0, w:0, h:0 };

  function initFrame(){
    var wr = document.getElementById('prev-wrap');
    F.w = wr.offsetWidth; F.h = wr.offsetHeight;
    F.x = 0; F.y = 0;
    applyFrame();
  }

  function applyFrame(){
    pifFrame.style.left   = F.x+'px';
    pifFrame.style.top    = F.y+'px';
    pifFrame.style.width  = F.w+'px';
    pifFrame.style.height = F.h+'px';
    // expose F ให้ export ใช้
    window._pifF = {x:F.x, y:F.y, w:F.w, h:F.h};
    // ยืดภาพ video ตาม frame (object-fit: fill ให้เต็มเสมอ)
    vid.style.position    = 'absolute';
    vid.style.left        = F.x+'px';
    vid.style.top         = F.y+'px';
    vid.style.width       = F.w+'px';
    vid.style.height      = F.h+'px';
    vid.style.objectFit   = 'fill';
  }

  function openPIF(){
    pifOn = true;
    initFrame();
    pifOverlay.classList.add('on');
  }
  // hideFrame: ซ่อนกรอบเหลืองอย่างเดียว — ภาพยังอยู่ตามที่ยืดไว้
  function hideFrame(){
    pifOn = false;
    pifOverlay.classList.remove('on');
    // ไม่ reset vid.style — ภาพยังอยู่ตำแหน่งเดิม (แต่ export จะใช้ค่าจาก window._pifF)
  }
  // resetPIF: ปิดกรอบ + reset ภาพกลับเต็มเฟรม
  function resetPIF(){
    pifOn = false;
    pifOverlay.classList.remove('on');
    vid.style.position = '';
    vid.style.left = vid.style.top = '';
    vid.style.width = ''; vid.style.height = '';
    vid.style.objectFit = '';
    applyARToPreview();
  }
  function closePIF(){ hideFrame(); } // backward compat

  // คลิก video:
  // - ถ้ากรอบเปิดอยู่ → ซ่อนกรอบ (ภาพค้างไว้)
  // - ถ้ากรอบปิดอยู่และยังไม่ได้ยืด → เปิดกรอบ
  // - ดับเบิลคลิก → reset ภาพกลับ
  document.getElementById('prev-vid').style.cursor = 'pointer';
  document.getElementById('prev-vid').addEventListener('click', function(e){
    e.stopPropagation();
    if(pifOn){ hideFrame(); } else { openPIF(); }
  });
  document.getElementById('prev-vid').addEventListener('dblclick', function(e){
    e.stopPropagation();
    resetPIF();
  });

  // Escape → ซ่อนกรอบ (ภาพค้างไว้)
  document.addEventListener('keydown', function(e){
    if(e.code==='Escape' && pifOn) hideFrame();
  });

  // คลิกที่ไหนก็ได้ที่ไม่ใช่ pif-frame → ซ่อนกรอบ (ภาพค้างไว้)
  document.addEventListener('mousedown', function(e){
    if(!pifOn) return;
    if(pifFrame.contains(e.target)) return;
    hideFrame();
  }, true);

  // Drag pif-frame body → move
  pifFrame.addEventListener('mousedown', function(e){
    if(e.target !== pifFrame) return;
    e.preventDefault();
    var sx=e.clientX, sy=e.clientY, ox=F.x, oy=F.y;
    var wr = document.getElementById('prev-wrap');
    function mm(e2){
      F.x = Math.max(-F.w/2, Math.min(wr.offsetWidth-F.w/2, ox+e2.clientX-sx));
      F.y = Math.max(-F.h/2, Math.min(wr.offsetHeight-F.h/2, oy+e2.clientY-sy));
      applyFrame();
    }
    function mu(){ document.removeEventListener('mousemove',mm); document.removeEventListener('mouseup',mu); }
    document.addEventListener('mousemove',mm); document.addEventListener('mouseup',mu);
  });

  // Drag handles → resize (stretch ภาพในเฟรม)
  var hdlDefs = [
    {cls:'tl', onDrag: function(dx,dy){ F.x+=dx; F.y+=dy; F.w-=dx; F.h-=dy; }},
    {cls:'tc', onDrag: function(dx,dy){ F.y+=dy; F.h-=dy; }},
    {cls:'tr', onDrag: function(dx,dy){ F.y+=dy; F.w+=dx; F.h-=dy; }},
    {cls:'ml', onDrag: function(dx,dy){ F.x+=dx; F.w-=dx; }},
    {cls:'mr', onDrag: function(dx,dy){ F.w+=dx; }},
    {cls:'bl', onDrag: function(dx,dy){ F.x+=dx; F.w-=dx; F.h+=dy; }},
    {cls:'bc', onDrag: function(dx,dy){ F.h+=dy; }},
    {cls:'br', onDrag: function(dx,dy){ F.w+=dx; F.h+=dy; }},
  ];
  var MIN_SZ = 80;
  pifFrame.querySelectorAll('.pif-hdl').forEach(function(hdl){
    var cls = hdl.className.replace('pif-hdl ','').trim();
    var def = hdlDefs.find(function(d){ return d.cls===cls; });
    if(!def) return;
    hdl.addEventListener('mousedown', function(e){
      e.preventDefault(); e.stopPropagation();
      var sx=e.clientX, sy=e.clientY;
      var ox=F.x, oy=F.y, ow=F.w, oh=F.h;
      function mm(e2){
        F.x=ox; F.y=oy; F.w=ow; F.h=oh;
        def.onDrag(e2.clientX-sx, e2.clientY-sy);
        F.w=Math.max(MIN_SZ,F.w); F.h=Math.max(MIN_SZ,F.h);
        applyFrame();
      }
      function mu(){ document.removeEventListener('mousemove',mm); document.removeEventListener('mouseup',mu); }
      document.addEventListener('mousemove',mm); document.addEventListener('mouseup',mu);
    });
  });
})();

// ═══════════════════════════════════════
// SNAP MARKERS (+) ระหว่างคลิปชนกัน
// ═══════════════════════════════════════
// S.transitions = {clipId: effectName}  เก็บ transition ที่ใส่แต่ละรอยต่อ
S.transitions = {};

function updateSnapMarkers(){
  var track = document.getElementById('tr-v1');
  // ลบ markers เก่า
  document.querySelectorAll('.snap-marker').forEach(function(m){ m.remove(); });

  var clips = Array.from(track.querySelectorAll('.clip'));
  if(clips.length < 2) return;
  clips.sort(function(a,b){ return parseFloat(a.style.left)-parseFloat(b.style.left); });

  var ps = pxSec();

  for(var i=0;i<clips.length-1;i++){
    var a = clips[i], b = clips[i+1];
    var aRight = parseFloat(a.style.left) + parseFloat(a.style.width);
    var bLeft  = parseFloat(b.style.left);
    var gapSec = Math.abs(bLeft - aRight) / ps;

    if(gapSec < 0.15){
      // วาง marker ตรงรอยต่อ — ใช้ position ใน track โดยตรง
      var m = document.createElement('div');
      m.className = 'snap-marker';
      // left ตรงกับ aRight ซึ่งเป็น pixel ใน track
      m.style.left = aRight + 'px';
      m.dataset.atcid = a.dataset.cid;
      m.dataset.trans = S.transitions[a.dataset.cid] || '';
      if(S.transitions[a.dataset.cid]) m.classList.add('has-trans');

      var dot = document.createElement('div');
      dot.className = 'snap-dot';
      dot.textContent = S.transitions[a.dataset.cid] ? '✦' : '+';
      dot.title = S.transitions[a.dataset.cid]
        ? 'Transition: '+S.transitions[a.dataset.cid]+' (คลิกเพื่อลบ)'
        : 'คลิกเพื่อเพิ่ม Transition หรือลาก Effect มาวาง';
      m.appendChild(dot);

      // Drop
      m.addEventListener('dragover', function(e){ e.preventDefault(); m.classList.add('drop-hover'); });
      m.addEventListener('dragleave', function(){ m.classList.remove('drop-hover'); });
      m.addEventListener('drop', function(e){
        e.preventDefault(); m.classList.remove('drop-hover');
        var fx = e.dataTransfer.getData('fx-trans');
        if(!fx) return;
        S.transitions[m.dataset.atcid] = fx;
        transEffect = fx;
        document.getElementById('trans-sel').value = fx.replace('dissolve','fade');
        m.classList.add('has-trans');
        dot.textContent='✦'; dot.title='Transition: '+fx+' (คลิกเพื่อลบ)';
        showToast('✨ ใส่ '+fx+' ที่รอยต่อแล้ว');
        updateSnapMarkers();
      });

      // Click toggle
      dot.addEventListener('click', function(e){
        e.stopPropagation();
        var cid = m.dataset.atcid;
        if(S.transitions[cid]){
          delete S.transitions[cid];
          m.classList.remove('has-trans');
          dot.textContent='+';
          dot.title='คลิกเพื่อเพิ่ม Transition';
          showToast('🗑 ลบ Transition แล้ว');
        } else {
          var fx = transEffect || 'fade';
          S.transitions[cid] = fx;
          m.classList.add('has-trans');
          dot.textContent='✦';
          dot.title='Transition: '+fx+' (คลิกเพื่อลบ)';
          showToast('✨ ใส่ '+fx+' — คลิกอีกทีเพื่อลบ');
        }
        updateSnapMarkers();
      });

      // วาง marker ใน track โดยตรง
      track.appendChild(m);
    }
  }
}

// อัปเดต snap markers ทุกครั้งที่ clip ขยับ (ลด delay เหลือ 30ms)
var _snapTimer=null;
function scheduleSnapUpdate(){
  clearTimeout(_snapTimer);
  _snapTimer=setTimeout(updateSnapMarkers, 30);
}
// เรียกทันทีตอนปล่อยเมาส์
function snapUpdateNow(){ clearTimeout(_snapTimer); updateSnapMarkers(); }

// ═══════════════════════════════════════
// FX TRANSITION LIBRARY — แผงซ้าย
// ═══════════════════════════════════════
var FX_LIST = [
  {id:'fade',      name:'Fade',       desc:'ค่อยๆ เข้า/ออก', ico:'🌅'},
  {id:'wipe',      name:'Wipe',       desc:'ปัดซ้ายขวา',      ico:'➡'},
  {id:'dissolve',  name:'Dissolve',   desc:'ละลายเข้ากัน',    ico:'💧'},
  {id:'zoom',      name:'Zoom In',    desc:'ซูมเข้า',          ico:'🔍'},
  {id:'slide-up',  name:'Slide Up',   desc:'เลื่อนขึ้น',       ico:'⬆'},
  {id:'slide-dn',  name:'Slide Down', desc:'เลื่อนลง',         ico:'⬇'},
  {id:'flash',     name:'Flash',      desc:'กะพริบขาว',        ico:'⚡'},
  {id:'blur',      name:'Blur',       desc:'เบลอเปลี่ยน',      ico:'🌫'},
  {id:'spin',      name:'Spin',       desc:'หมุน',              ico:'🌀'},
  {id:'none',      name:'ตัดตรง',     desc:'ไม่มี effect',      ico:'⬛'},
];
(function(){
  var grid = document.getElementById('fx-trans-grid');
  if(!grid) return;
  FX_LIST.forEach(function(fx){
    var card = document.createElement('div');
    card.className = 'fx-trans-card';
    card.draggable = true;
    card.dataset.fx = fx.id;
    card.innerHTML =
      '<span class="fx-trans-ico">'+fx.ico+'</span>'+
      '<div class="fx-trans-name">'+fx.name+'</div>'+
      '<div class="fx-trans-desc">'+fx.desc+'</div>';
    // Drag start → set fx-trans data
    card.addEventListener('dragstart', function(e){
      e.dataTransfer.setData('fx-trans', fx.id);
      e.dataTransfer.setData('type','fx-trans');
      card.classList.add('dragging');
    });
    card.addEventListener('dragend', function(){ card.classList.remove('dragging'); });
    // Double click → apply to all joints
    card.addEventListener('dblclick', function(){
      transEffect = fx.id;
      var sel = document.getElementById('trans-sel');
      if(sel) sel.value = fx.id==='dissolve'?'fade':fx.id;
      // apply to all existing joints
      var track = document.getElementById('tr-v1');
      var clips = Array.from(track.querySelectorAll('.clip'));
      clips.sort(function(a,b){return parseFloat(a.style.left)-parseFloat(b.style.left);});
      for(var i=0;i<clips.length-1;i++){
        var aRight=parseFloat(clips[i].style.left)+parseFloat(clips[i].style.width);
        var bLeft=parseFloat(clips[i+1].style.left);
        if(Math.abs(bLeft-aRight)<8){
          if(fx.id==='none') delete S.transitions[clips[i].dataset.cid];
          else S.transitions[clips[i].dataset.cid]=fx.id;
        }
      }
      showToast('✨ ใส่ '+fx.name+' ทุกรอยต่อ');
      updateSnapMarkers();
    });
    grid.appendChild(card);
  });
})();

// ═══════════════════════════════════════
// TEXT EDITOR — เพิ่มข้อความบน preview
// ═══════════════════════════════════════
var TXT = { layers:[], nid:1, selId:null, styles:{bold:false,italic:false,bg:false,stroke:false,shadow:false}, align:'left' };

// TEXT PRESETS
var TXT_PRESETS = {
  title:   { text:'Title ใหญ่', size:52, color:'#ffffff', bold:true,  italic:false, bg:false, stroke:true,  shadow:true,  x:50, y:30, align:'center' },
  subtitle:{ text:'Subtitle', size:28, color:'#eeeeee', bold:false, italic:false, bg:false, stroke:false, shadow:true,  x:50, y:55, align:'center' },
  lower3:  { text:'ชื่อ — ตำแหน่ง', size:22, color:'#ffffff', bold:true,  italic:false, bg:true,  stroke:false, shadow:false, x:5,  y:80, align:'left'   },
  caption: { text:'คำบรรยาย', size:18, color:'#ffffff', bold:false, italic:true,  bg:false, stroke:false, shadow:true,  x:50, y:90, align:'center' },
};

document.querySelectorAll('.txt-preset-btn').forEach(function(btn){
  // คลิก → เพิ่มที่ playhead ทันที
  btn.addEventListener('click', function(){
    var p = TXT_PRESETS[btn.dataset.preset];
    if(!p) return;
    applyPresetUI(p);
    addTextLayer(p);
  });

  // ทำให้ drag ได้
  btn.setAttribute('draggable', 'true');
  btn.style.cursor = 'grab';

  btn.addEventListener('dragstart', function(e){
    e.dataTransfer.setData('text/plain', btn.dataset.preset);
    e.dataTransfer.effectAllowed = 'copy';
    btn.style.opacity = '0.5';
  });
  btn.addEventListener('dragend', function(){
    btn.style.opacity = '1';
  });
});

function applyPresetUI(p){
  document.getElementById('txt-input').value = p.text;
  document.getElementById('txt-size').value  = p.size;
  document.getElementById('txt-size-v').textContent = p.size;
  document.getElementById('txt-color').value = p.color;
  TXT.styles.bold   = p.bold;   TXT.styles.italic = p.italic;
  TXT.styles.bg     = p.bg;     TXT.styles.stroke = p.stroke;
  TXT.styles.shadow = p.shadow; TXT.align = p.align;
  updateStyleBtns(); updateAlignBtns();
}

// Drop preset บน timeline → เพิ่ม text layer ที่เวลาที่ drop
(function(){
  var tlInner = document.getElementById('tl-inner');
  if(!tlInner) return;

  tlInner.addEventListener('dragover', function(e){
    if(!e.dataTransfer.types.includes('text/plain')) return;
    e.preventDefault();
    e.dataTransfer.dropEffect = 'copy';
    // แสดง ghost line
    var sc = document.getElementById('tl-scroll');
    var r = sc.getBoundingClientRect();
    var x = e.clientX - r.left + sc.scrollLeft;
    var ghost = document.getElementById('txt-drop-ghost');
    if(!ghost){
      ghost = document.createElement('div');
      ghost.id = 'txt-drop-ghost';
      ghost.style.cssText = 'position:absolute;top:0;width:2px;height:100%;background:var(--acc);pointer-events:none;z-index:99;opacity:.7;';
      tlInner.appendChild(ghost);
    }
    ghost.style.left = x + 'px';
  });

  tlInner.addEventListener('dragleave', function(e){
    var ghost = document.getElementById('txt-drop-ghost');
    if(ghost) ghost.remove();
  });

  tlInner.addEventListener('drop', function(e){
    e.preventDefault();
    var presetKey = e.dataTransfer.getData('text/plain');
    var p = TXT_PRESETS[presetKey];
    if(!p) return;

    // คำนวณเวลาจากตำแหน่ง drop
    var sc = document.getElementById('tl-scroll');
    var r = sc.getBoundingClientRect();
    var x = e.clientX - r.left + sc.scrollLeft;
    var dropTime = Math.max(0, x / pxSec());

    // ลบ ghost
    var ghost = document.getElementById('txt-drop-ghost');
    if(ghost) ghost.remove();

    // apply UI settings
    applyPresetUI(p);

    // เพิ่ม layer ที่เวลา dropTime — seek playhead ไปด้วย
    var ph = document.getElementById('tl-ph');
    if(ph) ph.style.left = (dropTime * pxSec()) + 'px';
    var pClone = Object.assign({}, p, { tIn: dropTime, tOut: dropTime + 3 });
    addTextLayer(pClone);
    showToast('✅ วางข้อความที่ ' + dropTime.toFixed(1) + 's — ลากขอบขวาเพื่อยืดระยะ');
  });
})();

// Style toggle buttons
document.querySelectorAll('.txt-style-btn').forEach(function(btn){
  btn.addEventListener('click', function(){
    var s = btn.dataset.style;
    TXT.styles[s] = !TXT.styles[s];
    btn.classList.toggle('on', TXT.styles[s]);
    if(TXT.selId) updateSelectedLayer();
  });
});
function updateStyleBtns(){
  ['bold','italic','bg','stroke','shadow'].forEach(function(s){
    var btn = document.getElementById('ts-'+s);
    if(btn) btn.classList.toggle('on', TXT.styles[s]);
  });
}

// Align buttons
document.querySelectorAll('.txt-align-btn').forEach(function(btn){
  btn.addEventListener('click', function(){
    document.querySelectorAll('.txt-align-btn').forEach(function(b){b.classList.remove('on');});
    btn.classList.add('on'); TXT.align = btn.dataset.align;
    if(TXT.selId) updateSelectedLayer();
  });
});
function updateAlignBtns(){
  document.querySelectorAll('.txt-align-btn').forEach(function(b){
    b.classList.toggle('on', b.dataset.align === TXT.align);
  });
}

// Size / Alpha live update
document.getElementById('txt-size').addEventListener('input', function(){
  document.getElementById('txt-size-v').textContent = this.value;
  if(TXT.selId) updateSelectedLayer();
});
document.getElementById('txt-alpha').addEventListener('input', function(){
  document.getElementById('txt-alpha-v').textContent = this.value+'%';
  if(TXT.selId) updateSelectedLayer();
});
['txt-color','txt-bg','txt-stroke'].forEach(function(id){
  document.getElementById(id).addEventListener('input', function(){
    if(TXT.selId) updateSelectedLayer();
  });
});
document.getElementById('txt-input').addEventListener('input', function(){
  if(TXT.selId) updateSelectedLayer();
});
document.getElementById('txt-font').addEventListener('change', function(){
  if(TXT.selId) updateSelectedLayer();
});

// Load font from device
document.getElementById('btn-load-font').addEventListener('click', function(){
  document.getElementById('fi-font').click();
});
document.getElementById('fi-font').addEventListener('change', function(){
  Array.from(this.files).forEach(function(f){
    var url = URL.createObjectURL(f);
    var name = f.name.replace(/\.[^.]+$/,'').replace(/[-_]/g,' ');
    var face = new FontFace(name, 'url('+url+')');
    face.load().then(function(loaded){
      document.fonts.add(loaded);
      // Add to select
      var opt = document.createElement('option');
      opt.value = name+',sans-serif'; opt.textContent = '📂 '+name;
      document.getElementById('txt-font').appendChild(opt);
      document.getElementById('txt-font').value = name+',sans-serif';
      // Show badge
      var badge = document.createElement('div');
      badge.className = 'font-badge on'; badge.textContent = name;
      document.getElementById('loaded-fonts').appendChild(badge);
      showToast('✅ โหลดฟอนต์ '+name+' แล้ว!');
    }).catch(function(){ showToast('❌ โหลดฟอนต์ไม่ได้: '+f.name); });
  });
});

// ADD TEXT
document.getElementById('btn-add-text').addEventListener('click', function(){
  var txt = document.getElementById('txt-input').value.trim();
  if(!txt){ showToast('⚠️ กรุณาพิมพ์ข้อความก่อน'); return; }
  var wr = document.getElementById('prev-wrap');
  if(!wr || wr.style.display==='none'){ showToast('⚠️ เปิดวิดีโอก่อน'); return; }
  addTextLayer({ text:txt, x:50, y:50, align:TXT.align });
});

function addTextLayer(preset){
  var wr = document.getElementById('prev-wrap');
  var W = wr.offsetWidth, H = wr.offsetHeight;
  var id = 'txt'+TXT.nid++;
  var txt = preset.text || document.getElementById('txt-input').value || 'ข้อความ';
  var size = preset.size || parseInt(document.getElementById('txt-size').value);
  var color = preset.color || document.getElementById('txt-color').value;
  var fontFam = document.getElementById('txt-font').value;
  var alpha = parseInt(document.getElementById('txt-alpha').value)/100;
  var bold   = (preset.bold!==undefined)   ? preset.bold   : TXT.styles.bold;
  var italic = (preset.italic!==undefined) ? preset.italic : TXT.styles.italic;
  var useBg  = (preset.bg!==undefined)     ? preset.bg     : TXT.styles.bg;
  var stroke = (preset.stroke!==undefined) ? preset.stroke : TXT.styles.stroke;
  var shadow = (preset.shadow!==undefined) ? preset.shadow : TXT.styles.shadow;
  var bgColor = document.getElementById('txt-bg').value;
  var strokeColor = document.getElementById('txt-stroke').value;
  var align  = preset.align || TXT.align;
  // position %
  var xp = (preset.x!==undefined) ? preset.x : 50;
  var yp = (preset.y!==undefined) ? preset.y : 50;
  var x = Math.floor((xp/100)*W);
  var y = Math.floor((yp/100)*H);

  // tIn/tOut = เวลา global ที่ text จะปรากฏ (default = playhead ถึง +3 วินาที)
  var ph = document.getElementById('tl-ph');
  var ps0 = pxSec();
  var gtNow = ph ? (parseFloat(ph.style.left)||0)/ps0 : 0;
  var layerTIn  = preset.tIn  !== undefined ? preset.tIn  : gtNow;
  var layerTOut = preset.tOut !== undefined ? preset.tOut : gtNow + 3;

  var layer = { id:id, text:txt, x:x, y:y, size:size, color:color, fontFam:fontFam,
                alpha:alpha, bold:bold, italic:italic, bg:useBg, bgColor:bgColor,
                stroke:stroke, strokeColor:strokeColor, shadow:shadow, align:align,
                w:0, h:0, tIn:layerTIn, tOut:layerTOut };
  TXT.layers.push(layer);
  renderTextLayer(layer);
  renderTextLayerList();
  renderTextTrack();
  selectTextLayer(id);
  showToast('✅ เพิ่มข้อความแล้ว ลากเพื่อย้ายตำแหน่ง');
}

function renderTextLayer(layer){
  var container = document.getElementById('txt-overlay-container');
  // remove old
  var old = document.getElementById('tl-'+layer.id);
  if(old) old.remove();

  var el = document.createElement('div');
  el.id = 'tl-'+layer.id;
  el.className = 'txt-overlay';
  el.dataset.tid = layer.id;
  el.contentEditable = false;
  el.textContent = layer.text;

  applyLayerStyle(el, layer);
  container.style.pointerEvents = 'all';

  // Drag to move
  el.addEventListener('mousedown', function(e){
    if(e.target.classList.contains('txt-hdl')) return;
    e.preventDefault(); e.stopPropagation();
    selectTextLayer(layer.id);
    var sx=e.clientX, sy=e.clientY, ox=layer.x, oy=layer.y;
    var wr=document.getElementById('prev-wrap');
    function mm(e2){
      layer.x=Math.max(0,Math.min(wr.offsetWidth-30,ox+e2.clientX-sx));
      layer.y=Math.max(0,Math.min(wr.offsetHeight-10,oy+e2.clientY-sy));
      el.style.left=layer.x+'px'; el.style.top=layer.y+'px';
    }
    function mu(){document.removeEventListener('mousemove',mm);document.removeEventListener('mouseup',mu);}
    document.addEventListener('mousemove',mm); document.addEventListener('mouseup',mu);
  });

  // Double click = edit text
  el.addEventListener('dblclick', function(e){
    e.stopPropagation();
    el.contentEditable = true;
    el.focus();
    el.style.cursor='text';
  });
  el.addEventListener('blur', function(){
    el.contentEditable=false; el.style.cursor='move';
    layer.text = el.textContent;
    renderTextLayerList();
  });

  container.appendChild(el);
}

function applyLayerStyle(el, layer){
  var alpha = layer.alpha||1;
  el.style.cssText = [
    'position:absolute',
    'left:'+layer.x+'px',
    'top:'+layer.y+'px',
    'font-size:'+layer.size+'px',
    'font-family:'+layer.fontFam,
    'color:'+hexToRgba(layer.color,alpha),
    'font-weight:'+(layer.bold?'bold':'normal'),
    'font-style:'+(layer.italic?'italic':'normal'),
    'text-align:'+layer.align,
    'background:'+(layer.bg?hexToRgba(layer.bgColor,0.7):'transparent'),
    'padding:'+(layer.bg?'3px 8px':'0'),
    'border-radius:'+(layer.bg?'4px':'0'),
    'text-shadow:'+(layer.shadow?'2px 2px 4px rgba(0,0,0,0.8),0 0 8px rgba(0,0,0,0.5)':'none'),
    '-webkit-text-stroke:'+(layer.stroke?'1px '+layer.strokeColor:'0'),
    'cursor:move',
    'z-index:18',
    'min-width:30px',
    'line-height:1.3',
    'user-select:none',
  ].join(';');
  // border for selection
  if(TXT.selId===layer.id) el.style.border='1.5px solid #f5c518';
  else el.style.border='1.5px solid transparent';
}

function selectTextLayer(id){
  TXT.selId = id;
  var layer = TXT.layers.find(function(l){return l.id===id;});
  if(!layer) return;
  // Update UI to match layer
  document.getElementById('txt-input').value = layer.text;
  document.getElementById('txt-size').value  = layer.size;
  document.getElementById('txt-size-v').textContent = layer.size;
  document.getElementById('txt-color').value = layer.color;
  document.getElementById('txt-font').value  = layer.fontFam;
  document.getElementById('txt-alpha').value = Math.round((layer.alpha||1)*100);
  document.getElementById('txt-alpha-v').textContent = Math.round((layer.alpha||1)*100)+'%';
  document.getElementById('txt-bg').value    = layer.bgColor||'#000000';
  document.getElementById('txt-stroke').value= layer.strokeColor||'#000000';
  TXT.styles.bold=layer.bold; TXT.styles.italic=layer.italic;
  TXT.styles.bg=layer.bg; TXT.styles.stroke=layer.stroke; TXT.styles.shadow=layer.shadow;
  TXT.align=layer.align;
  updateStyleBtns(); updateAlignBtns();
  // highlight
  TXT.layers.forEach(function(l){
    var el=document.getElementById('tl-'+l.id);
    if(el) applyLayerStyle(el,l);
  });
  renderTextLayerList();
}

// === TEXT LAYER: คลิก preview เพิ่ม / Delete ลบ / deselect ===
(function(){
  function isTextPanelActive(){
    // เช็กว่า panel ข้อความเปิดอยู่
    var p = document.getElementById('p-text');
    return p && (p.style.display === 'flex' || p.style.display === 'block');
  }

  function bindTextActions(){
    var wrap = document.getElementById('prev-wrap');
    if(!wrap){ setTimeout(bindTextActions, 500); return; }

    wrap.addEventListener('mousedown', function(e){
      // ถ้าคลิกบน text overlay → select (จัดการโดย renderTextLayer แล้ว)
      if(e.target.closest && e.target.closest('.txt-overlay')) return;
      // ถ้า pif หรือ crop active → skip
      if(typeof S !== 'undefined' && S.cropActive) return;
      var pifOv = document.getElementById('pif-overlay');
      if(pifOv && pifOv.classList.contains('on')) return;

      if(isTextPanelActive()){
        // อยู่ใน text mode → คลิก preview = เพิ่ม text layer ที่ตำแหน่งนั้น
        var rect = wrap.getBoundingClientRect();
        var xPx = e.clientX - rect.left;
        var yPx = e.clientY - rect.top;
        var xPct = (xPx / wrap.offsetWidth) * 100;
        var yPct = (yPx / wrap.offsetHeight) * 100;
        if(typeof addTextLayer === 'function'){
          addTextLayer({ x: xPct, y: yPct });
        }
      } else {
        // ไม่ใช่ text mode → deselect
        if(typeof TXT !== 'undefined' && TXT.selId){
          TXT.selId = null;
          document.querySelectorAll('.txt-overlay').forEach(function(el){
            el.style.border = '1.5px solid transparent';
          });
          if(typeof renderTextLayerList === 'function') renderTextLayerList();
        }
      }
    });
  }
  bindTextActions();

  // Delete / Backspace → ลบ text layer ที่เลือกอยู่
  document.addEventListener('keydown', function(e){
    if(e.key !== 'Delete' && e.key !== 'Backspace') return;
    // ถ้า focus อยู่ใน input/textarea → ไม่ลบ
    var tag = document.activeElement && document.activeElement.tagName;
    if(tag === 'INPUT' || tag === 'TEXTAREA') return;
    if(typeof TXT !== 'undefined' && TXT.selId){
      e.preventDefault();
      if(typeof deleteTextLayer === 'function') deleteTextLayer(TXT.selId);
    }
  });
})();


function updateSelectedLayer(){
  if(!TXT.selId) return;
  var layer = TXT.layers.find(function(l){return l.id===TXT.selId;});
  if(!layer) return;
  layer.text        = document.getElementById('txt-input').value;
  layer.size        = parseInt(document.getElementById('txt-size').value);
  layer.color       = document.getElementById('txt-color').value;
  layer.fontFam     = document.getElementById('txt-font').value;
  layer.alpha       = parseInt(document.getElementById('txt-alpha').value)/100;
  layer.bgColor     = document.getElementById('txt-bg').value;
  layer.strokeColor = document.getElementById('txt-stroke').value;
  layer.bold=TXT.styles.bold; layer.italic=TXT.styles.italic;
  layer.bg=TXT.styles.bg; layer.stroke=TXT.styles.stroke; layer.shadow=TXT.styles.shadow;
  layer.align=TXT.align;
  var el = document.getElementById('tl-'+layer.id);
  if(el){ el.textContent=layer.text; applyLayerStyle(el,layer); }
}

function renderTextLayerList(){
  var list = document.getElementById('txt-layers');
  list.innerHTML='';
  TXT.layers.forEach(function(layer){
    var item = document.createElement('div');
    item.className = 'txt-layer-item'+(TXT.selId===layer.id?' on':'');
    item.innerHTML =
      '<span style="font-size:13px;">T</span>'+
      '<span class="txt-layer-preview" style="font-size:11px;color:var(--tx);">'+layer.text.substring(0,28)+'</span>'+
      '<button class="txt-layer-del" data-tid="'+layer.id+'">✕</button>';
    item.addEventListener('click', function(e){
      if(e.target.dataset.tid) return;
      selectTextLayer(layer.id);
    });
    item.querySelector('.txt-layer-del').addEventListener('click', function(e){
      e.stopPropagation();
      deleteTextLayer(layer.id);
    });
    list.appendChild(item);
  });
}

function deleteTextLayer(id){
  var el = document.getElementById('tl-'+id);
  if(el) el.remove();
  TXT.layers = TXT.layers.filter(function(l){return l.id!==id;});
  if(TXT.selId===id) TXT.selId=null;
  renderTextLayerList();
  renderTextTrack();
  showToast('🗑 ลบข้อความแล้ว');
}


// ─── TEXT TRACK — แสดง text layers บน timeline ───
function renderTextTrack(){
  var track = document.getElementById('tr-t') || document.getElementById('tr-f');
  if(!track) return;
  // ลบ text clip เก่าออก
  track.querySelectorAll('.txt-tl-clip').forEach(function(el){ el.remove(); });
  var ps = pxSec();
  TXT.layers.forEach(function(layer){
    var tIn  = layer.tIn  || 0;
    var tOut = layer.tOut || (tIn + 3);
    var el = document.createElement('div');
    el.className = 'txt-tl-clip';
    el.dataset.lid = layer.id;
    el.style.cssText = [
      'position:absolute',
      'left:'+(tIn*ps)+'px',
      'width:'+Math.max(20,(tOut-tIn)*ps)+'px',
      'top:3px','height:calc(100% - 6px)',
      'background:rgba(245,197,24,0.35)',
      'border:1.5px solid var(--acc)',
      'border-radius:4px',
      'cursor:pointer',
      'display:flex','align-items:center',
      'padding:0 6px',
      'font-size:10px','color:#fff',
      'overflow:hidden','white-space:nowrap',
      'user-select:none','box-sizing:border-box',
    ].join(';');
    el.textContent = '✏ '+layer.text.substring(0,20);

    // คลิก → select layer
    el.addEventListener('mousedown', function(e){
      e.stopPropagation();
      if(typeof selectTextLayer === 'function') selectTextLayer(layer.id);
    });

    // ลาก clip ซ้าย-ขวา เพื่อย้ายเวลา
    el.addEventListener('mousedown', function(e){
      if(e.target.classList.contains('txt-tl-resize')) return;
      var startX = e.clientX;
      var startTIn = layer.tIn || 0;
      var dur = (layer.tOut||0) - startTIn;
      function onMove(e2){
        var dx = e2.clientX - startX;
        var dt = dx / pxSec();
        layer.tIn  = Math.max(0, startTIn + dt);
        layer.tOut = layer.tIn + dur;
        el.style.left = (layer.tIn * pxSec()) + 'px';
      }
      function onUp(){
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', onUp);
        renderTextTrack();
      }
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp);
    });

    // handle ขวา = resize tOut
    var rhdl = document.createElement('div');
    rhdl.className = 'txt-tl-resize';
    rhdl.style.cssText = 'position:absolute;right:0;top:0;width:8px;height:100%;cursor:ew-resize;background:rgba(255,255,255,0.2);';
    rhdl.addEventListener('mousedown', function(e){
      e.stopPropagation();
      var startX = e.clientX;
      var startTOut = layer.tOut || (layer.tIn + 3);
      function onMove(e2){
        var dx = e2.clientX - startX;
        var dt = dx / pxSec();
        layer.tOut = Math.max(layer.tIn + 0.2, startTOut + dt);
        el.style.width = Math.max(20,(layer.tOut - layer.tIn)*pxSec())+'px';
      }
      function onUp(){
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', onUp);
        renderTextTrack();
      }
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp);
    });
    el.appendChild(rhdl);

    // ลาก clip ข้าม track (vertical) — track เปลี่ยน tIn offset
    el.addEventListener('mousedown', function(e){
      if(e.target === rhdl) return;
      e.stopPropagation();
      selectTextLayer(layer.id);
      var startX = e.clientX;
      var startTIn = layer.tIn || 0;
      var dur = (layer.tOut || startTIn+3) - startTIn;
      var moved = false;
      function onMove(e2){
        moved = true;
        var dx = e2.clientX - startX;
        var dt = dx / pxSec();
        layer.tIn  = Math.max(0, startTIn + dt);
        layer.tOut = layer.tIn + dur;
        el.style.left = (layer.tIn * pxSec()) + 'px';
      }
      function onUp(){
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', onUp);
        if(moved) renderTextTrack();
      }
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp);
    });

    track.appendChild(el);
  });
}
function hexToRgba(hex,alpha){
  try{
    var r=parseInt(hex.slice(1,3),16);
    var g=parseInt(hex.slice(3,5),16);
    var b=parseInt(hex.slice(5,7),16);
    return 'rgba('+r+','+g+','+b+','+(alpha||1)+')';
  }catch(e){return hex;}
}
function openExp(){document.getElementById('exp-bg').classList.add('show');}
document.getElementById('btn-exp').addEventListener('click',openExp);
document.getElementById('exp-cancel').addEventListener('click',function(){document.getElementById('exp-bg').classList.remove('show');});
document.getElementById('exp-bg').addEventListener('click',function(e){if(e.target===this)this.classList.remove('show');});
document.querySelectorAll('.em-ar').forEach(function(el){
  el.addEventListener('click',function(){
    document.querySelectorAll('.em-ar').forEach(function(o){o.classList.remove('on');});
    el.classList.add('on');
  });
});
document.querySelectorAll('.em-res').forEach(function(b){
  b.addEventListener('click',function(){
    document.querySelectorAll('.em-res').forEach(function(x){x.classList.remove('on');});
    b.classList.add('on');S.expRes=b.dataset.eres;
  });
});
document.getElementById('btn-share').addEventListener('click',function(){
  var url=window.location.href.split('?')[0];
  if(navigator.clipboard){navigator.clipboard.writeText(url).then(function(){showToast('🔗 คัดลอกลิ้งแล้ว!');});}
  else{prompt('คัดลอกลิ้ง:',url);}
});
document.getElementById('btn-undo').addEventListener('click',function(){showToast('↩ ย้อนกลับ');});
document.getElementById('btn-redo').addEventListener('click',function(){showToast('↪ ทำซ้ำ');});

// ═══════════════════════════════════════
// EXPORT — รวมทุกคลิปในไทม์ไลน์แล้วดาวน์โหลด
// ═══════════════════════════════════════
document.getElementById('exp-go').addEventListener('click', async function(){
  buildQueue();
  if(!playQueue.length){ showToast('⚠️ ลากวิดีโอมาใส่ไทม์ไลน์ก่อน'); return; }
  var btn=this; btn.disabled=true; btn.textContent='⏳ กำลังโหลด FFmpeg...';
  var ok=await loadFFmpeg();
  if(!ok){ btn.disabled=false; btn.textContent='🚀 เริ่มรวมและส่งออก'; return; }

  var epw=document.getElementById('ep-wrap'); epw.style.display='block';
  var epf=document.getElementById('ep-fill'); epf.style.width='0';
  var eps=document.getElementById('ep-stat');
  var dl=document.getElementById('exp-dl');
  dl.style.display='none';
  btn.textContent='⏳ กำลังรวมวิดีโอ...';

  var steps=document.getElementById('exp-steps');
  var stepList=document.getElementById('exp-step-list');
  steps.style.display='block'; stepList.innerHTML='';
  playQueue.forEach(function(qItem,i){
    var div=document.createElement('div');
    div.id='exp-step-'+i;
    div.style.cssText='padding:4px 8px;background:var(--p2);border-radius:5px;font-size:11px;color:var(--tx2);border:1px solid var(--bd2);';
    div.textContent='⏳ ('+(i+1)+') '+qItem.entry.name;
    stepList.appendChild(div);
  });

  var fmtV=document.getElementById('exp-fmt').value;
  var isAudioOnly=(fmtV==='mp3'||fmtV==='aac');
  var outExt=fmtV==='webm'?'webm':fmtV==='mp3'?'mp3':fmtV==='aac'?'aac':'mp4';
  var crf=document.getElementById('exp-q').value;
  var resEl=document.querySelector('.em-res.on');
  var res=resEl?resEl.dataset.eres:'1280x720';
  var earEl=document.querySelector('.em-ar.on');
  var ear=earEl?earEl.dataset.ear:'16:9';
  var cropMap={'9:16':'scale=ih*9/16:ih,crop=ih*9/16:ih','1:1':'crop=min(iw\\,ih):min(iw\\,ih)','4:3':'scale=iw:iw*3/4,crop=iw:iw*3/4','4:5':'scale=iw:iw*5/4,crop=iw:iw*5/4'};

  var allSegData = []; // เก็บ ArrayBuffer ของแต่ละ seg

  try{
    for(var i=0;i<playQueue.length;i++){
      var qItem=playQueue[i];
      var entry=qItem.entry; var c=qItem.c;
      var stepEl=document.getElementById('exp-step-'+i);
      if(stepEl){ stepEl.style.color='var(--acc)'; stepEl.textContent='⚙️ ('+(i+1)+'/'+playQueue.length+') '+entry.name; }

      epf.style.width=Math.round((i/playQueue.length)*80)+'%';
      eps.textContent='⚙️ encode ('+(i+1)+'/'+playQueue.length+'): '+entry.name;

      var srcExt=entry.file.name.split('.').pop().toLowerCase()||'mp4';
      var inN='cin_'+i+'.'+srcExt;
      var segN='seg_'+i+'.mp4';

      var ps=pxSec();
      var totalFilePx = entry.dur * ps;
      var tIn  = (c.tIn  !== undefined) ? c.tIn  : 0;
      var tOut = (c.tOut !== undefined) ? c.tOut : entry.dur;
      if(tIn === 0 && tOut === entry.dur && c.w < totalFilePx - 1){
        var ratio = c.w / totalFilePx;
        tOut = Math.min(entry.dur, tIn + entry.dur * ratio);
      }
      var clipDurSec = Math.max(0.1, tOut - tIn);

      var args = [];
      if(tIn > 0.05) args.push('-ss', tIn.toFixed(3));
      args.push('-i', inN);
      args.push('-t', clipDurSec.toFixed(3));

      if(!isAudioOnly){
        var resParts = (res||'1280x720').split('x');
        // data-eres เก็บเป็น 16:9 เสมอ (เช่น 1920x1080)
        // ถ้า ear เป็น 9:16 หรือ portrait ต้อง swap width↔height
        var resW = parseInt(resParts[0])||1280;
        var resH = parseInt(resParts[1])||720;
        var tw, th;
        if(ear === '9:16' || ear === '4:5'){
          // portrait: swap → width=resH, height=resW
          tw = resH; th = resW;  // เช่น 1080x1920
        } else {
          tw = resW; th = resH;  // 1920x1080
        }

        // scale=tw:th ตรงๆ ไม่ใช้ expression — รองรับ ffmpeg.wasm v0.11
        var vf = 'scale='+tw+':'+th+',setsar=1';

        // PiP (Picture-in-Frame): ถ้าภาพถูกยืด/ย้ายใน prev-wrap → burn ลง export
        // F เก็บ x,y,w,h ใน pixel ของ prev-wrap (ขนาด = wr.offsetWidth x wr.offsetHeight)
        var wr = document.getElementById('prev-wrap');
        var wrW = wr ? wr.offsetWidth : tw;
        var wrH = wr ? wr.offsetHeight : th;
        // ดึง F จาก pifFrame IIFE — expose ออกมาผ่าน window
        var pifF = window._pifF;
        if(pifF && pifF.w > 0 && wrW > 0 && (
          Math.abs(pifF.x) > 2 || Math.abs(pifF.y) > 2 ||
          Math.abs(pifF.w - wrW) > 2 || Math.abs(pifF.h - wrH) > 2
        )){
          // แปลง pixel → ตำแหน่งใน output frame
          var pifX = Math.round(pifF.x / wrW * tw);
          var pifY = Math.round(pifF.y / wrH * th);
          var pifW = Math.round(pifF.w / wrW * tw);
          var pifH = Math.round(pifF.h / wrH * th);
          // ทำให้ even
          if(pifW%2!==0) pifW--;
          if(pifH%2!==0) pifH--;
          if(pifX < 0){
            // ภาพขยายออกนอก frame ซ้าย → crop ซ้าย
            var cropX = Math.round(Math.abs(pifX) / pifW * tw);
            vf = 'scale='+pifW+':'+pifH+',crop='+tw+':'+th+':'+Math.max(0,cropX)+':'+Math.max(0,Math.round(-pifY/pifH*th))+',setsar=1';
          } else if(pifW < tw || pifH < th){
            // ภาพเล็กกว่า frame → scale เล็กแล้ว pad (letterbox style)
            vf = 'scale='+pifW+':'+pifH+',pad='+tw+':'+th+':'+Math.max(0,pifX)+':'+Math.max(0,pifY)+':black,setsar=1';
          } else {
            // ภาพใหญ่กว่า frame → crop ตรงกลาง
            vf = 'scale='+pifW+':'+pifH+',crop='+tw+':'+th+':'+Math.max(0,-pifX)+':'+Math.max(0,-pifY)+',setsar=1';
          }
        }


        args.push('-vf', vf);
        args.push('-c:v','libx264',
                  '-crf', String(crf),
                  '-preset','ultrafast',
                  '-pix_fmt','yuv420p');
        // ถ้า clip ปิดเสียงต้นฉบับ → ไม่เอาเสียง video
        if(c.muted){
          args.push('-an');
        } else {
          args.push('-c:a','aac','-b:a','128k');
        }
      } else {
        if(fmtV==='mp3') args.push('-vn','-acodec','libmp3lame','-q:a','2');
        else args.push('-vn','-acodec','aac','-b:a','192k');
      }
      args.push(segN);
      await ffWrite(inN, entry.file);
      eps.textContent='⚙️ กำลัง encode ('+(i+1)+'/'+playQueue.length+'): '+entry.name+' ...';
      epf.style.width=Math.round((i/playQueue.length)*80)+'%';

      _ffmpegLib.setProgress(function(p){
        var pct = Math.round((p.ratio||0)*100);
        epf.style.width = Math.round((i/playQueue.length)*80 + pct*0.8/playQueue.length)+'%';
        eps.textContent = '⚙️ encode ('+(i+1)+'/'+playQueue.length+') '+pct+'%: '+entry.name;
      });

      await ffExec(args);
      _ffmpegLib.setProgress(function(){});

      var segData;
      try{ segData = _ffmpegLib.FS('readFile', segN); }
      catch(fsErr){ throw new Error('FS readFile ล้มเหลว ('+segN+'): '+fsErr.message); }
      if(!segData || segData.length===0){
        // ลองดูว่า ffmpeg log มีอะไร
        throw new Error('encode ได้ไฟล์ว่าง — ตรวจสอบว่าวิดีโอไม่เสียหาย หรือลองเปลี่ยน format เป็น MP4');
      }
      allSegData.push(segData.buffer);
      // cleanup waveform overlay file
      try{ _ffmpegLib.FS('unlink', 'wave_'+i+'.png'); }catch(e){}
      ffDel(inN); ffDel(segN);

      if(stepEl){ stepEl.style.color='#22c55e'; stepEl.textContent='✅ ('+(i+1)+') '+entry.name; }
      epf.style.width=Math.round(((i+1)/playQueue.length)*80)+'%';
    }

    // concat ถ้ามีหลายคลิป
    var finalN='final.'+outExt;
    eps.textContent='🔗 รวม '+allSegData.length+' คลิป...';
    epf.style.width='85%';

    var finalBuf;
    if(allSegData.length===1){
      finalBuf = allSegData[0];
    } else {
      // concat หลายคลิปด้วย v0.11 FS API
      var concatList = allSegData.map(function(_,i){ return "file 'seg_c_"+i+".mp4'"; }).join('\n');
      _ffmpegLib.FS('writeFile','concat.txt', new TextEncoder().encode(concatList));
      for(var ci=0;ci<allSegData.length;ci++){
        _ffmpegLib.FS('writeFile','seg_c_'+ci+'.mp4', new Uint8Array(allSegData[ci]));
      }
      await ffExec(['-f','concat','-safe','0','-i','concat.txt','-c','copy',finalN]);
      var concatResult = _ffmpegLib.FS('readFile', finalN);
      finalBuf = concatResult.buffer;
      ffDel('concat.txt'); ffDel(finalN);
      for(var ci2=0;ci2<allSegData.length;ci2++) ffDel('seg_c_'+ci2+'.mp4');
    }

    eps.textContent='📦 เตรียมไฟล์ดาวน์โหลด...';
    epf.style.width='95%';
    // Mix bgAudio (เพลงใน tr-a) ลงวิดีโอถ้ามี
    var audioClips = Object.values(S.clips).filter(function(c){ return c.type==='audio'; });
    if(audioClips.length > 0 && finalBuf){
      try{
        eps.textContent='🎵 Mix เสียงเพลง...'; epf.style.width='92%';
        // หา audio entry ตัวแรก
        var aClip = audioClips[0];
        var aEntry = S.files.find(function(f){ return f.id===aClip.fid; });
        if(aEntry){
          var aFileName = 'bgaudio_mix.'+aEntry.name.split('.').pop();
          var mixFinal = 'mix_final.mp4';
          // เขียน video final ลง FS
          _ffmpegLib.FS('writeFile', 'vid_premix.mp4', new Uint8Array(finalBuf));
          // เขียน audio file ลง FS
          await ffWrite(aFileName, aEntry.file);
          // calc audio offset (startSec ของ audio clip)
          var ps0 = pxSec();
          var aStart = (aClip.startSec!==undefined) ? aClip.startSec : (aClip.left/ps0);
          // mix: -itsoffset ทำให้เสียงเริ่มตรงตำแหน่ง
          var mixArgs = [
            '-i','vid_premix.mp4',
            '-itsoffset', aStart.toFixed(3),
            '-i', aFileName,
            '-c:v','copy',
            '-c:a','aac','-b:a','128k',
            '-shortest',
            mixFinal
          ];
          await ffExec(mixArgs);
          var mixResult = _ffmpegLib.FS('readFile', mixFinal);
          if(mixResult && mixResult.length > 1000){
            finalBuf = mixResult.buffer;
            console.log('[mix] bgAudio mixed OK, size:', mixResult.length);
          }
          try{ _ffmpegLib.FS('unlink','vid_premix.mp4'); }catch(e){}
          try{ _ffmpegLib.FS('unlink', aFileName); }catch(e){}
          try{ _ffmpegLib.FS('unlink', mixFinal); }catch(e){}
        }
      } catch(mixErr){
        console.warn('[mix] bgAudio mix failed, using original:', mixErr.message);
      }
    }

    // ══ WAVEFORM BURN-IN (Static PNG) ══
    var waves = window.S_WAVES || [];
    if(waves.length > 0 && finalBuf){
      try{
        eps.textContent='〰️ Burn waveform...'; epf.style.width='94%';
        var wCv = document.createElement('canvas');
        wCv.width = tw; wCv.height = th;
        var wCtx = wCv.getContext('2d');
        waves.forEach(function(clip){
          var style = WAVE_STYLES.find(function(s){return s.id===clip.styleId;})||WAVE_STYLES[0];
          var px=clip.pvX!==undefined?clip.pvX:5, py=clip.pvY!==undefined?clip.pvY:75;
          var pw=clip.pvW!==undefined?clip.pvW:90, ph=clip.pvH!==undefined?clip.pvH:15;
          var x=Math.floor(px/100*tw), y=Math.floor(py/100*th);
          var w=Math.max(10,Math.floor(pw/100*tw)), h=Math.max(4,Math.floor(ph/100*th));
          var sub=document.createElement('canvas'); sub.width=w; sub.height=h;
          var data=genWaveData(Math.max(20,Math.floor(w/5)), clip.seed||1);
          drawWaveAnimated(sub, style, data, 1.2); // mid-animation frame
          wCtx.drawImage(sub, x, y, w, h);
        });
        var wPng = dataURLtoUint8Array(wCv.toDataURL('image/png'));
        _ffmpegLib.FS('writeFile', 'wov.png', wPng);
        _ffmpegLib.FS('writeFile', 'vpw.mp4', new Uint8Array(finalBuf));
        // ใช้ -lavfi แทน -filter_complex เพื่อหลีก pthread issue
        await ffExec([
          '-i','vpw.mp4',
          '-i','wov.png',
          '-filter_complex','overlay=0:0',
          '-codec:a','copy',
          '-preset','ultrafast',
          'vwaved.mp4'
        ]);
        var r=_ffmpegLib.FS('readFile','vwaved.mp4');
        if(r&&r.length>10000){ finalBuf=r.buffer; console.log('[wave] burned OK',r.length); }
        try{_ffmpegLib.FS('unlink','vpw.mp4');}catch(e){}
        try{_ffmpegLib.FS('unlink','wov.png');}catch(e){}
        try{_ffmpegLib.FS('unlink','vwaved.mp4');}catch(e){}
      }catch(we){ console.warn('[wave] skip:',we.message); }
    }
    var mimeMap={'mp4':'video/mp4','webm':'video/webm','mp3':'audio/mpeg','aac':'audio/aac'};
    var blob=new Blob([finalBuf],{type:mimeMap[outExt]||'video/mp4'});
    var fname=(document.getElementById('exp-name').value||'suomsiang')+'.'+outExt;

    // วิธีดาวน์โหลดที่ทำงานได้ใน Chrome Extension (MV3)
    function triggerDownload(){
      // ลอง chrome.downloads API ก่อน (ต้องการ permission "downloads")
      // ถ้าไม่มี ใช้ <a> click แทน
      var dlUrl = URL.createObjectURL(blob);
      var a = document.createElement('a');
      a.href = dlUrl;
      a.download = fname;
      a.style.display = 'none';
      document.body.appendChild(a);
      a.click();
      setTimeout(function(){
        document.body.removeChild(a);
        URL.revokeObjectURL(dlUrl);
      }, 5000);
    }

    // อัปเดต dl button ให้ใช้ blob ใหม่ทุกครั้งที่กด (กัน URL หมดอายุ)
    var _lastBlob = blob;
    var _lastFname = fname;
    dl.href = '#';
    dl.onclick = function(e){
      e.preventDefault();
      var freshUrl = URL.createObjectURL(_lastBlob);
      var a2 = document.createElement('a');
      a2.href = freshUrl;
      a2.download = _lastFname;
      a2.style.display = 'none';
      document.body.appendChild(a2);
      a2.click();
      setTimeout(function(){ document.body.removeChild(a2); URL.revokeObjectURL(freshUrl); }, 5000);
    };
    dl.download = fname;
    dl.textContent='⬇ ดาวน์โหลด '+fname+' ('+(blob.size/1024/1024).toFixed(1)+' MB)';
    dl.style.display='block';
    epf.style.width='100%';
    eps.textContent='✅ รวม '+playQueue.length+' คลิปสำเร็จ! ขนาด: '+(blob.size/1024/1024).toFixed(1)+' MB';
    showToast('🎉 รวมวิดีโอสำเร็จ! กดดาวน์โหลดได้เลย');

    // ดาวน์โหลดอัตโนมัติทันที
    triggerDownload();

  }catch(err){
    eps.textContent='❌ '+err.message;
    showToast('❌ Export ไม่สำเร็จ: '+err.message.substring(0,50));
    console.error('[Export]',err);
  }
  btn.disabled=false; btn.textContent='🚀 เริ่มรวมและส่งออก';
});


// ═══════════════════════════════════════
// FFMPEG v0.12 — โหลดตรงใน Extension tab
// Extension pages ได้รับ crossOriginIsolated อัตโนมัติ
// ทำให้ SharedArrayBuffer และ Worker ทำงานได้
// ═══════════════════════════════════════
var _ffmpegLib = null;  // FFmpeg instance จาก @ffmpeg/ffmpeg v0.12

async function loadFFmpeg(){
  if(_ffmpegLib) return true;
  var ov     = document.getElementById('ff-ov');
  var msg    = document.getElementById('ff-msg');
  var bar    = document.getElementById('ff-pct-bar');
  var pctTxt = document.getElementById('ff-pct-txt');
  ov.classList.add('show');

  function setMsg(m){ if(msg) msg.textContent=m; }
  function setPct(p){
    p=Math.max(0,Math.min(100,Math.round(p)));
    if(bar) bar.style.width=p+'%';
    if(pctTxt) pctTxt.textContent=p+'%';
  }

  try{
    var FFLib = window.FFmpeg;
    if(!FFLib || !FFLib.createFFmpeg)
      throw new Error('window.FFmpeg ไม่พบ');

    var base = (typeof chrome!=='undefined' && chrome.runtime && chrome.runtime.getURL)
      ? chrome.runtime.getURL('')
      : (location.origin + location.pathname.replace(/[^/]*$/, ''));

    // URL ของไฟล์ใน extension — chrome.runtime.getURL ทำให้ worker importScripts ได้
    var coreUrl   = base + 'ffmpeg-core.js';
    var workerUrl = base + 'ffmpeg-core.worker.js';
    var wasmUrl   = base + 'ffmpeg-core.wasm';

    // โหลด wasm พร้อม progress bar (fetch ก่อนเพื่อแสดง progress)
    setMsg('กำลังโหลด ffmpeg-core.wasm (24MB)...'); setPct(5);
    var wasmResp = await fetch(wasmUrl);
    var total = parseInt(wasmResp.headers.get('content-length') || '24000000');
    var loaded = 0; var chunks = [];
    var rdr = wasmResp.body.getReader();
    while(true){
      var chunk = await rdr.read(); if(chunk.done) break;
      chunks.push(chunk.value); loaded += chunk.value.length;
      var pct = 5 + Math.round((loaded/total)*78);
      setPct(pct);
      setMsg('โหลด wasm ' + Math.round((loaded/total)*100) + '% (' +
             Math.round(loaded/1048576) + '/' + Math.round(total/1048576) + ' MB)');
    }
    var wasmArr = new Uint8Array(loaded); var woff = 0;
    chunks.forEach(function(c){ wasmArr.set(c, woff); woff += c.length; });
    var wasmBlob  = new Blob([wasmArr], {type:'application/wasm'});
    var wasmBlobUrl = URL.createObjectURL(wasmBlob);

    setMsg('กำลัง initialize FFmpeg...'); setPct(85);

    var ff = FFLib.createFFmpeg({
      log: true,
      // corePath = chrome-extension:// URL — worker จะ importScripts(corePath) ได้
      corePath:            coreUrl,
      // workerPath = URL ของ worker script
      workerPath:          workerUrl,
      // mainScriptUrlOrBlob = URL ที่ส่งไปให้ worker ผ่าน postMessage
      // ต้องเป็น chrome-extension:// ไม่ใช่ blob: เพราะ CSP บล็อก blob: ใน worker
      mainScriptUrlOrBlob: coreUrl,
      // locateFile — ให้ wasm ใช้ blob URL ที่โหลดมาแล้ว
      locateFile: function(path, scriptDir){
        if(path.endsWith('.wasm')) return wasmBlobUrl;
        if(path.endsWith('.worker.js')) return workerUrl;
        return scriptDir + path;
      },
    });

    setMsg('กำลัง ff.load()...'); setPct(90);
    await ff.load();

    // revoke wasm blob หลัง load เสร็จ (ffmpeg copy wasm ไปแล้ว)
    setTimeout(function(){ try{ URL.revokeObjectURL(wasmBlobUrl); }catch(e){} }, 10000);

    _ffmpegLib = ff;
    ffmpeg = {_v11:true};
    setMsg('✅ พร้อมแล้ว!'); setPct(100);
    setTimeout(function(){ ov.classList.remove('show'); }, 600);
    showToast('✅ FFmpeg พร้อมส่งออกแล้ว!');
    return true;

  }catch(e){
    console.error('[loadFFmpeg]', e);
    setMsg('❌ ' + e.message); setPct(0);
    setTimeout(function(){ ov.classList.remove('show'); }, 5000);
    showToast('❌ โหลด FFmpeg ไม่สำเร็จ: ' + e.message.substring(0,50));
    return false;
  }
}
// v0.11 FS helpers
async function ffWrite(name, fileObj){
  var buf=await fileObj.arrayBuffer();
  _ffmpegLib.FS('writeFile',name,new Uint8Array(buf));
}
function ffRead(name){
  var data=_ffmpegLib.FS('readFile',name);
  return data; // Uint8Array
}
function ffDel(name){ try{_ffmpegLib.FS('unlink',name);}catch(e){} }
async function ffExec(args){
  console.log('[ffExec] running:', args.join(' '));
  var timeoutId;
  var timeoutP = new Promise(function(_,rej){
    timeoutId = setTimeout(function(){ rej(new Error('FFmpeg timeout (5 นาที) — วิดีโออาจใหญ่เกินไป')); }, 300000);
  });
  try{
    await Promise.race([ _ffmpegLib.run.apply(_ffmpegLib, args), timeoutP ]);
    clearTimeout(timeoutId);
    console.log('[ffExec] completed ok');
  }catch(e){
    clearTimeout(timeoutId);
    console.error('[ffExec] error:', e);
    throw e;
  }
}
// ส่ง run command ไปยัง sandbox และรอผล
async function sbRun(args, inputFiles, outName){
  // แปลงแต่ละไฟล์เป็น ArrayBuffer
  var filesData = await Promise.all(inputFiles.map(async function(f){
    return {name:f.name, data: await f.file.arrayBuffer()};
  }));
  var transfers = filesData.map(function(f){return f.data;});

  return new Promise(function(resolve,reject){
    var id = 'r'+(++_sbMsgId);
    _sbCallbacks[id] = {resolve:resolve, reject:reject};
    setTimeout(function(){
      if(_sbCallbacks[id]){
        delete _sbCallbacks[id];
        reject(new Error('FFmpeg timeout'));
      }
    }, 600000); // 10 นาที
    sbSend({type:'RUN', _id:id, args:args, files:filesData, outName:outName}, transfers);
  });
}


// แสดงคำแนะนำวิธีเปิดผ่าน localhost
function showFFmpegHelp(){
  var ov=document.getElementById('ff-ov');
  ov.classList.add('show');
  ov.innerHTML='<div style="background:#1e1e1e;border:1px solid #f5c518;border-radius:16px;padding:28px;max-width:480px;margin:20px;text-align:center;">'+
    '<div style="font-size:28px;margin-bottom:12px;">⚠️</div>'+
    '<div style="font-size:16px;font-weight:700;color:#f5c518;margin-bottom:10px;">FFmpeg ต้องเปิดผ่าน localhost</div>'+
    '<div style="font-size:13px;color:#888;margin-bottom:16px;line-height:1.6;">Chrome บล็อก FFmpeg เมื่อเปิดจาก <code>file://</code><br>ต้องเปิดผ่าน local server แทน</div>'+
    '<div style="background:#111;border-radius:8px;padding:12px;margin-bottom:16px;text-align:left;">'+
      '<div style="font-size:11px;color:#888;margin-bottom:6px;font-weight:600;">วิธีที่ 1 — VS Code (ง่ายที่สุด)</div>'+
      '<div style="font-size:12px;color:#e8e8e8;">1. เปิด VS Code<br>2. ติดตั้ง Extension "Live Server"<br>3. คลิกขวาที่ไฟล์ → Open with Live Server</div>'+
    '</div>'+
    '<div style="background:#111;border-radius:8px;padding:12px;margin-bottom:16px;text-align:left;">'+
      '<div style="font-size:11px;color:#888;margin-bottom:6px;font-weight:600;">วิธีที่ 2 — Command Line</div>'+
      '<div style="font-size:12px;color:#e8e8e8;font-family:monospace;">npx serve .<br>แล้วเปิด http://localhost:3000</div>'+
    '</div>'+
    '<div style="background:#111;border-radius:8px;padding:12px;margin-bottom:16px;text-align:left;">'+
      '<div style="font-size:11px;color:#888;margin-bottom:6px;font-weight:600;">วิธีที่ 3 — Python</div>'+
      '<div style="font-size:12px;color:#e8e8e8;font-family:monospace;">python -m http.server 8080<br>แล้วเปิด http://localhost:8080</div>'+
    '</div>'+
    '<button onclick="document.getElementById(\'ff-ov\').classList.remove(\'show\')" style="background:#f5c518;color:#000;border:none;padding:10px 24px;border-radius:8px;font-weight:700;cursor:pointer;font-size:14px;">✕ ปิด</button>'+
  '</div>';
}

// ═══════════════════════════════════════
// UTILS
// ═══════════════════════════════════════
function fmt(s){if(!s||isNaN(s))return'0:00.0';var m=Math.floor(s/60),sc=(s%60).toFixed(1);return m+':'+(sc<10?'0':'')+sc;}
function showToast(msg){var t=document.getElementById('toast');t.textContent=msg;t.classList.add('show');setTimeout(function(){t.classList.remove('show');},3000);}

// INIT
drawRuler();
// ตรวจว่าเปิดจาก file:// หรือไม่
if(window.location.protocol==='file:'){
  var lw=document.getElementById('localhost-warn');
  if(lw){
    lw.style.display='flex';
    lw.addEventListener('click',function(){
      if(typeof chrome === 'undefined' || !chrome.runtime) showFFmpegHelp();
    });
  }
}
// ซ่อน trim markers
['tl-trim-in','tl-trim-out','tl-trim-zone'].forEach(function(id){
  var el=document.getElementById(id);
  if(el) el.style.display='none';
});
// drag-drop จาก media list ไปยัง timeline tracks
['tr-v1','tr-v2','tr-a','tr-f','tr-t'].forEach(function(id){
  var el=document.getElementById(id);
  if(el) setupTrackDrop(el);
});
// ปุ่ม "+ เพิ่มเสียง" ใน label
var lblAudio=document.getElementById('lbl-add-audio');
if(lblAudio){
  lblAudio.addEventListener('click',function(){
    var fi2=document.createElement('input');
    fi2.type='file'; fi2.accept='audio/*'; fi2.multiple=true;
    fi2.onchange=function(){addFiles(Array.from(fi2.files));};
    fi2.click();
  });
}
// fi ให้รับทั้งวิดีโอและเสียง
document.getElementById('fi').accept='video/*,audio/*';
document.getElementById('dz').querySelector('.dz-s').innerHTML=
  'คลิกหรือลากหลายไฟล์มาวาง<br><b style="color:var(--acc)">วิดีโอ MP4/MOV/AVI และเสียง MP3/WAV</b>';

// ═══════════════════════════════════════════════════════
// WAVEFORM STICKER SYSTEM — v2
// ═══════════════════════════════════════════════════════
var WAVE_STYLES = [
  { id:'bars',   name:'Bars',   ico:'📊', color:'#f5c518', bg:'transparent', desc:'แท่งกราฟ' },
  { id:'line',   name:'Line',   ico:'〰️', color:'#00e5ff', bg:'transparent', desc:'เส้นโค้ง' },
  { id:'mirror', name:'Mirror', ico:'🪞', color:'#ff4fc8', bg:'transparent', desc:'สะท้อนกลาง' },
  { id:'dots',   name:'Dots',   ico:'⠿',  color:'#7fff00', bg:'transparent', desc:'จุดกระจาย' },
  { id:'neon',   name:'Neon',   ico:'💡', color:'#ff6f00', bg:'transparent', desc:'นีออน glow' },
  { id:'circle', name:'Circle', ico:'🔵', color:'#b388ff', bg:'transparent', desc:'วงกลม' },
];
if(!window.S_WAVES) window.S_WAVES = [];

function genWaveData(n, seed){
  var d = []; var s = seed || 1;
  for(var i=0;i<n;i++){
    s = (s*9301+49297)%233280;
    d.push(0.15 + (s/233280)*0.85);
  }
  return d;
}

// วาด waveform ที่เคลื่อนไหวตาม audioTime
function drawWaveAnimated(canvas, style, baseData, audioTime){
  var W=canvas.width, H=canvas.height, n=baseData.length;
  var ctx=canvas.getContext('2d');
  ctx.clearRect(0,0,W,H);
  // ไม่วาด background ถ้าเป็น transparent
  if(style.bg && style.bg !== 'transparent'){
    ctx.fillStyle=style.bg; ctx.fillRect(0,0,W,H);
  } else {
    ctx.clearRect(0,0,W,H);
  }

  // animate data ตาม time
  var data = baseData.map(function(v,i){
    var wave = Math.sin(audioTime*4 + i*0.4)*0.3;
    return Math.max(0.05, Math.min(1, v + wave));
  });

  var c=style.color;
  if(style.id==='bars'){
    var bw=Math.max(1,W/n-1);
    for(var i=0;i<n;i++){
      var h=data[i]*H*0.9;
      var grd=ctx.createLinearGradient(0,H-h,0,H);
      grd.addColorStop(0,c); grd.addColorStop(1,c+'33');
      ctx.fillStyle=grd;
      ctx.fillRect(i*(W/n),H-h,bw,h);
    }
  } else if(style.id==='line'){
    ctx.beginPath(); ctx.strokeStyle=c; ctx.lineWidth=2.5;
    ctx.shadowBlur=4; ctx.shadowColor=c;
    for(var i=0;i<n;i++){
      var x=i*(W/n), y=H-data[i]*H*0.85;
      i===0?ctx.moveTo(x,y):ctx.lineTo(x,y);
    }
    ctx.stroke();
    ctx.shadowBlur=0;
    ctx.lineTo(W,H); ctx.lineTo(0,H); ctx.closePath();
    ctx.fillStyle=c+'22'; ctx.fill();
  } else if(style.id==='mirror'){
    for(var i=0;i<n;i++){
      var h2=data[i]*H*0.43, x2=i*(W/n), bw2=Math.max(1,W/n-1);
      var grd2=ctx.createLinearGradient(0,H/2-h2,0,H/2);
      grd2.addColorStop(0,c); grd2.addColorStop(1,c+'44');
      ctx.fillStyle=grd2;
      ctx.fillRect(x2,H/2-h2,bw2,h2);
      ctx.fillRect(x2,H/2,bw2,h2);
    }
  } else if(style.id==='dots'){
    for(var i=0;i<n;i++){
      var steps=Math.floor(data[i]*6)+1;
      for(var j=0;j<steps;j++){
        var dotY=H-(j/steps)*H*0.85-4;
        ctx.fillStyle=c+Math.floor((j/steps)*255).toString(16).padStart(2,'0');
        ctx.beginPath();
        ctx.arc(i*(W/n)+3,dotY,2.5,0,Math.PI*2);
        ctx.fill();
      }
    }
  } else if(style.id==='neon'){
    ctx.shadowBlur=10; ctx.shadowColor=c;
    [2, 1.5, 0.8].forEach(function(lw, li){
      ctx.beginPath(); ctx.strokeStyle=li===0?c:c+'88'; ctx.lineWidth=lw;
      for(var i=0;i<n;i++){
        var x=i*(W/n), y=H/2-data[i]*(H*0.4);
        i===0?ctx.moveTo(x,y):ctx.lineTo(x,y);
      }
      ctx.stroke();
      ctx.beginPath();
      for(var i=0;i<n;i++){
        var x=i*(W/n), y=H/2+data[i]*(H*0.4);
        i===0?ctx.moveTo(x,y):ctx.lineTo(x,y);
      }
      ctx.stroke();
    });
    ctx.shadowBlur=0;
  } else if(style.id==='circle'){
    var cx=W/2, cy=H/2, r=Math.min(W,H)*0.32;
    ctx.shadowBlur=6; ctx.shadowColor=c;
    for(var i=0;i<n;i++){
      var angle=(i/n)*Math.PI*2;
      var len=data[i]*r*0.7;
      ctx.beginPath();
      ctx.moveTo(cx+Math.cos(angle)*r, cy+Math.sin(angle)*r);
      ctx.lineTo(cx+Math.cos(angle)*(r+len), cy+Math.sin(angle)*(r+len));
      ctx.strokeStyle=c; ctx.lineWidth=2; ctx.stroke();
    }
    ctx.shadowBlur=0;
  }
}

// สร้าง card preview ใน panel
function buildWaveCards(){
  var grid = document.getElementById('fx-wave-grid');
  if(!grid) return;
  grid.innerHTML = '';
  WAVE_STYLES.forEach(function(style){
    var seed = style.id.charCodeAt(0)*7+1;
    var data = genWaveData(40, seed);
    var card = document.createElement('div');
    card.className = 'fx-wave-card';
    card.setAttribute('draggable','true');
    card.title = style.name+' — คลิกเพิ่มที่ playhead หรือลากมาวางบน timeline';

    var cv = document.createElement('canvas');
    cv.width = 200; cv.height = 72;
    drawWaveAnimated(cv, style, data, 0);
    card.appendChild(cv);

    // animate preview
    var animT = 0;
    var animId = null;
    card.addEventListener('mouseenter',function(){
      animId = setInterval(function(){
        animT += 0.08;
        drawWaveAnimated(cv, style, data, animT);
      }, 50);
    });
    card.addEventListener('mouseleave',function(){
      clearInterval(animId);
      drawWaveAnimated(cv, style, data, 0);
    });

    var lbl = document.createElement('div');
    lbl.style.cssText = 'font-size:10px;color:'+style.color+';font-weight:600;margin-top:3px;';
    lbl.textContent = style.ico+' '+style.name+' — '+style.desc;
    card.appendChild(lbl);

    // color picker
    var colorRow = document.createElement('div');
    colorRow.style.cssText = 'display:flex;align-items:center;gap:4px;margin-top:4px;';
    var colorLbl = document.createElement('span');
    colorLbl.style.cssText = 'font-size:9px;color:var(--tx2);';
    colorLbl.textContent = 'สี:';
    var colorPick = document.createElement('input');
    colorPick.type = 'color'; colorPick.value = style.color;
    colorPick.style.cssText = 'width:20px;height:16px;border:none;cursor:pointer;background:none;';
    colorPick.addEventListener('input',function(e){
      style.color = e.target.value;
      lbl.style.color = style.color;
      drawWaveAnimated(cv, style, data, animT);
      // อัปเดต clip ที่ใช้ style นี้
      (window.S_WAVES||[]).forEach(function(c){
        if(c.styleId===style.id) renderWaveClip(c);
      });
    });
    colorRow.appendChild(colorLbl); colorRow.appendChild(colorPick);
    card.appendChild(colorRow);

    // drag
    card.addEventListener('dragstart',function(e){
      e.dataTransfer.setData('wave-style', style.id);
      e.dataTransfer.effectAllowed = 'copy';
      card.style.opacity = '0.5';
    });
    card.addEventListener('dragend',function(){ card.style.opacity='1'; });

    // click
    card.addEventListener('click',function(e){
      if(e.target===colorPick) return;
      var ph = document.getElementById('tl-ph');
      var ps = pxSec();
      var dropTime = ph ? (parseFloat(ph.style.left)||0)/ps : 0;
      addWaveClip(style.id, dropTime, 5);
      showToast('〰️ เพิ่ม '+style.name+' ที่ '+dropTime.toFixed(1)+'s');
    });

    grid.appendChild(card);
  });
}

function addWaveClip(styleId, startSec, durSec){
  durSec = durSec || 5;
  var id = 'wv'+Date.now();
  var clip = {id:id, styleId:styleId, startSec:startSec, dur:durSec, seed:Math.floor(Math.random()*999)};
  window.S_WAVES.push(clip);
  renderWaveClip(clip);
  return clip;
}

function renderWaveClip(clip){
  var old = document.getElementById('wc-'+clip.id);
  if(old) old.remove();

  var track = document.getElementById('tr-f');
  if(!track) return;

  var style = WAVE_STYLES.find(function(s){return s.id===clip.styleId;}) || WAVE_STYLES[0];
  var ps = pxSec();
  var w = Math.max(40, clip.dur*ps);
  var n = Math.max(20, Math.floor(w/5));
  var data = genWaveData(n, clip.seed||1);

  var el = document.createElement('div');
  el.id = 'wc-'+clip.id;
  el.className = 'wave-clip';
  el.style.left = (clip.startSec*ps)+'px';
  el.style.width = w+'px';
  el.style.border = '1.5px solid '+style.color+'66';
  el.style.background = 'transparent';
  el.title = style.name+' — ลากย้าย | ลาก handle ขวาเพื่อยืด';

  var cv = document.createElement('canvas');
  cv.width = w; cv.height = 54;
  drawWaveAnimated(cv, style, data, 0);
  el.appendChild(cv);

  // live animation เมื่อเล่น
  var animId = null;
  function startAnim(){
    if(animId) return;
    animId = setInterval(function(){
      var t = (window.playQueueOffset||0) + (document.getElementById('prev-vid').currentTime||0);
      drawWaveAnimated(cv, style, data, t*2);
    }, 50);
  }
  function stopAnim(){
    clearInterval(animId); animId=null;
    drawWaveAnimated(cv, style, data, 0);
  }

  var vidEl = document.getElementById('prev-vid');
  if(vidEl){
    vidEl.addEventListener('play', startAnim);
    vidEl.addEventListener('pause', stopAnim);
    vidEl.addEventListener('ended', stopAnim);
  }

  // delete
  var del = document.createElement('div');
  del.className = 'wc-del'; del.textContent = '✕';
  del.addEventListener('mousedown',function(e){
    e.stopPropagation();
    stopAnim();
    if(vidEl){ vidEl.removeEventListener('play',startAnim); vidEl.removeEventListener('pause',stopAnim); }
    el.remove();
    // ลบ preview overlay ด้วย
    var pvEl = document.getElementById('wcp-'+clip.id);
    if(pvEl) pvEl.remove();
    window.S_WAVES = (window.S_WAVES||[]).filter(function(c){return c.id!==clip.id;});
    showToast('🗑 ลบ waveform แล้ว');
  });
  el.appendChild(del);

  // resize right handle
  var hdl = document.createElement('div');
  hdl.className = 'wc-hdl';
  hdl.addEventListener('mousedown',function(e){
    e.stopPropagation(); e.preventDefault();
    var sx=e.clientX, sw=clip.dur;
    function onMove(e2){
      clip.dur = Math.max(0.5, sw+(e2.clientX-sx)/pxSec());
      var nw = Math.max(40, clip.dur*pxSec());
      el.style.width = nw+'px';
      cv.width = nw;
      var nd = genWaveData(Math.max(20,Math.floor(nw/5)), clip.seed||1);
      drawWaveAnimated(cv, style, nd, 0);
    }
    function onUp(){ document.removeEventListener('mousemove',onMove); document.removeEventListener('mouseup',onUp); }
    document.addEventListener('mousemove',onMove);
    document.addEventListener('mouseup',onUp);
  });
  el.appendChild(hdl);

  // drag move (left/right ซิงค์เวลา)
  el.addEventListener('mousedown',function(e){
    if(e.target===hdl||e.target===del) return;
    e.preventDefault(); e.stopPropagation();
    var sx=e.clientX, ss=clip.startSec;
    function onMove(e2){
      clip.startSec = Math.max(0, ss+(e2.clientX-sx)/pxSec());
      el.style.left = (clip.startSec*pxSec())+'px';
    }
    function onUp(){ document.removeEventListener('mousemove',onMove); document.removeEventListener('mouseup',onUp); }
    document.addEventListener('mousemove',onMove);
    document.addEventListener('mouseup',onUp);
  });

  // คลิกขวา → context menu
  el.addEventListener('contextmenu', function(e){
    e.preventDefault(); e.stopPropagation();
    // ลบ menu เก่า
    var om = document.getElementById('wave-ctx-menu');
    if(om) om.remove();
    var menu = document.createElement('div');
    menu.id = 'wave-ctx-menu';
    menu.style.cssText = 'position:fixed;left:'+e.clientX+'px;top:'+e.clientY+'px;background:#1a1a1a;border:1px solid #444;border-radius:8px;padding:4px 0;z-index:9999;min-width:150px;box-shadow:0 4px 20px rgba(0,0,0,.7);font-size:13px;color:#fff;';
    function mi(ico,lbl,fn){
      var item=document.createElement('div');
      item.style.cssText='padding:8px 14px;cursor:pointer;display:flex;gap:8px;';
      item.innerHTML='<span>'+ico+'</span><span>'+lbl+'</span>';
      item.onmouseenter=function(){item.style.background='rgba(245,197,24,.15)';};
      item.onmouseleave=function(){item.style.background='';};
      item.addEventListener('mousedown',function(ev){ev.stopPropagation();fn();menu.remove();});
      return item;
    }
    menu.appendChild(mi('📋','ทำซ้ำ (ต่อท้าย)',function(){
      var nc={id:'wv'+Date.now(),styleId:clip.styleId,startSec:clip.startSec+clip.dur,dur:clip.dur,seed:Math.floor(Math.random()*9999)};
      window.S_WAVES.push(nc);
      renderWaveClip(nc);
      renderWavePreview(nc);
      showToast('📋 ทำซ้ำต่อท้ายแล้ว');
    }));
    menu.appendChild(mi('📋','ทำซ้ำ (ที่เดิม)',function(){
      var nc={id:'wv'+Date.now(),styleId:clip.styleId,startSec:clip.startSec,dur:clip.dur,seed:Math.floor(Math.random()*9999)};
      window.S_WAVES.push(nc);
      renderWaveClip(nc);
      renderWavePreview(nc);
      showToast('📋 ทำซ้ำแล้ว');
    }));
    var sep=document.createElement('div');
    sep.style.cssText='height:1px;background:#333;margin:4px 0;';
    menu.appendChild(sep);
    menu.appendChild(mi('🗑','ลบ',function(){
      stopAnim();
      el.remove();
      var pv=document.getElementById('wcp-'+clip.id);
      if(pv) pv.remove();
      window.S_WAVES=(window.S_WAVES||[]).filter(function(c){return c.id!==clip.id;});
      showToast('🗑 ลบ waveform แล้ว');
    }));
    document.body.appendChild(menu);
    setTimeout(function(){
      document.addEventListener('mousedown',function cls(ev){
        if(!menu.contains(ev.target)){menu.remove();document.removeEventListener('mousedown',cls);}
      });
    },10);
  });

  track.appendChild(el);

  // แสดง overlay บน preview ด้วย
  renderWavePreview(clip, style, data, startAnim, stopAnim);

  return el;
}

// วาด waveform overlay บน preview — คลิกได้ ลากย้าย ยืดหด
function renderWavePreview(clip, style, data, _sa, _so){
  var oldPv = document.getElementById('wcp-'+clip.id);
  if(oldPv) oldPv.remove();

  var wrap = document.getElementById('prev-wrap');
  if(!wrap) return;

  style = style || WAVE_STYLES.find(function(s){return s.id===clip.styleId;})||WAVE_STYLES[0];
  data  = data  || genWaveData(60, clip.seed||1);

  // state ตำแหน่งและขนาดบน preview (% ของ wrap)
  if(clip.pvX  === undefined) clip.pvX  = 5;   // %
  if(clip.pvY  === undefined) clip.pvY  = 75;  // %
  if(clip.pvW  === undefined) clip.pvW  = 90;  // %
  if(clip.pvH  === undefined) clip.pvH  = 15;  // %

  var pvEl = document.createElement('div');
  pvEl.id = 'wcp-'+clip.id;
  pvEl.style.cssText = [
    'position:absolute',
    'left:'+clip.pvX+'%',
    'top:'+clip.pvY+'%',
    'width:'+clip.pvW+'%',
    'height:'+clip.pvH+'%',
    'pointer-events:all',
    'z-index:14',
    'cursor:move',
    'border-radius:6px',
    'box-shadow:none',
    'overflow:hidden',
    'display:none',
  ].join(';');

  var cv = document.createElement('canvas');
  cv.style.cssText = 'width:100%;height:100%;display:block;';
  pvEl.appendChild(cv);

  // resize handle (มุมขวาล่าง)
  var rsz = document.createElement('div');
  rsz.style.cssText = 'position:absolute;right:0;bottom:0;width:14px;height:14px;cursor:se-resize;background:rgba(255,255,255,.25);border-radius:3px 0 4px 0;z-index:2;';
  rsz.innerHTML = '<svg width="10" height="10" viewBox="0 0 10 10" style="display:block;margin:2px auto;"><path d="M2 8 L8 8 L8 2" stroke="white" stroke-width="1.5" fill="none"/></svg>';
  pvEl.appendChild(rsz);

  // delete X
  var delPv = document.createElement('div');
  delPv.style.cssText = 'position:absolute;top:3px;right:3px;width:18px;height:18px;background:rgba(0,0,0,.7);color:#fff;border-radius:50%;display:none;align-items:center;justify-content:center;font-size:11px;cursor:pointer;z-index:3;';
  delPv.textContent = '✕';
  pvEl.appendChild(delPv);
  pvEl.addEventListener('mouseenter',function(){ delPv.style.display='flex'; });
  pvEl.addEventListener('mouseleave',function(){ delPv.style.display='none'; });

  // ลบทั้ง preview และ track clip
  delPv.addEventListener('mousedown',function(e){
    e.stopPropagation();
    pvEl.remove();
    var trackEl = document.getElementById('wc-'+clip.id);
    if(trackEl) trackEl.remove();
    window.S_WAVES = (window.S_WAVES||[]).filter(function(c){return c.id!==clip.id;});
    showToast('🗑 ลบ waveform แล้ว');
  });

  // ลาก preview ย้ายตำแหน่ง
  pvEl.addEventListener('mousedown',function(e){
    if(e.target===rsz||e.target.parentNode===rsz||e.target===delPv) return;
    e.preventDefault(); e.stopPropagation();
    var wr = wrap.getBoundingClientRect();
    var sx=e.clientX, sy=e.clientY;
    var ox=clip.pvX, oy=clip.pvY;
    function onMove(e2){
      clip.pvX = Math.max(0, Math.min(100-clip.pvW, ox+(e2.clientX-sx)/wr.width*100));
      clip.pvY = Math.max(0, Math.min(100-clip.pvH, oy+(e2.clientY-sy)/wr.height*100));
      pvEl.style.left = clip.pvX+'%';
      pvEl.style.top  = clip.pvY+'%';
    }
    function onUp(){ document.removeEventListener('mousemove',onMove); document.removeEventListener('mouseup',onUp); }
    document.addEventListener('mousemove',onMove);
    document.addEventListener('mouseup',onUp);
  });

  // resize handle ยืดหด
  rsz.addEventListener('mousedown',function(e){
    e.preventDefault(); e.stopPropagation();
    var wr = wrap.getBoundingClientRect();
    var sx=e.clientX, sy=e.clientY;
    var ow=clip.pvW, oh=clip.pvH;
    function onMove(e2){
      clip.pvW = Math.max(10, Math.min(100, ow+(e2.clientX-sx)/wr.width*100));
      clip.pvH = Math.max(5,  Math.min(50,  oh+(e2.clientY-sy)/wr.height*100));
      pvEl.style.width  = clip.pvW+'%';
      pvEl.style.height = clip.pvH+'%';
      // redraw canvas
      cv.width = pvEl.offsetWidth; cv.height = pvEl.offsetHeight;
      var d2 = genWaveData(Math.max(20,Math.floor(cv.width/6)), clip.seed||1);
      drawWaveAnimated(cv, style, d2, 0);
    }
    function onUp(){ document.removeEventListener('mousemove',onMove); document.removeEventListener('mouseup',onUp); }
    document.addEventListener('mousemove',onMove);
    document.addEventListener('mouseup',onUp);
  });

  wrap.appendChild(pvEl);

  // sync ตาม globalTime
  var vidEl = document.getElementById('prev-vid');
  var pvAnimId = null;

  function pvDraw(t){
    if(!pvEl.offsetParent && pvEl.style.display==='none') return;
    cv.width  = pvEl.offsetWidth  || 200;
    cv.height = pvEl.offsetHeight || 50;
    var d2 = genWaveData(Math.max(20,Math.floor(cv.width/5)), clip.seed||1);
    drawWaveAnimated(cv, style, d2, t);
  }

  function syncPv(){
    var gt=(window.playQueueOffset||0)+(vidEl?vidEl.currentTime||0:0);
    var visible = gt>=clip.startSec && gt<(clip.startSec+clip.dur);
    pvEl.style.display = visible ? 'block' : 'none';
    if(visible) pvDraw(gt*2);
  }

  if(vidEl){
    vidEl.addEventListener('timeupdate', syncPv);
    vidEl.addEventListener('play',function(){
      pvAnimId=setInterval(syncPv,50);
    });
    vidEl.addEventListener('pause',function(){
      clearInterval(pvAnimId); syncPv();
    });
    vidEl.addEventListener('ended',function(){
      clearInterval(pvAnimId); pvEl.style.display='none';
    });
  }

  // sync playhead seek
  var ph = document.getElementById('tl-ph');
  if(ph){
    var phObs = new MutationObserver(syncPv);
    phObs.observe(ph, {attributes:true, attributeFilter:['style']});
  }

  setTimeout(function(){ pvDraw(0); syncPv(); }, 150);
}

function refreshWaveClips(){
  // ลบ preview overlays เก่าก่อน
  document.querySelectorAll('[id^="wcp-"]').forEach(function(el){ el.remove(); });
  (window.S_WAVES||[]).forEach(function(clip){ renderWaveClip(clip); });
}

// Drop zone สำหรับ waveform — ผูกกับ tl-content ทั้งก้อน
(function setupWaveDrop(){
  function trySetup(){
    var sc = document.getElementById('tl-scroll');
    var trf = document.getElementById('tr-f');
    if(!sc || !trf){ setTimeout(trySetup,300); return; }
    trf.style.minHeight = '54px';

    // ให้ tr-f รับ drop โดยตรง
    trf.addEventListener('dragover', function(e){
      if(e.dataTransfer.types.indexOf('wave-style')<0) return;
      e.preventDefault(); e.dataTransfer.dropEffect='copy';
      trf.style.outline = '2px dashed var(--acc)';
    });
    trf.addEventListener('dragleave', function(){
      trf.style.outline = '';
    });
    trf.addEventListener('drop', function(e){
      var styleId = e.dataTransfer.getData('wave-style');
      if(!styleId) return;
      e.preventDefault();
      trf.style.outline = '';
      var r = sc.getBoundingClientRect();
      var x = e.clientX - r.left + sc.scrollLeft;
      var dropTime = Math.max(0, x/pxSec());
      addWaveClip(styleId, dropTime, 5);
      showToast('〰️ วาง waveform ที่ '+dropTime.toFixed(1)+'s');
    });

    // fallback: sc ก็รับได้
    sc.addEventListener('dragover', function(e){
      if(e.dataTransfer.types.indexOf('wave-style')<0) return;
      e.preventDefault(); e.dataTransfer.dropEffect='copy';
    });
    sc.addEventListener('drop', function(e){
      var styleId = e.dataTransfer.getData('wave-style');
      if(!styleId) return;
      e.preventDefault();
      var r = sc.getBoundingClientRect();
      var x = e.clientX - r.left + sc.scrollLeft;
      var dropTime = Math.max(0, x/pxSec());
      addWaveClip(styleId, dropTime, 5);
      showToast('〰️ วาง waveform ที่ '+dropTime.toFixed(1)+'s');
    });
  }
  trySetup();
})();

// Init panel เมื่อคลิก fx
(function(){
  function tryBind(){
    var btns = document.querySelectorAll('.ib[data-p]');
    if(!btns.length){ setTimeout(tryBind,300); return; }
    btns.forEach(function(btn){
      btn.addEventListener('click',function(){
        if(btn.dataset.p==='fx') setTimeout(buildWaveCards,50);
      });
    });
    buildWaveCards();
  }
  if(document.readyState==='loading'){
    document.addEventListener('DOMContentLoaded', tryBind);
  } else {
    tryBind();
  }
})();
// สร้าง waveform overlay PNG สำหรับ export
// คืนค่า base64 PNG ที่มีขนาด tw x th โดย waveform อยู่ที่ตำแหน่ง pvX,pvY
function buildWaveOverlayPNG(tw, th, clipStartSec, clipDurSec){
  var waves = window.S_WAVES || [];
  if(!waves.length) return null;

  // กรอง wave ที่ overlap กับ clip นี้
  var active = waves.filter(function(w){
    return w.startSec < (clipStartSec+clipDurSec) && (w.startSec+w.dur) > clipStartSec;
  });
  if(!active.length) return null;

  var cv = document.createElement('canvas');
  cv.width = tw; cv.height = th;
  var ctx = cv.getContext('2d');

  active.forEach(function(clip){
    var style = WAVE_STYLES.find(function(s){return s.id===clip.styleId;})||WAVE_STYLES[0];
    // ตำแหน่งบน frame
    var px = clip.pvX  !== undefined ? clip.pvX  : 5;   // %
    var py = clip.pvY  !== undefined ? clip.pvY  : 75;  // %
    var pw = clip.pvW  !== undefined ? clip.pvW  : 90;  // %
    var ph = clip.pvH  !== undefined ? clip.pvH  : 15;  // %

    var x = Math.floor(px/100*tw);
    var y = Math.floor(py/100*th);
    var w = Math.floor(pw/100*tw);
    var h = Math.floor(ph/100*th);

    // วาดลงใน subcanvas
    var sub = document.createElement('canvas');
    sub.width = w; sub.height = h;
    var data = genWaveData(Math.max(20, Math.floor(w/5)), clip.seed||1);
    // วาดที่ t=0 (static สำหรับ export)
    drawWaveAnimated(sub, style, data, 0);
    ctx.drawImage(sub, x, y, w, h);
  });

  return cv.toDataURL('image/png');
}

// แปลง dataURL เป็น Uint8Array
function dataURLtoUint8Array(dataURL){
  var b64 = dataURL.split(',')[1];
  var bin = atob(b64);
  var arr = new Uint8Array(bin.length);
  for(var i=0;i<bin.length;i++) arr[i]=bin.charCodeAt(i);
  return arr;
}


// ── FIX: แสดงปุ่มส่งออกใน Electron ──
(function(){
  var btn = document.getElementById('btn-export');
  if(btn){
    btn.style.display = '';
    btn.style.marginRight = '6px';
  }
  // แก้ export path สำหรับ Electron (file:// → ใช้ native dialog)
  if(typeof window.electronAPI !== 'undefined'){
    var exportBtn = document.getElementById('btn-export');
    if(exportBtn){
      exportBtn.addEventListener('click', function(e){
        // ถ้ากำลังจะ export ผ่าน ffmpeg.wasm แต่รัน Electron
        // ให้ redirect ไป native export แทน
        if(document.getElementById('btn-native-export')){
          e.stopImmediatePropagation();
          document.getElementById('btn-native-export').click();
        }
      }, true);
    }
  }
})();
