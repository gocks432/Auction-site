/* Pure domain logic. Trailing underscores keep helpers private to Apps Script. */
function fail_(message) { throw new Error(message); }
function num_(n, min, max) { n = Number(n); if (!Number.isSafeInteger(n) || n < min || n > max) fail_('숫자 범위를 확인해 주세요: '+min+'~'+max); return n; }
function str_(v, max) { return String(v == null ? '' : v).trim().slice(0,max); }
function student_(s,id) { var u=s.students.find(function(x){return x.id===id;}); if(!u) fail_('학생을 찾을 수 없습니다.'); return u; }
function auction_(s,id) { var a=s.auctions.find(function(x){return x.id===id;}); if(!a) fail_('경매를 찾을 수 없습니다.'); return a; }
function latest_(s,a) { var map={}; s.bids.forEach(function(b){if(b.auction===a.id) map[b.student]=b;}); return Object.keys(map).map(function(k){return map[k];}).filter(function(b){return b.amount>0;}); }
function ranked_(s,a) { return latest_(s,a).sort(function(x,y){return y.amount-x.amount || x.seq-y.seq;}); }
function held_(s,id,except) { return s.auctions.reduce(function(sum,a){ if(a.paymentMode==='external'||a.status!=='open'||a.id===except)return sum; var b=a.mode==='public'?ranked_(s,a).slice(0,1):latest_(s,a);return sum+b.filter(function(x){return x.student===id;}).reduce(function(v,x){return v+x.amount;},0);},0); }
function log_(s,student,amount,reason,now,key) { s.ledger.push({id:key,student:student,amount:amount,reason:reason,at:now}); }
function settle_(s,now) { var changed=false; s.auctions.forEach(function(a){if(a.status==='open'&&now>=a.end){var winner=ranked_(s,a)[0];a.status='closed';a.closedAt=now;a.winner=winner?winner.student:'';a.price=winner?winner.amount:0;if(winner){var u=student_(s,winner.student);if(a.paymentMode==='external'){a.paymentStatus='pending';}else{if(u.balance<winner.amount)fail_('정산 잔액 불일치: '+a.title);u.balance-=winner.amount;log_(s,u.id,-winner.amount,'낙찰: '+a.title,now,'settle:'+a.id);}s.coupons.push({id:'coupon:'+a.id,auction:a.id,student:u.id,title:a.title,description:a.description,expiry:now+a.validDays*86400000,status:a.paymentMode==='external'?'paymentPending':'unused'});}changed=true;}}); return changed; }
function placeBid_(s,id,p,now) { var a=auction_(s,p.auction),u=student_(s,id);if(a.status!=='open'||now<a.start||now>=a.end)fail_('입찰 가능한 시간이 아닙니다.');if(a.allowed.length&&a.allowed.indexOf(id)<0)fail_('이 경매의 참여 대상이 아닙니다.');var amount=num_(p.amount,0,100000000),old=latest_(s,a).find(function(b){return b.student===id;});if(amount===0){if(a.mode!=='sealed')fail_('공개 경매는 입찰을 철회할 수 없습니다.');if(!old)fail_('철회할 입찰이 없습니다.');}else{if(amount<a.minimum||(amount-a.minimum)%a.step!==0)fail_('시작 금액과 입찰 단위를 확인해 주세요.');if(old&&old.amount===amount)fail_('이미 같은 금액으로 입찰했습니다.');var highest=ranked_(s,a)[0];if(a.mode==='public'&&highest&&amount<highest.amount+a.step)fail_('최고 금액이 변경되었습니다. 더 높은 금액으로 입찰해 주세요.');if(a.paymentMode!=='external'&&amount>u.balance-held_(s,id,a.id))fail_('사용 가능한 GOLD가 부족합니다.');}s.seq++;s.bids.push({auction:a.id,student:id,amount:amount,at:now,seq:s.seq}); }
function newAuction_(s,p,now,id) { var title=str_(p.title,80),description=str_(p.description,1500);if(!title)fail_('경매 이름을 입력해 주세요.');if(['public','sealed'].indexOf(p.mode)<0)fail_('경매 방식을 선택해 주세요.');var timing=p.timing;if(['duration','scheduled'].indexOf(timing)<0)fail_('시간 설정을 확인해 주세요.');var start=0,end=0,minutes=num_(p.minutes||5,1,1440);if(timing==='scheduled'){start=Number(p.start);end=Number(p.end);if(!Number.isSafeInteger(start)||!Number.isSafeInteger(end)||start<=now||end<=start||end-start>7*86400000)fail_('시작은 현재 이후, 종료는 시작 이후 7일 이내여야 합니다.');}var allowed=Array.isArray(p.allowed)?p.allowed:[];allowed=allowed.filter(function(x,i){student_(s,x);return allowed.indexOf(x)===i;});var a={id:id,paymentMode:'external',paymentStatus:'none',title:title,description:description,mode:p.mode,minimum:num_(p.minimum,1,100000000),step:num_(p.step,1,100000000),validDays:num_(p.validDays||14,1,365),timing:timing,minutes:minutes,start:start,end:end,allowed:allowed,status:timing==='duration'?'draft':'open',image:p.image||'',created:now};s.auctions.push(a);return a; }
function changeGold_(s,p,now,key) { var rows=p.rows;if(!Array.isArray(rows)||!rows.length||rows.length>100)fail_('GOLD 변경 목록을 확인해 주세요.');var reason=str_(p.reason,200);if(!reason)fail_('변경 사유가 필요합니다.');var seen={};rows.forEach(function(r){if(seen[r.id])fail_('학생이 중복되었습니다.');seen[r.id]=true;var u=student_(s,r.id),amount=num_(r.amount,-100000000,100000000);if(u.balance+amount<held_(s,u.id)||u.balance+amount>100000000)fail_(u.number+'번: 잔액 또는 묶인 GOLD를 확인해 주세요.');});rows.forEach(function(r){var u=student_(s,r.id),amount=Number(r.amount);u.balance+=amount;log_(s,u.id,amount,reason,now,key+':'+u.id);}); }
function validateImport_(s,rows) { if(!Array.isArray(rows)||!rows.length||rows.length>100)fail_('학생 목록은 1~100명이어야 합니다.');var ns={},cs={};return rows.map(function(r){var number=num_(r.number,1,999),code=str_(r.code,32).toUpperCase(),pin=String(r.pin||'').trim(),name=str_(r.name,40)||number+'번';if(!/^[A-Z0-9_-]{4,32}$/.test(code)||!/^\d{4,12}$/.test(pin))fail_(number+'번: 아이디 또는 PIN 형식 오류');if(ns[number]||cs[code])fail_('번호 또는 아이디가 중복되었습니다.');ns[number]=true;cs[code]=true;return {number:number,code:code,pin:pin,name:name};}).map(function(r,i,all){var target=s.students.find(function(u){return u.number===r.number;}),collision=s.students.find(function(u){return u.code===r.code&&(!target||u.id!==target.id);});if(collision&&!all.some(function(x){return x.number===collision.number&&x.code!==r.code;}))fail_('다른 학생이 사용 중인 아이디입니다: '+r.number+'번');return r;}); }
function parseLoginText_(text) {
  text=String(text).replace(/\r/g,'').replace(/：/g,':').replace(/[\u200B-\u200D\uFEFF]/g,'');
  var marker=/(?:학생\s*코드|학생\s*(?:아이디|ID)|아이디|Student\s*(?:Code|ID))\s*:?\s*([A-Z0-9_-]{4,32})/gi;
  var hits=Array.from(text.matchAll(marker)),rows=[];
  hits.forEach(function(hit,i){
    var before=text.slice(i?hits[i-1].index+hits[i-1][0].length:0,hit.index);
    var after=text.slice(hit.index+hit[0].length,i+1<hits.length?hits[i+1].index:text.length);
    var pin=after.match(/(?:2\s*차\s*)?(?:P\s*I\s*N|비밀\s*번호)\s*:?\s*([0-9]{4,12})(?![0-9])/i);
    if(!pin){rows.push({number:'',name:'',code:hit[1].toUpperCase(),pin:'',note:'PIN을 인식하지 못했습니다.'});return;}
    var nums=Array.from(before.matchAll(/(?:^|\n)[ \t]*(\d{1,3})[ \t]*번[ \t]*(?=\n|$)/g));
    // Repeated matching number can be the nickname (e.g. 8번); conflicting numbers remain unresolved.
    var unique=nums.map(function(x){return Number(x[1]);}).filter(function(v,j,a){return a.indexOf(v)===j;});
    var number=unique.length===1?unique[0]:'',name='';
    if(number&&nums.length){var last=nums[nums.length-1],tail=before.slice(last.index+last[0].length).trim().split('\n').map(function(x){return x.trim();}).filter(Boolean);if(tail.length===1&&!/PIN|코드|학교|로그인/i.test(tail[0]))name=tail[0].slice(0,40);}
    rows.push({number:number,name:name|| (number?number+'번':''),code:hit[1].toUpperCase(),pin:pin[1],note:number?'원본 대조 필요':'학생 번호 확인 필요'});
  });
  return rows;
}

// Idempotent upgrade: open/draft auctions switch to bidding-only. Closed history stays intact.
function enableExternalPayment_(s,now){
 if(s.paymentMode==='external')return false;
 s.paymentMode='external';s.auctions.forEach(function(a){if(a.status==='open'||a.status==='draft'){a.paymentMode='external';a.paymentStatus='none';}});
 s.audit.push({at:now,action:'paymentMode',detail:'외부 GOLD 수동 처리 방식으로 전환 (미종료 경매 포함)'});return true;
}
function confirmPayment_(s,p,now){
 var a=auction_(s,p.id);
 if(a.paymentMode!=='external'||a.status!=='closed'||!a.winner)fail_('처리할 낙찰 내역이 없습니다.');
 if(a.paymentStatus==='paid')fail_('이미 처리 완료된 내역입니다.');
 var note=str_(p.note,300);if(!note)fail_('학급 홈페이지에서 처리한 내용을 적어 주세요.');
 a.paymentStatus='paid';a.paymentAt=now;a.paymentNote=note;
 var c=s.coupons.find(function(x){return x.auction===a.id;});
 if(c&&c.status==='paymentPending'){c.status='unused';c.expiry=now+a.validDays*86400000;}
}
