# ReciboFácil

Sistema desktop para Windows de **emissão, gerenciamento, consulta e impressão de recibos de prestação de serviços**.

| | |
|---|---|
| Linguagem | Python 3.13 |
| Interface | CustomTkinter (menu lateral + dashboard) |
| Banco de dados | MariaDB 10.4.13 · InnoDB · utf8mb4 (sem SQLite) |
| Acesso ao banco | PyMySQL (consultas parametrizadas e transações) |
| PDF | ReportLab (A4) |

## Funcionalidades

- **Dashboard:** total de recibos, valor emitido, recibos e valor do mês, recebido, pendente, últimos recibos e evolução dos últimos 6 meses.
- **Clientes:** cadastro com CPF/CNPJ validado (inclui o CNPJ alfanumérico), telefone, e-mail e endereço; pesquisa, edição, consulta e desativação/reativação.
- **Emissão de recibos:** numeração automática (`2026/000012`), data, cliente, vários serviços (quantidade × valor unitário), desconto opcional, observações e, se quiser, o pagamento já na emissão.
- **Histórico:** pesquisa por número, cliente (nome ou CPF/CNPJ) e período, com paginação; detalhes completos do recibo; registro de pagamentos parciais; cancelamento com motivo.
- **PDF profissional em A4:** cabeçalho do emitente, valor por extenso, tabela de serviços, totais, pagamentos, observações, assinatura, rodapé e marca d'água em recibos cancelados.
- **Impressão** pelo visualizador de PDF padrão do Windows.
- **Configurações:** dados do emitente, rodapé e pasta dos PDFs, formas de pagamento.

## Instalação (Windows)

**Pré-requisitos:** Python 3.13 (com *Add python.exe to PATH*), MariaDB 10.4.13 instalado com o serviço iniciado e a senha do usuário `root`. HeidiSQL é opcional.

### Jeito rápido

1. Dê dois cliques em **`instalar.bat`**. Ele cria o ambiente virtual, instala as dependências, abre o `.env` para você definir `DB_PASSWORD` (e conferir `DB_PORT`) e prepara o banco (pede a senha do `root`).
2. Para usar o sistema, dê dois cliques em **`iniciar.bat`**.

### Passo a passo manual (PowerShell, dentro da pasta do projeto)

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
notepad .env                      # defina DB_PASSWORD e confira DB_PORT
python database/setup_db.py       # cria banco, tabelas, dados iniciais e o usuário do sistema
python main.py                    # abre o sistema
```

> Se o PowerShell bloquear a ativação do ambiente virtual: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

### O que o `setup_db.py` faz

Conecta com o administrador do MariaDB (pede a senha no terminal se `DB_ADMIN_PASSWORD` estiver vazio), executa `schema.sql` e `seed.sql`, cria o usuário `recibofacil_user` com acesso **somente** ao banco `recibofacil` e verifica a estrutura (tabelas, engine, collation, chaves estrangeiras e restrições). Pode ser executado quantas vezes quiser sem apagar dados.

Opções: `--sem-seed` (não carrega os dados iniciais) e `--verificar` (apenas confere a estrutura).

Alternativa pelo HeidiSQL: execute `database/schema.sql` e depois `database/seed.sql`.

### Primeiro uso

1. Abra **Configurações → Emitente** e troque o emitente de exemplo pelos seus dados (ele aparece no cabeçalho e na assinatura dos recibos).
2. Em **Clientes**, cadastre os clientes (o seed traz dois clientes fictícios para teste).
3. Em **Novo recibo**, emita e abra o PDF.

## Testes

```powershell
pytest -v
```

São mais de 120 testes: validadores (CPF/CNPJ, valor por extenso, máscaras), estrutura do banco, repositórios, regras de negócio, geração de PDF e interface. Os testes usam emitente e cliente próprios e removem tudo ao final; **não consomem a numeração do seu emitente real**. Os testes de interface são ignorados automaticamente se não houver ambiente gráfico.

## Arquitetura

```
ReciboFacil/
├── main.py                  ponto de entrada
├── instalar.bat / iniciar.bat
├── requirements.txt / .env.example
├── database/
│   ├── schema.sql           banco, tabelas, FKs, índices, CHECKs, view (comentado)
│   ├── seed.sql             formas de pagamento, configurações, emitente e clientes de exemplo
│   ├── consultas.sql        consultas prontas para o HeidiSQL
│   └── setup_db.py          prepara e verifica o banco
├── docs/modelo_de_dados.md  diagrama de relacionamentos e decisões de modelagem
├── app/
│   ├── config/              leitura do .env
│   ├── core/                conexão e transações, exceções, log
│   ├── models/              dataclasses (Cliente, Emitente, Recibo, ItemRecibo, Pagamento...)
│   ├── repositories/        SQL parametrizado, um por tabela/agregado
│   ├── validators/          CPF/CNPJ, cadastros, recibos e pagamentos
│   ├── services/            regras de negócio e transações
│   ├── documents/           PDF (ReportLab) e abertura/impressão
│   ├── ui/                  janela principal, telas, diálogos e componentes
│   └── utils/               dinheiro (Decimal), formatação, valor por extenso
├── output/pdfs/             PDFs gerados
├── logs/                    log da aplicação (criado em tempo de execução)
└── tests/                   testes automatizados
```

Fluxo de dependência: **interface → serviços → repositórios → conexão**. A interface nunca escreve SQL; os repositórios nunca contêm regra de negócio; quem define os limites de cada transação (COMMIT/ROLLBACK) é o serviço.

### Decisões importantes

- **Histórico preservado:** o recibo guarda uma cópia dos dados do cliente e do emitente na emissão. Editar ou desativar um cliente não altera recibos já emitidos nem o PDF reimpresso.
- **Numeração segura:** o número é reservado com `SELECT ... FOR UPDATE` na mesma transação que grava o recibo, com índice único `(emitente, ano, número)` como segunda barreira. Emissões simultâneas não repetem número e uma falha não deixa buracos na sequência. Recibos cancelados mantêm o número.
- **Emissão atômica:** número, cabeçalho, itens, pagamento e histórico são gravados em uma única transação.
- **Dinheiro:** `DECIMAL` no banco e `Decimal` no Python, com arredondamento em centavos (meio para cima). Nunca `float`.
- **Exclusão lógica:** clientes são desativados, recibos são cancelados (com motivo). Chaves estrangeiras `RESTRICT` impedem apagar o que tem histórico.
- **Segurança:** consultas parametrizadas (inclusive pesquisas com `%`, `_` e aspas), usuário do banco com privilégios mínimos, mensagens de erro claras sem expor detalhes técnicos (que vão para o log).
- **Validações:** CPF e CNPJ (numérico e alfanumérico), campos obrigatórios e tamanhos, e-mail, telefone, CEP, UF, valores negativos ou zerados, desconto maior que o subtotal, datas futuras e pagamento acima do pendente. Os erros são acumulados e mostrados juntos.

O modelo de dados completo está em [`docs/modelo_de_dados.md`](docs/modelo_de_dados.md).

## Roteiro sugerido para a apresentação

1. **Visão geral:** abra o **Dashboard** e explique os indicadores.
2. **Clientes:** cadastre um cliente (mostre a validação digitando um CPF inválido e a máscara automática), pesquise e desative.
3. **Novo recibo:** selecione o cliente, adicione dois serviços, aplique um desconto, registre o pagamento e emita. Abra o PDF.
4. **Histórico:** pesquise por número, cliente e período; abra os detalhes; registre um pagamento parcial; cancele um recibo e mostre o PDF com a marca d'água.
5. **Integridade:** edite o cliente e mostre que o recibo antigo continua com os dados da emissão.
6. **Banco no HeidiSQL:** mostre as tabelas, o diagrama (`docs/modelo_de_dados.md`) e execute algumas consultas de `database/consultas.sql`.
7. **Qualidade:** rode `pytest -v`.

## Solução de problemas

| Mensagem ou sintoma | O que fazer |
|---|---|
| *Não foi possível conectar ao MariaDB* | Confirme que o serviço do MariaDB está iniciado e que `DB_HOST` e `DB_PORT` do `.env` estão corretos (veja a porta no HeidiSQL). |
| *Lost connection to MySQL server during query* | Pode haver dois servidores de banco disputando a porta 3306 (por exemplo XAMPP e MariaDB). Pare um deles ou mude a porta e ajuste `DB_PORT`. |
| *Access denied* no `setup_db.py` | Usuário ou senha do administrador incorretos. Deixe `DB_ADMIN_PASSWORD` vazio no `.env` para digitar a senha no terminal. |
| *Acesso negado ao banco de dados* ao abrir o sistema | `DB_USER`/`DB_PASSWORD` do `.env` diferentes dos usados no `setup_db.py`. Rode o `setup_db.py` novamente. |
| *O banco de dados não existe* | Rode `python database/setup_db.py`. |
| O PDF não abre | Verifique se há um programa padrão para arquivos PDF no Windows. O arquivo fica na pasta configurada (padrão `output/pdfs`). |
| O botão Imprimir só abre o PDF | Alguns visualizadores não aceitam impressão direta. Imprima com **Ctrl+P** no próprio visualizador. |
| Erro inesperado | Os detalhes ficam em `logs/recibofacil.log`. |

## Observações

- O CNPJ alfanumérico (em vigor desde julho de 2026) é aceito e validado.
- Recibos emitidos são imutáveis (apenas pagamentos e cancelamento). Para corrigir um recibo, cancele-o e emita outro.
- Os dados de exemplo (emitente e dois clientes) usam documentos fictícios com dígitos verificadores válidos.
