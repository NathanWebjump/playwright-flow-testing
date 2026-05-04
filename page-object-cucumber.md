# Page Object Model com Cucumber

Cucumber é um framework de BDD (Behavior-Driven Development) que permite escrever testes em linguagem natural (Gherkin). Combinado com Page Object Model, você obtém testes legíveis por stakeholders não-técnicos, com código de automação organizado e manutenível.

## Como as peças se encaixam

```
Feature Files (.feature)       ← escritos pelo time (QA, PO, devs)
    └── Step Definitions       ← "cola" entre Gherkin e código
         └── Page Objects      ← encapsulam a interação com a UI
              └── Playwright / Selenium / WebdriverIO
```

- **Feature files**: descrevem o comportamento em linguagem natural (Gherkin)
- **Step definitions**: mapeiam cada linha Gherkin para código TypeScript/JavaScript
- **Page Objects**: abstraem os seletores e ações da interface

## Estrutura de pastas

```
projeto/
├── playwright.config.ts
├── cucumber.config.ts            ← configuração do Cucumber
├── features/                     ← arquivos .feature (Gherkin)
│   ├── auth/
│   │   ├── login.feature
│   │   └── logout.feature
│   ├── checkout/
│   │   └── purchase-flow.feature
│   └── dashboard/
│       └── dashboard.feature
├── step-definitions/             ← mapeamento Gherkin → código
│   ├── auth/
│   │   ├── login.steps.ts
│   │   └── logout.steps.ts
│   ├── checkout/
│   │   └── purchase-flow.steps.ts
│   └── shared/
│       └── common.steps.ts       ← steps reutilizados entre features
├── pages/                        ← Page Objects
│   ├── BasePage.ts
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   ├── CheckoutPage.ts
│   └── components/
│       ├── Header.ts
│       └── Modal.ts
├── support/                      ← configurações e hooks do Cucumber
│   ├── world.ts                  ← World: contexto compartilhado entre steps
│   └── hooks.ts                  ← Before/After hooks (setup e teardown)
└── reports/                      ← relatórios gerados (HTML, JSON)
```

## Instalação

```bash
npm install --save-dev \
  @cucumber/cucumber \
  @playwright/test \
  playwright \
  ts-node \
  typescript
```

## Configuração do Cucumber

```typescript
// cucumber.config.ts
import { defineConfig } from '@cucumber/cucumber';

export default defineConfig({
  paths: ['features/**/*.feature'],
  require: [
    'support/world.ts',
    'support/hooks.ts',
    'step-definitions/**/*.steps.ts',
  ],
  requireModule: ['ts-node/register'],
  format: [
    'progress-bar',
    'html:reports/cucumber-report.html',
    'json:reports/cucumber-report.json',
  ],
  parallel: 2,
});
```

## World: o contexto compartilhado

O `World` é o objeto que existe durante toda a execução de um cenário. É nele que você guarda a instância do browser, das Page Objects e qualquer estado que precisa ser compartilhado entre steps.

```typescript
// support/world.ts
import { setWorldConstructor, World, IWorldOptions } from '@cucumber/cucumber';
import { Browser, BrowserContext, Page, chromium } from 'playwright';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';
import { CheckoutPage } from '../pages/CheckoutPage';

export interface CustomWorld extends World {
  browser: Browser;
  context: BrowserContext;
  page: Page;
  // Page Objects disponíveis para todos os steps
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  checkoutPage: CheckoutPage;
}

setWorldConstructor(
  class extends World implements CustomWorld {
    browser!: Browser;
    context!: BrowserContext;
    page!: Page;
    loginPage!: LoginPage;
    dashboardPage!: DashboardPage;
    checkoutPage!: CheckoutPage;

    constructor(options: IWorldOptions) {
      super(options);
    }
  }
);
```

## Hooks: setup e teardown

```typescript
// support/hooks.ts
import { Before, After, BeforeAll, AfterAll } from '@cucumber/cucumber';
import { chromium } from 'playwright';
import { CustomWorld } from './world';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';
import { CheckoutPage } from '../pages/CheckoutPage';

BeforeAll(async function () {
  // Executado uma vez antes de todos os cenários
});

Before(async function (this: CustomWorld) {
  this.browser = await chromium.launch({ headless: true });
  this.context = await this.browser.newContext();
  this.page    = await this.context.newPage();

  // Instancia as Page Objects no World
  this.loginPage    = new LoginPage(this.page);
  this.dashboardPage = new DashboardPage(this.page);
  this.checkoutPage  = new CheckoutPage(this.page);
});

After(async function (this: CustomWorld, scenario) {
  // Captura screenshot em caso de falha
  if (scenario.result?.status === 'FAILED') {
    const screenshot = await this.page.screenshot({ fullPage: true });
    this.attach(screenshot, 'image/png');
  }

  await this.page.close();
  await this.context.close();
  await this.browser.close();
});
```

## Feature file (Gherkin)

```gherkin
# features/auth/login.feature
Feature: Autenticação de usuários

  Como um usuário registrado
  Quero fazer login na plataforma
  Para acessar meu painel

  Background:
    Given que estou na página de login

  Scenario: Login com credenciais válidas
    When preencho o e-mail com "usuario@exemplo.com"
    And preencho a senha com "senha123"
    And clico em Entrar
    Then devo ser redirecionado para o dashboard
    And devo ver a mensagem de boas-vindas

  Scenario: Login com senha incorreta
    When preencho o e-mail com "usuario@exemplo.com"
    And preencho a senha com "senha-errada"
    And clico em Entrar
    Then devo ver a mensagem de erro "Credenciais inválidas"

  Scenario Outline: Login com diferentes perfis
    When preencho o e-mail com "<email>"
    And preencho a senha com "<senha>"
    And clico em Entrar
    Then devo ser redirecionado para o dashboard

    Examples:
      | email                 | senha    |
      | admin@exemplo.com     | admin123 |
      | gerente@exemplo.com   | ger456   |
      | operador@exemplo.com  | op789    |
```

## Step Definitions

```typescript
// step-definitions/auth/login.steps.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { CustomWorld } from '../../support/world';

Given('que estou na página de login', async function (this: CustomWorld) {
  await this.loginPage.navigate();
});

When('preencho o e-mail com {string}', async function (this: CustomWorld, email: string) {
  await this.loginPage.fillEmail(email);
});

When('preencho a senha com {string}', async function (this: CustomWorld, password: string) {
  await this.loginPage.fillPassword(password);
});

When('clico em Entrar', async function (this: CustomWorld) {
  await this.loginPage.submit();
});

Then('devo ser redirecionado para o dashboard', async function (this: CustomWorld) {
  await expect(this.page).toHaveURL('/dashboard');
});

Then('devo ver a mensagem de boas-vindas', async function (this: CustomWorld) {
  await expect(this.dashboardPage.welcomeMessage).toBeVisible();
});

Then('devo ver a mensagem de erro {string}', async function (this: CustomWorld, message: string) {
  const error = await this.loginPage.getErrorMessage();
  expect(error).toBe(message);
});
```

## Page Objects (mesma estrutura do POM puro)

Os Page Objects no contexto do Cucumber são idênticos aos do POM convencional. A diferença está em como são instanciados — no World, e não dentro do teste.

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
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

  async fillEmail(email: string) {
    await this.emailInput.fill(email);
  }

  async fillPassword(password: string) {
    await this.passwordInput.fill(password);
  }

  async submit() {
    await this.submitButton.click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.errorMessage.textContent();
  }
}
```

## Steps compartilhados

Steps que se repetem em múltiplas features ficam em `step-definitions/shared/`:

```typescript
// step-definitions/shared/common.steps.ts
import { Given } from '@cucumber/cucumber';
import { CustomWorld } from '../../support/world';

Given('que o usuário está autenticado', async function (this: CustomWorld) {
  await this.loginPage.navigate();
  await this.loginPage.fillEmail('usuario@exemplo.com');
  await this.loginPage.fillPassword('senha123');
  await this.loginPage.submit();
  await this.page.waitForURL('/dashboard');
});

Given('que estou na página {string}', async function (this: CustomWorld, path: string) {
  await this.page.goto(path);
});
```

## Tags e organização de cenários

Cucumber suporta tags para categorizar e filtrar cenários:

```gherkin
@smoke @regressao
Scenario: Login com credenciais válidas
  ...

@critico @smoke
Scenario: Fluxo de compra completo
  ...
```

Execute apenas um subconjunto:

```bash
# Apenas cenários marcados com @smoke
npx cucumber-js --tags "@smoke"

# Excluir cenários @wip (work in progress)
npx cucumber-js --tags "not @wip"

# Combinação
npx cucumber-js --tags "@smoke and not @flaky"
```

## Scripts no package.json

```json
{
  "scripts": {
    "test":         "cucumber-js",
    "test:smoke":   "cucumber-js --tags @smoke",
    "test:headful": "HEADLESS=false cucumber-js",
    "test:report":  "cucumber-js && open reports/cucumber-report.html"
  }
}
```

## Comparação: Cucumber vs Playwright puro

| Aspecto | Playwright puro (`.spec.ts`) | Playwright + Cucumber |
|---|---|---|
| Legibilidade técnica | Alta | Alta |
| Legibilidade para negócio | Baixa | Alta (Gherkin) |
| Velocidade de escrita | Mais rápida | Mais lenta |
| Documentação viva | Não | Sim (`.feature` files) |
| Curva de aprendizado | Menor | Maior |
| Indicado para | Times técnicos | Times com PO/QA envolvidos |

## Resumo da arquitetura

```
features/           → linguagem de negócio (Gherkin)
step-definitions/   → ponte entre Gherkin e código
support/            → infraestrutura (World, hooks, browser)
pages/              → abstração da UI (Page Objects)
reports/            → saída visual dos testes
```

Essa separação garante que cada camada tenha uma única responsabilidade: negócio, orquestração, infraestrutura e interface.
