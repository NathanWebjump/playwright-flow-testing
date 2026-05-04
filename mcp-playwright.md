# MCP do Playwright

O [Playwright MCP](https://github.com/microsoft/playwright-mcp) é um servidor MCP oficial da Microsoft que expõe as capacidades do Playwright como tools consumíveis por agentes de IA. Com ele, você pode dar ao seu agente a capacidade de controlar um browser real — navegar, clicar, preencher formulários, tirar screenshots e escrever testes — tudo via linguagem natural.

## Como funciona

O servidor Playwright MCP roda como um processo local e se comunica com o cliente MCP (ex: GitHub Copilot no VS Code) via `stdio`. Quando você instrui o agente a interagir com uma página web, ele invoca as tools do servidor, que por sua vez controlam uma instância do Playwright.

```
 VS Code Copilot
      │
      │ MCP (stdio)
      ▼
 playwright-mcp (servidor)
      │
      │ Playwright API
      ▼
  Chromium / Firefox / WebKit
```

O servidor usa, por padrão, um **snapshot de acessibilidade** (accessibility tree) da página ao invés de screenshots para transmitir o estado da UI ao modelo. Isso é mais eficiente em tokens e mais preciso para interações com elementos.

## Instalação

### Pré-requisitos

- Node.js >= 18
- VS Code com a extensão GitHub Copilot

### Configuração no VS Code

Abra (ou crie) o arquivo `.vscode/mcp.json` na raiz do seu projeto:

```json
{
  "servers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

Alternativamente, configure globalmente em `settings.json`:

```json
{
  "mcp": {
    "servers": {
      "playwright": {
        "command": "npx",
        "args": ["@playwright/mcp@latest"]
      }
    }
  }
}
```

Após salvar, o VS Code exibirá um botão **Start** ao lado da configuração do servidor. Clique para iniciá-lo.

## Tools disponíveis

O servidor expõe as seguintes tools para o agente:

| Tool | Descrição |
|---|---|
| `browser_navigate` | Navega para uma URL |
| `browser_click` | Clica em um elemento |
| `browser_type` | Digita texto em um campo |
| `browser_snapshot` | Captura o estado de acessibilidade da página |
| `browser_screenshot` | Tira screenshot da página |
| `browser_hover` | Passa o mouse sobre um elemento |
| `browser_select_option` | Seleciona uma opção em um `<select>` |
| `browser_check` | Marca/desmarca checkboxes |
| `browser_fill` | Preenche um campo de formulário |
| `browser_press_key` | Pressiona uma tecla |
| `browser_wait_for` | Aguarda um seletor aparecer |
| `browser_evaluate` | Executa JavaScript na página |
| `browser_close` | Fecha o browser |

## Modos de operação

### Modo padrão (accessibility snapshot)
O servidor envia ao modelo um snapshot da árvore de acessibilidade da página. É o modo recomendado — mais rápido e sem custo de processamento de imagem.

### Modo com visão (screenshot)
Para páginas com UI complexa onde o accessibility tree é insuficiente, você pode solicitar explicitamente um screenshot:

```
"Tire um screenshot da página atual"
```

### Modo headless
Por padrão, o browser abre em modo headful (você vê a janela). Para CI ou uso em background:

```json
{
  "servers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

## Casos de uso

- **Geração de testes E2E**: instrua o agente a navegar pelo fluxo e gerar o arquivo de teste automaticamente
- **Debugging de testes**: deixe o agente reproduzir uma falha e investigar o estado da página
- **Scraping assistido**: explore páginas e extraia dados com supervisão do agente
- **Documentação automática**: gere screenshots e descreva fluxos de UI automaticamente

## Referências

- [Repositório oficial: microsoft/playwright-mcp](https://github.com/microsoft/playwright-mcp)
- [Documentação do Playwright](https://playwright.dev)
- [Configuração de MCP no VS Code](https://code.visualstudio.com/docs/copilot/chat/mcp-servers)
