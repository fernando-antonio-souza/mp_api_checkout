# MP Checkout

Checkout com integração direta à **API do Mercado Pago** (Checkout API / Pagamentos Transparentes), construído com **FastAPI** e frontend em HTML/TailwindCSS. Permite pagamentos via **Cartão de crédito**, **Pix** e **Boleto bancário**.

## Stack

| Camada | Tecnologia |
|---|---|
| Backend / API | [FastAPI](https://fastapi.tiangolo.com/) |
| Servidor ASGI | [Uvicorn](https://www.uvicorn.org/) |
| Templates | Jinja2 |
| Frontend | HTML5, [TailwindCSS (CDN)](https://tailwindcss.com/), JavaScript vanilla |
| Pagamentos | [Mercado Pago — API de Payments](https://www.mercadopago.com.br/developers/en/reference/online-payments/checkout-api-payments/create-payment/post) (`requests`, sem SDK) |
| Configuração | [python-decouple](https://github.com/thekevinscott/python-decouple) (`.env`) |

## Estrutura do projeto

```
mp_checkout/
├── app.py                  # Rotas FastAPI (GET renderiza o checkout, POST cria o pagamento)
├── services/
│   ├── __init__.py         # Exporta a classe MercadoPago
│   └── mercadopago.py      # Cliente HTTP da API MP (card_tokens + payments)
├── templates/
│   └── checkout.html       # Página de checkout (Tailwind + JS puro)
└── .env                    # Variáveis de ambiente (não versionar!)
```

## Configuração

### 1. Variáveis de ambiente

Crie um arquivo `.env` na raiz:

```env
MP_BASE_API_URL=https://api.mercadopago.com
MP_ACCESS_TOKEN=SUA_CREDENCIAL_AQUI
```

- **Testes (sandbox):** use o **Access Token de teste** do [painel de desenvolvedores](https://www.mercadopago.com.br/developers/panel/app) → Sua aplicação → Credenciais de teste.
- **Produção:** troque pelo Access Token de produção.
- ⚠️ Nunca versione o `.env` nem exponha o Access Token no frontend.

### 2. Dependências

```bash
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
# .venv\Scripts\activate         # Windows

pip install fastapi "uvicorn[standard]" jinja2 python-decouple requests
```

## Como rodar

```bash
source .venv/bin/activate
uvicorn app:app --reload
```

Acesse **http://127.0.0.1:8000/** — o checkout será renderizado.

## Funcionalidades do checkout

- **Seleção animada entre 3 métodos de pagamento** (cartão, Pix, boleto), com painéis dinâmicos
- Validações client-side: **CPF** (dígitos verificadores), e-mail, campos obrigatórios e formatação de valor
- **Máscaras de input**: CPF, CEP, número do cartão, validade (MM/AA), CVV, UF
- **Detecção de bandeira** enquanto digita (Visa, Mastercard, Amex, Elo/Hipercard)
- Seletor de **parcelamento em até 12x** com preview de valor
- Tela de resultado com verificação animada e dados específicos por método:
  - **Cartão** → status `approved` / `pending`
  - **Pix** → QR Code (base64) + código "Copia e Cola" com botão copiar
  - **Boleto** → link em `transaction_details.external_resource_url` + linha digitável
- Toast de erros com as mensagens exatas devolvidas pela API do MP

## Fluxo da API

### `GET /`
Renderiza `templates/checkout.html`.

### `POST /`
Recebe o JSON do checkout, monta os payloads por método e chama a API do MP.

**Payload enviado pelo frontend:**

```jsonc
{
  "payment_method": "card",          // "card" | "pix" | "boleto"
  "transaction_amount": 299.90,
  "description": "Plano Premium Anual",
  "email": "comprador@email.com",
  "identification_number": "11144477735",   // CPF (somente números)

  // apenas para "card":
  "card_number": "5031433215406351",
  "expiration_month": 11,
  "expiration_year": 2030,
  "security_code": "123",
  "cardholder_name": "APRO",
  "installments": 12,

  // apenas para "boleto":
  "first_name": "João", "last_name": "Silva",
  "zip_code": "06233-200", "street_name": "Av. das Nações",
  "street_number": "123", "neighborhood": "Centro",
  "city": "Osasco", "federal_unit": "SP"
}
```

**Como o backend traduz para a API do MP:**

| Método | Endpoints usados | Observações |
|---|---|---|
| Cartão | `POST /v1/card_tokens` → `POST /v1/payments` | Gera o token do cartão no backend e envia com `installments` |
| Pix | `POST /v1/payments` | `payment_method_id: "pix"`; resposta traz QR em `point_of_interaction` |
| Boleto | `POST /v1/payments` | `payment_method_id: "bolbradesco"`; exige `payer.address.*` completo |

Cada requisição envia header `X-Idempotency-Key` (UUID) para evitar cobranças duplicadas em retries.

**Respostas de erro:**

- `400` — método inválido ou campos obrigatórios ausentes (com a lista dos campos faltantes)
- `502` — erro retornado pela API do MP, repassado com a mensagem do MP
- `503`-like / rede — falha de conexão tratada no frontend com toast

## Testes no sandbox

- O **Access Token de teste** gera pagamentos reais "fake" — nada é cobrado.
- Para cartões, use os [cartões de teste](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/additional-content/your-integrations/test/cards) oficiais (ex: `5031 4332 1540 6351`, CVV `123`, validade futura, nome `APRO` = aprovado).
- E-mail do comprador deve estar no formato `@testuser.com`.
- Meios offline (Pix/boleto) terminam sempre em **`pending`** — é o comportamento correto; a confirmação viria via webhook.
- ⚠️ **Limitação conhecida:** a página do boleto de sandbox (`/sandbox/payments/...`) só renderiza para **usuários de teste logados** no MP na mesma sessão do navegador; caso contrário exibe "No fue posible completar tu pago". Em produção isso não acontece.

## Próximos passos (roadmap)

- [ ] **Webhooks** (IPN/notifications) para confirmar `approved` do Pix/boleto de forma assíncrona
- [ ] **Gerar o token do cartão no frontend** com o SDK JS do MP (o número do cartão não deve passar pelo servidor em produção)
- [ ] Paginação/estado real do pedido (atualmente o valor e a descrição são fixos no HTML)
- [ ] Tailwind via build (em vez de CDN) para produção
- [ ] Testes automatizados com mocks da API do MP

## Segurança

- O `MP_ACCESS_TOKEN` fica apenas no backend (`.env`)
- Os dados de cartão atualmente geram *card token* no backend — mover para o SDK JS antes de ir para produção
- Idempotência ativa nas chamadas de pagamento
