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
