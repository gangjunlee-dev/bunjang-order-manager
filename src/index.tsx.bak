import { Hono } from 'hono';

type Bindings = {
  BUNJANG_ACCESS_KEY: string;
  BUNJANG_SECRET_KEY: string;
  BUNJANG_API_URL?: string;
};

const app = new Hono<{ Bindings: Bindings }>();

function base64url(buf: ArrayBuffer): string {
  const bytes = new Uint8Array(buf);
  let s = '';
  for (let i = 0; i < bytes.length; i++) s += String.fromCharCode(bytes[i]);
  return btoa(s).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
function base64urlEncode(str: string): string {
  return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
function decodeBase64(b64: string): Uint8Array {
  const bin = atob(b64);
  const arr = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) arr[i] = bin.charCodeAt(i);
  return arr;
}
function sleep(ms: number) { return new Promise(r => setTimeout(r, ms)); }

async function generateJWT(accessKey: string, secretKey: string, method: string): Promise<string> {
  const header = base64urlEncode(JSON.stringify({ alg: 'HS256', typ: 'JWT' }));
  const payload: any = { accessKey, iat: Math.floor(Date.now() / 1000) };
  const m = method.toUpperCase();
  if (m === 'POST' || m === 'PUT' || m === 'DELETE') payload.nonce = crypto.randomUUID();
  const payloadB64 = base64urlEncode(JSON.stringify(payload));
  const data = new TextEncoder().encode(header + '.' + payloadB64);
  const keyBytes = decodeBase64(secretKey);
  const cryptoKey = await crypto.subtle.importKey('raw', keyBytes, { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  const sig = await crypto.subtle.sign('HMAC', cryptoKey, data);
  return header + '.' + payloadB64 + '.' + base64url(sig);
}

async function bunjangFetch(env: Bindings, path: string, method = 'GET', body?: any): Promise<any> {
  const baseUrl = env.BUNJANG_API_URL || 'https://openapi.bunjang.co.kr';
  const token = await generateJWT(env.BUNJANG_ACCESS_KEY, env.BUNJANG_SECRET_KEY, method);
  const opts: RequestInit = { method, headers: { 'Authorization': 'Bearer ' + token, 'Content-Type': 'application/json' } };
  if (body) opts.body = JSON.stringify(body);
  for (let i = 0; i <= 2; i++) {
    const resp = await fetch(baseUrl + path, opts);
    if (resp.status === 429) { await sleep(parseInt(resp.headers.get('Retry-After') || '2') * 1000); continue; }
    if (resp.status === 204) return { success: true };
    const text = await resp.text();
    try { return JSON.parse(text); } catch { return { rawResponse: text, status: resp.status }; }
  }
  throw new Error('Max retries exceeded');
}

function toISO(d: Date): string { return d.toISOString().replace(/\.\d{3}Z$/, 'Z'); }
function dateWindowsDesc(totalDays: number): Array<{ start: string; end: string }> {
  const windows: Array<{ start: string; end: string }> = [];
  const now = new Date(); let cursor = now; let remaining = totalDays;
  while (remaining > 0) {
    const chunk = Math.min(remaining, 15);
    const end = new Date(cursor); const start = new Date(cursor.getTime() - chunk * 86400000);
    windows.push({ start: toISO(start), end: toISO(end) }); cursor = start; remaining -= chunk;
  }
  return windows;
}

// ── API Routes ──
app.get('/api/settings/check', (c) => c.json({
  hasAccessKey: !!c.env.BUNJANG_ACCESS_KEY,
  hasSecretKey: !!c.env.BUNJANG_SECRET_KEY,
  hasApiUrl: !!c.env.BUNJANG_API_URL
}));

app.get('/api/orders', async (c) => {
  const page = c.req.query('page') || '0';
  const size = c.req.query('size') || '100';
  const now = new Date();
  const start = c.req.query('statusUpdateStartDate') || toISO(new Date(now.getTime() - 15 * 86400000));
  const end = c.req.query('statusUpdateEndDate') || toISO(now);
  return c.json(await bunjangFetch(c.env, `/api/v1/orders?statusUpdateStartDate=${encodeURIComponent(start)}&statusUpdateEndDate=${encodeURIComponent(end)}&page=${page}&size=${size}`));
});

app.get('/api/orders/extended', async (c) => {
  const days = Math.min(parseInt(c.req.query('days') || '90'), 180);
  const windows = dateWindowsDesc(days);
  const results: any[] = [];
  const seen = new Set<number>();
  const errors: string[] = [];
  for (const w of windows) {
    for (let p = 0; p < 50; p++) {
      try {
        const d = await bunjangFetch(c.env, `/api/v1/orders?statusUpdateStartDate=${encodeURIComponent(w.start)}&statusUpdateEndDate=${encodeURIComponent(w.end)}&page=${p}&size=100`);
        for (const o of (d.data || [])) { if (!seen.has(o.id)) { seen.add(o.id); results.push(o); } }
        if (p + 1 >= (d.totalPages || 1)) break;
        await sleep(200);
      } catch (e: any) { errors.push(w.start + '~' + w.end + ' p' + p + ': ' + (e as Error).message); break; }
    }
  }
  return c.json({ data: results, total: results.length, errors });
});

app.get('/api/orders/:orderId', async (c) => c.json(await bunjangFetch(c.env, `/api/v1/orders/${c.req.param('orderId')}`)));
app.post('/api/orders', async (c) => c.json(await bunjangFetch(c.env, '/api/v2/orders', 'POST', await c.req.json())));
app.post('/api/orders/:orderId/confirm', async (c) => c.json(await bunjangFetch(c.env, `/api/v1/orders/${c.req.param('orderId')}/confirm`, 'POST')));

app.post('/api/orders/search-by-invoice-fast', async (c) => {
  const { invoiceNo, orderIds } = await c.req.json();
  if (!invoiceNo || !orderIds) return c.json({ error: 'invoiceNo and orderIds required' }, 400);
  const matched: any[] = [];
  for (const id of orderIds) {
    try {
      const det = await bunjangFetch(c.env, `/api/v1/orders/${id}`);
      const d = det.data || {};
      const inv = d.delivery?.invoice?.no || '';
      const retInvs = (d.returns || []).map((r: any) => r.invoice?.no || '');
      if (inv.includes(invoiceNo) || retInvs.some((n: string) => n.includes(invoiceNo))) matched.push(d);
      await sleep(50);
    } catch { }
  }
  return c.json({ data: matched, totalFound: matched.length });
});

app.get('/api/products/search', async (c) => {
  const keyword = c.req.query('keyword') || '';
  const cursor = c.req.query('cursor') || '';
  let path = `/api/v1/products?keyword=${encodeURIComponent(keyword)}`;
  if (cursor) path += `&cursor=${encodeURIComponent(cursor)}`;
  return c.json(await bunjangFetch(c.env, path));
});
app.get('/api/products/:pid', async (c) => c.json(await bunjangFetch(c.env, `/api/v1/products/${c.req.param('pid')}`)));
app.get('/api/categories', async (c) => c.json(await bunjangFetch(c.env, '/api/v1/categories')));
app.get('/api/brands', async (c) => c.json(await bunjangFetch(c.env, '/api/v1/brands')));

// ── SPA ──
app.get('*', (c) => {
  const html = `<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Bunjang Order Manager</title>
<script src="https://cdn.tailwindcss.com"><\/script>
<script src="//t1.daumcdn.net/mapjsapi/bundle/postcode/prod/postcode.v2.js"><\/script>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>📦</text></svg>">
<script>
tailwind.config = {
  theme: {
    extend: {
      fontFamily: { sans: ['Inter', 'system-ui', 'sans-serif'] },
      colors: {
        brand: { 50:'#eff6ff', 100:'#dbeafe', 200:'#bfdbfe', 300:'#93c5fd', 400:'#60a5fa', 500:'#3b82f6', 600:'#2563eb', 700:'#1d4ed8' },
        surface: { 50:'#f8fafc', 100:'#f1f5f9', 200:'#e2e8f0' }
      }
    }
  }
}
<\/script>
<style>
body { font-family: 'Inter', system-ui, sans-serif; }
.line-clamp-2{display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden}
#toast{transition:all 0.4s cubic-bezier(.4,0,.2,1)}
.tab-pill { position:relative; padding:10px 20px; border-radius:12px; font-size:14px; font-weight:500; color:#64748b; background:transparent; transition:all 0.25s ease; cursor:pointer; border:none; }
.tab-pill:hover { color:#334155; background:#f1f5f9; }
.tab-pill.active { color:#fff; background:linear-gradient(135deg, #3b82f6, #2563eb); box-shadow:0 4px 15px rgba(59,130,246,0.35); }
.card { background:#fff; border-radius:16px; border:1px solid #e2e8f0; box-shadow:0 1px 3px rgba(0,0,0,0.04); transition:box-shadow 0.2s; }
.card:hover { box-shadow:0 4px 20px rgba(0,0,0,0.06); }
.order-row { border:1px solid #e2e8f0; border-radius:12px; padding:14px 16px; margin-bottom:8px; cursor:pointer; transition:all 0.2s ease; background:#fff; }
.order-row:hover { border-color:#93c5fd; background:#eff6ff; transform:translateY(-1px); box-shadow:0 4px 12px rgba(59,130,246,0.1); }
.btn { display:inline-flex; align-items:center; gap:6px; padding:8px 18px; border-radius:10px; font-size:13px; font-weight:600; transition:all 0.2s ease; cursor:pointer; border:none; }
.btn-primary { background:linear-gradient(135deg, #3b82f6, #2563eb); color:#fff; }
.btn-primary:hover { box-shadow:0 4px 15px rgba(59,130,246,0.4); transform:translateY(-1px); }
.btn-secondary { background:#f1f5f9; color:#475569; }
.btn-secondary:hover { background:#e2e8f0; }
.btn-success { background:linear-gradient(135deg, #22c55e, #16a34a); color:#fff; }
.btn-success:hover { box-shadow:0 4px 15px rgba(34,197,94,0.4); }
.btn-danger { background:#fee2e2; color:#dc2626; }
.btn-danger:hover { background:#fecaca; }
.btn-yellow { background:linear-gradient(135deg, #fbbf24, #f59e0b); color:#78350f; }
.btn-yellow:hover { box-shadow:0 4px 15px rgba(245,158,11,0.4); }
.input-field { width:100%; padding:10px 14px; border:1.5px solid #e2e8f0; border-radius:10px; font-size:14px; transition:all 0.2s; background:#fff; outline:none; }
.input-field:focus { border-color:#3b82f6; box-shadow:0 0 0 3px rgba(59,130,246,0.1); }
.input-field::placeholder { color:#94a3b8; }
.badge { display:inline-flex; align-items:center; padding:4px 10px; border-radius:20px; font-size:11px; font-weight:600; letter-spacing:0.3px; }
.modal-overlay { backdrop-filter:blur(4px); background:rgba(15,23,42,0.5); }
.modal-content { border-radius:20px; animation:modalIn 0.3s ease; }
@keyframes modalIn { from { opacity:0; transform:scale(0.95) translateY(10px); } to { opacity:1; transform:scale(1) translateY(0); } }
.section-title { font-size:13px; font-weight:700; text-transform:uppercase; letter-spacing:0.8px; color:#94a3b8; margin-bottom:12px; }
</style>
</head>
<body class="bg-surface-50 min-h-screen">
<div class="max-w-5xl mx-auto px-4 py-6">

  <!-- Header -->
  <div class="card p-5 mb-6">
    <div class="flex items-center justify-between">
      <div class="flex items-center gap-4">
        <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-brand-500 to-brand-700 flex items-center justify-center text-white text-lg shadow-lg shadow-brand-500/25"><i class="fas fa-box"></i></div>
        <div><h1 class="text-lg font-bold text-slate-800">번개장터 주문 관리</h1><p class="text-xs text-slate-400 mt-0.5">Order Management Dashboard</p></div>
      </div>
      <div id="connStatus" class="flex items-center gap-2 text-sm text-slate-400"><div class="w-2 h-2 rounded-full bg-slate-300 animate-pulse"></div>연결 확인 중...</div>
    </div>
  </div>

  <!-- Tabs -->
  <div class="flex flex-wrap gap-2 mb-6 p-1 bg-white rounded-2xl border border-surface-200 shadow-sm">
    <button onclick="switchTab('cancelled')" id="tab-cancelled" class="tab-pill active"><i class="fas fa-rotate-left mr-1.5"></i>취소/환불</button>
    <button onclick="switchTab('all')" id="tab-all" class="tab-pill"><i class="fas fa-list mr-1.5"></i>전체 주문</button>
    <button onclick="switchTab('invoice')" id="tab-invoice" class="tab-pill"><i class="fas fa-truck mr-1.5"></i>송장 검색</button>
    <button onclick="switchTab('products')" id="tab-products" class="tab-pill"><i class="fas fa-search mr-1.5"></i>상품 검색</button>
    <button onclick="switchTab('neworder')" id="tab-neworder" class="tab-pill"><i class="fas fa-cart-plus mr-1.5"></i>수동 주문</button>
    <button onclick="switchTab('address')" id="tab-address" class="tab-pill"><i class="fas fa-address-book mr-1.5"></i>주소록</button>
  </div>

  <!-- 취소/환불 -->
  <div id="panel-cancelled" class="panel">
    <div class="card p-6">
      <div class="flex items-center justify-between mb-4">
        <div><h2 class="text-base font-bold text-slate-800">취소/환불 주문</h2><p class="text-xs text-slate-400 mt-1">취소 및 환불 내역</p></div>
        <div class="flex items-center gap-2">
          <select id="days-cancelled" class="input-field text-sm py-1.5 w-auto">
            <option value="7">7일</option>
            <option value="15">15일</option>
            <option value="30" selected>30일</option>
            <option value="60">60일</option>
            <option value="90">90일</option>
            <option value="180">180일</option>
          </select>
          <button onclick="loadOrders('cancelled')" class="btn btn-primary"><i class="fas fa-sync-alt"></i>조회</button>
        </div>
      </div>
      <div id="cancelledList" class="text-slate-400 text-sm text-center py-12"><i class="fas fa-inbox text-3xl text-slate-200 block mb-3"></i>조회 버튼을 눌러주세요.</div>
    </div>
  </div>

  <!-- 전체 주문 -->
  <div id="panel-all" class="panel hidden">
    <div class="card p-6">
      <div class="flex items-center justify-between mb-4">
        <div><h2 class="text-base font-bold text-slate-800">전체 주문</h2><p class="text-xs text-slate-400 mt-1">전체 주문 내역</p></div>
        <div class="flex items-center gap-2">
          <select id="days-all" class="input-field text-sm py-1.5 w-auto">
            <option value="7">7일</option>
            <option value="15">15일</option>
            <option value="30" selected>30일</option>
            <option value="60">60일</option>
            <option value="90">90일</option>
            <option value="180">180일</option>
          </select>
          <button onclick="loadOrders('all')" class="btn btn-primary"><i class="fas fa-sync-alt"></i>조회</button>
        </div>
      </div>
      <div id="allList" class="text-slate-400 text-sm text-center py-12"><i class="fas fa-inbox text-3xl text-slate-200 block mb-3"></i>조회 버튼을 눌러주세요.</div>
    </div>
  </div>

  <!-- 송장 검색 -->
  <div id="panel-invoice" class="panel hidden">
    <div class="card p-6">
      <div class="mb-4"><h2 class="text-base font-bold text-slate-800">송장번호로 주문 검색</h2><p class="text-xs text-slate-400 mt-1">처음 검색 시 주문 목록을 자동으로 불러옵니다.</p></div>
      <div class="flex gap-2 mb-4">
        <input id="invoiceInput" type="text" placeholder="송장번호를 입력하세요" class="input-field flex-1" onkeydown="if(event.key==='Enter')searchInvoice()">
        <button onclick="searchInvoice()" class="btn btn-primary"><i class="fas fa-search"></i>검색</button>
      </div>
      <div id="invoiceProgress" class="hidden text-sm text-slate-500 mb-2"></div>
      <div id="invoiceResult" class="text-slate-400 text-sm text-center py-12"><i class="fas fa-truck text-3xl text-slate-200 block mb-3"></i>송장번호를 입력하고 검색하세요.</div>
    </div>
  </div>

  <!-- 상품 검색 -->
  <div id="panel-products" class="panel hidden">
    <div class="card p-6">
      <div class="mb-4"><h2 class="text-base font-bold text-slate-800">상품 검색</h2><p class="text-xs text-slate-400 mt-1">상품명 또는 PID로 검색하세요.</p></div>
      <div class="flex gap-2 mb-4">
        <input id="productSearch" type="text" placeholder="상품명 또는 PID" class="input-field flex-1" onkeydown="if(event.key==='Enter')searchProducts()">
        <button onclick="searchProducts()" class="btn btn-primary"><i class="fas fa-search"></i>검색</button>
      </div>
      <div id="productList" class="text-slate-400 text-sm text-center py-12"><i class="fas fa-box-open text-3xl text-slate-200 block mb-3"></i>검색어를 입력하세요.</div>
    </div>
  </div>

  <!-- 수동 주문 -->
  <div id="panel-neworder" class="panel hidden">
    <div class="card p-6">
      <div class="mb-5"><h2 class="text-base font-bold text-slate-800">수동 주문 생성</h2><p class="text-xs text-slate-400 mt-1">상품을 조회하고 주문을 생성합니다.</p></div>
      <div class="section-title">상품 정보</div>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
        <div>
          <label class="block text-xs font-semibold text-slate-500 mb-1.5">상품 PID</label>
          <div class="flex gap-2"><input id="orderPid" type="text" class="input-field flex-1" placeholder="상품 PID 입력"><button onclick="lookupProduct()" class="btn btn-secondary"><i class="fas fa-search"></i>조회</button></div>
          <div id="productPreview" class="mt-2 text-sm"></div>
        </div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">수량</label><input id="orderQty" type="number" value="1" min="1" class="input-field"></div>
      </div>
      <div class="section-title">배송 정보</div>
      <div class="mb-4">
        <label class="block text-xs font-semibold text-slate-500 mb-1.5">저장된 주소 선택</label>
        <select id="addressSelect" onchange="applyAddress()" class="input-field"><option value="">-- 직접 입력 --</option></select>
      </div>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">수령인 이름</label><input id="shipName" type="text" class="input-field"></div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">연락처</label><input id="shipPhone" type="text" class="input-field" placeholder="01012345678"></div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">우편번호</label>
          <div class="flex gap-2"><input id="shipZip" type="text" class="input-field flex-1" readonly><button onclick="openPostcode('ship')" class="btn btn-yellow"><i class="fas fa-search"></i>주소 검색</button></div></div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">주소</label><input id="shipAddr1" type="text" class="input-field" readonly></div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">상세주소</label><input id="shipAddr2" type="text" class="input-field" placeholder="동/호수 입력"></div>
        <div><label class="block text-xs font-semibold text-slate-500 mb-1.5">배송 메모</label><input id="shipExtra" type="text" class="input-field"></div>
      </div>
      <div class="mt-5 flex gap-2">
        <button onclick="previewOrder()" class="btn btn-secondary"><i class="fas fa-eye"></i>미리보기</button>
        <button onclick="submitOrder()" class="btn btn-primary"><i class="fas fa-paper-plane"></i>주문 생성</button>
      </div>
      <div id="orderResult" class="mt-4 text-sm"></div>
    </div>
  </div>

  <!-- 주소록 -->
  <div id="panel-address" class="panel hidden">
    <div class="card p-6">
      <div class="mb-5"><h2 class="text-base font-bold text-slate-800"><i class="fas fa-address-book text-brand-500 mr-2"></i>주소록 관리</h2><p class="text-xs text-slate-400 mt-1">자주 쓰는 배송지를 저장하고 수동 주문 시 빠르게 선택하세요.</p></div>
      <div class="bg-gradient-to-br from-brand-50 to-surface-100 rounded-2xl p-5 mb-5 border border-brand-100">
        <div class="section-title" style="color:#3b82f6">새 주소 추가</div>
        <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">별칭 (필수)</label><input id="addrLabel" type="text" class="input-field text-sm" placeholder="예: 대만 본사"></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">수령인</label><input id="addrName" type="text" class="input-field text-sm"></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">연락처</label><input id="addrPhone" type="text" class="input-field text-sm" placeholder="01012345678"></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">우편번호</label>
            <div class="flex gap-1"><input id="addrZip" type="text" class="input-field text-sm flex-1" readonly><button onclick="openPostcode('addr')" class="btn btn-yellow text-xs py-2 px-3"><i class="fas fa-search"></i></button></div></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">주소</label><input id="addrAddr1" type="text" class="input-field text-sm" readonly></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">상세주소</label><input id="addrAddr2" type="text" class="input-field text-sm" placeholder="동/호수"></div>
          <div><label class="block text-xs font-semibold text-slate-500 mb-1">배송 메모</label><input id="addrExtra" type="text" class="input-field text-sm"></div>
        </div>
        <button onclick="addAddress()" class="mt-3 btn btn-primary"><i class="fas fa-plus"></i>주소 추가</button>
      </div>
      <div class="section-title">저장된 주소</div>
      <div id="addressList" class="text-slate-400 text-sm text-center py-8"><i class="fas fa-map-marker-alt text-3xl text-slate-200 block mb-3"></i>저장된 주소가 없습니다.</div>
    </div>
  </div>

  <!-- Modal -->
  <div id="orderModal" class="hidden fixed inset-0 modal-overlay z-50 flex items-center justify-center p-4">
    <div class="modal-content bg-white shadow-2xl max-w-2xl w-full max-h-[90vh] overflow-y-auto p-6">
      <div class="flex justify-between items-center mb-5">
        <h3 class="font-bold text-lg text-slate-800" id="modalTitle">주문 상세</h3>
        <button onclick="closeModal()" class="w-8 h-8 rounded-lg bg-surface-100 hover:bg-surface-200 flex items-center justify-center text-slate-400 hover:text-slate-600 transition">&times;</button>
      </div>
      <div id="orderDetail"></div>
    </div>
  </div>

  <!-- Toast -->
  <div id="toast" class="fixed bottom-6 right-6 bg-slate-800 text-white px-5 py-3 rounded-xl shadow-2xl opacity-0 pointer-events-none text-sm font-medium z-50" style="transform:translateY(10px)"></div>
</div>

<script>
var currentTab = 'cancelled';
var allOrdersCache = [];
var cachedProductInfo = null;

document.addEventListener('DOMContentLoaded', function() {
  checkConnection();
  renderAddressList();
  refreshAddressSelect();
});

function apiFetch(url, opts) {
  opts = opts || {};
  opts.headers = opts.headers || {};
  opts.headers['Content-Type'] = 'application/json';
  return fetch(url, opts).then(function(r) {
    if (r.status === 204) return { success: true };
    return r.json().then(function(data) {
      if (!r.ok) throw new Error(data.error || data.message || ('HTTP ' + r.status));
      return data;
    });
  });
}

function checkConnection() {
  apiFetch('/api/settings/check').then(function(d) {
    var el = document.getElementById('connStatus');
    if (d.hasAccessKey && d.hasSecretKey) {
      el.innerHTML = '<div class="w-2 h-2 rounded-full bg-emerald-400 shadow-lg shadow-emerald-400/50"></div><span class="text-emerald-600 font-medium">API 연결됨</span>';
    } else {
      el.innerHTML = '<div class="w-2 h-2 rounded-full bg-red-400"></div><span class="text-red-500 font-medium">API 키 미설정</span>';
    }
  }).catch(function() {
    document.getElementById('connStatus').innerHTML = '<div class="w-2 h-2 rounded-full bg-red-400"></div><span class="text-red-500">연결 실패</span>';
  });
}

function toast(msg, dur) {
  var el = document.getElementById('toast');
  el.textContent = msg;
  el.style.opacity = '1';
  el.style.transform = 'translateY(0)';
  setTimeout(function() { el.style.opacity = '0'; el.style.transform = 'translateY(10px)'; }, dur || 3000);
}

function switchTab(name) {
  currentTab = name;
  var tabs = ['cancelled','all','invoice','products','neworder','address'];
  for (var i = 0; i < tabs.length; i++) {
    var t = tabs[i];
    document.getElementById('tab-' + t).className = (t === name) ? 'tab-pill active' : 'tab-pill';
    document.getElementById('panel-' + t).className = (t === name) ? 'panel' : 'panel hidden';
  }
  if (name === 'neworder') refreshAddressSelect();
}

function getStatusLabel(s) {
  var m = {'PAYMENT_RECEIVED':'결제완료','SHIP_READY':'배송준비','IN_TRANSIT':'배송중','DELIVERED':'배송완료','PURCHASE_CONFIRM':'구매확정','CANCELLED':'취소','REFUNDED':'환불','RETURN_REQUESTED':'반품요청','RETURN_IN_TRANSIT':'반품배송중','RETURN_COMPLETED':'반품완료'};
  return m[s] || s;
}
function getStatusClass(s) {
  if (s === 'CANCELLED' || s === 'REFUNDED') return 'badge bg-red-50 text-red-600';
  if (s === 'RETURN_REQUESTED' || s === 'RETURN_IN_TRANSIT' || s === 'RETURN_COMPLETED') return 'badge bg-orange-50 text-orange-600';
  if (s === 'PURCHASE_CONFIRM' || s === 'DELIVERED') return 'badge bg-emerald-50 text-emerald-600';
  if (s === 'IN_TRANSIT') return 'badge bg-blue-50 text-blue-600';
  if (s === 'SHIP_READY') return 'badge bg-amber-50 text-amber-600';
  return 'badge bg-slate-50 text-slate-600';
}
function escapeHtml(t) { if (!t) return ''; return String(t).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;'); }
function formatDate(d) {
  if (!d) return '-';
  try { var dt = new Date(d); return dt.getFullYear() + '-' + pad2(dt.getMonth()+1) + '-' + pad2(dt.getDate()) + ' ' + pad2(dt.getHours()) + ':' + pad2(dt.getMinutes()); } catch(e) { return d; }
}
function pad2(n) { return n < 10 ? '0' + n : '' + n; }

/* ══ Kakao Postcode ══ */
function openPostcode(target) {
  new daum.Postcode({
    oncomplete: function(data) {
      var addr = data.userSelectedType === 'R' ? data.roadAddress : data.jibunAddress;
      if (target === 'ship') {
        document.getElementById('shipZip').value = data.zonecode;
        document.getElementById('shipAddr1').value = addr;
        document.getElementById('shipAddr2').value = '';
        document.getElementById('shipAddr2').focus();
      } else if (target === 'addr') {
        document.getElementById('addrZip').value = data.zonecode;
        document.getElementById('addrAddr1').value = addr;
        document.getElementById('addrAddr2').value = '';
        document.getElementById('addrAddr2').focus();
      }
    }
  }).open();
}

/* ══ Address Book ══ */
function getAddresses() { try { return JSON.parse(localStorage.getItem('bunjang_addresses') || '[]'); } catch(e) { return []; } }
function saveAddresses(list) { localStorage.setItem('bunjang_addresses', JSON.stringify(list)); }

function addAddress() {
  var label = document.getElementById('addrLabel').value.trim();
  if (!label) { toast('별칭을 입력하세요'); return; }
  var addr = { label: label, name: document.getElementById('addrName').value.trim(), phone: document.getElementById('addrPhone').value.trim(), zip: document.getElementById('addrZip').value.trim(), addr1: document.getElementById('addrAddr1').value.trim(), addr2: document.getElementById('addrAddr2').value.trim(), extra: document.getElementById('addrExtra').value.trim() };
  var list = getAddresses(); list.push(addr); saveAddresses(list);
  toast('"' + label + '" 주소 추가됨');
  var ids = ['addrLabel','addrName','addrPhone','addrZip','addrAddr1','addrAddr2','addrExtra'];
  for (var i = 0; i < ids.length; i++) document.getElementById(ids[i]).value = '';
  renderAddressList(); refreshAddressSelect();
}

function deleteAddress(idx) {
  var list = getAddresses();
  if (!confirm('"' + list[idx].label + '" 주소를 삭제하시겠습니까?')) return;
  list.splice(idx, 1); saveAddresses(list); toast('삭제됨');
  renderAddressList(); refreshAddressSelect();
}

function editAddress(idx) {
  var list = getAddresses(); var a = list[idx];
  document.getElementById('addrLabel').value = a.label || '';
  document.getElementById('addrName').value = a.name || '';
  document.getElementById('addrPhone').value = a.phone || '';
  document.getElementById('addrZip').value = a.zip || '';
  document.getElementById('addrAddr1').value = a.addr1 || '';
  document.getElementById('addrAddr2').value = a.addr2 || '';
  document.getElementById('addrExtra').value = a.extra || '';
  list.splice(idx, 1); saveAddresses(list);
  renderAddressList(); refreshAddressSelect();
  toast('수정할 주소를 불러왔습니다.');
}

function renderAddressList() {
  var list = getAddresses(); var el = document.getElementById('addressList');
  if (list.length === 0) { el.innerHTML = '<div class="text-center py-8"><i class="fas fa-map-marker-alt text-3xl text-slate-200 block mb-3"></i><span class="text-slate-400">저장된 주소가 없습니다.</span></div>'; return; }
  var h = '';
  for (var i = 0; i < list.length; i++) {
    var a = list[i];
    h += '<div class="order-row flex justify-between items-start">';
    h += '<div><div class="font-semibold text-slate-800"><i class="fas fa-map-marker-alt text-brand-400 mr-1.5"></i>' + escapeHtml(a.label) + '</div>';
    h += '<div class="text-sm text-slate-500 mt-1">' + escapeHtml(a.name) + ' / ' + escapeHtml(a.phone) + '</div>';
    h += '<div class="text-sm text-slate-400">[' + escapeHtml(a.zip) + '] ' + escapeHtml(a.addr1) + ' ' + escapeHtml(a.addr2) + '</div>';
    if (a.extra) h += '<div class="text-xs text-slate-300 mt-1">메모: ' + escapeHtml(a.extra) + '</div>';
    h += '</div><div class="flex gap-1 shrink-0">';
    h += '<button onclick="editAddress(' + i + ')" class="btn btn-secondary text-xs py-1.5 px-2.5"><i class="fas fa-edit"></i></button>';
    h += '<button onclick="deleteAddress(' + i + ')" class="btn btn-danger text-xs py-1.5 px-2.5"><i class="fas fa-trash"></i></button>';
    h += '</div></div>';
  }
  el.innerHTML = h;
}

function refreshAddressSelect() {
  var sel = document.getElementById('addressSelect'); if (!sel) return;
  var list = getAddresses();
  var h = '<option value="">-- 직접 입력 --</option>';
  for (var i = 0; i < list.length; i++) h += '<option value="' + i + '">' + escapeHtml(list[i].label) + ' (' + escapeHtml(list[i].name) + ')</option>';
  sel.innerHTML = h;
}

function applyAddress() {
  var sel = document.getElementById('addressSelect'); var idx = sel.value;
  if (idx === '') return;
  var list = getAddresses(); var a = list[parseInt(idx)]; if (!a) return;
  document.getElementById('shipName').value = a.name || '';
  document.getElementById('shipPhone').value = a.phone || '';
  document.getElementById('shipZip').value = a.zip || '';
  document.getElementById('shipAddr1').value = a.addr1 || '';
  document.getElementById('shipAddr2').value = a.addr2 || '';
  document.getElementById('shipExtra').value = a.extra || '';
  toast('"' + a.label + '" 주소 적용됨');
}

/* ══ Orders ══ */
function loadOrders(type) {
  var container = (type === 'cancelled') ? document.getElementById('cancelledList') : document.getElementById('allList');
  var daysEl = document.getElementById('days-' + type);
  var days = daysEl ? daysEl.value : '30';
  container.innerHTML = '<div class="text-center py-12"><div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-brand-50 mb-3"><i class="fas fa-spinner fa-spin text-xl text-brand-500"></i></div><div class="text-slate-500 text-sm">주문 로딩 중... (최근 ' + days + '일)</div></div>';
  apiFetch('/api/orders/extended?days=' + days).then(function(d) {
    var orders = d.data || []; allOrdersCache = orders;
    if (type === 'cancelled') {
      orders = orders.filter(function(o) { return o.orderItems && o.orderItems.some(function(it) {
        return it.status === 'CANCELLED' || it.status === 'REFUNDED' || it.status === 'RETURN_REQUESTED' || it.status === 'RETURN_IN_TRANSIT' || it.status === 'RETURN_COMPLETED';
      }); });
    }
    if (orders.length === 0) { container.innerHTML = '<div class="text-center py-12"><i class="fas fa-inbox text-3xl text-slate-200 block mb-3"></i><span class="text-slate-400">주문이 없습니다.</span></div>'; return; }
    var h = '';
    for (var i = 0; i < orders.length; i++) {
      var o = orders[i]; var items = o.orderItems || []; var statusBadges = ''; var productIds = '';
      for (var j = 0; j < items.length; j++) {
        statusBadges += '<span class="' + getStatusClass(items[j].status) + '">' + getStatusLabel(items[j].status) + '</span> ';
        productIds += (j > 0 ? ', ' : '') + items[j].product.id;
      }
      var dateStr = items.length > 0 ? formatDate(items[0].statusUpdatedAt) : '-';
      h += '<div class="order-row flex justify-between items-center" onclick="viewOrder(' + o.id + ')">';
      h += '<div><div class="font-semibold text-slate-800">주문 #' + o.id + '</div>';
      h += '<div class="text-xs text-slate-400 mt-1"><i class="fas fa-box text-slate-300 mr-1"></i>' + escapeHtml(productIds) + '</div>';
      h += '<div class="text-xs text-slate-300 mt-0.5"><i class="fas fa-clock text-slate-200 mr-1"></i>' + dateStr + '</div></div>';
      h += '<div class="flex flex-wrap gap-1 justify-end">' + statusBadges + '</div></div>';
    }
    container.innerHTML = h; toast(orders.length + '건 로드 완료');
  }).catch(function(e) { container.innerHTML = '<div class="text-center py-12 text-red-500"><i class="fas fa-exclamation-circle text-2xl block mb-2"></i>오류: ' + escapeHtml(e.message) + '</div>'; });
}

/* ══ Order Detail ══ */
function viewOrder(orderId) {
  document.getElementById('modalTitle').textContent = '주문 상세';
  document.getElementById('orderModal').classList.remove('hidden');
  document.getElementById('orderDetail').innerHTML = '<div class="text-center py-12"><div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-brand-50 mb-3"><i class="fas fa-spinner fa-spin text-xl text-brand-500"></i></div></div>';
  apiFetch('/api/orders/' + orderId).then(function(d) {
    var data = d.data || d; var order = data.order || data; var seller = data.seller || {};
    var delivery = data.delivery || {}; var returns = data.returns || []; var items = order.orderItems || [];
    var h = '<div class="space-y-4">';

    // 주문 정보
    h += '<div class="bg-surface-50 rounded-xl p-4"><div class="section-title">주문 정보</div><div class="grid grid-cols-2 gap-3 text-sm">';
    h += '<div class="text-slate-500">주문 ID</div><div class="font-semibold text-slate-800">' + order.id + '</div>';
    h += '<div class="text-slate-500">총 금액</div><div class="font-bold text-brand-600">' + Number(order.totalPrice || 0).toLocaleString() + '원</div>';
    h += '<div class="text-slate-500">상품 금액</div><div>' + Number(order.totalProductPrice || 0).toLocaleString() + '원</div>';
    h += '<div class="text-slate-500">배송비</div><div>' + Number(order.deliveryPrice || 0).toLocaleString() + '원</div>';
    h += '<div class="text-slate-500">주문일</div><div>' + formatDate(order.orderDoneAt) + '</div>';
    h += '<div class="text-slate-500">승인일</div><div>' + formatDate(order.approvedAt) + '</div>';
    h += '</div></div>';

    // 주문 아이템 (이미지 포함)
    h += '<div class="bg-surface-50 rounded-xl p-4"><div class="section-title">주문 아이템 (' + items.length + ')</div>';
    for (var i = 0; i < items.length; i++) {
      var it = items[i];
      var pid = (it.product ? it.product.id : '');
      h += '<div class="border-b border-surface-200 pb-3 mb-3 last:border-0 last:pb-0 last:mb-0 text-sm">';
      h += '<div class="flex gap-3">';
      h += '<div id="orderItemImg-' + i + '" class="shrink-0"></div>';
      h += '<div class="flex-1">';
      h += '<span class="' + getStatusClass(it.status) + '">' + getStatusLabel(it.status) + '</span>';
      h += ' <span class="text-slate-600 ml-1">상품 #' + pid + '</span>';
      if (it.product && it.product.name) h += '<span class="text-slate-500"> - ' + escapeHtml(it.product.name) + '</span>';
      if (it.product && it.product.price) h += '<span class="font-semibold text-brand-600 ml-1">' + Number(it.product.price).toLocaleString() + '원</span>';
      h += '<div class="text-xs text-slate-400 mt-1">상태 변경: ' + formatDate(it.statusUpdatedAt) + '</div>';
      if (it.purchaseConfirmedAt) h += '<div class="text-xs text-emerald-500">구매확정: ' + formatDate(it.purchaseConfirmedAt) + '</div>';
      if (it.refundedAt) h += '<div class="text-xs text-red-500">환불: ' + formatDate(it.refundedAt) + '</div>';
      if (pid) h += '<a href="https://m.bunjang.co.kr/products/' + pid + '" target="_blank" class="inline-flex items-center gap-1 text-xs text-brand-500 hover:text-brand-700 mt-1"><i class="fas fa-external-link-alt"></i>번개장터에서 보기</a>';
      h += '</div></div></div>';
    }
    h += '</div>';

    // 배송 정보
    if (delivery.invoice || delivery.address) {
      h += '<div class="bg-blue-50 rounded-xl p-4"><div class="section-title" style="color:#3b82f6">배송 정보</div><div class="text-sm">';
      if (delivery.invoice) h += '<div class="mb-1">송장: <strong class="text-brand-600">' + escapeHtml(delivery.invoice.no) + '</strong> <span class="text-slate-400">(' + escapeHtml(delivery.invoice.companyName || delivery.invoice.companyCode || '') + ')</span></div>';
      if (delivery.shipDoneAt) h += '<div class="text-slate-500">발송일: ' + formatDate(delivery.shipDoneAt) + '</div>';
      if (delivery.address) {
        h += '<div class="mt-2 text-slate-600">' + escapeHtml(delivery.address.name) + ' / ' + escapeHtml(delivery.address.phone) + '</div>';
        h += '<div class="text-slate-500">[' + escapeHtml(delivery.address.zipCode) + '] ' + escapeHtml(delivery.address.address1) + ' ' + escapeHtml(delivery.address.address2) + '</div>';
      }
      h += '</div></div>';
    }

    // 반품 정보
    if (returns.length > 0) {
      h += '<div class="bg-red-50 rounded-xl p-4"><div class="section-title" style="color:#dc2626">반품 정보 (' + returns.length + ')</div>';
      for (var r = 0; r < returns.length; r++) {
        var ret = returns[r];
        h += '<div class="text-sm border-b border-red-100 pb-2 mb-2 last:border-0"><div>반품 ID: ' + ret.id + '</div>';
        if (ret.invoice) h += '<div>반품 송장: <strong class="text-red-600">' + escapeHtml(ret.invoice.no) + '</strong> (' + escapeHtml(ret.invoice.companyName || '') + ')</div>';
        if (ret.shipDoneAt) h += '<div class="text-slate-500">반품 발송: ' + formatDate(ret.shipDoneAt) + '</div>';
        h += '</div>';
      }
      h += '</div>';
    }

    // 판매자 + 링크
    if (seller.id) {
      h += '<div class="bg-surface-50 rounded-xl p-4 text-sm"><div class="section-title">판매자</div>';
      h += '<div class="text-slate-600 mb-3">ID: ' + seller.id + (seller.shopName ? (' / ' + escapeHtml(seller.shopName)) : '') + '</div>';
      h += '<div class="flex flex-wrap gap-2">';
      h += '<a href="https://m.bunjang.co.kr/shop/' + seller.id + '/products" target="_blank" class="btn btn-secondary text-xs"><i class="fas fa-store"></i>판매자 상점</a>';
      var chatPid = (items.length > 0 && items[0].product) ? items[0].product.id : '';
      if (chatPid) h += '<a href="https://m.bunjang.co.kr/products/' + chatPid + '" target="_blank" class="btn btn-primary text-xs"><i class="fas fa-comment-dots"></i>상품 페이지 (번개톡)</a>';
      h += '</div></div>';
    }

    // 액션 버튼
    h += '<div class="flex gap-2 pt-3">';
    var firstPid = (items.length > 0 && items[0].product) ? items[0].product.id : '';
    if (firstPid) h += '<button onclick="reorder(' + firstPid + ')" class="btn btn-primary"><i class="fas fa-redo"></i>재주문</button>';
    var hasCancel = items.some(function(it) { return it.status === 'CANCELLED' || it.status === 'REFUNDED' || it.status === 'RETURN_REQUESTED' || it.status === 'RETURN_IN_TRANSIT' || it.status === 'RETURN_COMPLETED'; });
    if (!hasCancel) h += '<button onclick="confirmPurchase(' + order.id + ')" class="btn btn-success"><i class="fas fa-check"></i>구매확정</button>';
    h += '<button onclick="closeModal()" class="btn btn-secondary"><i class="fas fa-times"></i>닫기</button>';
    h += '</div></div>';

    document.getElementById('orderDetail').innerHTML = h;

    // 비동기 이미지 로딩
    for (var k = 0; k < items.length; k++) {
      (function(idx, prodId) {
        if (!prodId) return;
        apiFetch('/api/products/' + prodId).then(function(pd) {
          var pp = pd.data || pd;
          var imgUrl = (pp.imageUrls && pp.imageUrls.length > 0) ? pp.imageUrls[0] : (pp.imageUrlTemplate ? pp.imageUrlTemplate.replace('{}', '1') : '');
          var el = document.getElementById('orderItemImg-' + idx);
          if (el && imgUrl) {
            el.innerHTML = '<img src="' + escapeHtml(imgUrl) + '" class="w-20 h-20 object-cover rounded-lg shadow-sm cursor-pointer" onclick="showProductInModal(' + prodId + ')" onerror="this.remove()">';
          }
        }).catch(function() {});
      })(k, items[k].product ? items[k].product.id : null);
    }
  }).catch(function(e) { document.getElementById('orderDetail').innerHTML = '<div class="text-red-500 text-center py-8"><i class="fas fa-exclamation-circle text-2xl block mb-2"></i>오류: ' + escapeHtml(e.message) + '</div>'; });
}

function closeModal() { document.getElementById('orderModal').classList.add('hidden'); }

function showProductInModal(pid) {
  document.getElementById('modalTitle').textContent = '상품 상세';
  apiFetch('/api/products/' + pid).then(function(d) {
    var p = d.data || d;
    var imgs = p.imageUrls || [];
    if (imgs.length === 0 && p.imageUrlTemplate) imgs.push(p.imageUrlTemplate.replace('{}', '1'));
    var condLabel = {'NEW':'새상품','LIKE_NEW':'거의 새것','USED':'중고','DAMAGED':'하자 있음'};
    var saleLabel = {'SELLING':'판매중','RESERVED':'예약중','SOLD_OUT':'판매완료'};
    var h = '<div class="space-y-4">';
    if (imgs.length > 0) {
      h += '<div class="flex overflow-x-auto snap-x gap-2 pb-2 -mx-2 px-2" style="scroll-snap-type:x mandatory;-webkit-overflow-scrolling:touch">';
      for (var i = 0; i < imgs.length; i++) {
        h += '<img src="' + escapeHtml(imgs[i]) + '" class="flex-shrink-0 max-h-80 rounded-xl object-contain snap-center" style="scroll-snap-align:center" onerror="this.remove()">';
      }
      h += '</div>';
      if (imgs.length > 1) h += '<div class="text-xs text-slate-400 text-center"><i class="fas fa-images mr-1"></i>' + imgs.length + '장 (좌우 스크롤)</div>';
    }
    h += '<div class="font-bold text-lg text-slate-800">' + escapeHtml(p.name) + '</div>';
    h += '<div class="text-2xl font-bold text-brand-600">' + Number(p.price || 0).toLocaleString() + '원</div>';
    h += '<div class="grid grid-cols-2 gap-3 text-sm">';
    if (p.condition) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">상태</div><div class="font-semibold text-slate-700 mt-0.5">' + (condLabel[p.condition] || p.condition) + '</div></div>';
    if (p.quantity !== undefined) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">수량</div><div class="font-semibold text-slate-700 mt-0.5">' + p.quantity + '개</div></div>';
    if (p.shippingFee !== undefined) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">배송비</div><div class="font-semibold text-slate-700 mt-0.5">' + Number(p.shippingFee).toLocaleString() + '원</div></div>';
    if (p.saleStatus) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">판매상태</div><div class="font-semibold text-slate-700 mt-0.5">' + (saleLabel[p.saleStatus] || p.saleStatus) + '</div></div>';
    h += '</div>';
    if (p.description) h += '<div class="p-3 bg-surface-50 rounded-lg text-sm text-slate-600 whitespace-pre-wrap max-h-40 overflow-y-auto">' + escapeHtml(p.description).replace(/\\\\n/g, '<br>').replace(/\\n/g, '<br>') + '</div>';
    h += '<div class="flex gap-2"><button onclick="useProduct(' + (p.pid || p.id) + ')" class="btn btn-primary"><i class="fas fa-cart-plus"></i>이 상품으로 주문</button>';
    h += '<a href="https://m.bunjang.co.kr/products/' + (p.pid || p.id) + '" target="_blank" class="btn btn-secondary"><i class="fas fa-external-link-alt"></i>번개장터에서 보기</a></div></div>';
    document.getElementById('orderDetail').innerHTML = h;
  }).catch(function(e) { toast('상품 조회 실패: ' + e.message); });
}

function confirmPurchase(orderId) {
  if (!confirm('주문 #' + orderId + ' 구매확정 하시겠습니까?')) return;
  apiFetch('/api/orders/' + orderId + '/confirm', { method: 'POST' }).then(function() { toast('구매확정 완료!'); closeModal(); }).catch(function(e) { toast('오류: ' + e.message, 5000); });
}

function reorder(pid) { closeModal(); switchTab('neworder'); document.getElementById('orderPid').value = pid; lookupProduct(); }

/* ══ Invoice Search ══ */
function searchInvoice() {
  var inv = document.getElementById('invoiceInput').value.trim();
  if (!inv) { toast('송장번호를 입력하세요'); return; }
  var container = document.getElementById('invoiceResult');
  var progress = document.getElementById('invoiceProgress');
  if (allOrdersCache.length === 0) {
    container.innerHTML = '<div class="text-center py-8"><div class="inline-flex items-center justify-center w-10 h-10 rounded-full bg-brand-50 mb-3"><i class="fas fa-spinner fa-spin text-brand-500"></i></div><div class="text-sm text-slate-500">주문 목록을 먼저 불러오는 중...</div></div>';
    apiFetch('/api/orders/extended?days=90').then(function(d) { allOrdersCache = d.data || []; doInvoiceSearch(inv, container, progress); }).catch(function(e) { container.innerHTML = '<div class="text-red-500">주문 로드 실패: ' + escapeHtml(e.message) + '</div>'; });
  } else { doInvoiceSearch(inv, container, progress); }
}

function doInvoiceSearch(inv, container, progress) {
  var ids = allOrdersCache.map(function(o) { return o.id; });
  container.innerHTML = '<div class="text-center py-8"><div class="inline-flex items-center justify-center w-10 h-10 rounded-full bg-brand-50 mb-3"><i class="fas fa-spinner fa-spin text-brand-500"></i></div><div class="text-sm text-slate-500">' + ids.length + '건의 주문에서 검색 중...</div></div>';
  progress.className = 'text-sm text-slate-500 mb-2'; progress.textContent = '검색 대상: ' + ids.length + '건';
  apiFetch('/api/orders/search-by-invoice-fast', { method: 'POST', body: JSON.stringify({ invoiceNo: inv, orderIds: ids }) }).then(function(d) {
    progress.className = 'hidden'; var matched = d.data || [];
    if (matched.length === 0) { container.innerHTML = '<div class="text-center py-12"><i class="fas fa-search text-3xl text-slate-200 block mb-3"></i><span class="text-slate-400">일치하는 주문이 없습니다.</span></div>'; return; }
    renderInvoiceResults(container, matched); toast(matched.length + '건 찾음');
  }).catch(function(e) { progress.className = 'hidden'; container.innerHTML = '<div class="text-red-500">오류: ' + escapeHtml(e.message) + '</div>'; });
}

function renderInvoiceResults(container, orders) {
  var h = '';
  for (var i = 0; i < orders.length; i++) {
    var d = orders[i]; var order = d.order || d; var delivery = d.delivery || {}; var returns = d.returns || []; var items = order.orderItems || [];
    h += '<div class="order-row" onclick="viewOrder(' + order.id + ')">';
    h += '<div class="flex justify-between items-center"><div class="font-semibold text-slate-800">주문 #' + order.id + '</div><div class="font-bold text-brand-600">' + Number(order.totalPrice || 0).toLocaleString() + '원</div></div>';
    if (delivery.invoice) h += '<div class="text-sm mt-1.5"><i class="fas fa-truck text-brand-400 mr-1"></i>배송 송장: <strong class="text-brand-600">' + escapeHtml(delivery.invoice.no) + '</strong> <span class="text-slate-400">(' + escapeHtml(delivery.invoice.companyName || '') + ')</span></div>';
    for (var r = 0; r < returns.length; r++) { if (returns[r].invoice) h += '<div class="text-sm text-red-500"><i class="fas fa-undo mr-1"></i>반품 송장: <strong>' + escapeHtml(returns[r].invoice.no) + '</strong></div>'; }
    h += '<div class="mt-2 flex flex-wrap gap-1">';
    for (var j = 0; j < items.length; j++) h += '<span class="' + getStatusClass(items[j].status) + '">' + getStatusLabel(items[j].status) + '</span>';
    h += '</div></div>';
  }
  container.innerHTML = h;
}

/* ══ Products ══ */
function searchProducts() {
  var kw = document.getElementById('productSearch').value.trim(); if (!kw) { toast('검색어를 입력하세요'); return; }
  var container = document.getElementById('productList');
  container.innerHTML = '<div class="text-center py-12"><div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-brand-50 mb-3"><i class="fas fa-spinner fa-spin text-xl text-brand-500"></i></div></div>';
  var url = /^[0-9]+$/.test(kw) ? '/api/products/' + kw : '/api/products/search?keyword=' + encodeURIComponent(kw);
  apiFetch(url).then(function(d) {
    if (d.data && Array.isArray(d.data)) renderProductList(container, d.data);
    else if (d.data && (d.data.pid || d.data.id)) renderProductDetail(container, d.data);
    else if (d.pid || d.id) renderProductDetail(container, d);
    else container.innerHTML = '<div class="text-center py-12 text-slate-400">결과가 없습니다.</div>';
  }).catch(function(e) { container.innerHTML = '<div class="text-red-500">오류: ' + escapeHtml(e.message) + '</div>'; });
}

function renderProductList(container, products) {
  if (!products || products.length === 0) { container.innerHTML = '<div class="text-center py-12 text-slate-400">결과가 없습니다.</div>'; return; }
  var h = '<div class="grid grid-cols-1 md:grid-cols-2 gap-3">';
  for (var i = 0; i < products.length; i++) {
    var p = products[i]; var imgUrl = p.imageUrlTemplate ? p.imageUrlTemplate.replace('{}', '1') : '';
    h += '<div class="order-row flex gap-3" onclick="useProduct(' + p.pid + ')">';
    if (imgUrl) h += '<img src="' + escapeHtml(imgUrl) + '" class="w-16 h-16 object-cover rounded-lg shadow-sm" onerror="this.remove()">';
    h += '<div class="flex-1 min-w-0"><div class="font-semibold text-sm text-slate-800 line-clamp-2">' + escapeHtml(p.name) + '</div>';
    h += '<div class="text-xs text-slate-400 mt-1">PID: ' + p.pid + '</div>';
    h += '<div class="text-sm font-bold text-brand-600 mt-1">' + Number(p.price || 0).toLocaleString() + '원</div></div></div>';
  }
  h += '</div>'; container.innerHTML = h;
}

function renderProductDetail(container, p) {
  var imgs = p.imageUrls || [];
  var imgTpl = p.imageUrlTemplate || '';
  var imgCount = p.imageCount || 0;
  if (imgs.length === 0 && imgTpl && imgCount > 0) {
    for (var ii = 1; ii <= imgCount; ii++) imgs.push(imgTpl.replace('{}', '' + ii));
  }
  if (imgs.length === 0 && imgTpl) imgs.push(imgTpl.replace('{}', '1'));
  var condLabel = {'NEW':'새상품','LIKE_NEW':'거의 새것','USED':'중고','DAMAGED':'하자 있음'};
  var saleLabel = {'SELLING':'판매중','RESERVED':'예약중','SOLD_OUT':'판매완료'};

  var h = '<div class="card overflow-hidden">';
  if (imgs.length > 0) {
    h += '<div class="relative bg-slate-100">';
    h += '<div class="flex overflow-x-auto snap-x snap-mandatory" style="scroll-snap-type:x mandatory;-webkit-overflow-scrolling:touch">';
    for (var i = 0; i < imgs.length; i++) {
      h += '<div class="flex-shrink-0 w-full snap-center" style="scroll-snap-align:center"><img src="' + escapeHtml(imgs[i]) + '" class="w-full max-h-96 object-contain" onerror="this.remove()"></div>';
    }
    h += '</div>';
    if (imgs.length > 1) h += '<div class="absolute bottom-3 left-1/2 -translate-x-1/2 bg-slate-800/60 text-white text-xs px-2.5 py-1 rounded-full"><i class="fas fa-images mr-1"></i>' + imgs.length + '장 (좌우 스크롤)</div>';
    h += '</div>';
  }
  h += '<div class="p-5">';
  h += '<div class="font-bold text-lg text-slate-800">' + escapeHtml(p.name) + '</div>';
  h += '<div class="text-sm text-slate-400 mt-1">PID: ' + (p.pid || p.id) + '</div>';
  h += '<div class="text-2xl font-bold text-brand-600 mt-3">' + Number(p.price || 0).toLocaleString() + '원</div>';
  h += '<div class="grid grid-cols-2 gap-3 mt-4 text-sm">';
  if (p.condition) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">상태</div><div class="font-semibold text-slate-700 mt-0.5">' + (condLabel[p.condition] || p.condition) + '</div></div>';
  if (p.quantity !== undefined) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">수량</div><div class="font-semibold text-slate-700 mt-0.5">' + p.quantity + '개</div></div>';
  if (p.shippingFee !== undefined) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">배송비</div><div class="font-semibold text-slate-700 mt-0.5">' + Number(p.shippingFee).toLocaleString() + '원</div></div>';
  if (p.saleStatus) h += '<div class="bg-surface-50 rounded-lg p-2.5"><div class="text-slate-400 text-xs">판매상태</div><div class="font-semibold text-slate-700 mt-0.5">' + (saleLabel[p.saleStatus] || p.saleStatus) + '</div></div>';
  h += '</div>';
  if (p.description) h += '<div class="mt-4 p-3 bg-surface-50 rounded-lg text-sm text-slate-600 whitespace-pre-wrap max-h-40 overflow-y-auto">' + escapeHtml(p.description).replace(/\\\\n/g, '<br>').replace(/\\n/g, '<br>') + '</div>';
  h += '<div class="flex gap-2 mt-4">';
  h += '<button onclick="useProduct(' + (p.pid || p.id) + ')" class="btn btn-primary"><i class="fas fa-cart-plus"></i>이 상품으로 주문</button>';
  h += '<a href="https://m.bunjang.co.kr/products/' + (p.pid || p.id) + '" target="_blank" class="btn btn-secondary"><i class="fas fa-external-link-alt"></i>번개장터에서 보기</a>';
  h += '</div></div></div>';
  container.innerHTML = h;
}

function useProduct(pid) { closeModal(); switchTab('neworder'); document.getElementById('orderPid').value = pid; lookupProduct(); }

/* ══ New Order ══ */
function lookupProduct() {
  var pid = document.getElementById('orderPid').value.trim(); if (!pid) return;
  var el = document.getElementById('productPreview');
  el.innerHTML = '<div class="flex items-center gap-2 text-slate-400"><i class="fas fa-spinner fa-spin"></i>조회 중...</div>';
  apiFetch('/api/products/' + pid).then(function(d) {
    var p = d.data || d; cachedProductInfo = p;
    var imgUrl = (p.imageUrls && p.imageUrls.length > 0) ? p.imageUrls[0] : (p.imageUrlTemplate ? p.imageUrlTemplate.replace('{}', '1') : '');
    var h = '<div class="bg-brand-50 rounded-xl p-3 border border-brand-100 flex gap-3">';
    if (imgUrl) h += '<img src="' + escapeHtml(imgUrl) + '" class="w-16 h-16 object-cover rounded-lg" onerror="this.remove()">';
    h += '<div><div class="font-semibold text-slate-800">' + escapeHtml(p.name) + '</div>';
    h += '<div class="text-brand-600 font-bold mt-1">' + Number(p.price || 0).toLocaleString() + '원</div>';
    h += '<div class="text-xs text-slate-400 mt-0.5">배송비: ' + Number(p.shippingFee || 0).toLocaleString() + '원</div>';
    if (p.saleStatus) h += '<div class="text-xs text-slate-400 mt-0.5">' + p.saleStatus + '</div>';
    h += '</div></div>';
    el.innerHTML = h;
  }).catch(function(e) { el.innerHTML = '<div class="text-red-500 text-sm">조회 실패: ' + escapeHtml(e.message) + '</div>'; cachedProductInfo = null; });
}

function previewOrder() {
  var payload = buildPayload(); if (!payload) return;
  document.getElementById('orderResult').innerHTML = '<pre class="bg-surface-50 rounded-xl p-4 text-xs overflow-auto border border-surface-200">' + escapeHtml(JSON.stringify(payload, null, 2)) + '</pre>';
}

function submitOrder() {
  var payload = buildPayload(); if (!payload) return;
  if (!confirm('주문을 생성하시겠습니까?')) return;
  document.getElementById('orderResult').innerHTML = '<div class="flex items-center gap-2 text-slate-500"><i class="fas fa-spinner fa-spin"></i>주문 생성 중...</div>';
  apiFetch('/api/orders', { method: 'POST', body: JSON.stringify(payload) }).then(function(d) {
    document.getElementById('orderResult').innerHTML = '<div class="bg-emerald-50 text-emerald-700 rounded-xl p-4 border border-emerald-200"><i class="fas fa-check-circle mr-2"></i>주문 생성 성공! ID: ' + (d.data ? d.data.id : JSON.stringify(d)) + '</div>';
    toast('주문 생성 완료!');
  }).catch(function(e) { document.getElementById('orderResult').innerHTML = '<div class="bg-red-50 text-red-600 rounded-xl p-4 border border-red-200"><i class="fas fa-exclamation-circle mr-2"></i>오류: ' + escapeHtml(e.message) + '</div>'; });
}

function buildPayload() {
  var pid = document.getElementById('orderPid').value.trim();
  if (!pid) { toast('상품 PID를 입력하세요'); return null; }
  if (!cachedProductInfo || !cachedProductInfo.price) { toast('상품을 먼저 조회하세요'); return null; }
  var price = cachedProductInfo.price;
  var shipFee = cachedProductInfo.shippingFee || 0;
  return {
    product: { id: parseInt(pid), price: parseInt(price) }, deliveryPrice: parseInt(shipFee),
    shippingAddress: {
      name: document.getElementById('shipName').value.trim(), phone: document.getElementById('shipPhone').value.trim(),
      zipCode: document.getElementById('shipZip').value.trim(), address1: document.getElementById('shipAddr1').value.trim(),
      address2: document.getElementById('shipAddr2').value.trim(), extraInfo: document.getElementById('shipExtra').value.trim()
    }
  };
}

<\/script>
</body>
</html>`;
  return c.html(html);
});

export default app;
