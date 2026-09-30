# NFC Wi-Fi Connect 📶

Sistema completo, moderno e profissional para gerenciamento de estabelecimentos comerciais e conexão Wi-Fi via **placas NFC de aproximação** e **cartões digitais responsivos**.

---

## 📋 Sumário
1. [Visão Geral & Arquitetura (GitHub Pages + Firebase)](#visão-geral--arquitetura-github-pages--firebase)
2. [Como Criar o Projeto no Firebase](#1-como-criar-o-projeto-no-firebase)
3. [Como Ativar o Google Authentication e Autorizar o Domínio do GitHub](#2-como-ativar-o-google-authentication-e-autorizar-o-domínio-do-github)
4. [Como Configurar o Cloud Firestore](#3-como-configurar-o-cloud-firestore)
5. [Como Criar o Firebase Storage](#4-como-criar-o-firebase-storage)
6. [Como Configurar as Regras de Segurança](#5-como-configurar-as-regras-de-segurança)
7. [Variáveis de Ambiente (VITE_PUBLIC_BASE_URL)](#6-variáveis-de-ambiente-vite_public_base_url)
8. [Como Definir Seu E-mail como Administrador Exclusivo](#7-como-definir-seu-e-mail-como-administrador-exclusivo)
9. [Como Executar Localmente](#8-como-executar-localmente)
10. [Como Fazer Build](#9-como-fazer-build)
11. [Como Publicar no GitHub Pages (Deploy Automático)](#10-como-publicar-no-github-pages-deploy-automático)
12. [Como Cadastrar o Primeiro Cliente e Gerar Link NFC](#11-como-cadastrar-o-primeiro-cliente-e-gerar-link-nfc)
13. [Como Gravar a URL em uma Placa / Tag NFC](#12-como-gravar-a-url-em-uma-placa--tag-nfc)

---

## 💡 Visão Geral & Arquitetura (GitHub Pages + Firebase)

O frontend é hospedado no **GitHub Pages**, enquanto o **Firebase** é utilizado exclusivamente para autenticação administrativa (Google Login), banco de dados (Cloud Firestore) e imagens (Firebase Storage).

```
[ Placa NFC Física no Balcão ]
         │ (Aproximação do smartphone)
         ▼
[ URL GitHub Pages: https://USUARIO.github.io/REPOSITORIO/?wifi=PWyRj6rpV ]
         │ (Carrega diretamente index.html sem erros 404 de SPA)
         ▼
[ Firestore: /publicWifiPages/PWyRj6rpV ]
         │ (Consulta pública individual sem exigência de login)
         ▼
[ Cartão Digital Mobile-First ]
  • Logo em alta resolução
  • Nome da Rede (SSID)
  • Senha com botão "COPIAR SENHA"
  • Contraste e cores automáticas
```

### Arquitetura de Segurança de Dados
Para evitar raspagem e enumeração de dados de todos os clientes por terceiros:
1. **Coleção Privada `/clients/{clientId}`**: Acesso restrito exclusivamente aos administradores autenticados com e-mail autorizado na whitelist. Permite listagem completa, métricas, edição e exclusão.
2. **Coleção de Projeção Pública `/publicWifiPages/{publicId}`**: Utiliza o identificador público aleatório como ID do documento.
   - **Listagem desabilitada (`allow list: if false`)**: Ninguém na internet consegue baixar a lista com todos os clientes ou senhas.
   - **Leitura pontual permitida (`allow get: if isValidId(publicId)`)**: Apenas quem possui a URL exata gravada na placa física consegue visualizar os dados do Wi-Fi daquele estabelecimento específico.

---

## 1. Como Criar o Projeto no Firebase

1. Acesse o [Firebase Console](https://console.firebase.google.com/).
2. Clique em **"Adicionar projeto"** (ou "Criar um projeto").
3. Digite o nome do projeto (ex: `nfc-wifi-connect`).
4. Desative o Google Analytics (ou mantenha ativo se desejar).
5. Clique em **"Criar projeto"** e aguarde alguns segundos.
6. Na página inicial do projeto, clique no ícone **Web (`</>`)** para registrar a aplicação web:
   - Apelido do app: `nfc-wifi-web`
   - Marque a opção **"Configurar também o Firebase Hosting para este app"**
   - Clique em **"Registrar app"**.
   - Guarde as credenciais do `firebaseConfig`.

---

## 2. Como Ativar o Google Authentication

1. No menu lateral do Firebase Console, acesse **Build > Authentication**.
2. Clique em **"Começar"** (Get Started).
3. Na aba **"Sign-in method"** (Método de login), selecione o provedor **Google**.
4. Ative a chave seletora **"Ativar"**.
5. Em **"E-mail de suporte do projeto"**, selecione o seu e-mail Google.
6. Clique em **"Salvar"**.
7. Na aba **"Settings" > "Authorized domains"** (Domínios autorizados), verifique se o seu domínio (ex: `localhost`, `seudominio.com` ou o subdomínio `.web.app`) está listado.

---

## 3. Como Configurar o Cloud Firestore

1. No menu lateral, acesse **Build > Firestore Database**.
2. Clique em **"Criar banco de dados"**.
3. Escolha o local do banco de dados (ex: `southamerica-east1` em São Paulo, ou `us-east1`).
4. Em modo de segurança, selecione **"Iniciar no modo de produção"** e confirme.

---

## 4. Como Criar o Firebase Storage

1. No menu lateral, acesse **Build > Storage**.
2. Clique em **"Começar"** (Get Started).
3. Mantenha as regras padrão e selecione a mesma região do seu Firestore.
4. Clique em **"Concluído"**.

---

## 5. Como Configurar as Regras de Segurança

### Regras do Firestore
No Firebase Console, acesse **Firestore Database > Regras** e publique o conteúdo do arquivo `firestore.rules`:

```javascript
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    // Bloqueio padrão global
    match /{document=**} {
      allow read, write: if false;
    }

    function isValidId(id) {
      return id is string && id.size() >= 3 && id.size() <= 128 && id.matches('^[a-zA-Z0-9_\\-]+$');
    }

    function incoming() {
      return request.resource.data;
    }

    function existing() {
      return resource.data;
    }

    function isSignedIn() {
      return request.auth != null;
    }

    function isVerified() {
      return isSignedIn() && request.auth.token.email_verified == true;
    }

    function isAdmin() {
      return isVerified() && (
        request.auth.token.email in ['caioperatone.beiral@gmail.com'] ||
        exists(/databases/$(database)/documents/admins/$(request.auth.uid))
      );
    }

    function isValidClient(data) {
      return data.keys().hasAll(['publicId', 'businessName', 'ssid', 'active', 'backgroundColor', 'createdAt', 'updatedAt']) &&
             data.keys().hasOnly(['publicId', 'businessName', 'ssid', 'wifiPassword', 'logoUrl', 'backgroundColor', 'active', 'customTitle', 'instructions', 'createdAt', 'updatedAt']) &&
             data.publicId is string && data.publicId.size() >= 6 && data.publicId.size() <= 64 && data.publicId.matches('^[a-zA-Z0-9_\\-]+$') &&
             data.businessName is string && data.businessName.size() >= 1 && data.businessName.size() <= 120 &&
             data.ssid is string && data.ssid.size() >= 1 && data.ssid.size() <= 64 &&
             (!('wifiPassword' in data) || (data.wifiPassword is string && data.wifiPassword.size() <= 128)) &&
             (!('logoUrl' in data) || (data.logoUrl is string && data.logoUrl.size() <= 2048)) &&
             data.backgroundColor is string && data.backgroundColor.size() >= 4 && data.backgroundColor.size() <= 16 && data.backgroundColor.matches('^#[0-9a-fA-F]{3,8}$') &&
             data.active is bool &&
             (!('customTitle' in data) || (data.customTitle is string && data.customTitle.size() <= 120)) &&
             (!('instructions' in data) || (data.instructions is string && data.instructions.size() <= 500)) &&
             data.createdAt is timestamp &&
             data.updatedAt is timestamp;
    }

    function isValidPublicWifiPage(data) {
      return data.keys().hasAll(['publicId', 'businessName', 'ssid', 'active', 'backgroundColor', 'updatedAt']) &&
             data.keys().hasOnly(['publicId', 'businessName', 'ssid', 'wifiPassword', 'logoUrl', 'backgroundColor', 'active', 'customTitle', 'instructions', 'updatedAt']) &&
             data.publicId is string && data.publicId.size() >= 6 && data.publicId.size() <= 64 && data.publicId.matches('^[a-zA-Z0-9_\\-]+$') &&
             data.businessName is string && data.businessName.size() >= 1 && data.businessName.size() <= 120 &&
             data.ssid is string && data.ssid.size() >= 1 && data.ssid.size() <= 64 &&
             (!('wifiPassword' in data) || (data.wifiPassword is string && data.wifiPassword.size() <= 128)) &&
             (!('logoUrl' in data) || (data.logoUrl is string && data.logoUrl.size() <= 2048)) &&
             data.backgroundColor is string && data.backgroundColor.size() >= 4 && data.backgroundColor.size() <= 16 && data.backgroundColor.matches('^#[0-9a-fA-F]{3,8}$') &&
             data.active is bool &&
             (!('customTitle' in data) || (data.customTitle is string && data.customTitle.size() <= 120)) &&
             (!('instructions' in data) || (data.instructions is string && data.instructions.size() <= 500)) &&
             data.updatedAt is timestamp;
    }

    // Coleção privada de clientes administrativos
    match /clients/{clientId} {
      allow get, list: if isAdmin();
      allow create: if isAdmin() && isValidId(clientId) && isValidClient(incoming()) &&
                       incoming().createdAt == request.time && incoming().updatedAt == request.time;
      allow update: if isAdmin() && isValidId(clientId) && isValidClient(incoming()) &&
                       incoming().updatedAt == request.time &&
                       incoming().createdAt == existing().createdAt &&
                       incoming().publicId == existing().publicId;
      allow delete: if isAdmin() && isValidId(clientId);
    }

    // Coleção pública de páginas Wi-Fi - consulta individual autorizada por ID público
    match /publicWifiPages/{publicId} {
      // Consulta individual pelo ID da tag
      allow get: if isValidId(publicId);
      // Listagem ou download de toda a base proibido
      allow list: if false;

      allow create: if isAdmin() && isValidId(publicId) && isValidPublicWifiPage(incoming()) && incoming().publicId == publicId;
      allow update: if isAdmin() && isValidId(publicId) && isValidPublicWifiPage(incoming()) && incoming().publicId == publicId && incoming().publicId == existing().publicId;
      allow delete: if isAdmin() && isValidId(publicId);
    }

    // Tabela de UIDs de administradores
    match /admins/{uid} {
      allow get: if isSignedIn() && (request.auth.uid == uid || isAdmin());
      allow list: if isAdmin();
      allow create, update: if isAdmin() && isValidId(uid);
      allow delete: if isAdmin() && isValidId(uid) && request.auth.uid != uid;
    }
  }
}
```

### Regras do Firebase Storage
No Firebase Console, acesse **Storage > Regras** e publique:

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /logos/{allPaths=**} {
      // Qualquer visitante pode visualizar a logo do estabelecimento
      allow read: if true;
      // Apenas administradores autenticados podem enviar ou remover logos
      allow write, delete: if request.auth != null;
    }
  }
}
```

---

## 6. Onde Colocar as Variáveis de Ambiente & Configurações

As credenciais públicas do seu projeto Firebase são mantidas no arquivo `firebase-applet-config.json` na raiz do projeto:

```json
{
  "projectId": "seu-projeto-firebase",
  "appId": "1:123456789:web:abcdef123456",
  "apiKey": "AIzaSy...",
  "authDomain": "seu-projeto-firebase.firebaseapp.com",
  "firestoreDatabaseId": "(default)",
  "storageBucket": "seu-projeto-firebase.firebasestorage.app",
  "messagingSenderId": "123456789",
  "measurementId": "",
  "oAuthClientId": ""
}
```

> **Nota de Segurança:** As chaves de configuração web do Firebase (`apiKey`, `projectId`, etc.) são públicas por design no Firebase. A segurança real do sistema é garantida pelas **Regras de Segurança do Firestore** e pela **Whitelist de E-mails**.

---

## 7. Como Definir Seu E-mail como Administrador (Whitelist)

No arquivo `src/firebase/config.ts`:

```typescript
export const DEFAULT_ADMIN_EMAILS: string[] = [
  'caioperatone.beiral@gmail.com'
  // Adicione outros e-mails se desejar: 'outro-socio@gmail.com'
];
```

E sincronize o e-mail no arquivo `firestore.rules`:
```javascript
request.auth.token.email in ['caioperatone.beiral@gmail.com']
```

Caso uma pessoa não autorizada tente entrar com uma conta Google qualquer, ela recebe uma notificação clara de **"Acesso Negado: seu e-mail não possui permissão administrativa"** e é imediatamente desconectada.

---

## 8. Como Executar Localmente

Certifique-se de ter o [Node.js](https://nodejs.org/) instalado (versão 18 ou superior).

```bash
# Instalar dependências
npm install

# Iniciar servidor de desenvolvimento
npm run dev
```

Abra seu navegador em: `http://localhost:3000`

---

## 9. Como Fazer Build

Para gerar os arquivos otimizados para produção:

```bash
npm run build
```

Os arquivos prontos para publicação serão gerados no diretório `dist/`.

---

## 10. Como Publicar no Firebase Hosting

1. Instale o Firebase CLI globalmente:
```bash
npm install -g firebase-tools
```

2. Faça login na sua conta Google:
```bash
firebase login
```

3. Inicialize o hosting (se ainda não fez):
```bash
firebase init hosting
```
- Selecione o seu projeto criado.
- Pasta pública: digite `dist`.
- Configurar como SPA (Single-page app): digite `Yes` (Y).
- Sobrescrever index.html: digite `No` (N).

4. Faça o build e publique com um comando:
```bash
npm run build && firebase deploy --only hosting
```

Seu site estará no ar na URL fornecida (ex: `https://seu-projeto.web.app`).

---

## 11. Como Configurar Domínio Próprio

1. No Firebase Console, acesse **Hosting > Adicionar domínio personalizado**.
2. Digite o seu domínio (ex: `nfcwifi.com.br` ou `conectar.meusite.com`).
3. O Firebase fornecerá os registros DNS (tipo `A` e `TXT`).
4. Acesse o painel onde registrou seu domínio (Registro.br, Cloudflare, GoDaddy, Hostinger, etc.) e adicione as entradas DNS correspondentes.
5. Em poucas horas o certificado SSL gratuito será emitido automaticamente pelo Firebase.

---

## 12. Como Cadastrar o Primeiro Cliente

1. Acesse `/admin` e faça login com seu e-mail Google autorizado.
2. No dashboard, clique no botão azul **"Novo Cliente"**.
3. Preencha os campos:
   - **Nome do Estabelecimento:** Ex: *Bella Cucina Restaurante*
   - **Nome da Rede (SSID):** Ex: *Bella_Clientes_5G*
   - **Senha do Wi-Fi:** Ex: *pasta2026* (ou marque "Sem Senha" se a rede for aberta)
   - **Logo do Estabelecimento:** Envie a imagem da marca (PNG, JPG, WEBP ou SVG)
   - **Cor de Fundo da Página:** Escolha uma cor moderna na paleta ou digite o hexadecimal (ex: `#0f172a`, `#000000`, `#064e3b`)
   - **Título / Instrução (Opcionais):** Ex: *Conecte-se ao Wi-Fi*
4. Clique em **"Criar e Gerar Link NFC"**.
5. O sistema criará o cliente e gerará um ID público exclusivo (ex: `a8K92mP4x`).

---

## 13. Como Obter o Link da Página

No card do cliente recém-criado:
- Clique no botão **"Copiar"** ao lado da URL pública.
- O link completo será copiado para a área de transferência:
  `https://seudominio.com/wifi/a8K92mP4x`
- Você também pode clicar em **"Visualizar"** para testar como o cliente final verá a página, ou no botão **"QR Code"** para baixar o código QR.

---

## 14. Como Gravar a URL em uma Placa / Tag NFC

Você precisa gravar a placa NFC apenas **uma vez**:

1. **Baixe o aplicativo gravador no celular:**
   - Para Android ou iPhone: Baixe o **NFC Tools** (gratuito) na Google Play Store ou App Store.
2. **Abra o app NFC Tools:**
   - Toque na aba **"Escrever"** (Write).
   - Toque em **"Adicionar um registro"** (Add a record).
   - Selecione a opção **"URL / URI"**.
   - Cole o link copiado no painel (ex: `https://seudominio.com/wifi/a8K92mP4x`).
   - Toque em **"OK"**.
3. **Grave na placa física:**
   - Toque no botão azul **"Escrever / Write"**.
   - Aproxime a placa NFC na parte traseira do seu celular.
   - O aplicativo emitirá um som/vibração confirmando o sucesso da gravação.
4. **Pronto!**
   - Agora, qualquer pessoa que aproximar o celular da placa abrirá o cartão do Wi-Fi instantaneamente.
   - Quando o dono do restaurante mudar a senha do Wi-Fi no futuro, você só precisa atualizar no painel `/admin`. A placa NFC continuará funcionando perfeitamente sem precisar ser regravada!
