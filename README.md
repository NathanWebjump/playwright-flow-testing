# Escrevendo testes com o Playwright MCP

Com o Playwright MCP configurado, você pode gerar testes E2E completos simplesmente descrevendo os fluxos em linguagem natural para o agente. Este guia mostra como fazer isso de forma eficiente.

## Pré-requisitos

- Playwright MCP configurado (veja [mcp-playwright.md](./mcp-playwright.md))
- Playwright instalado no projeto:

```bash
npm init playwright@latest
```

Isso cria a estrutura básica com `playwright.config.ts` e a pasta `tests/`.

## Fluxo de trabalho

O fluxo básico para gerar um teste é:

1. Inicie o servidor MCP no VS Code
2. Abra o chat do Copilot no modo **Agent** (`@agent` ou `Ctrl+Alt+I`)
3. Descreva o fluxo que deseja testar
4. O agente navega pela aplicação, inspeciona os elementos e gera o arquivo `.spec.ts`
5. Revise e execute o teste gerado

## Exemplos de prompts

### Teste de login

```
Navegue para http://localhost:3000/login, faça login com o usuário
"admin@exemplo.com" e senha "senha123", verifique que o dashboard
é exibido após o login e gere um teste Playwright para esse fluxo.
```

### Teste de formulário

```
Acesse http://localhost:3000/cadastro, preencha o formulário de
novo usuário com dados válidos, submeta e confirme que a mensagem
de sucesso aparece. Salve como tests/cadastro.spec.ts.
```

### Teste de fluxo completo (E2E)

```
Teste o fluxo completo de compra na loja em http://localhost:3000:
1. Busque pelo produto "Teclado mecânico"
2. Adicione ao carrinho
3. Vá para o checkout
4. Preencha os dados de entrega
5. Confirme o pedido
Gere o arquivo tests/checkout.spec.ts com assertions em cada etapa.
```

### Teste de regressão a partir de um bug

```
O botão "Salvar" em http://localhost:3000/perfil não funciona quando
o campo "telefone" está vazio. Reproduza o problema e gere um teste
de regressão que garante que esse bug não volte.
```

## Estrutura do teste gerado

O agente gera arquivos no padrão do Playwright. Um exemplo típico:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Fluxo de login', () => {
  test('deve autenticar usuário com credenciais válidas', async ({ page }) => {
    await page.goto('http://localhost:3000/login');

    await page.getByLabel('E-mail').fill('admin@exemplo.com');
    await page.getByLabel('Senha').fill('senha123');
    await page.getByRole('button', { name: 'Entrar' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  });

  test('deve exibir erro com credenciais inválidas', async ({ page }) => {
    await page.goto('http://localhost:3000/login');

    await page.getByLabel('E-mail').fill('invalido@exemplo.com');
    await page.getByLabel('Senha').fill('senhaerrada');
    await page.getByRole('button', { name: 'Entrar' }).click();

    await expect(page.getByText('Credenciais inválidas')).toBeVisible();
  });
});
```

Note que o agente usa locators semânticos (`getByLabel`, `getByRole`, `getByText`) ao invés de seletores frágeis como `div.btn-submit` — isso é a melhor prática do Playwright e o MCP segue esse padrão.

## Executando os testes

```bash
# Rodar todos os testes
npx playwright test

# Rodar um arquivo específico
npx playwright test tests/login.spec.ts

# Rodar em modo UI (recomendado para debug)
npx playwright test --ui

# Gerar relatório HTML
npx playwright test --reporter=html
```

## Dicas para prompts mais eficientes

**Seja específico sobre assertions**
> Em vez de: "teste o login"
> Use: "teste o login e verifique que o usuário é redirecionado para `/dashboard` e que o nome dele aparece no header"

**Indique onde salvar o arquivo**
> "...e salve o teste em `tests/auth/login.spec.ts`"

**Peça múltiplos cenários de uma vez**
> "Gere testes para os cenários: login com sucesso, senha errada, e-mail não cadastrado e conta bloqueada"

**Aproveite o contexto visual**
> Se a página tem UI complexa, peça um screenshot antes: "tire um screenshot e depois gere o teste para o formulário de pagamento"

**Itere sobre testes gerados**
> "O teste falhou na asserção da linha 12 porque o texto mudou para 'Bem-vindo'. Corrija o teste."

## Inspecionando o que o agente vê

Durante a navegação, você pode pedir ao agente para mostrar o estado atual da página:

```
"Mostre o snapshot de acessibilidade da página atual"
```

Isso é útil para entender por que o agente não está encontrando um elemento — talvez o componente esteja dentro de um iframe, ou o atributo `aria-label` esteja faltando.

---

## Arquitetura de automações

Testes gerados pelo agente com uma linha de código funcionam bem para validações pontuais, mas projetos maiores precisam de uma estrutura que garanta **manutenibilidade** e **legibilidade** a longo prazo. Os dois guias abaixo cobrem as abordagens mais adotadas:

### [Page Object Model (POM)](./page-object-model.md)

Organiza o código criando uma camada de abstração entre os testes e a interface. Cada tela da aplicação vira uma classe TypeScript que encapsula seletores e ações, deixando os arquivos `.spec.ts` focados apenas em comportamento.

**Quando usar:** projetos Playwright puros, times que preferem testes escritos diretamente em TypeScript.

Destaques:
- `BasePage.ts` com utilitários compartilhados
- Locators declarados no construtor da classe
- Componentes reutilizáveis em `pages/components/`
- Boas práticas: seletores semânticos, asserções fora da Page Object, nomes de métodos no domínio do negócio

### [Page Object Model com Cucumber](./page-object-cucumber.md)

Combina Page Object Model com BDD (Behavior-Driven Development) usando o framework Cucumber e sintaxe Gherkin. Testes ficam legíveis por stakeholders não-técnicos, enquanto o código de automação permanece organizado nas Page Objects.

**Quando usar:** times multidisciplinares (QA + PO + devs), projetos onde os critérios de aceite viram cenários de teste.

Destaques:
- Feature files `.feature` escritos em linguagem natural
- Step definitions como "cola" entre Gherkin e TypeScript
- `World` para compartilhar estado entre steps do mesmo cenário
- Hooks de `Before`/`After` para setup e teardown
- Configuração de relatórios HTML e JSON

### Comparação rápida

| | POM puro | POM + Cucumber |
|---|---|---|
| Linguagem dos testes | TypeScript | Gherkin (linguagem natural) |
| Curva de aprendizado | Baixa | Média |
| Legibilidade para não-devs | Moderada | Alta |
| Dependências extras | Nenhuma | `@cucumber/cucumber`, `ts-node` |
| Ideal para | Times de engenharia | Times multidisciplinares |
