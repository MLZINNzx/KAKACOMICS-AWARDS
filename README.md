# KakaComics Awards

Site de votação estático + API serverless + Supabase. Funciona na **Vercel** (atual) e mantém compatibilidade com a antiga hospedagem Netlify como backup.

## Estrutura

```
api/index.js            → Function da Vercel ( espelho de netlify/functions/api.js )
public/                 → site estático (index.html, admin.html, style.css, assets/)
supabase/schema.sql     → schema do banco (não precisa rodar de novo)
vercel.json             → rotas: /api/* → function, /admin → admin.html
netlify/ + netlify.toml → deploy antigo (manter até validar a Vercel)
```

## Banco de dados (NÃO recriar)

Os votos, categorias e concorrentes já estão no **mesmo projeto Supabase**. A migração não altera nada no banco:
não rode `schema.sql` de novo e não crie outro projeto Supabase.

## Deploy na Vercel

1. Faça upload desta pasta para um repositório Git e importe o projeto na Vercel.
   - Framework: **Other / Static** (ou deixe a Vercel detectar).
   - Output Directory: `public` (já definido em `vercel.json`).
   - Build Command: vazio.
2. Em **Settings → Environment Variables**, adicione **exatamente os mesmos valores** usados na Netlify:

   | Variável | Onde encontrar |
   |---|---|
   | `SUPABASE_URL` | Supabase → Project Settings → API → Project URL |
   | `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Project Settings → API → `service_role` (**nunca** exponha no front) |
   | `ADMIN_EMAIL` | e-mail de admin atual |
   | `ADMIN_PASSWORD` | senha de admin atual |
   | `SESSION_SECRET` | **copie o valor antigo da Netlify** (ver aviso abaixo) |
   | `STRICT_IP` | `true` ou `false` (igual ao atual) |

3. Deploy. A Vercel cria automaticamente a function `api/index.js`.
4. Em **Settings → Domains**, adicione o **mesmo domínio customizado** usado na Netlify e aponte o DNS (CNAME/A) para a Vercel.
   Mantenha a Netlify no ar até conferir tudo; depois você pode desligá-la.

> ⚠️ **Não troque o `SESSION_SECRET`.** Ele gera o `voter_hash` e o `ip_hash` dos votos e assina o cookie de admin. Trocar o valor permite que eleitores antigos votem de novo (o cookie `kv` deixaria de corresponder ao hash gravado) e desloga o admin. Use o valor idêntico ao da Netlify.

> ⚠️ **Use o mesmo domínio.** O cookie `kv` que impede voto duplo é gravado por domínio. Se o site mudar de domínio, quem já votou não leva o cookie e poderá votar novamente. Por isso configure o domínio final na Vercel antes de liberar a votação.

> ℹ️ Se `STRICT_IP=true`: a Netlify usava `x-nf-client-connection-ip` e a Vercel usa `x-real-ip`. A function aceita ambos. Os hashes de IP de votos antigos não são recalculados, mas o bloqueio principal continua sendo `voter_hash` (cookie) + constraint `unique(category_id, voter_hash)` no banco.

## Validação após o deploy

- `GET /api/public` → JSON com `settings` e `categories` (sem erro 500).
- `GET /admin` → tela de login; entre e confirme que categorias, concorrentes e ranking aparecem com os votos antigos.
- Faça um voto de teste numa categoria e confirme que o contador atualiza e que um segundo voto é recusado.
- Verifique em **Runtime Logs** da Vercel que não há erros.

Se algo falhar, volta o DNS para a Netlify: os arquivos e a function antiga continuam no projeto.

## Rodando local

Crie um arquivo `.env` com as variáveis acima e use `npx netlify dev` (compatibilidade antiga) ou `npx vercel dev` (equivalente Vercel, requer Vercel CLI e login).
Coloque a logo em `public/assets/logo.png`.

## Limpeza futura

Depois que a Vercel estiver estável por alguns dias, você pode remover `netlify/`, `netlify.toml` e a entrada do deploy antigo. O banco e os votos permanecem intactos.
