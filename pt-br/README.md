# OpenCode: Orquestrador cloud + Ollama local

Este repositório reúne uma configuração enxuta para executar o OpenCode em um fluxo híbrido:

- **Orquestrador Cloud**: analisa o pedido, divide o trabalho em microtarefas e audita os resultados.
- **Worker local com Ollama**: executa uma tarefa pequena por vez, valida a alteração e devolve um relatório padronizado.
- **Contingência local**: grava o último resultado em `.opencode/state/last-result.md`, permitindo continuar ou diagnosticar mesmo quando o retorno do subagente vier vazio.

A GPU é usada de forma sequencial: o worker nunca recebe duas microtarefas em paralelo.

## Como o fluxo funciona

1. `/orquestrar <pedido>` envia o pedido ao agente principal `orchestrator`.
2. O orquestrador faz o mínimo de leitura necessário e cria um plano em tarefas, subtarefas e microtarefas.
3. Cada microtarefa é delegada uma por vez ao `local-worker`.
4. O worker altera somente o escopo autorizado, executa uma validação e grava o contrato final em `last-result.md`.
5. O orquestrador verifica o resultado:
   - `DONE`: avança para a próxima microtarefa.
   - `FAILED` ou `BLOCKED`: envia uma correção cirúrgica.
   - Após três falhas seguidas na mesma microtarefa, interrompe o ciclo e pede orientação.

Formato interno do worker:

```text
TAREFA
STATUS: DONE | FAILED | BLOCKED
ARQUIVOS_ALTERADOS
VERIFICACAO
PROBLEMA_ENCONTRADO
PROXIMO_PASSO_SUGERIDO
```

## Arquivos deste acervo

```text
.
├── opencode.json                     # Configuração ativa: provedores, agentes e permissões
├── README.md                         # Guia de instalação e calibragem de hardware
├── agents/
│   ├── orchestrator.md               # Agente primário Cloud
│   └── local-worker.md               # Executor Ollama local
├── commands/
│   └── orquestrar.md                 # Comando que inicia o fluxo híbrido
```

O diretório global utilizado pelo OpenCode é normalmente `~/.config/opencode/`. Em Windows, corresponde a `C:\Users\<usuario>\.config\opencode\`.

Também é possível instalar a mesma estrutura em um projeto específico:

```text
seu-projeto/
├── opencode.json
└── .opencode/
    ├── agents/
    │   ├── orchestrator.md
    │   └── local-worker.md
    ├── commands/
    │   └── orquestrar.md
    └── state/
        └── last-result.md
```

## Configuração atual

- O agente `orchestrator` é primário e pode delegar exclusivamente ao `local-worker`.
- O `local-worker` está configurado para `ollama/gemma4:e4b#fast`.
- As chamadas locais têm limite de 15 passos por microtarefa.
- As capacidades do Ollama configuradas nesta instalação incluem contexto de até 64k tokens.
- A interface está configurada para solicitar permissões interativamente.

Antes de usar em outra máquina, ajuste o provedor/modelo Cloud selecionado pelo OpenCode e confirme os modelos disponíveis no Ollama.

## Instalação

### 1. Instale e execute o Ollama

```bash
ollama pull gemma4:e4b
ollama serve
```

Confirme o endpoint padrão:

```text
http://localhost:11434
```

### 2. Copie a configuração

Para configuração global no Windows:

```powershell
Copy-Item .\opencode.json "$env:USERPROFILE\.config\opencode\opencode.json"
Copy-Item .\cli.json "$env:USERPROFILE\.config\opencode\cli.json"
Copy-Item .\agents\* "$env:USERPROFILE\.config\opencode\agents\" -Recurse -Force
Copy-Item .\commands\* "$env:USERPROFILE\.config\opencode\commands\" -Recurse -Force
```

Para instalação por projeto, copie `opencode.json` para a raiz do projeto e os diretórios `agents` e `commands` para `.opencode/`.

### 3. Ajuste o modelo Cloud

Escolha no OpenCode o modelo Cloud que deseja usar como agente primário. A configuração também pode apontar um modelo padronizado diretamente no perfil do `orchestrator`, conforme a versão do OpenCode instalada.

## Uso

Na raiz do projeto:

```bash
opencode
```

---
> **Selecione o modelo Cloud** desejado (podendo ser um modelo Free), por exemplo o `DeepSeek V4.1 Flash Free`. Digite `/models` para ver os modelos disponíveis, pode consultar os modelos Free, digitando `Free` na consulta.
---

Selecione o agente `orchestrator` ou execute diretamente:

```text
/orquestrar Implemente a validação de CPF no formulário de cadastro e adicione testes unitários.
```

O OpenCode apresentará o plano antes das delegações. O executor local responderá em um formato curto e objetivo, sem raciocínio detalhado ou logs longos.

## Requisitos de hardware sugeridos

Os arquivos de configuração foram calibrados inicialmente para os requisitos de entrada, como por exemplo uma GPU RTX 3070 Ti 8 GB com Ollama 64k. Abaixo, um comparativo de perfis de hardware para diferentes níveis de uso:

### Comparativo de Configuração por Perfil

| Componente | Mínimo (Entrada) | Recomendado (Custo-Benefício) | Alto (Entusiasta / Workstation) |
| :--- | :--- | :--- | :--- |
| **GPU (VRAM)** | **8 GB** (ex: RTX 3060 8GB, RTX 4060) | **16 GB** (ex: RTX 4060 Ti 16GB, RTX 4070 Ti Super, RTX 5060 Ti 16GB) | **24 GB a 48 GB+** (ex: RTX 3090, RTX 4090, RTX 5090 ou dual GPU) |
| **Memória RAM** | **16 GB** DDR4/DDR5 | **32 GB** DDR4/DDR5 | **64 GB a 128 GB** DDR5 (preferência por *Dual/Quad Channel*) |
| **Processador (CPU)** | 6 núcleos modernos (ex: Ryzen 5 5600 / Core i5-12400) | 8 núcleos (ex: Ryzen 7 5700X / Core i5-13600K) | 12 a 16+ núcleos com suporte AVX-512 (ex: Ryzen 9 7900/9900X ou Core i7/i9) |
| **Armazenamento** | SSD NVMe 500 GB | SSD NVMe 1 TB (PCIe 4.0) | SSD NVMe 2 TB+ (PCIe 4.0/5.0) |
| **Modelos Suportados (100% GPU)** | **2B a 9B** (contexto curto/médio) | **2B a 24B** (com folga de contexto) | **32B a 70B+** (dependendo da VRAM e quantização) |


O modelo `gemma4:e4b` é a alternativa usada atualmente. Outros modelos podem exigir ajuste de VRAM, contexto e variante. Para 64k de contexto no Ollama, podem ser úteis:

```powershell
$env:OLLAMA_FLASH_ATTENTION = "1"
$env:OLLAMA_KV_CACHE_TYPE = "q8_0"
ollama serve
```

### Calibragem de Hardware para perfil de entrada (RTX 3070 Ti 8 GB + Ollama 64k)

Analisando alguns dos modelos disponíveis no Ollama:

| Modelo Local | Peso | Recomendação para `local-worker` |
|---|---|---|
| `qwen3.5:4b` | 3.3 GB | Alternativa de baixo consumo, excelente para tarefas simples e pouca VRAM |
| `gemma4:e4b` | 6.6 GB | Alternativa recomendada, bom desempenho em tarefas complexas |
| `deepseek-r1:8b` | 5.2 GB | **Evitar para o worker.** Modelos de raciocínio longo geram muito texto interno e costumam quebrar contratos de saída curta. |

### Dicas Extras para Economizar VRAM no Ollama
1. **Mantenha `small_model` igual ao `local-worker`:** No `opencode.json`, confirme se estão iguais. Isso evita que o Ollama descarregue e recarregue modelos diferentes na sua GPU de 8 GB a cada turno.
2. **Ative Flash Attention e KV Cache quantizado no Ollama** (opcional, recomendado para aproveitar os 64k sem estourar 8 GB de VRAM):
   - `OLLAMA_FLASH_ATTENTION=1`
   - `OLLAMA_KV_CACHE_TYPE=q8_0`

> ---
> #### Configuração de Cache KV (`OLLAMA_KV_CACHE_TYPE`)
>
> Define a precisão do histórico de contexto na VRAM. O uso de quantização reduz o consumo de memória e permite janelas de contexto maiores (`num_ctx`).
>
> ##### Opções Disponíveis
>
> | Configuração | Consumo de VRAM | Impacto na Qualidade | Caso de Uso Ideal |
> | :--- | :--- | :--- | :--- |
> | **`f16`** *(padrão)* | 100% | Nenhum | GPUs com folga de memória ou contextos curtos (4k–8k). |
> | **`q8_0`** | ~50% | Imperceptível | **Padrão recomendado:** equilibra economia de VRAM e precisão. |
> | **`q4_0`** | ~25% | Perceptível em textos longos | Cenários críticos de falta de VRAM ou contextos extremos. |
>
> ##### Recomendações por Capacidade de VRAM
>
> * **6 GB – 8 GB:** Use **`q8_0`** como padrão para modelos 7B–9B (`Q4_K_M`) com contexto de 8k–16k sem recorrer à RAM. Use **`q4_0`** apenas para documentos longos (32k+) no limite da GPU.
> * **10 GB – 12 GB:** Use **`q8_0`** para modelos 12B–14B em tarefas de código/chat ou para rodar modelos 7B–8B com contextos altos (32k–64k).
> * **16 GB – 24 GB:** Use **`q8_0`** para modelos 22B–35B (16k–32k de contexto) e para conter o consumo padrão de 32k aplicado automaticamente pelo Ollama a partir de 24 GiB. Use **`f16`** apenas em modelos ≤14B com contexto curto.
> * **48 GB ou mais:** Use **`f16`** para modelos médios ou contextos até 32k com velocidade máxima. Use **`q8_0`** para modelos 70B+ ou janelas extremas (128k–256k), onde o `f16` pode exceder 40 GB só de cache.
>
> ##### Requisitos e Pontos de Atenção
>
> * **Flash Attention obrigatório:** A quantização só funciona com `OLLAMA_FLASH_ATTENTION=1` ativado simultaneamente.
> * **Sensibilidade por modelo:** Arquiteturas com *Grouped-Query Attention* (GQA) agressiva (ex.: variantes menores da família Qwen) sofrem maior degradação com `q4_0`.
> * **Escopo global:** A variável aplica-se a todos os modelos do servidor e exige a reinicialização do serviço Ollama para entrar em vigor.
> ---
## Segurança

- Revise a configuração antes de executar.
- Altere as permissões somente se entender quais ferramentas o orquestrador e o worker podem acessar.

## Personalização

- Ajuste o agente Cloud em `agents/orchestrator.md`.
- Ajuste o comportamento do worker local em `agents/local-worker.md`.
- Ajuste o texto e o fluxo do comando em `commands/orquestrar.md`.
- Ajuste modelos, limites de contexto, variantes e permissões em `opencode.json`.
- Ajuste aparência e comportamento da CLI em `cli.json`.