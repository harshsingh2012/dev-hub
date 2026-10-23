<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>SplitEasy – Hostel Expense Splitter</title>

<style>
  /* ---------- Basic look ---------- */
  * { box-sizing: border-box; }
  body {
    margin: 0;
    background: #eef2f7;
    color: #14213d;
    font-family: Arial, sans-serif;
    line-height: 1.5;
  }
  main { max-width: 640px; margin: 0 auto; padding: 20px 16px 40px; }
  h1 { margin: 8px 0 4px; }
  h2 { font-size: 1.1rem; margin: 20px 0 8px; }
  .sub, .muted { color: #5c6b85; }

  /* ---------- Cards, inputs, buttons ---------- */
  .card {
    background: #fff;
    border: 1px solid #d9e0ea;
    border-radius: 12px;
    padding: 14px;
    margin-bottom: 12px;
  }
  .row { display: flex; gap: 8px; flex-wrap: wrap; }
  .row > * { flex: 1; min-width: 130px; }
  label { display: grid; gap: 4px; font-size: .85rem; color: #5c6b85; }
  input, select {
    font: inherit;
    padding: 10px;
    border: 1px solid #d9e0ea;
    border-radius: 8px;
    width: 100%;
  }
  button {
    font: inherit;
    background: #2a5bd7;
    color: #fff;
    border: 0;
    border-radius: 8px;
    padding: 11px 14px;
    cursor: pointer;
  }
  button.ghost { background: none; color: #14213d; border: 1px solid #d9e0ea; }

  /* ---------- Lists ---------- */
  ul { list-style: none; margin: 0; padding: 0; display: grid; gap: 8px; }
  li {
    background: #fff;
    border: 1px solid #d9e0ea;
    border-radius: 10px;
    padding: 10px 12px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 10px;
  }
  .good { color: #2f9e6e; font-weight: bold; }
  .owe  { color: #d6336c; font-weight: bold; }
</style>
</head>

<body>
<main>
  <h1>SplitEasy</h1>
  <p class="sub">Add friends, log who paid, and see who owes whom.</p>

  <!-- 1. Add members -->
  <div class="card">
    <label>Add a member
      <div class="row">
        <input id="memberName" maxlength="20" placeholder="e.g. Aarav">
        <button id="addMemberBtn" style="flex:0 0 auto">Add member</button>
      </div>
    </label>
    <p id="memberList" class="muted" style="margin:8px 0 0">No members yet.</p>
  </div>

  <!-- 2. Add an expense -->
  <div class="card">
    <div class="row">
      <label>What was it for?
        <input id="desc" maxlength="40" placeholder="e.g. Pizza">
      </label>
      <label>Amount (₹)
        <input id="amount" type="number" min="1" step="0.01">
      </label>
    </div>
    <div class="row" style="margin-top:8px">
      <label>Paid by
        <select id="payer"></select>
      </label>
      <button id="addExpenseBtn" style="align-self:end">Add expense</button>
    </div>
  </div>

  <!-- 3. Output -->
  <h2>Expenses</h2>
  <ul id="expenseList"></ul>

  <h2>Who pays whom</h2>
  <ul id="settleList"></ul>

  <p><button class="ghost" id="resetBtn">Clear everything</button></p>
</main>

<script>
/* ============================================================
   DATA
   members  : ["Aarav", "Riya", ...]
   expenses : [{ desc, amount, payer, sharedBy: [names] }, ...]
   ============================================================ */
const STORAGE_KEY = "splitEasyData";
let data = { members: [], expenses: [] };

// Load saved data (if any) when the page opens
try {
  const saved = JSON.parse(localStorage.getItem(STORAGE_KEY));
  if (saved) data = saved;
} catch (e) { /* ignore, start fresh */ }

function save() {
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(data)); } catch (e) {}
}

const $ = id => document.getElementById(id);

// Stops names typed by users from being treated as HTML
function escapeHtml(text) {
  return text.replace(/[&<>"']/g, c => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;"
  }[c]));
}

/* ============================================================
   CALCULATION
   balance > 0  -> person should RECEIVE money
   balance < 0  -> person should PAY money
   ============================================================ */
function calculateBalances() {
  const balance = {};
  data.members.forEach(name => balance[name] = 0);

  data.expenses.forEach(exp => {
    const share = exp.amount / exp.sharedBy.length;   // equal split
    balance[exp.payer] += exp.amount;                 // payer paid the full bill
    exp.sharedBy.forEach(name => balance[name] -= share); // everyone owes a share
  });
  return balance;
}

// Match people who owe money with people who should receive it,
// using the fewest possible payments.
function calculateSettlements(balance) {
  const receivers = [], payers = [];
  for (const name in balance) {
    const amt = Math.round(balance[name] * 100) / 100;
    if (amt > 0.01) receivers.push([name, amt]);
    else if (amt < -0.01) payers.push([name, -amt]);
  }

  const result = [];
  let i = 0, j = 0;
  while (i < payers.length && j < receivers.length) {
    const pay = Math.min(payers[i][1], receivers[j][1]);
    result.push({ from: payers[i][0], to: receivers[j][0], amount: pay });
    payers[i][1] -= pay;
    receivers[j][1] -= pay;
    if (payers[i][1] < 0.01) i++;
    if (receivers[j][1] < 0.01) j++;
  }
  return result;
}

/* ============================================================
   SHOW EVERYTHING ON SCREEN
   ============================================================ */
function render() {
  // Members line + "Paid by" dropdown
  $("memberList").textContent = data.members.length
    ? "Members: " + data.members.join(", ")
    : "No members yet.";
  $("payer").innerHTML = data.members
    .map(n => `<option>${escapeHtml(n)}</option>`).join("");

  // Expense list
  $("expenseList").innerHTML = data.expenses.length
    ? data.expenses.map((e, index) => `
        <li>
          <span>${escapeHtml(e.desc)}<br>
            <small class="muted">${escapeHtml(e.payer)} paid, split ${e.sharedBy.length} ways</small>
          </span>
          <span>₹${e.amount.toFixed(2)}
            <button class="ghost" data-index="${index}" aria-label="Delete">×</button>
          </span>
        </li>`).join("")
    : '<li class="muted">No expenses yet.</li>';

  // Settlement list
  const settlements = calculateSettlements(calculateBalances());
  $("settleList").innerHTML = settlements.length
    ? settlements.map(s => `
        <li>
          <span><b>${escapeHtml(s.from)}</b> pays <b>${escapeHtml(s.to)}</b></span>
          <span class="owe">₹${s.amount.toFixed(2)}</span>
        </li>`).join("")
    : '<li class="good">Everyone is settled up.</li>';
}

/* ============================================================
   BUTTON ACTIONS
   ============================================================ */
$("addMemberBtn").onclick = () => {
  const name = $("memberName").value.trim();
  if (!name) return alert("Enter a name.");
  if (data.members.includes(name)) return alert("That member already exists.");
  data.members.push(name);
  $("memberName").value = "";
  save(); render();
};

$("addExpenseBtn").onclick = () => {
  const desc = $("desc").value.trim();
  const amount = parseFloat($("amount").value);
  const payer = $("payer").value;

  if (data.members.length < 2) return alert("Add at least 2 members first.");
  if (!desc || !(amount > 0)) return alert("Enter a description and an amount above 0.");

  data.expenses.push({ desc, amount, payer, sharedBy: [...data.members] });
  $("desc").value = "";
  $("amount").value = "";
  save(); render();
};

// Delete one expense (uses the × button's data-index)
$("expenseList").onclick = e => {
  if (e.target.dataset.index !== undefined) {
    data.expenses.splice(Number(e.target.dataset.index), 1);
    save(); render();
  }
};

$("resetBtn").onclick = () => {
  if (confirm("Delete all members and expenses?")) {
    data = { members: [], expenses: [] };
    save(); render();
  }
};

render();   // first draw when the page opens
</script>
</body>
</html>
