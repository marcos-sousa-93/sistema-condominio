# 🔄 Como o Código Conversa — Traço Completo de Execução
## 🎬 Parte 0 — A metáfora
### Pense em cada arquivo como uma pessoa numa empresa:

`run.py` = porteiro de entrada do prédio (chama o gerente)

`app/__init__.py` = gerente que contrata todos os departamentos

`config.py` = manual de regras da empresa

`database.py` = bibliotecário (só ele mexe nos arquivos)

`utils.py` = segurança interna (verifica crachá)

`auth.py` = recepção (login/usuários)

`api.py` = atendente universal (CRUD)

`assistant.py` = ouvidoria inteligente

`schema.sql` = planta do arquivo

`app.js` = o cliente (quem faz os pedidos)

`index.html` / `login.html` = as salas onde o cliente é atendido

## Parte 1 — Boot: python run.py (o primeiro segundo)
# Linha 1 — Terminal
```text
D:\...> python run.py
```
Python começa a executar o arquivo. Primeiro, ele interpreta os imports.

`run.py` linha 1

```python
from app import create_app
```

### O que acontece nos bastidores:

1. Python vê que existe uma pasta `app/` ao lado de `run.py`

2. Entra na pasta `app/`

3. Encontra `__init__.py`

4. Executa o arquivo inteiro `app/__init__.py` do topo até o final, definindo tudo que estiver lá

5. Só ao terminar, extrai o nome `create_app` e o disponibiliza em `run.py`

Por que isso importa: quando `app/__init__.py` roda, ele executa seus próprios imports internos:

from .database import init_db, get_db → executa app/database.py

from .auth import auth_bp → executa app/auth.py

from .api import api_bp, ENTITIES → executa app/api.py

from .assistant import assistant_bp → executa app/assistant.py

Isto é a "reação em cadeia dos imports":

text
run.py
  └─→ importa app/__init__.py
       ├─→ importa app/database.py
       ├─→ importa app/auth.py
       │    ├─→ importa app/database.py (já carregado, reaproveita)
       │    └─→ importa app/utils.py
       │         └─→ importa app/database.py (já carregado)
       ├─→ importa app/api.py
       │    ├─→ importa app/database.py
       │    └─→ importa app/utils.py (já carregado)
       └─→ importa app/assistant.py
            ├─→ importa app/database.py
            └─→ importa app/utils.py (já carregado)
Nada de função é chamada ainda. Só definições. Nenhum SQL executado. Isso se chama "fase de importação" e termina quando run.py chega na linha 3.

run.py linha 3
python
app = create_app()
Agora sim algo é executado. Vamos entrar em create_app.

Dentro de create_app() (app/init.py)
python
app = Flask(__name__, template_folder="../templates", static_folder="../static")
O que faz: cria o "objeto servidor HTTP" do Flask. A partir daqui, o app sabe onde estão HTMLs e CSS/JS.

python
from config import Config
Import tardio — o arquivo config.py só é carregado agora, dentro da função. Isso quebra dependência circular.

python
app.config.from_object(Config)
Comunicação: Config (classe em config.py) fala com app.config (dicionário do Flask). A partir daqui, qualquer módulo faz current_app.config["DATABASE"] e obtém o caminho absoluto.

python
os.makedirs(app.config["UPLOAD_FOLDER"], exist_ok=True)
Cria static/uploads/ fisicamente no disco. Fim do diálogo com o SO.

python
init_db(app)
Chamada para fora. create_app chama init_db (definida em app/database.py).

init_db(app) — app/database.py
python
with app.app_context():
Cria um contexto artificial — porque get_db() usa flask.g, que só existe dentro de um contexto (normalmente request). Aqui estamos ainda no boot, então criamos um "contexto fake".

python
db = get_db()
Chamada em cadeia:

text
init_db → get_db → sqlite3.connect → abre portaria.db
get_db() vê que g.db não existe

Abre conexão com o arquivo em current_app.config["DATABASE"]

Guarda em g.db para reutilização

Retorna a conexão

python
with open(app.config["SCHEMA"], encoding="utf-8") as f:
    db.executescript(f.read())
Diálogo do Python com o SQLite:

Abre o arquivo schema.sql do disco

Lê o texto inteiro

Pede ao SQLite para executar tudo de uma vez (o arquivo tem todos os CREATE TABLE)

As tabelas nascem (ou já existem, graças a IF NOT EXISTS)

python
db.commit()
Salva no arquivo .db.

python
app.teardown_appcontext(close_db)
Registra um callback: toda vez que uma request acabar, o Flask chama close_db() sozinho. A partir de agora nunca mais haverá conexão aberta sobrando.

Retorno: init_db volta para create_app sem devolver nada.

_auto_seed(app) — app/init.py
python
with app.app_context():
    db = get_db()   # reaproveita a mesma conexão de novo
    total = db.execute("SELECT COUNT(*) FROM usuarios").fetchone()[0]
Diálogo:

_auto_seed pergunta ao banco: "quantos usuários tem?"

Banco responde: 0 (primeira vez) ou 2 (já rodou antes)

Se 0 → executa 2 INSERT com generate_password_hash(...) (werkzeug gera hash com salt)

Se 2 → não faz nada

_auto_permissoes(app) — app/init.py
Diálogo:

text
_auto_permissoes → ENTITIES (do api.py)
  → para cada chave (visitantes, encomendas, ...)
    → INSERT OR IGNORE INTO permissoes ... VALUES ('porteiro', ?, 1, 1, 1, 0)
Comunicação cruzada: app/__init__.py (este arquivo) importou ENTITIES de app/api.py no topo. A lista de módulos vem de lá. Sem essa importação, o auto-seed não saberia quais módulos existem.

Registro dos blueprints
python
app.register_blueprint(auth_bp)
app.register_blueprint(api_bp)
app.register_blueprint(assistant_bp)
O que acontece por baixo:

Cada blueprint já tinha suas rotas registradas internamente (via decorator @auth_bp.route(...) etc.) — isso ocorreu durante os imports do topo

register_blueprint copia essas rotas para o "mapa de rotas" do app principal

A partir daqui, o Flask sabe: GET /login → auth.login_page, POST /api/<entity> → api.crud_collection, etc.

Cada blueprint vem com seu url_prefix:

auth_bp → sem prefixo (rotas na raiz: /login, /api/me, etc.)

api_bp → url_prefix="/api"

assistant_bp → url_prefix="/api/assistant"

Rota raiz
python
@app.route("/")
def index():
    ...
Só define a função. Ninguém chamou ainda.

return app
create_app termina devolvendo o app pronto. Volta para run.py linha 3.

run.py linha 5-6
python
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
Última etapa do boot:

Abre a porta 5000 em todas interfaces

Fica em loop esperando conexões HTTP

print(" * Running on http://0.0.0.0:5000")

Fim do boot. O servidor agora está "ouvindo".

Parte 2 — Primeiro GET (usuário abre o navegador)
text
Usuário digita http://192.168.1.10:5000/
O que o Flask faz antes de chamar código Python
Aceita a conexão TCP na porta 5000

Lê a linha GET / HTTP/1.1

Consulta seu mapa de rotas (construído no register_blueprint)

Encontra / → função index() do app/__init__.py

index() — app/init.py
python
if "user_id" not in session:
Diálogo com o cookie: o Flask leu o cookie session, desassinou com SECRET_KEY, e reconstruiu o dict session. Como é a primeira visita, está vazio.

python
return redirect(url_for("auth.login_page"))
Duas chamadas encadeadas:

url_for("auth.login_page") → pergunta ao mapa de rotas: "qual URL corresponde ao endpoint login_page do blueprint auth?" → resposta: /login

redirect("/login") → devolve HTTP 302 com Location: /login

Navegador segue o redirect
text
GET /login HTTP/1.1
Flask → login_page() (app/auth.py)
python
@auth_bp.route("/login", methods=["GET"])
def login_page():
    return render_template("login.html")
render_template("login.html"):

Flask procura login.html em templates/ (configurado em create_app)

Renderiza o Jinja2 (aqui não há variáveis, é HTML puro)

Devolve HTML com Content-Type: text/html

Navegador recebe, baixa style.css (link <link rel="stylesheet">), e mostra a tela

Primeira página pronta. Nenhuma consulta SQL foi feita — só HTML estático.

Parte 3 — Login (form submit → sessão criada)
Usuário preenche e clica "Entrar"
login.html — inline <script>
javascript
document.getElementById('login-form').onsubmit = async (e) => {
Sequestra o submit do form. Sem isso, o navegador faria um POST nativo e recarregaria a página.

javascript
e.preventDefault();
Cancela o comportamento padrão.

javascript
const fd = new FormData(e.target);
const r = await fetch('/api/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ usuario: fd.get('usuario'), senha: fd.get('senha') })
});
Requisição HTTP:

text
POST /api/login HTTP/1.1
Content-Type: application/json

{"usuario":"admin","senha":"admin123"}
Flask roteia
Mapa de rotas: POST /api/login → auth.api_login

api_login() — app/auth.py
python
data = request.get_json() or request.form
Diálogo request: Flask parseia o corpo JSON e o request.get_json() retorna {"usuario":"admin","senha":"admin123"}.

python
usuario = (data.get("usuario") or "").strip().lower()
senha = data.get("senha") or ""
Normaliza: "admin" e "admin123".

python
db = get_db()
Conversa com database.py:

get_db vê que g.db já existe (mesma requisição? não — nova requisição → nova conexão, mas o padrão é uma por request)

Abre conexão nova ao arquivo .db

python
row = db.execute(
    "SELECT * FROM usuarios WHERE lower(usuario)=? AND ativo=1", (usuario,)
).fetchone()
SQL enviado ao SQLite:

sql
SELECT * FROM usuarios WHERE lower(usuario)='admin' AND ativo=1;
SQLite busca, encontra a linha do admin, devolve como Row (dict-like).

python
if not row or not check_password_hash(row["senha_hash"], senha):
    return jsonify({"error": "Usuário ou senha inválidos"}), 401
check_password_hash:

Pega o hash salvo no banco (ex.: pbkdf2:sha256:600000$...)

Aplica o mesmo algoritmo com o mesmo salt na senha digitada

Compara os dois resultados

Retorna True ou False

Se falso → 401 (o erro que você viu antes quando o banco estava vazio). Se verdadeiro:

python
session["user_id"] = row["id"]
session["usuario"] = row["usuario"]
session["nome"] = row["nome"]
session["perfil"] = row["perfil"]
O que acontece no Flask:

session é um dict especial

Cada atribuição marca o dict como "modificado"

No final da request, Flask serializa o dict → assina com SECRET_KEY → coloca no cookie session=...

Navegador recebe Set-Cookie: session=eyJ1c2VyX2lkIjo...

python
log_audit("Login", row["usuario"])
Chama utils.log_audit:

python
db = get_db()  # mesma conexão da request
db.execute("INSERT INTO auditoria ... VALUES (?,?,?,?)", (...))
db.commit()
Grava em auditoria (não bloqueia a resposta — acontece antes do retorno).

python
return jsonify({"ok": True})
Resposta HTTP 200 com {"ok": true}.

login.html — de volta ao JS
javascript
const j = await r.json();
if (!r.ok) { msg.textContent = j.error || 'Erro'; return; }
location.href = '/';
r.ok é true → redireciona para /.

Navegador faz novo GET
text
GET / HTTP/1.1
Cookie: session=eyJ1c2VyX2lkIjo...
Flask → index() (app/init.py)
python
if "user_id" not in session:
Agora sim o Flask:

Lê o cookie session=...

Verifica a assinatura com SECRET_KEY

Se válida, decodifica em dict → {"user_id": 1, "usuario": "admin", ...}

"user_id" in session → True

python
return render_template("index.html")
Devolve a SPA.

Fim da parte "servidor HTML". A partir daqui o navegador carrega o JS e conversa via API.

Parte 4 — SPA carrega (o cliente começa a conversar)
Navegador carrega index.html
html
<script src="/static/js/app.js"></script>
Navegador faz GET /static/js/app.js.

Flask serve o arquivo estático
Aqui não há Python envolvido: o Flask tem um servidor interno que entrega arquivos de static/ diretamente. Sem passar por create_app.

app.js — execução
Arquivo inteiro roda de cima para baixo:

const $ = ... — define helpers (não executa nada com eles ainda)

const ENTITIES = {...} — constrói o grande dicionário de configuração das 24 entidades. Nada é enviado ao servidor ainda — só prepara

const MENU = [...] — constrói a lista de itens de menu

let session = null; ... — variáveis de estado

Última linha: boot(); — chama o ponto de entrada

boot() — app.js
javascript
try {
  session = await api('/api/me');
} catch { location.href = '/login'; return; }
Chama api('/api/me'). Vamos abrir api:

api() — app.js
javascript
async function api(path, opts={}) {
  const r = await fetch(path, {headers:{'Content-Type':'application/json'}, ...opts});
Requisição HTTP:

text
GET /api/me HTTP/1.1
Cookie: session=...
Flask roteia → api_me() — app/auth.py
python
@auth_bp.route("/api/me")
@login_required
def api_me():
Decorator login_required executa primeiro (o Python embrulha a função). Verifica "user_id" not in session:

Se não tem → devolve 401

Se tem → chama api_me "de verdade"

Como o usuário logou, session tem os dados. A função roda:

python
from .api import ENTITIES
Import tardio (import circular evitado). Traz o dicionário de entidades.

python
db = get_db()
perfil = session.get("perfil")   # "admin"
todas = list(ENTITIES.keys()) + ESPECIAIS  # 24 + 3 = 27
python
if perfil == "admin":
    perms = {ent: {"ver": True, "criar": True, "editar": True, "excluir": True}
             for ent in todas}
Admin → todos os 27 módulos liberados em 4 ações.

python
return jsonify({
    "usuario": session.get("usuario"),
    "nome": session.get("nome"),
    "perfil": perfil,
    "permissoes": perms,
})
Resposta JSON:

json
{
  "usuario": "admin",
  "nome": "Administrador",
  "perfil": "admin",
  "permissoes": { "visitantes": {"ver":true,...}, ... }
}
api() — de volta ao JS
javascript
if (r.status === 401) { ... }   // não é 401
const j = await r.json();        // parseia JSON
if (!r.ok) throw new Error(...);  // 200, ok
return j;                        // devolve para boot()
boot() continua
javascript
window.PERMS = session.permissoes || {};
Guarda as permissões globalmente — qualquer parte do JS pode consultar window.PERMS['encomendas'].excluir.

javascript
$('#user-name').textContent = session.nome;
$('#user-role').textContent = session.perfil === 'admin' ? 'Administrador' : 'Porteiro';
$('#user-avatar').textContent = session.nome.trim()[0].toUpperCase();
Preenche o avatar e nome no canto da sidebar.

javascript
unidadesCache = await api('/api/unidades').catch(()=>[]);
Nova requisição GET /api/unidades → api.listar_unidades():

SELECT DISTINCT unidade FROM moradores ORDER BY unidade

Retorna ["A-101", "A-102", "A-201", ...]

Guardado em unidadesCache para uso em <datalist> nos formulários

javascript
montarNav();
montarNav() — app.js
javascript
const isAdmin = session.perfil === 'admin';   // true
const perms = window.PERMS || {};
$('#nav').innerHTML = MENU.map(m => {...}).join('');
Para cada item de MENU:

Se tem group → <div class="group">Portaria</div>

Se m.admin && !isAdmin → retorna '' (pula)

Se m.ent → consulta perms[m.ent]; se ver === false → pula; senão cria <button>

Senão (item comum) → cria <button>

Resultado: o <nav id="nav"> (antes vazio) é preenchido com 35+ botões.

javascript
$$('#nav button').forEach(b => b.onclick = ()=>navigate(b.dataset.nav));
Liga o clique de cada botão à função navigate com o nome da view.

javascript
navigate('dashboard');
navigate('dashboard') — app.js
javascript
const item = MENU.find(m => m.view === 'dashboard');
Acha {view:'dashboard', ico:'📊', lbl:'Painel'}.

javascript
if (item.admin && !isAdmin) return toast('Acesso restrito.', true);   // não
const p = perms['dashboard'];   // undefined (admin vê tudo, mas não tem entrada em PERMS)
if (!isAdmin && p && p.ver === false) { ... }   // pulado porque isAdmin
Atualiza títulos:

javascript
$('#view-title').textContent = 'Painel';
$('#topbar-right').innerHTML = new Date().toLocaleDateString(...);
javascript
if (view === 'dashboard') return renderDashboard();
renderDashboard() — app.js
javascript
const s = await api('/api/dashboard/stats');
Requisição GET /api/dashboard/stats.

Flask → dashboard_stats() — app/api.py
python
if not pode("dashboard", "ver"):
    return jsonify({"error": "sem permissão"}), 403
Chama utils.pode("dashboard","ver"):

session["perfil"] == "admin" → retorna True imediatamente (curto-circuito)

python
def s(sql):
    return db.execute(sql).fetchone()[0]
Define helper local.

python
return jsonify({
    "dentro": s("SELECT COUNT(*) FROM visitantes WHERE status='dentro'"),
    "entradas_hoje": s("SELECT COUNT(*) FROM visitantes WHERE date(entrada)=date('now')"),
    ...
})
9 SELECTs executados um por um, cada um conta uma coisa.

Resposta:

json
{
  "dentro": 0, "entradas_hoje": 0, "encomendas_pend": 0, "ocorrencias_abertas": 0,
  "manutencoes_pend": 0, "estoque_baixo": 0, "moradores": 0, "caixas_alerta": 0,
  "tags_pendentes": 0
}
renderDashboard() continua
Monta os 9 cards:

javascript
$('#view').innerHTML = `
  <div class="stats">${cards.map(([l,n,i,c])=>`
    <div class="stat"><div class="ic ${c}">${i}</div><div><div class="num">${n}</div><div class="lbl">${l}</div></div></div>
  `).join('')}</div>
  <section class="card">...ações rápidas...</section>
`;
A página aparece. O usuário vê os cards.

javascript
$$('[data-quick]').forEach(b => b.onclick = () => {...});
$$('[data-nav]').forEach(b => b.onclick = () => navigate(b.dataset.nav));
Liga os botões "Nova entrada", "Nova encomenda", etc.

Fim do ciclo de boot. Agora o navegador está "vivo", esperando cliques.

Parte 5 — Ciclo de uma interação CRUD
Cenário: usuário clica "Visitantes" no menu
montarNav → botão.onclick → navigate('visitantes')
javascript
currentView = 'visitantes';
$$('#nav button').forEach(b => b.classList.toggle('active', b.dataset.nav === 'visitantes'));
$('#view-title').textContent = 'Visitantes';
...
if (item.ent) return renderEntity(item.ent);
renderEntity('visitantes') — app.js
javascript
const cfg = ENTITIES['visitantes'];
const p = window.PERMS?.['visitantes'];   // {ver,criar,editar,excluir} ou undefined
Monta o cabeçalho:

javascript
$('#view').innerHTML = `
  <div class="toolbar">
    <input class="search" id="search" placeholder="🔎 Buscar...">
    <div class="spacer"></div>
    ${(isAdmin || p.criar) ? `<button class="btn primary" data-action="new">➕ Novo</button>` : ''}
  </div>
  <section class="card"><div class="tbl-wrap"><table class="tbl" id="tbl">
    <thead><tr>${cfg.cols.map(c => `<th>${campoLabel(cfg, c)}</th>`).join('')}<th>Ações</th></tr></thead>
    <tbody><tr><td colspan="${cfg.cols.length+1}" class="empty">Carregando...</td></tr></tbody>
  </table></div></section>
`;
Depois:

javascript
const load = async () => {
  const q = $('#search').value.trim();
  const rows = await api(`/api/visitantes${q ? '?q='+encodeURIComponent(q) : ''}`);
  ...
};
load();
api('/api/visitantes') → GET /api/visitantes
Flask → crud_collection('visitantes') — app/api.py
python
if entity not in ENTITIES:
    return jsonify({"error": "entidade desconhecida"}), 404
'visitantes' in ENTITIES → True.

python
acao = "ver" if request.method == "GET" else "criar"
acao = "ver".

python
if not pode(entity, acao):
    return jsonify({"error": "sem permissão"}), 403
utils.pode("visitantes","ver") — utils.py
python
if session.get("perfil") == "admin":
    return True
Admin → True (curto-circuito). Para porteiro, seria consulta ao banco:

python
row = db.execute("SELECT pode_ver, ... FROM permissoes WHERE perfil=? AND entidade=?", (perfil, "visitantes")).fetchone()
return bool(row["pode_ver"])
De volta ao crud_collection
python
cfg = ENTITIES['visitantes']
db = get_db()

q = request.args.get("q","").strip()  # ""
filtros = request.args.to_dict()      # {}

sql = "SELECT * FROM visitantes"
wheres, params = [], []

if q and cfg.get("search"):  # q vazio → pula
    ...

for k, v in filtros.items():  # vazio → pula
    ...

sql += " ORDER BY entrada DESC"
rows = db.execute(sql, params).fetchall()
return jsonify([dict(r) for r in rows])
SQL enviado:

sql
SELECT * FROM visitantes ORDER BY entrada DESC;
Resposta:

json
[{"id":1,"nome":"Pedro Santos","unidade":"A-101",...}, ...]
load() continua — app.js
javascript
if (!rows.length) {
  tbody.innerHTML = `<tr><td colspan="...">Nenhum registro.</td></tr>`;
  return;
}
Se vazio → mostra mensagem. Se tem linhas:

javascript
tbody.innerHTML = rows.map(r => {
  const canEdit = isAdmin || p.editar;  // true (admin)
  const canDel  = isAdmin || p.excluir; // true
  const rowActionsHTML = cfg.rowActions(r); // "Registrar saída" se status='dentro'
  const btnEdit = canEdit ? `<button data-edit="${r.id}">Editar</button>` : '';
  const btnDel  = canDel  ? `<button data-del="${r.id}">Excluir</button>` : '';
  return `<tr>
    ${cfg.cols.map(c => `<td>${cellValue(c, r[c])}</td>`).join('')}
    <td><div class="actions">${rowActionsHTML}${btnEdit}${btnDel}</div></td>
  </tr>`;
}).join('');
Tabela preenchida.

javascript
$$('[data-edit]', tbody).forEach(b => b.onclick = ()=>openForm('visitantes', rows.find(r=>r.id==b.dataset.edit)));
$$('[data-del]', tbody).forEach(b => b.onclick = ()=>deleteItem('visitantes', b.dataset.del));
$$('[data-act]', tbody).forEach(b => b.onclick = ()=>rowAction('visitantes', b.dataset.act, b.dataset.id));
Cada botão agora tem um handler ligado à linha correta.

Usuário clica "Novo"
openForm('visitantes') — app.js
javascript
const cfg = ENTITIES['visitantes'];
const bodyHTML = cfg.fields.map(f => renderField(f, item?.[f.n])).join('');
renderField gera HTML para cada campo:

Exemplo: {n:'nome', l:'Nome', t:'text', req:true} →

html
<label class="field"><span>Nome *</span><input name="nome" type="text" value="" required></label>
Exemplo: {n:'entrada', l:'Entrada', t:'datetime-local', req:true, def:()=>new Date().toISOString().slice(0,16)} →

html
<label class="field"><span>Entrada *</span><input name="entrada" type="datetime-local" value="2024-01-15T14:30" required></label>
def() é executado — a data/hora atual é preenchida.

javascript
openModal({ title: 'Novo visitante', body: bodyHTML, wide: false, onSave: async (formData) => {...} });
openModal() — app.js
Cria <div class="overlay"><div class="modal"><form id="mform">...

javascript
$('#mform', root).onsubmit = async e => {
  e.preventDefault();
  const fd = new FormData(e.target);
  try { await onSave(fd); close(); } catch(err){ toast(err.message, true); }
};
O modal está na tela. Aguardando o usuário preencher e clicar "Salvar".

Usuário preenche e clica "Salvar"
onSave(formData) — de volta em openForm
javascript
let data = {};
for (const f of cfg.fields) {
  let v = formData.get(f.n);
  if (f.t === 'file') { ... }           // não há file em visitantes
  if (v === '') v = null;
  if (f.t === 'number' && v !== null) v = Number(v);
  data[f.n] = v;
}
data montado:

json
{"nome":"Maria Silva","documento":"12.345.678-9","unidade":"A-101","motivo":"visita","entrada":"2024-01-15T14:30","saida":null,"status":"dentro"}
javascript
try {
  await api(`/api/visitantes`, {method:'POST', body: JSON.stringify(data)});
  toast('Cadastrado com sucesso.');
  renderEntity('visitantes');  // recarrega a tabela
} catch (err) { toast(err.message, true); }
api('/api/visitantes', {method:'POST', body:...})
Requisição HTTP:

text
POST /api/visitantes HTTP/1.1
Content-Type: application/json

{"nome":"Maria Silva",...}
Flask → crud_collection('visitantes') (mesmo endpoint, outro método)
python
acao = "ver" if request.method == "GET" else "criar"  # "criar"
if not pode("visitantes", "criar"): ...                # admin → True
python
data = request.get_json()   # {"nome":"Maria Silva",...}
campos = [f for f in cfg["fields"] if f in data and data[f] not in (None, "")]
campos = ["nome","documento","unidade","motivo","entrada","status"]
(saida foi None → excluída)

python
sql = "INSERT INTO visitantes (nome,documento,unidade,motivo,entrada,status) VALUES (?,?,?,?,?,?)"
cur = db.execute(sql, [data[f] for f in campos])
SQL:

sql
INSERT INTO visitantes (nome,documento,unidade,motivo,entrada,status)
VALUES ('Maria Silva','12.345.678-9','A-101','visita','2024-01-15T14:30','dentro');
python
db.commit()
log_audit("Criar visitantes", "Maria Silva")
return jsonify({"id": cur.lastrowid}), 201
Resposta: {"id": 4} com status 201.

onSave — de volta ao JS
toast('Cadastrado com sucesso.') → openModal fecha (close()) → renderEntity('visitantes') re-executa o load() → novo GET /api/visitantes → a nova linha aparece na tabela.

Ciclo completo de criação. 🔄

Parte 6 — Ciclo de permissões (admin)
Admin clica "🔐 Permissões" no menu
navigate('permissoes') → renderPermissoes()
javascript
async function renderPermissoes() {
  const data = await api('/api/permissoes');
Flask → api_permissoes_list() — app/auth.py
python
@auth_bp.route("/api/permissoes", methods=["GET"])
@login_required
@admin_required
def api_permissoes_list():
Ordem real de execução:

admin_required outer wrapper verifica session["perfil"] == "admin" → True

Chama inner wrapper de login_required → verifica "user_id" in session → True

Chama api_permissoes_list() de verdade

python
from .api import ENTITIES
ESPECIAIS = ["dashboard", "assistente", "auditoria"]
db = get_db()
rows = db.execute("SELECT * FROM permissoes WHERE perfil='porteiro'").fetchall()
return jsonify({
    "entidades": list(ENTITIES.keys()) + ESPECIAIS,
    "especiais": ESPECIAIS,
    "permissoes": {r["entidade"]: {...} for r in rows},
})
Resposta:

json
{
  "entidades": ["visitantes","encomendas",...,"dashboard","assistente","auditoria"],
  "especiais": ["dashboard","assistente","auditoria"],
  "permissoes": {
    "visitantes": {"ver":true,"criar":true,"editar":true,"excluir":false},
    "encomendas": {"ver":true,"criar":true,"editar":true,"excluir":false},
    ...
  }
}
renderPermissoes() monta a tabela com toggles
javascript
const linhas = data.entidades.map(ent => {
  const p = data.permissoes[ent];
  const isEsp = especiais.has(ent);
  ...
  const toggle = (acao, checked, disabled=false) => `
    <label class="toggle">
      <input type="checkbox" data-ent="${ent}" data-acao="${acao}"
             ${checked ? 'checked' : ''} ${disabled ? 'disabled' : ''}>
      <span></span>
    </label>`;
  return `<tr>
    <td>...</td>
    <td class="center">${toggle('ver', p.ver)}</td>
    ...
  </tr>`;
}).join('');
Cada linha da tabela tem 4 toggles.

Admin desativa "Excluir" em Encomendas
javascript
cb.onchange = async () => {
  const ent = cb.dataset.ent;   // "encomendas"
  const estado = {};
  $$(`input[data-ent="${ent}"]`).forEach(x => { estado[x.dataset.acao] = x.checked; });
  // estado = { ver:true, criar:true, editar:true, excluir:false }
  await api(`/api/permissoes/${ent}`, {method:'PUT', body: JSON.stringify(estado)});
  toast(`Permissões de ${ent} atualizadas.`);
};
Flask → api_permissoes_update('encomendas') — app/auth.py
python
data = request.get_json()   # {ver:true, criar:true, editar:true, excluir:false}
campos = {k: 1 if data.get(k) else 0 for k in ("ver","criar","editar","excluir")}
# campos = {ver:1, criar:1, editar:1, excluir:0}
existe = db.execute("SELECT id FROM permissoes WHERE perfil='porteiro' AND entidade='encomendas'").fetchone()
if existe:
    db.execute("UPDATE permissoes SET pode_ver=?,... WHERE perfil='porteiro' AND entidade=?", (1,1,1,0,'encomendas'))
else:
    db.execute("INSERT INTO permissoes ...")
db.commit()
log_audit("Alterar permissões", "encomendas: {ver:1, criar:1, editar:1, excluir:0}")
return jsonify({"ok": True, "permissoes": campos})
Banco atualizado. Na próxima requisição do porteiro:

Porteiro tenta excluir encomenda
javascript
// No frontend, o botão nem aparece porque p.excluir = false
Mas se ele abrir o console e forçar:

javascript
fetch('/api/encomendas/5', {method:'DELETE'})
Flask → crud_item('encomendas', 5)
python
acao = {"GET":"ver","PUT":"editar","DELETE":"excluir"}[request.method]  # "excluir"
if not pode("encomendas", "excluir"):
    return jsonify({"error": "sem permissão"}), 403
pode("encomendas", "excluir") — utils.py
python
if session.get("perfil") == "admin":   # porteiro → False
    return True
perfil = "porteiro"
row = db.execute("SELECT pode_excluir FROM permissoes WHERE perfil='porteiro' AND entidade='encomendas'").fetchone()
# row = {pode_excluir: 0}
return bool(0)   # False
Retorna 403.

Frontend recebe 403
javascript
if (!r.ok) throw new Error(j.error || 'Erro');   // Error("sem permissão")
No .catch(err) { toast(err.message, true); } → mostra "sem permissão".

Segurança em camadas: UI esconde + backend recusa. 🛡️

Parte 7 — Assistente (chat)
Usuário clica "Assistente" → renderAssistente()
javascript
$('#view').innerHTML = `<div class="chat-wrap">...`;
$('#chat-form').onsubmit = e => {
  e.preventDefault();
  const t = $('#chat-text').value.trim();
  if (!t) return;
  $('#chat-text').value = '';
  enviarMsg(t);
};
Usuário digita "Resumo" e envia
enviarMsg('Resumo') — app.js
javascript
appendMsg('user', 'Resumo');   // bolha azul do usuário
// adiciona "digitando..."
const r = await api('/api/assistant/chat', {method:'POST', body: JSON.stringify({message:'Resumo'})});
appendMsg('bot', r.reply);
Flask → chat() — app/assistant.py
python
@assistant_bp.route("/chat", methods=["POST"])
@login_required
def chat():
    if not pode("assistente", "ver"):
        return jsonify({"reply": "🔒 ..."})
Admin → pode retorna True imediatamente.

python
data = request.get_json()   # {"message":"Resumo"}
msg = "Resumo"
t = norm("Resumo")   # "resumo"
db = get_db()
unidade = _achar_unidade("Resumo", db)   # None
Cascata de if re.search(...)
python
if re.search(r"\b(resumo|panorama|status|situacao geral|visao geral)\b", t):
Match! "resumo" casa com o padrão.

python
dentro   = db.execute("SELECT COUNT(*) FROM visitantes WHERE status='dentro'").fetchone()[0]
entradas = db.execute("SELECT COUNT(*) FROM visitantes WHERE date(entrada)=date('now')").fetchone()[0]
encom    = db.execute("SELECT COUNT(*) FROM encomendas WHERE status='pendente'").fetchone()[0]
...
9 SELECTs, um por métrica. Cada um conta uma coisa diferente.

python
reply = (
    "📊 *Resumo — " + date.today().strftime("%d/%m/%Y") + "*\n\n"
    "• Pessoas no local: " + str(dentro) + "\n"
    ...
)
return jsonify({"reply": reply})
Resposta:

json
{"reply": "📊 *Resumo — 15/01/2024*\n\n• Pessoas no local: 0\n• Entradas hoje: 0\n..."}
Frontend recebe
javascript
typing.remove();
appendMsg('bot', r.reply);
Bolha cinza com o texto. Fim da interação.

Se fosse "quem está na portaria?"
chat() percorre a cascata:

não é saudação

não é ajuda

não é resumo

não é encomenda

cai no if re.search(r"\b(visitante|visita|quem esta|dentro|portaria|entrada|saida)\b", t): → match

SELECT * FROM visitantes WHERE status='dentro'

monta "👥 *N pessoa(s) no local:*\n\n• Nome → Unidade (hora)"

retorna {"reply": ...}

Se fosse "manutenção da piscina"
cai no if re.search(r"\b(manutencao|...)\b", t)

tipos_map contém "piscina": "piscina"

loop: if "piscina" in t: alvo = "piscina"; break

SELECT * FROM manutencoes WHERE tipo='piscina'

formata 3 campos: título, status, data_prevista

A cascata de ifs é a "árvore de decisão" do assistente.

📊 Diagrama Geral de Quem Chama Quem








































🔗 Tabela de Comunicação entre Arquivos
Origem	Função/linha	Chamada	Destino	Passa	Recebe
run.py:1	import	—	app/__init__.py	—	módulo carregado
run.py:3	app = create_app()	—	create_app	—	app Flask
create_app	init_db(app)	—	database.init_db	app	—
init_db	get_db()	—	database.get_db	—	conexão
init_db	db.executescript()	—	SQLite	DDL	—
create_app	_auto_seed(app)	—	mesma pasta	app	—
_auto_seed	db.execute("SELECT COUNT")	—	SQLite	SQL	0 ou 2
create_app	_auto_permissoes(app)	—	mesma pasta	app	—
_auto_permissoes	ENTITIES (importado)	—	api.py (dict)	—	lista
create_app	register_blueprint(...)	—	Flask	blueprint	—
Navegador	GET /login	HTTP	Flask router	—	HTML
Flask router	login_page()	—	auth.py	—	HTML
Navegador	POST /api/login	HTTP	Flask router	JSON	JSON
Flask router	api_login()	—	auth.py	—	JSON
api_login	check_password_hash	—	werkzeug	senha+hash	True/False
api_login	get_db()	—	database.py	—	conexão
api_login	session[...]=	—	Flask	—	cookie
api_login	log_audit	—	utils.py	ação	—
log_audit	db.execute("INSERT")	—	SQLite	SQL	—
Navegador	GET /api/me	HTTP	Flask router	cookie	JSON
Flask router	api_me()	—	auth.py	—	JSON
api_me	ENTITIES (import)	—	api.py	—	dict
api_me	SELECT FROM permissoes	—	SQLite	SQL	rows
Navegador	GET /api/dashboard/stats	HTTP	Flask router	—	JSON
Flask router	dashboard_stats()	—	api.py	—	JSON
dashboard_stats	pode("dashboard","ver")	—	utils.py	ent,acao	True
pode	session.get("perfil")	—	Flask	—	"admin"
dashboard_stats	9x SELECT COUNT	—	SQLite	SQL	números
Navegador	GET /api/visitantes	HTTP	Flask router	—	JSON
Flask router	crud_collection('visitantes')	—	api.py	—	JSON
crud_collection	pode('visitantes','ver')	—	utils.py	—	True
crud_collection	SELECT FROM visitantes	—	SQLite	SQL	rows
Navegador	POST /api/visitantes	HTTP	Flask router	JSON	JSON+201
crud_collection	pode('visitantes','criar')	—	utils.py	—	True
crud_collection	INSERT INTO visitantes	—	SQLite	SQL+params	lastrowid
crud_collection	log_audit	—	utils.py	—	—
Navegador	PUT /api/permissoes/encomendas	HTTP	Flask router	JSON	JSON
Flask router	api_permissoes_update('encomendas')	—	auth.py	—	JSON
api_permissoes_update	UPDATE permissoes	—	SQLite	SQL	—
api_permissoes_update	log_audit	—	utils.py	—	—
Navegador	POST /api/assistant/chat	HTTP	Flask router	JSON	JSON
Flask router	chat()	—	assistant.py	—	JSON
chat	pode('assistente','ver')	—	utils.py	—	True
chat	_achar_unidade	—	mesma função	texto	"A-101" ou None
chat	Nx SELECT	—	SQLite	SQL	linhas
chat	return jsonify	—	Flask	dict	JSON
🎯 A Grande Narrativa
Boot: run.py chama create_app. O gerente (init) contrata todos os funcionários (imports), cria o arquivo (database), verifica se as senhas padrão existem (auto-seed), garante que cada módulo tem permissões (auto-permissões), e diz ao Flask "estas são as rotas" (register_blueprint). O servidor senta e espera.

Primeira visita: o navegador bate na porta, o gerente (index) diz "você não tem crachá, vá para a recepção" (redirect). A recepção (login.html) mostra o formulário.

Login: o cliente entrega usuário+senha. A recepção (api_login) confere no arquivo (banco) e emite um crachá (cookie de sessão assinado). Agora o cliente pode entrar.

SPA: o gerente entrega a sala principal (index.html). O JS (app.js) pergunta "quem sou eu?" (/api/me), recebe crachá + lista do que pode fazer. Monta o menu condicionalmente (montarNav) e abre o painel (renderDashboard).

Interações: cada clique é uma chamada HTTP → Flask → pode() (o segurança interno) → SQL → resposta → o JS injeta HTML. O padrão é sempre o mesmo: validar → consultar → responder → desenhar.

Segurança: duas camadas. O frontend esconde botões sem permissão. O backend recusa a operação mesmo que alguém burle a UI. Tudo é auditado.

O assistente: regex em cascata decide a intenção. Cada ramo faz uma ou mais consultas SQL e devolve texto formatado. Sem LLM, sem internet, 100% local.

