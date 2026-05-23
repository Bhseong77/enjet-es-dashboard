# 🔧 ENJET 통합 워크플로우 - 영업/구매 모듈 수정 가이드

ES 모듈 Phase 4 완성에 맞춰 **영업 모듈**과 **구매 모듈**도 일부 수정이 필요합니다.

> 영업 모듈(1.4MB)과 구매 모듈(1MB) 자체는 워낙 커서 한 응답에 같이 수정 못해요. 
> 이 가이드대로 직접 추가하시거나, 다음 응답에서 부분별로 패치 드릴게요.

---

## 📋 영업 모듈 수정 사항 (3개)

### 1️⃣ 견적 모달 - "ES사업부에 요청" 옵션 추가

**위치**: `openQuoteModal(actId)` 함수 (line 14438)

**수정 방법**: 모달 열리기 전에 분기 다이얼로그 표시

```javascript
// 기존 견적서 작성 버튼 onclick 수정
onclick="openQuoteModalWithBranch(null)"

// 새 함수 추가
function openQuoteModalWithBranch(actId) {
  // 분기 모달 표시
  const html = `
    <div id="quoteBranchModal" style="position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:10000;display:flex;align-items:center;justify-content:center">
      <div style="background:#fff;padding:30px;border-radius:12px;max-width:500px">
        <h3>💰 견적을 어떻게 작성하시겠어요?</h3>
        <div style="display:grid;gap:14px;margin-top:20px">
          <button onclick="document.getElementById('quoteBranchModal').remove();openQuoteModal('${actId}')" 
            style="padding:14px;border:2px solid var(--p);background:#fff;border-radius:8px;cursor:pointer;text-align:left">
            <strong>📝 직접 작성 (기존)</strong><br>
            <span style="font-size:12px;color:#666">영업이 BOM/단가 직접 입력 (표준 제품, 빠른 견적)</span>
          </button>
          <button onclick="requestESQuote('${actId}')" 
            style="padding:14px;border:2px solid var(--pu);background:#fff;border-radius:8px;cursor:pointer;text-align:left">
            <strong>📋 ES사업부에 요청</strong><br>
            <span style="font-size:12px;color:#666">제안서/BOM을 ES가 작성 → 구매팀 거쳐 정확한 원가</span>
          </button>
          <button onclick="document.getElementById('quoteBranchModal').remove()" 
            style="padding:8px;background:#eee;border:none;border-radius:6px;cursor:pointer">취소</button>
        </div>
      </div>
    </div>
  `;
  document.body.insertAdjacentHTML('beforeend', html);
}

async function requestESQuote(actId) {
  document.getElementById('quoteBranchModal').remove();
  const act = slData.find(x => x.id === actId);
  if (!act) return;
  
  // ES에 견적 요청 생성
  const request = {
    id: `REQ-${Date.now()}`,
    salesActivityId: actId,
    projectCode: act.projectCode || act.pjcode,
    company: act.company,
    projectName: act.projectName || act.subject,
    salesperson: _meName(),
    requestedAt: new Date().toISOString(),
    status: "pending",
    requestType: "quote_request"
  };
  
  // es-quote-requests.json에 추가 (ES 모듈이 폴링으로 가져감)
  const path = "🔧 ES사업부/99.Dashboard(real-time)/es-quote-requests.json";
  const existing = await gGet(`/drives/${spDrive}/root:/${encodeURIComponent(path)}:/content`).catch(() => null);
  const requests = existing?.requests || [];
  requests.unshift(request);
  
  await gPut(path, { requests, lastUpdate: new Date().toISOString() });
  
  alert(`✅ ES사업부에 견적 요청 완료\n\n프로젝트: ${request.projectCode}\nES가 BOM 작성 후 → 구매팀 거쳐 → 영업에 BOM 전달`);
}
```

### 2️⃣ 발주서 모달 - 결제 Term 추가

**위치**: `orderModal` (line 21864) HTML + `openOrderModal` (line 19531) 함수

**HTML 추가**: 발주서 모달 안에 결제 Term 섹션

```html
<!-- 결제 조건 (NEW) -->
<div style="background:#fef3c7;padding:14px;border-radius:10px;margin:14px 0;border:1px solid #f59e0b">
  <h4>💳 결제 조건 (계약금 / 중도금 / 잔금)</h4>
  
  <!-- 비율 입력 모드 -->
  <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin-top:10px">
    <div>
      <label>계약금 %</label>
      <input type="number" id="order-deposit-pct" value="30" oninput="recalcOrderPayment()">
      <input type="date" id="order-deposit-date" placeholder="수금 예정">
    </div>
    <div>
      <label>중도금 %</label>
      <input type="number" id="order-interim-pct" value="40" oninput="recalcOrderPayment()">
      <select id="order-interim-trigger">
        <option value="출하시">출하시</option>
        <option value="입고시">입고시</option>
        <option value="custom">날짜 지정</option>
      </select>
    </div>
    <div>
      <label>잔금 %</label>
      <input type="number" id="order-balance-pct" value="30" oninput="recalcOrderPayment()">
      <select id="order-balance-trigger">
        <option value="FAT후">FAT 완료 후</option>
        <option value="납품확인후">납품확인 후</option>
        <option value="custom">날짜 지정</option>
      </select>
    </div>
  </div>
  
  <!-- 자동 계산 결과 -->
  <div id="order-payment-summary" style="margin-top:10px;padding:10px;background:#fff;border-radius:6px;font-size:12px">
    합계 100% 정상 (계약금 30% + 중도금 40% + 잔금 30%)
  </div>
</div>
```

**JS 함수 추가**:

```javascript
function recalcOrderPayment() {
  const d = parseInt(document.getElementById('order-deposit-pct').value || 0);
  const i = parseInt(document.getElementById('order-interim-pct').value || 0);
  const b = parseInt(document.getElementById('order-balance-pct').value || 0);
  const total = d + i + b;
  const amount = parseInt(document.getElementById('order-total-amount').value.replace(/,/g, '') || 0);
  
  const summary = document.getElementById('order-payment-summary');
  if (total === 100) {
    summary.style.color = 'green';
    summary.innerHTML = `✅ 합계 100% 정상<br>
      계약금: ${_fmtNum(amount * d / 100)}원 (${d}%) ·
      중도금: ${_fmtNum(amount * i / 100)}원 (${i}%) ·
      잔금: ${_fmtNum(amount * b / 100)}원 (${b}%)`;
  } else {
    summary.style.color = 'red';
    summary.innerHTML = `⚠️ 합계 ${total}% (100%이 되어야 합니다)`;
  }
}

// saveOrder 함수 수정 - paymentTerm 추가
function saveOrder() {
  // ... 기존 코드 ...
  data.paymentTerm = {
    deposit: { pct: parseInt(document.getElementById('order-deposit-pct').value), date: document.getElementById('order-deposit-date').value },
    interim: { pct: parseInt(document.getElementById('order-interim-pct').value), trigger: document.getElementById('order-interim-trigger').value },
    balance: { pct: parseInt(document.getElementById('order-balance-pct').value), trigger: document.getElementById('order-balance-trigger').value }
  };
  // ... 저장 ...
}
```

### 3️⃣ Todo 담당자 - ES사업부 옵션 추가

영업 Todo 모달의 담당자 셀렉트에 "ES사업부" 옵션 추가:

```javascript
// 영업 모듈에 이미 DEPT_OPTIONS에 "ES사업부" 등록되어 있음 (line 18071)
// Todo 작성 시 dept을 "ES사업부"로 지정하면 자동으로 ES 모듈에 표시됨

// Todo 데이터 저장 시 추가 필드:
{
  ...
  assignedDept: "ES사업부",  // ES 모듈이 이걸로 필터링
  assignedTeam: "SE",        // 또는 PE, AE
  projectCode: "PRJ-2026-XXX"  // 필수
}
```

---

## 📋 구매 모듈 수정 사항 (2개)

### 1️⃣ 전자결재 결재완료 → 자동 등록

**위치**: 구매 모듈에 새 함수 추가

```javascript
// 구매 모듈 init 시 호출 - 매 1분마다 결재완료된 ES 구매품의 확인
async function pollESApprovals() {
  setInterval(async () => {
    const approvalPath = "📋 전자결재/99.Dashboard(real-time)/approval-master.json";
    const data = await fetchJson(approvalPath);
    if (!data?.approvals) return;
    
    // ES에서 상신했고 결재완료된 + 아직 처리 안 된 것
    const newApprovals = data.approvals.filter(a => 
      a.source === "es_module" && 
      a.status === "완료" && 
      !a.purchaseRegistered  // 구매에서 처리 안 됐음 표시
    );
    
    for (const approval of newApprovals) {
      // pr-master.json에 자동 등록
      const prData = await fetchJson(PR_SP_PATH);
      const prs = prData?.items || [];
      
      prs.unshift({
        id: `PR-${Date.now()}`,
        prNo: approval.id.replace("APR", "PR"),
        date: new Date().toISOString().substring(0,10),
        company: approval.formData.customer,
        projectCode: approval.formData.projectCode,
        items: approval.formData.items,
        totalAmount: approval.formData.totalAmount,
        specialNote: approval.formData.specialNote,
        source: "es_module_auto",
        esApprovalId: approval.id,
        status: "신규"
      });
      
      await savePR(prs);
      
      // 결재에 처리 표시
      approval.purchaseRegistered = true;
      approval.purchaseRegisteredAt = new Date().toISOString();
      await saveApproval(data);
      
      alert(`✅ ES 구매품의서가 구매 모듈에 자동 등록되었습니다.\nPR 번호: ${prs[0].id}`);
    }
  }, 60000);
}
```

### 2️⃣ 구매 BOM 완성 시 → 영업에 전달

```javascript
// 구매 모듈의 BOM/견적이 확정되면 영업 모듈에 전달
async function sendBomToSales(prId) {
  const pr = _prData.find(p => p.id === prId);
  if (!pr) return;
  
  // 영업 모듈 견적 모달에서 이 데이터를 가져갈 수 있도록 저장
  const bomReadyPath = "📦영업/99.Dashboard(real-time)/bom-ready-for-quote.json";
  const existing = await fetchJson(bomReadyPath);
  const items = existing?.items || [];
  
  items.unshift({
    id: pr.id,
    projectCode: pr.projectCode,
    customer: pr.company,
    bom: pr.items.map(i => ({
      partNo: i.partNo,
      name: i.name,
      spec: i.spec,
      qty: i.qty,
      unitPrice: i.unitPrice,  // ← 구매가 (영업이 마진 추가)
      totalAmount: i.totalAmount
    })),
    purchaseTotal: pr.totalAmount,
    readyAt: new Date().toISOString()
  });
  
  await gPut(bomReadyPath, { items, lastUpdate: new Date().toISOString() });
  
  alert(`✅ 영업 모듈에 BOM 전달 완료\n프로젝트: ${pr.projectCode}\n영업은 견적 작성 시 [구매 BOM 사용] 옵션 선택 가능`);
}

// 영업 모듈 견적 모달에 [🛒 구매 BOM 사용] 버튼 추가:
// 클릭 시 bom-ready-for-quote.json에서 해당 프로젝트 BOM 가져와서 채우기
```

---

## 🚀 작업 우선순위 권장

### 즉시 (필수):
1. ✅ ES 모듈 v2.0 배포 (이미 만듦)
2. 영업 모듈: 견적 모달 분기 추가 (#1) 
3. 영업 모듈: 발주서 결제 Term (#2)

### 다음 응답에서:
4. 구매 모듈: ES 결재완료 자동 등록 (#1)
5. 구매 모듈: BOM → 영업 전달 (#2)

### 그 다음:
6. 일정 동기화 (구매 납기 → ES)
7. 입고 → 출고 흐름
8. SAT/FAT 관리
9. 세금계산서 자동 트리거

---

## 📊 데이터 흐름 요약

```
영업 → ES 견적 요청
   📦영업/99.Dashboard(real-time)/sales-activities.json (status: 견적_ES요청)
   🔧 ES사업부/99.Dashboard(real-time)/es-quote-requests.json ← ES가 폴링

ES → 전자결재 상신
   📋 전자결재/99.Dashboard(real-time)/approval-master.json
   (source: "es_module", formId: "PURCHASE")

전자결재 완료 → 구매 자동 등록
   🏭 구매/99.Dashboard(real-time)/pr-master.json
   (source: "es_module_auto")

구매 BOM → 영업 견적
   📦영업/99.Dashboard(real-time)/bom-ready-for-quote.json ← 영업이 폴링
   (영업 견적 모달에서 [구매 BOM 사용] 클릭)

영업 Todo (담당:ES) → ES 표시
   📦영업/99.Dashboard(real-time)/sales-todos.json
   (assignedDept: "ES사업부")
   ES 모듈 SE팀 탭에 자동 표시

ES → 영업 Todo 완료
   영업 sales-todos.json 업데이트 (status: 완료, completedFromES: true)
   Teams 채널 자동 게시

회의록 양방향
   영업: 📦영업/99.Dashboard(real-time)/mtg-logs.json
   ES:   🔧 ES사업부/99.Dashboard(real-time)/es-meetings.json
   ES 내부 회의록 작성 시 영업 mtg-logs.json에도 자동 추가

자료 업로드 양방향
   ES → 영업 customerFiles 자동 추가
   Teams 채널 자동 게시
```

---

이 가이드대로 영업/구매 모듈에 코드 추가하시면 **완전 자동화**가 완성됩니다.

다음 응답에서 영업/구매 모듈 자체에 직접 코드 패치해드릴 수 있으니 말씀해주세요!
