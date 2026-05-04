# Page Object Model (POM)

Page Object Model é um design pattern de automação de testes que organiza o código criando uma camada de abstração entre os testes e a interface do usuário. Cada página (ou componente significativo) da aplicação é representada por uma classe que encapsula os seletores e as ações disponíveis naquela tela.

## Por que usar POM?

Sem o padrão, testes tendem a repetir seletores e lógica de interação diretamente nos arquivos `.spec.ts`. Quando a UI muda — um `id` é renomeado, um botão muda de posição — você precisa atualizar cada teste que usa aquele elemento.

Com POM:

- **Manutenibilidade**: seletores e ações ficam em um só lugar
- **Legibilidade**: os testes descrevem *comportamento*, não mecânica de cliques
- **Reuso**: a mesma Page Object é usada por múltiplos testes
- **Separação de responsabilidades**: testes definem *o quê* testar; Page Objects definem *como* interagir

## Estrutura de pastas

```
projeto/
├── playwright.config.ts
├── tests/
│   ├── auth/
│   │   ├── login.spec.ts
│   │   └── logout.spec.ts
│   ├── checkout/
│   │   └── purchase-flow.spec.ts
│   └── dashboard/
│       └── dashboard.spec.ts
└── pages/                        ← Page Objects aqui
    ├── BasePage.ts               ← classe base com utilidades comuns
    ├── LoginPage.ts
    ├── DashboardPage.ts
    ├── CheckoutPage.ts
    └── components/               ← componentes reutilizáveis
        ├── Header.ts
        ├── Modal.ts
        └── Navbar.ts
```

> A pasta `pages/` fica no mesmo nível que `tests/`. Cada arquivo representa uma tela ou componente da aplicação.

## Anatomia de uma Page Object

### BasePage.ts

Centraliza utilitários compartilhados por todas as páginas.

```typescript
// pages/BasePage.ts
import { Page, Locator } from '@playwright/test';

export abstract class BasePage {
  constructor(protected readonly page: Page) {}

  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }
}
```

### LoginPage.ts

Encapsula seletores e ações da tela de login.

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  // Seletores declarados como propriedades da classe
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput    = page.getByLabel('E-mail');
    this.passwordInput = page.getByLabel('Senha');
    this.submitButton  = page.getByRole('button', { name: 'Entrar' });
    this.errorMessage  = page.getByTestId('login-error');
  }

  async navigate() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.errorMessage.textContent();
  }
}
```

### Usando a Page Object no teste

```typescript
// tests/auth/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/LoginPage';
import { DashboardPage } from '../../pages/DashboardPage';

test.describe('Login', () => {
  test('login com credenciais válidas redireciona para o dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboardPage = new DashboardPage(page);

    await loginPage.navigate();
    await loginPage.login('usuario@exemplo.com', 'senha123');

    await expect(page).toHaveURL('/dashboard');
    await expect(dashboardPage.welcomeMessage).toBeVisible();
  });

  test('login com senha errada exibe mensagem de erro', async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.navigate();
    await loginPage.login('usuario@exemplo.com', 'senha-errada');

    expect(await loginPage.getErrorMessage()).toBe('Credenciais inválidas');
  });
});
```

## Componentes reutilizáveis

Partes da UI que aparecem em múltiplas páginas (header, modal de confirmação, navbar) merecem sua própria classe em `pages/components/`.

```typescript
// pages/components/Modal.ts
import { Page, Locator } from '@playwright/test';

export class ConfirmationModal {
  private readonly confirmButton: Locator;
  private readonly cancelButton: Locator;
  private readonly titleText: Locator;

  constructor(private readonly page: Page) {
    this.confirmButton = page.getByRole('button', { name: 'Confirmar' });
    this.cancelButton  = page.getByRole('button', { name: 'Cancelar' });
    this.titleText     = page.getByTestId('modal-title');
  }

  async confirm() {
    await this.confirmButton.click();
  }

  async cancel() {
    await this.cancelButton.click();
  }

  async getTitle(): Promise<string | null> {
    return this.titleText.textContent();
  }
}
```

Uso em uma Page Object:

```typescript
// pages/CheckoutPage.ts
import { Page } from '@playwright/test';
import { BasePage } from './BasePage';
import { ConfirmationModal } from './components/Modal';

export class CheckoutPage extends BasePage {
  readonly modal: ConfirmationModal;

  constructor(page: Page) {
    super(page);
    this.modal = new ConfirmationModal(page);
  }

  async finalizarCompra() {
    await this.page.getByRole('button', { name: 'Finalizar compra' }).click();
    await this.modal.confirm();
  }
}
```

## Boas práticas

| Prática | Motivo |
|---|---|
| Declarar locators no construtor | Evita recriar objetos a cada chamada |
| Usar `getByRole`, `getByLabel`, `getByTestId` | Seletores semânticos são mais estáveis que CSS/XPath |
| Não usar `expect()` dentro da Page Object | Page Objects interagem com a UI; asserções ficam nos testes |
| Um arquivo por tela/componente | Mantém os arquivos pequenos e focados |
| Métodos com nomes no domínio do negócio | `finalizarCompra()` em vez de `clickButton('#btn-checkout')` |
| Herdar de `BasePage` | Compartilha utilitários sem duplicação |

## Playwright Fixtures com POM

Para não instanciar Page Objects manualmente em cada teste, você pode criar fixtures customizadas:

```typescript
// tests/fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

type Fixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect } from '@playwright/test';
```

```typescript
// tests/auth/login.spec.ts
import { test, expect } from '../fixtures';

test('login válido', async ({ loginPage, dashboardPage }) => {
  await loginPage.navigate();
  await loginPage.login('usuario@exemplo.com', 'senha123');

  await expect(dashboardPage.welcomeMessage).toBeVisible();
});
```

## Resumo

```
testes (.spec.ts)
    └── descrevem comportamento e fazem asserções
         └── chamam Page Objects (pages/)
              └── encapsulam seletores e ações da UI
```

Page Object Model não é exclusivo do Playwright — o mesmo padrão se aplica a Selenium, Cypress e qualquer outro framework de automação. A estrutura de pastas e os princípios permanecem os mesmos.
