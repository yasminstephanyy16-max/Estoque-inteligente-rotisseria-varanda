// app.js - interface em português
const apiUrl = '/api/items';
const logsUrl = '/api/logs';

async function fetchItems() {
  const res = await fetch(apiUrl);
  const data = await res.json();
  renderTable(data);
}

function formatDate(d){
  if(!d) return '';
  try { return new Date(d).toISOString().slice(0,10); } catch(e){ return d; }
}

function daysUntil(dateStr){
  if(!dateStr) return Infinity;
  const today = new Date();
  const d = new Date(dateStr + 'T00:00:00');
  const diff = Math.ceil((d - new Date(today.getFullYear(), today.getMonth(), today.getDate())) / (1000*60*60*24));
  return diff;
}

function renderTable(items) {
  const tbody = document.querySelector('#items-table tbody');
  tbody.innerHTML = '';
  const filtroNome = document.getElementById('filtro-nome').value.toLowerCase();
  const status = document.getElementById('filtro-status').value;

  items.forEach(item => {
    if (filtroNome) {
      const found = (item.nome || '').toLowerCase().includes(filtroNome) || (item.localizacao||'').toLowerCase().includes(filtroNome);
      if (!found) return;
    }
    const tr = document.createElement('tr');

    const dVal = item.validade ? item.validade : null;
    const dias = daysUntil(dVal);
    if (status === 'vence-logo' && !(dias >=0 && dias <=7)) return;
    if (status === 'vencidos' && !(dias < 0)) return;
    if (status === 'menos' && !(Number(item.quantidade) <= 2)) return;

    if (dias < 0) tr.classList.add('row-vencido');
    else if (dias <= 7) tr.classList.add('row-vence-logo');
    if (Number(item.quantidade) <= 2) tr.classList.add('row-baixa');

    tr.innerHTML = `
      <td>${item.id}</td>
      <td>${item.nome}</td>
      <td>${Number(item.quantidade)}</td>
      <td>${item.unidade || ''}</td>
      <td>${formatDate(item.fabricacao)}</td>
      <td>${formatDate(item.validade)} ${dias !== Infinity ? (dias < 0 ? ' (Vencido)' : dias === 0 ? ' (Vence hoje)' : dias <=7 ? ` (em ${dias}d)` : '') : ''}</td>
      <td>${item.localizacao || ''}</td>
      <td class="actions-col">
        <button data-id="${item.id}" class="edit">Editar</button>
        <button data-id="${item.id}" class="entrada">Entrada</button>
        <button data-id="${item.id}" class="saida">Saída</button>
        <button data-id="${item.id}" class="delete">Excluir</button>
      </td>
    `;
    tbody.appendChild(tr);
  });
}

// form actions
document.getElementById('item-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const id = document.getElementById('item-id').value;
  const body = {
    nome: document.getElementById('nome').value,
    unidade: document.getElementById('unidade').value,
    quantidade: Number(document.getElementById('quantidade').value),
    fabricacao: document.getElementById('fabricacao').value || null,
    validade: document.getElementById('validade').value || null,
    localizacao: document.getElementById('localizacao').value,
    descricao: document.getElementById('descricao').value
  };

  if (!body.nome) return alert('Digite o nome do produto.');
  if (id) {
    await fetch(`${apiUrl}/${id}`, { method:'PUT', headers:{'Content-Type':'application/json'}, body: JSON.stringify(body) });
    alert('Atualizado!');
  } else {
    await fetch(apiUrl, { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(body) });
    alert('Produto adicionado!');
  }
  clearForm();
  fetchItems();
});

document.getElementById('limpar').addEventListener('click', clearForm);
function clearForm(){
  document.getElementById('item-id').value='';
  document.getElementById('nome').value='';
  document.getElementById('unidade').value='';
  document.getElementById('quantidade').value=0;
  document.getElementById('fabricacao').value='';
  document.getElementById('validade').value='';
  document.getElementById('localizacao').value='';
  document.getElementById('descricao').value='';
}

// table actions (delegation)
document.querySelector('#items-table tbody').addEventListener('click', async (e) => {
  const id = e.target.dataset.id;
  if (!id) return;
  if (e.target.classList.contains('edit')) {
    const res = await fetch(`${apiUrl}/${id}`);
    const item = await res.json();
    document.getElementById('item-id').value = item.id;
    document.getElementById('nome').value = item.nome;
    document.getElementById('unidade').value = item.unidade || '';
    document.getElementById('quantidade').value = item.quantidade;
    document.getElementById('fabricacao').value = item.fabricacao ? item.fabricacao.slice(0,10) : '';
    document.getElementById('validade').value = item.validade ? item.validade.slice(0,10) : '';
    document.getElementById('localizacao').value = item.localizacao || '';
    document.getElementById('descricao').value = item.descricao || '';
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
  if (e.target.classList.contains('delete')) {
    if (confirm('Deseja excluir esse produto?')) {
      await fetch(`${apiUrl}/${id}`, { method:'DELETE' });
      fetchItems();
    }
  }
  if (e.target.classList.contains('entrada') || e.target.classList.contains('saida')) {
    const tipo = e.target.classList.contains('entrada') ? 'entrada' : 'saida';
    const q = prompt(`${tipo === 'entrada' ? 'Quantidade adicionada' : 'Quantidade retirada'} (use ponto para decimais):`, '1');
    if (q === null) return;
    const qtd = Number(q);
    if (isNaN(qtd) || qtd <= 0) return alert('Quantidade inválida.');
    const obs = prompt('Observação (opcional):', '') || '';
    const resp = await fetch(`${apiUrl}/${id}/move`, { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ tipo, quantidade: qtd, observacao: obs }) });
    const json = await resp.json();
    if (resp.ok) {
      alert(`Movimento registrado. Nova quantidade: ${json.novaQtd}`);
      fetchItems();
    } else {
      alert(json.error || 'Erro ao registrar movimento');
    }
  }
});

// filtros
document.getElementById('filtro-nome').addEventListener('input', fetchItems);
document.getElementById('filtro-status').addEventListener('change', fetchItems);

// logs
document.getElementById('ver-logs').addEventListener('click', async () => {
  const sec = document.getElementById('logs-section');
  sec.classList.remove('hidden');
  const res = await fetch(logsUrl);
  const logs = await res.json();
  const tbody = document.querySelector('#logs-table tbody');
  tbody.innerHTML = '';
  logs.forEach(l => {
    const tr = document.createElement('tr');
    const d = new Date(l.data_movimento).toLocaleString();
    tr.innerHTML = `<td>${d}</td><td>${l.item_nome || ('Item ' + l.item_id)}</td><td>${l.tipo}</td><td>${l.quantidade}</td><td>${l.observacao||''}</td>`;
    tbody.appendChild(tr);
  });
});
document.getElementById('fechar-logs').addEventListener('click', () => document.getElementById('logs-section').classList.add('hidden'));

// export CSV
document.getElementById('export-csv').addEventListener('click', () => {
  window.location.href = '/api/export/csv';
});

// init
fetchItems();
/* styles.css */
:root { --bg:#f4f6f8; --card:#fff; --accent:#2b7a78; --danger:#d9534f; --warn:#f0ad4e; --muted:#666; }
*{box-sizing:border-box}
body{font-family:Arial,Helvetica,sans-serif;margin:0;background:var(--bg);color:#222}
header{background:var(--accent);color:#fff;padding:16px}
header h1{margin:0;font-size:20px}
.subtitle{margin:6px 0 0;font-size:13px;opacity:0.9}
main{display:flex;gap:18px;padding:18px;flex-wrap:wrap}
.form-section, .list-section, .logs-section{background:var(--card);padding:14px;border-radius:8px;box-shadow:0 2px 10px rgba(0,0,0,0.05)}
.form-section{width:340px}
.list-section{flex:1;min-width:420px}
label{display:block;margin-bottom:8px;font-size:13px}
input[type="text"], input[type="number"], input[type="date"], select{width:100%;padding:8px;margin-top:4px;border:1px solid #ddd;border-radius:4px}
.actions{display:flex;gap:8px;margin-top:10px}
button{padding:8px 10px;border:none;background:var(--accent);color:#fff;border-radius:6px;cursor:pointer}
button#limpar{background:#6c757d}
button#export-csv{background:#1976d2;margin-top:10px}
.filtros{display:flex;gap:8px;margin-bottom:10px}
#items-table{width:100%;border-collapse:collapse}
th,td{padding:8px;border-bottom:1px solid #eee;text-align:left;font-size:13px}
td.actions-col button{margin-right:6px}
.row-vencido{background:rgba(217,83,79,0.06)}
.row-vence-logo{background:rgba(240,173,78,0.04)}
.row-baixa{background:rgba(91,192,222,0.04)}
.hidden{display:none}
.logs-section{width:100%}
@media(max-width:900px){
  main{flex-direction:column}
  .form-section{width:100%}
  .list-section{min-width:0}
}
// server.js - Geladeira Inteligente (PT-BR)
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const path = require('path');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
const DB_FILE = path.join(__dirname, 'geladeira.db');

app.use(cors());
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

// abrir/criar DB
const db = new sqlite3.Database(DB_FILE, (err) => {
  if (err) return console.error('Erro DB:', err.message);
  console.log('Conectado ao SQLite.');
});

// criar tabelas
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nome TEXT NOT NULL,
    unidade TEXT,
    quantidade REAL DEFAULT 0,
    fabricacao TEXT,
    validade TEXT,
    localizacao TEXT,
    descricao TEXT,
    criado_em DATETIME DEFAULT CURRENT_TIMESTAMP
  );`);

  db.run(`CREATE TABLE IF NOT EXISTS logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    item_id INTEGER,
    tipo TEXT, -- "entrada" ou "saida"
    quantidade REAL,
    data_movimento DATETIME DEFAULT CURRENT_TIMESTAMP,
    observacao TEXT,
    FOREIGN KEY(item_id) REFERENCES items(id)
  );`);
});

// util helpers
function parseDateSafe(d) {
  if (!d) return null;
  const parsed = new Date(d);
  return isNaN(parsed.getTime()) ? null : parsed.toISOString().slice(0,10);
}

// API - itens
app.get('/api/items', (req, res) => {
  db.all('SELECT * FROM items ORDER BY id DESC', (err, rows) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(rows);
  });
});

app.get('/api/items/:id', (req, res) => {
  db.get('SELECT * FROM items WHERE id = ?', [req.params.id], (err, row) => {
    if (err) return res.status(500).json({ error: err.message });
    if (!row) return res.status(404).json({ error: 'Item não encontrado' });
    res.json(row);
  });
});

app.post('/api/items', (req, res) => {
  const { nome, unidade, quantidade = 0, fabricacao, validade, localizacao, descricao } = req.body;
  const f = parseDateSafe(fabricacao);
  const v = parseDateSafe(validade);
  const stmt = db.prepare('INSERT INTO items (nome, unidade, quantidade, fabricacao, validade, localizacao, descricao) VALUES (?,?,?,?,?,?,?)');
  stmt.run([nome, unidade, quantidade, f, v, localizacao, descricao], function(err) {
    if (err) return res.status(500).json({ error: err.message });
    res.status(201).json({ id: this.lastID });
  });
  stmt.finalize();
});

app.put('/api/items/:id', (req, res) => {
  const { nome, unidade, quantidade, fabricacao, validade, localizacao, descricao } = req.body;
  const f = parseDateSafe(fabricacao);
  const v = parseDateSafe(validade);
  db.run(
    `UPDATE items SET nome=?, unidade=?, quantidade=?, fabricacao=?, validade=?, localizacao=?, descricao=? WHERE id=?`,
    [nome, unidade, quantidade, f, v, localizacao, descricao, req.params.id],
    function(err) {
      if (err) return res.status(500).json({ error: err.message });
      if (this.changes === 0) return res.status(404).json({ error: 'Item não encontrado' });
      res.json({ updated: true });
    }
  );
});

app.delete('/api/items/:id', (req, res) => {
  db.run('DELETE FROM items WHERE id=?', [req.params.id], function(err) {
    if (err) return res.status(500).json({ error: err.message });
    if (this.changes === 0) return res.status(404).json({ error: 'Item não encontrado' });
    // também deletar logs relacionados
    db.run('DELETE FROM logs WHERE item_id=?', [req.params.id]);
    res.json({ deleted: true });
  });
});

// Movimentação (entrada/saída)
app.post('/api/items/:id/move', (req, res) => {
  const itemId = req.params.id;
  const { tipo, quantidade = 0, observacao = '' } = req.body; // tipo: 'entrada' ou 'saida'
  if (!['entrada','saida'].includes(tipo)) return res.status(400).json({ error: 'tipo inválido' });

  db.get('SELECT quantidade FROM items WHERE id=?', [itemId], (err, row) => {
    if (err) return res.status(500).json({ error: err.message });
    if (!row) return res.status(404).json({ error: 'Item não encontrado' });

    let novaQtd = Number(row.quantidade);
    const q = Number(quantidade);
    if (tipo === 'entrada') novaQtd += q;
    else novaQtd -= q;

    if (novaQtd < 0) return res.status(400).json({ error: 'Quantidade insuficiente' });

    db.run('UPDATE items SET quantidade = ? WHERE id = ?', [novaQtd, itemId], function(err2) {
      if (err2) return res.status(500).json({ error: err2.message });
      const stmt = db.prepare('INSERT INTO logs (item_id, tipo, quantidade, observacao) VALUES (?,?,?,?)');
      stmt.run([itemId, tipo, quantidade, observacao], function(err3) {
        if (err3) return res.status(500).json({ error: err3.message });
        res.json({ moved: true, novaQtd });
      });
      stmt.finalize();
    });
  });
});

// Logs de movimentação
app.get('/api/logs', (req, res) => {
  db.all(`SELECT l.id, l.item_id, l.tipo, l.quantidade, l.data_movimento, l.observacao, i.nome as item_nome
          FROM logs l LEFT JOIN items i ON l.item_id = i.id
          ORDER BY l.data_movimento DESC LIMIT 500`, (err, rows) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(rows);
  });
});

// Export CSV simples (itens)
app.get('/api/export/csv', (req, res) => {
  db.all('SELECT * FROM items ORDER BY id', (err, rows) => {
    if (err) return res.status(500).send('Erro');
    const headers = ['id','nome','unidade','quantidade','fabricacao','validade','localizacao','descricao','criado_em'];
    const lines = [headers.join(',')];
    rows.forEach(r => {
      const vals = headers.map(h => {
        let v = r[h] === null || r[h] === undefined ? '' : String(r[h]).replace(/"/g, '""');
        if (v.includes(',') || v.includes('"') || v.includes('\n')) v = `"${v}"`;
        return v;
      });
      lines.push(vals.join(','));
    });
    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', 'attachment; filename="itens_estoque.csv"');
    res.send(lines.join('\n'));
  });
});

// fallback - serve index
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));
{
  "name": "geladeira-inteligente",
  "version": "1.0.0",
  "description": "Controle de estoque de alimentos refrigerados (entrada/saída, validade, histórico)",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sqlite3": "^5.1.6",
    "body-parser": "^1.20.2",
    "cors": "^2.8.5"
  }
}
