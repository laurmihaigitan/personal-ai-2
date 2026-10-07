let currentConvId = null;
let currentMode = "short";

if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js').catch(() => {});
}

document.addEventListener('DOMContentLoaded', () => {
  loadDirectories();
  setupEventListeners();
  checkApiStatus();
});

function setupEventListeners() {
  const btnMode = document.getElementById('btn-mode');
  btnMode.addEventListener('click', () => {
    currentMode = currentMode === "short" ? "detailed" : "short";
    btnMode.textContent = currentMode === "short" ? "Scurt" : "Detaliat";
    btnMode.classList.toggle('text-sky-400', currentMode === "detailed");
  });

  document.getElementById('chat-form').addEventListener('submit', handleSendMessage);

  document.getElementById('btn-new-chat').addEventListener('click', () => {
    currentConvId = null;
    document.getElementById('chat-box').innerHTML = `
      <div class="text-center text-xs text-zinc-500 my-auto" id="empty-state">
        Conversație nouă începută.
      </div>
    `;
    document.getElementById('token-indicator').textContent = "Model gata";
  });

  document.getElementById('btn-docs').addEventListener('click', openDocsModal);
  document.getElementById('btn-settings').addEventListener('click', () => toggleModal('modal-settings', true));
  document.getElementById('btn-history').addEventListener('click', openHistoryModal);

  document.querySelectorAll('.btn-close-modal').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('[id^="modal-"]').forEach(m => m.classList.add('hidden'));
    });
  });

  document.getElementById('btn-save-keys').addEventListener('click', saveApiKeys);
  document.getElementById('btn-do-upload').addEventListener('click', handleUploadFile);
}

function toggleModal(id, show) {
  document.getElementById(id).classList.toggle('hidden', !show);
}

async function loadDirectories() {
  const res = await fetch('/api/directories');
  const dirs = await res.json();
  const select = document.getElementById('select-dir');
  select.innerHTML = '<option value="">Fără context</option>';
  dirs.forEach(d => {
    const opt = document.createElement('option');
    opt.value = d.id;
    opt.textContent = d.name;
    select.appendChild(opt);
  });
  if (dirs.length > 0) select.value = "vama";
}

async function handleSendMessage(e) {
  e.preventDefault();
  const input = document.getElementById('prompt-input');
  const text = input.value.trim();
  if (!text) return;

  const emptyState = document.getElementById('empty-state');
  if (emptyState) emptyState.remove();

  appendMessage('user', text);
  input.value = '';

  const modelVal = document.getElementById('select-model').value.split(':');
  const provider = modelVal[0];
  const model = modelVal[1];
  const dirId = document.getElementById('select-dir').value;
  const useWeb = document.getElementById('check-web').checked;

  const btnSend = document.getElementById('btn-send');
  btnSend.disabled = true;
  btnSend.textContent = '...';

  const assistantBubble = appendMessage('assistant', 'Se procesează...');

  try {
    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        conv_id: currentConvId,
        provider: provider,
        model: model,
        dir_id: dirId || null,
        prompt: text,
        mode: currentMode,
        use_web: useWeb
      })
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.detail || "Eroare la procesarea răspunsului.");
    }

    const data = await res.json();
    currentConvId = data.conv_id;
    assistantBubble.textContent = data.reply;
    document.getElementById('token-indicator').textContent = `${model} | Tokeni: ${data.tokens} | Cost: ${data.cost}`;
  } catch (err) {
    assistantBubble.textContent = `Eroare: ${err.message}`;
    assistantBubble.classList.add('text-red-400');
  } finally {
    btnSend.disabled = false;
    btnSend.textContent = 'Trimite';
  }
}

function appendMessage(role, text) {
  const box = document.getElementById('chat-box');
  const msg = document.createElement('div');
  msg.className = role === 'user' 
    ? 'self-end bg-sky-950 text-sky-100 border border-sky-800 rounded-lg px-3 py-2 max-w-[85%] break-words'
    : 'self-start bg-zinc-900 border border-zinc-800 rounded-lg px-3 py-2 max-w-[90%] break-words whitespace-pre-wrap';
  msg.textContent = text;
  box.appendChild(msg);
  box.scrollTop = box.scrollHeight;
  return msg;
}

async function openDocsModal() {
  const dirSelect = document.getElementById('select-dir');
  const dirId = dirSelect.value;
  if (!dirId) {
    alert("Selectează mai întâi un director din antet.");
    return;
  }
  document.getElementById('current-dir-name').textContent = dirSelect.options[dirSelect.selectedIndex].text;
  toggleModal('modal-docs', true);
  loadDocumentList(dirId);
}

async function loadDocumentList(dirId) {
  const container = document.getElementById('doc-list');
  container.innerHTML = 'Încărcare...';
  const res = await fetch(`/api/documents?dir_id=${dirId}`);
  const docs = await res.json();
  if (docs.length === 0) {
    container.innerHTML = '<span class="text-zinc-500 italic">Nu există documente încărcate în acest director.</span>';
    return;
  }
  container.innerHTML = '';
  docs.forEach(doc => {
    const item = document.createElement('div');
    item.className = 'flex justify-between items-center bg-zinc-950 p-2 rounded border border-zinc-800';
    item.innerHTML = `
      <span class="truncate max-w-[200px]">${doc.filename}</span>
      <button onclick="deleteDoc(${doc.id})" class="text-red-400 hover:text-red-300">Șterge</button>
    `;
    container.appendChild(item);
  });
}

async function deleteDoc(docId) {
  if (!confirm("Ștergi acest document și fragmentele sale indexate?")) return;
  await fetch(`/api/documents/${docId}`, { method: 'DELETE' });
  const dirId = document.getElementById('select-dir').value;
  loadDocumentList(dirId);
}

async function handleUploadFile() {
  const dirId = document.getElementById('select-dir').value;
  const fileInput = document.getElementById('file-upload-input');
  const status = document.getElementById('upload-status');
  if (!fileInput.files[0]) {
    status.textContent = "Alege un fișier mai întâi.";
    return;
  }

  const formData = new FormData();
  formData.append('dir_id', dirId);
  formData.append('file', fileInput.files[0]);

  status.textContent = "Extragere text și indexare...";
  try {
    const res = await fetch('/api/upload', { method: 'POST', body: formData });
    const data = await res.json();
    if (!res.ok) throw new Error(data.detail);
    status.textContent = `Succes: ${data.filename} (${data.chunks} fragmente indexate)`;
    fileInput.value = '';
    loadDocumentList(dirId);
  } catch (err) {
    status.textContent = `Eroare: ${err.message}`;
  }
}

async function saveApiKeys() {
  const openai = document.getElementById('input-openai-key').value.trim();
  const gemini = document.getElementById('input-gemini-key').value.trim();
  const status = document.getElementById('settings-status');

  await fetch('/api/settings', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      openai_key: openai || null,
      gemini_key: gemini || null
    })
  });
  status.textContent = "Cheile au fost salvate local în siguranță.";
  setTimeout(() => toggleModal('modal-settings', false), 1200);
}

async function checkApiStatus() {
  const res = await fetch('/api/settings/check');
  const data = await res.json();
  if (!data.openai_configured && !data.gemini_configured) {
    toggleModal('modal-settings', true);
    document.getElementById('settings-status').textContent = "Configurează cel puțin o cheie API pentru a începe.";
  }
}

async function openHistoryModal() {
  toggleModal('modal-history', true);
  const container = document.getElementById('conv-list');
  container.innerHTML = 'Încărcare istoric...';
  const res = await fetch('/api/conversations');
  const convs = await res.json();
  if (convs.length === 0) {
    container.innerHTML = '<span class="text-zinc-500">Nu există conversații salvate.</span>';
    return;
  }
  container.innerHTML = '';
  convs.forEach(c => {
    const item = document.createElement('div');
    item.className = 'p-2 bg-zinc-950 border border-zinc-800 rounded cursor-pointer hover:border-zinc-700';
    item.textContent = c.title || c.id;
    item.onclick = () => loadExistingConversation(c.id);
    container.appendChild(item);
  });
}

async function loadExistingConversation(convId) {
  currentConvId = convId;
  toggleModal('modal-history', false);
  const box = document.getElementById('chat-box');
  box.innerHTML = '';
  const res = await fetch(`/api/conversations/${convId}`);
  const msgs = await res.json();
  msgs.forEach(m => appendMessage(m.role, m.content));
}
