# Painel de Vendas · JCA Matriz + Omie

Painel de faturamento da **JCA Matriz** alimentado pela API Omie (NF-e de saída, somente leitura).
A filial tem repositório próprio: [FATURAMENTOLOJAJCA](https://github.com/lelisson/FATURAMENTOLOJAJCA).
As **chaves da API não ficam no código**: no PC são variáveis de usuário do Windows; no GitHub, Secrets do Actions.

## Estrutura

```
painel-omie-matriz/
  .env                 ← você cria (copie de .env.example)
  Iniciar Painel.bat   ← painel local em http://localhost:8001
  backend/app/         ← FastAPI + cliente Omie
  frontend/index.html  ← painel (Chart.js)
  scripts/             ← publicação, segredos e testes
  streamlit_app.py     ← app do Streamlit Cloud
```

## Chaves (variáveis de usuário do Windows)

- `OMIE_APP_KEY` / `OMIE_APP_SECRET`: aplicativo Omie da matriz (as mesmas dos scripts de estoque).
- `PAINEL_MATRIZ_DATA_KEY`: chave Fernet que cifra os dados publicados. É diferente da chave da filial.

`python scripts/enviar_secrets_github.py` cadastra no GitHub, sem exibir, `OMIE_MATRIZ_APP_KEY`, `OMIE_MATRIZ_APP_SECRET` e `PAINEL_DATA_KEY`.

## Streamlit (deploy)

- **Publicação:** o GitHub Actions (`.github/workflows/publicar.yml`) roda `scripts/publicar_dados.py` a cada 20 min, mesmo com o PC desligado. Ele consulta a Omie (somente leitura), cifra os dados com `PAINEL_DATA_KEY` e sobrescreve `painel.enc` no branch `dados`. Para publicar na hora: **Actions > Publicar dados do painel > Run workflow**.
- **Botões Faturamento / Financeiro:** o cabeçalho do app troca de painel. O Financeiro (títulos, movimentos e custos da matriz; código em `financeiro/`) é gerado pelo mesmo robô, no máximo a cada hora (`FIN_INTERVALO_MIN=55`), de `FIN_INICIO=2026-01-01` até hoje, e publicado como `financeiro.enc` (cifrado com `PAINEL_DATA_KEY`) ao lado do `painel.enc`. Meses fechados ficam em `data/cache_meses.enc` (cifrado, no cache do Actions).
- **App:** no Streamlit Community Cloud, arquivo principal `streamlit_app.py`; em **Settings > Secrets**, o conteúdo gerado por `python scripts/gerar_secrets.py --nova-senha` (`.streamlit/secrets.toml`, fora do git). A senha nova fica em `%USERPROFILE%\.painel-omie-matriz\senha-streamlit.txt`; só o hash vai para o Streamlit. Após 5 erros a sessão fica bloqueada por 5 min.
- **Painel sempre ativo:** `manter-ativo.yml` visita o app a cada ~2 h. Cadastre a URL do app no secret `STREAMLIT_APP_URL`.
- **Meses fechados** (após o dia 1 às 23:59 do mês seguinte) ficam congelados em `backend/data/cache_congelado_matriz.enc` (cifrado, no cache do Actions) e não são consultados de novo; no **domingo** o robô reconsulta. `PEDIDOS_MAX_SEG` limita o tempo de ConsultarPedido por execução.
- A matriz não tem PDV: não consulta CuponsFiscais/NFC-e e o canal PDV fica oculto.
- Sem dados publicados nem chaves, o app abre com `backend/data/vendas_demo.json` (dados fictícios).

## Painel local

1. `copy .env.example .env` e ajuste `DATA_SOURCE=omie`.
2. Duplo clique em **Iniciar Painel.bat** (http://localhost:8001).
3. Teste de conexão: `python scripts/test_conexao.py`.

## Segurança

- `.env` e `.streamlit/secrets.toml` estão no `.gitignore`.
- O branch `dados` contém só o arquivo cifrado; sem `PAINEL_DATA_KEY` ele é ilegível.
- O frontend nunca recebe app_key/app_secret.
