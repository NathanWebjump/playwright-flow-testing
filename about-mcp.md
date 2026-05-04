# O que é MCP (Model Context Protocol)

MCP é um protocolo aberto desenvolvido pela Anthropic que define uma forma padronizada de conectar modelos de linguagem (LLMs) a ferramentas, dados e sistemas externos. Pense nele como uma "API universal" entre agentes de IA e o mundo ao redor deles.

## Problema que o MCP resolve

Antes do MCP, cada integração entre um LLM e uma ferramenta externa (banco de dados, browser, sistema de arquivos, APIs) era feita de forma ad hoc: cada provider de IA tinha sua própria convenção de "function calling", seus próprios formatos e seus próprios SDKs. Isso gerava:

- **Fragmentação**: integrações não eram reutilizáveis entre providers
- **Retrabalho**: a mesma ferramenta precisava ser reimplementada para cada modelo
- **Lock-in**: mudar de LLM significava reescrever as integrações

O MCP resolve isso com um protocolo único, independente de model, que qualquer cliente (IDE, agente, chatbot) pode consumir.

## Arquitetura

O MCP segue um modelo cliente-servidor:

```
┌─────────────────────┐        ┌──────────────────────┐
│   MCP Client        │        │   MCP Server         │
│                     │        │                      │
│  (VS Code Copilot,  │◄──────►│  (Playwright, DB,    │
│   Claude Desktop,   │  JSON  │   filesystem, APIs)  │
│   agentes custom)   │  -RPC  │                      │
└─────────────────────┘        └──────────────────────┘
```

- **MCP Client**: o agente de IA ou IDE que consome as capacidades
- **MCP Server**: processo (local ou remoto) que expõe ferramentas e recursos
- **Transporte**: comunicação via `stdio` (processo local) ou `SSE/HTTP` (remoto)

## Primitivas do protocolo

Um servidor MCP pode expor três tipos de primitivas:

### Tools
Funções que o modelo pode invocar. São a primitiva mais usada — equivalente ao "function calling" de outros providers, mas padronizado.

```json
{
  "name": "browser_navigate",
  "description": "Navega para uma URL no browser",
  "inputSchema": {
    "type": "object",
    "properties": {
      "url": { "type": "string" }
    },
    "required": ["url"]
  }
}
```

### Resources
Dados que o servidor expõe para o modelo ler — arquivos, linhas de banco de dados, respostas de API. São somente leitura e identificados por URI.

### Prompts
Templates de prompt pré-definidos pelo servidor que o usuário pode invocar diretamente no cliente.

## Ciclo de vida de uma chamada

1. O cliente lista as tools disponíveis via `tools/list`
2. O LLM decide invocar uma tool e retorna um `tool_call`
3. O cliente envia `tools/call` para o servidor MCP
4. O servidor executa a ação e retorna o resultado
5. O resultado é injetado no contexto do LLM para a próxima resposta

## Por que é relevante para desenvolvedores

- **Reutilizável**: um servidor MCP funciona com qualquer cliente compatível (VS Code, Claude Desktop, Cursor, etc.)
- **Composável**: você pode conectar múltiplos servidores MCP ao mesmo cliente simultaneamente
- **Local-first**: servidores rodam como processos locais — sem tráfego de dados sensíveis para terceiros
- **Extensível**: qualquer time pode publicar seu próprio servidor MCP

## Referências

- [Especificação oficial do MCP](https://modelcontextprotocol.io)
- [SDK TypeScript](https://github.com/modelcontextprotocol/typescript-sdk)
- [SDK Python](https://github.com/modelcontextprotocol/python-sdk)
- [Repositório de servidores MCP da comunidade](https://github.com/punkpeye/awesome-mcp-servers)
