# Atividade Prática: Autenticação com Firebase, Perfis de Acesso (RBAC) e Deploy no Render

**Objetivo:** Implementar autenticação com **Firebase Authentication** (E-mail/Senha e Google) e controle de acesso baseado em perfis (**User**, **Técnico/Analista de TI** e **Admin**) no sistema de **Chamados de TI**, vinculando os registros no banco de dados e publicando a aplicação em produção no **Render**.

---

## 1. Preparação do Ambiente e Repositório

1. **Fork e Clone:**
   * Acesse o repositório base: `https://github.com/profjacques/chamados-ti`
   * Clique em **Fork** (no canto superior direito) para criar uma cópia do projeto na sua conta pessoal do GitHub.
   * Clone o **seu** repositório para a máquina local:
     ```bash
     git clone https://github.com/SEU-USUARIO/chamados-ti.git
     cd chamados-ti
     ```
2. **Análise da Estrutura:**
   * Inspecione os arquivos existentes no projeto (HTML, CSS, JavaScript e configuração de persistência) para identificar onde os chamados são criados, listados, atualizados e excluídos.

---

## 2. Configuração do Firebase

1. Acesse o [Console do Firebase](https://console.firebase.google.com/) e crie um novo projeto (ex: `chamados-ti-rbac`).
2. **Ativar Métodos de Login:**
   * No menu lateral, acesse **Criação (Build) > Authentication > Sign-in method**.
   * Ative o provedor **E-mail/senha** (habilite apenas a primeira opção; mantenha o link por e-mail desativado).
   * Ative o provedor **Google**, informe um e-mail de suporte do projeto e salve.
3. **Ativar Banco de Dados (Firestore ou Realtime Database):**
   * Crie o banco de dados em modo de teste inicialmente para armazenar a coleção de usuários (`usuarios`) e de chamados (`chamados`).
4. **Registrar Aplicativo Web:**
   * Na visão geral do projeto, clique no ícone da Web (`</>`) para registrar o app e copie o objeto `firebaseConfig`.
5. **Domínios Autorizados:**
   * Em **Authentication > Settings > Authorized domains**, verifique se `localhost` está listado. Após o deploy, você deverá adicionar aqui o domínio gerado pelo Render (`seu-projeto.onrender.com`).

---

## 3. Modelagem dos 3 Perfis de Usuário (RBAC)

Como o **Firebase Authentication** gerencia apenas a identidade (login), você deve criar uma coleção `usuarios` no banco de dados vinculada pelo `uid` de cada conta autenticada, contendo o campo `perfil`:

```json
{
  "uid": "abc123xyz",
  "nome": "João Silva",
  "email": "joao@empresa.com",
  "perfil": "user"
}
```

### Regras de Negócio por Perfil

| Perfil (`role`) | Descrição | Permissões no Sistema |
| :--- | :--- | :--- |
| **`user`** *(Padrão)* | Usuário comum / Solicitante | • **Criar** novos chamados.<br>• **Visualizar** apenas os próprios chamados (`uid == user.uid`).<br>• Não pode alterar status técnico nem excluir chamados de terceiros. |
| **`tecnico`** | Técnico / Analista de TI | • **Visualizar** todos os chamados abertos no sistema.<br>• **Assumir** chamados, adicionar parecer técnico e **alterar o status** (`Aberto`, `Em Atendimento`, `Resolvido`).<br>• Não pode gerenciar perfis de usuários nem excluir chamados. |
| **`admin`** | Administrador do Sistema | • **Acesso total** a todos os chamados (criar, editar, alterar status e excluir).<br>• **Painel de Gestão de Usuários:** visualizar usuários cadastrados e alterar o campo `perfil` (`user`, `tecnico`, `admin`). |

> **Regra de Primeiro Acesso:** Todo usuário que criar conta via **E-mail/Senha** ou fizer o primeiro login via **Google** deve ser salvo automaticamente na coleção `usuarios` com o perfil padrão `"user"` (caso o registro ainda não exista no banco). Para testar o painel administrativo, altere manualmente o campo `perfil` do seu próprio usuário para `"admin"` diretamente no Console do Firebase.

---

## 4. Requisitos Técnicos de Implementação

### A. Tela de Login e Cadastro (`login.html` ou seção dedicada)
* Formulário com campos de **Nome**, **E-mail** e **Senha**, com botões para **Entrar** (`signInWithEmailAndPassword`) e **Criar Conta** (`createUserWithEmailAndPassword`).
* Botão dedicado para **Entrar com Google** (`signInWithPopup` com `GoogleAuthProvider`).
* Ao concluir o cadastro ou o primeiro login com Google, verificar se o `uid` já existe na coleção `usuarios`; caso não exista, gravar o documento inicial com `perfil: "user"`.
* Exibição clara de mensagens de erro amigáveis (credenciais inválidas, e-mail já em uso, pop-up fechado).

### B. Proteção de Rotas e Renderização Condicional
* Utilize o observador `onAuthStateChanged(auth, async (user) => { ... })` na página principal (`index.html`):
  * **Se `user` for nulo:** Redirecione imediatamente para `login.html`.
  * **Se `user` estiver autenticado:** Busque o documento correspondente na coleção `usuarios` para obter o `perfil`, exiba o **nome/e-mail e a tag do perfil atual** no cabeçalho e habilite o botão **Sair (Logout)** com `signOut(auth)`.
* **Controle de Interface (UI):**
  * Exiba o painel/aba de **Gestão de Usuários** apenas se `perfil === "admin"`.
  * Exiba os controles de **Atendimento / Alteração de Status** apenas se `perfil === "tecnico"` ou `perfil === "admin"`.
  * Exiba o botão de **Excluir Chamado** apenas se `perfil === "admin"`.

### C. Integração com o Banco de Dados de Chamados
* Ao registrar um novo chamado, grave obrigatoriamente os dados do solicitante e do atendimento:
  * `autorUid`: `user.uid`
  * `autorNome`: `user.displayName || nomeCadastrado`
  * `autorEmail`: `user.email`
  * `status`: `"Aberto"`
  * `tecnicoResponsavel`: `null` *(preenchido quando um `tecnico` ou `admin` assumir/atualizar o chamado)*
* Filtre a consulta/exibição da listagem: se o usuário logado tiver `perfil === "user"`, liste apenas os chamados onde `autorUid === user.uid`.
* *(Opcional / Bônus)* Configure as **Security Rules** do Firebase para validar `request.auth != null` e restringir alterações de perfil apenas a administradores.

---

## 5. Deploy no Render

1. Envie suas alterações para o seu repositório no GitHub:
   ```bash
   git add .
   git commit -m "feat: implementa login firebase, perfis RBAC (user, tecnico, admin) e integracao com banco"
   git push origin main
   ```
2. Acesse o [Dashboard do Render](https://dashboard.render.com/), clique em **New + > Static Site** (ou **Web Service**, caso utilize back-end Node.js) e conecte o seu repositório `chamados-ti`.
3. Configure os campos de deploy:
   * **Build Command:** *(em branco para HTML/CSS/JS puro, ou `npm install` se houver dependências Node)*
   * **Publish Directory:** `.` *(ou a pasta pública onde está o `index.html`)*
4. Após finalizar o deploy, copie a URL gerada (`https://...onrender.com`).
5. **Volte ao Console do Firebase** (`Authentication > Settings > Authorized domains`) e adicione a URL do Render para permitir o login com o Google em produção.

---

## 6. Critérios de Entrega e Avaliação

1. Atualize o arquivo `README.md` na raiz do seu repositório no GitHub contendo, logo no topo:
   * **Nome completo do(a) aluno(a)**
   * **Link da aplicação no Render:** `https://...onrender.com`
   * **Credenciais de teste** (ex: e-mail de um usuário comum, de um técnico e do admin criados para validação).
2. Teste todos os fluxos na URL do Render (cadastro padrão como `user`, abertura de chamado restrita, alteração de status como `tecnico` e promoção de perfil no painel `admin`).
3. Entregue exclusivamente o **link do seu repositório no GitHub** na tarefa da plataforma.

| Critério | Pontuação |
| :--- | :---: |
| Autenticação funcional via E-mail/Senha e Google (`Firebase Auth`) | 2,0 pts |
| Criação automática do documento de usuário com perfil padrão (`user`) | 1,5 pts |
| Regras de visualização e criação restritas para `user` (apenas próprios chamados) | 2,0 pts |
| Fluxo de atendimento e alteração de status para `tecnico` / Analista de TI | 1,5 pts |
| Painel de gestão de usuários e exclusão de chamados exclusivo para `admin` | 1,5 pts |
| Deploy funcional no Render (domínio autorizado no Firebase) e `README.md` atualizado | 1,5 pts |
