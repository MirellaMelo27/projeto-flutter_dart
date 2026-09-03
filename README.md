# 📝 App de Correção Automática de Provas

App Android para gerar provas com questões embaralhadas, imprimir com QR code/gabarito, corrigir automaticamente via leitura da folha de respostas, e gerar estatísticas/relatórios de notas.

> **Contexto:** aplicação desenvolvida para substituir sistemas como GradePen e Prova Fácil, unindo o que cada um faz bem (GradePen gera prova + embaralha; Prova Fácil gera estatística de alternativa mais marcada) e cobrindo o que os dois deixam a desejar (informar qual alternativa o aluno marcou, permitir ajustar o layout da prova, individualizar a folha de resposta por aluno).

---

## 👥 Equipe e Divisão de Responsabilidades

| Pessoa | Responsabilidade |
|--------|-------------------|
| **Pessoa 1(Karol)** | Turmas & Alunos — cadastro do professor, turmas, importação de lista de alunos |
| **Pessoa 2** | Banco de Questões & Criação de Provas — cadastro de questões, montagem de provas, embaralhamento |
| **Pessoa 3** | Geração da Prova & Gabarito (QR Code) — layout de impressão, folha de resposta, individualização por aluno |
| **Pessoa 4** | Correção Automática — leitura de QR code, leitura da folha de respostas, cálculo de nota |
| **Pessoa 5(Mirella)** | Resultados & Estatísticas — notas por turma, exportação Excel, estatística de alternativa mais marcada |

---

## 🗓️ Ordem de Execução

A ordem abaixo respeita as dependências reais entre as partes: não dá pra corrigir uma prova (Pessoa 4) que ainda não foi gerada com QR code (Pessoa 3), e não dá pra gerar prova (Pessoa 3) sem que existam questões e provas montadas (Pessoa 2) nem alunos cadastrados (Pessoa 1) pra individualizar a folha de resposta.

```
Etapa 1 — Pessoa 1 + Pessoa 2 (em paralelo)
  ├── Pessoa 1: cadastro de professor, turmas e alunos (+ importar lista)
  └── Pessoa 2: banco de questões + montagem de provas + lógica de embaralhamento
        ↓
Etapa 2 — Pessoa 3
  └── Gera a prova pra impressão (layout) e a folha de resposta com QR code,
      vinculando cada folha a uma VariacaoDeProva (Pessoa 2) e a um Aluno (Pessoa 1)
        ↓
Etapa 3 — Pessoa 4
  └── Lê o QR code gerado pela Pessoa 3, lê as respostas marcadas na folha,
      compara com o gabarito da variação e calcula a nota automaticamente
        ↓
Etapa 4 — Pessoa 5
  └── Consome as correções salvas pela Pessoa 4 pra montar a tela de notas,
      a exportação em Excel e a estatística de alternativa mais marcada por questão
```

**Na prática:** Pessoas 1 e 2 podem começar juntas desde o dia 1. Pessoa 3 só consegue avançar de verdade depois que Pessoa 2 tiver pelo menos uma prova/variação de teste pronta (e o ideal é já ter alunos da Pessoa 1 pra testar a individualização). Pessoa 4 depende de ter uma folha real gerada pela Pessoa 3 pra testar a leitura. Pessoa 5 é a última porque só tem o que mostrar depois que existir correção de verdade.

Isso não significa que as Pessoas 3, 4 e 5 ficam paradas esperando — elas podem adiantar telas com dados mock (é literalmente o que a N1 pede) e só plugar na parte anterior quando ela estiver pronta de verdade (N2).

---

## 🗃️ Modelo de Dados (entidades principais)

| Entidade | Descrição | Depende de |
|----------|-----------|------------|
| `Professor` | Usuário que usa o app | — |
| `Turma` | Turma de um professor | `Professor` |
| `Aluno` | Aluno de uma turma | `Turma` |
| `Questao` | Enunciado + alternativas + gabarito | — |
| `Prova` | Conjunto de questões selecionadas | `Questao` |
| `VariacaoDeProva` | Uma versão embaralhada da prova, vinculada (opcionalmente) a um aluno | `Prova`, `Aluno` |
| `Correcao` | Respostas lidas + nota calculada | `VariacaoDeProva`, `Aluno` |

---

## 🔍 Detalhamento por Pessoa

### Pessoa 1 — Turmas & Alunos
**Depende de:** nada, pode começar imediatamente.

- Tela de login/cadastro do professor
- Tela "Minhas Turmas" (criar/editar/excluir)
- Tela de alunos dentro da turma (cadastro manual)
- Importação de lista de alunos (CSV/planilha)

| Fase | Entrega |
|------|---------|
| N1 | Telas navegáveis com turma/aluno mockados |
| N2 | CRUD real no Firestore |
| N3 | Fluxo completo testado: turma criada → alunos cadastrados/importados → disponíveis pros outros módulos |

### Pessoa 2 — Banco de Questões & Criação de Provas
**Depende de:** nada, pode começar imediatamente (em paralelo com Pessoa 1).

- Tela "Banco de Questões" (enunciado + alternativas + qual é a correta)
- Tela "Nova Prova" (selecionar questões e montar a prova)
- Lógica de embaralhamento: gerar N variações, embaralhando ordem de questões e alternativas

| Fase | Entrega |
|------|---------|
| N1 | Criação de prova com questões mock; embaralhamento pode ser simulado |
| N2 | Questões/provas salvas no Firestore; variações reais e persistidas |
| N3 | Garantir que cada variação tem o gabarito certo vinculado (a Pessoa 4 depende disso) |

### Pessoa 3 — Geração da Prova & Gabarito (QR Code)
**Depende de:** Pessoa 2 (precisa de uma `VariacaoDeProva` pra gerar) e Pessoa 1 (precisa de `Aluno` pra individualizar a folha).

- Montagem do layout final pra impressão (com opção de ajustar quebra de página)
- Geração da folha de resposta com QR code, identificando unicamente qual variação/aluno é aquela prova
- Exportação em .doc/PDF

| Fase | Entrega |
|------|---------|
| N1 | Prova mock com QR code fake, só pra mostrar o fluxo |
| N2 | QR code real gerado a partir dos IDs salvos no Firestore |
| N3 | Testar que o QR gerado aqui é lido corretamente pela Pessoa 4 |

### Pessoa 4 — Correção Automática (peça mais crítica do produto)
**Depende de:** Pessoa 3 (precisa de uma folha real com QR code pra testar a leitura).

- Tela "Corrigir Prova": abre câmera, lê o QR code (identifica variação/aluno)
- Leitura da folha de respostas (reconhecimento das marcações)
- Comparação automática com o gabarito → cálculo da nota

| Fase | Entrega |
|------|---------|
| N1 | Fluxo simulado: escaneia QR mock → mostra nota mockada |
| N2 | Leitura real do QR + gravação da correção no Firestore; leitura da folha pode começar simplificada se o reconhecimento de imagem for complexo demais pro prazo — alinhar com o grupo |
| N3 | Fluxo ponta a ponta: ler QR → ler respostas → nota salva e visível pra Pessoa 5 |

### Pessoa 5 — Resultados & Estatísticas
**Depende de:** Pessoa 4 (precisa de `Correcao` real pra ter o que mostrar).

- Tela "Notas da Turma" (lista de alunos com nota, por prova)
- Exportação para Excel
- Estatística por questão: qual alternativa foi mais marcada (diferencial competitivo do produto)

| Fase | Entrega |
|------|---------|
| N1 | Tela de notas e estatística com dados mock |
| N2 | Puxando `Correcao` reais do Firestore |
| N3 | Exportação Excel funcionando + estatística validada com dados reais |

---

## ✅ Checklist de Entrega

### Pessoa 1
- [ ] Cadastro/login do professor
- [ ] CRUD de turmas
- [ ] Cadastro manual de alunos
- [ ] Importação de lista de alunos
- [ ] Integração com Firestore (N2)

### Pessoa 2
- [ ] Cadastro de questões (com gabarito)
- [ ] Tela de montagem de prova
- [ ] Embaralhamento de questões e alternativas gerando variações
- [ ] Integração com Firestore (N2)

### Pessoa 3
- [ ] Layout de impressão da prova
- [ ] Geração da folha de resposta com QR code
- [ ] Vinculação prova ↔ aluno (individualização)
- [ ] Exportação .doc/PDF

### Pessoa 4
- [ ] Leitura de QR code
- [ ] Leitura da folha de respostas
- [ ] Cálculo automático da nota
- [ ] Gravação da correção no Firestore (N2)

### Pessoa 5
- [ ] Tela de notas por turma/aluno
- [ ] Exportação para Excel
- [ ] Estatística de alternativa mais marcada por questão
- [ ] 
