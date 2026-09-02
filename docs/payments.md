# Merchant Payments Infra (GFK/GFC)

## Objectif

Permettre a chaque tenant/site de brancher son propre agregateur (PayPal, CinetPay, Flutterwave, etc.) sans modification frontend obligatoire, via l'API backend.

## Ce qui est en place

- Stockage des configurations de paiement par tenant/site dans `merchant_payment_configs`
- Chiffrement AES-256-GCM des credentials (cle derivee de `MERCHANT_CRED_KEY` ou `ENCRYPTION_KEY`)
- Initialisation checkout via provider choisi + enregistrement `payment_intents`
- Verification serveur (`verify`) + synchronisation du statut commande (`PAID` / `CANCELLED`)
- Routes dashboard JWT + routes DevAPI par cle API (`gfk_` et `gfc_`)

## Routes JWT (dashboard interne)

- `GET /v1/tenants/:tenantId/payment-providers`
- `POST /v1/tenants/:tenantId/payment-providers`
- `PATCH /v1/tenants/:tenantId/payment-providers/:configId/toggle`
- `PATCH /v1/tenants/:tenantId/payment-providers/:configId/default`
- `DELETE /v1/tenants/:tenantId/payment-providers/:configId`
- `GET /v1/projects/:projectId/payment-options`
- `POST /v1/projects/:projectId/checkout/pay`
- `GET /v1/projects/:projectId/checkout/verify/:providerRef`

## Routes DevAPI (X-Api-Key gfk_/gfc_)

- `GET /v1/cloud/payments/providers/available`
- `GET /v1/cloud/payments/providers/schema/:provider`
- `GET /v1/cloud/payments/providers`
- `POST /v1/cloud/payments/providers` (ADMIN)
- `PATCH /v1/cloud/payments/providers/:configId/toggle` (ADMIN)
- `PATCH /v1/cloud/payments/providers/:configId/default` (ADMIN)
- `DELETE /v1/cloud/payments/providers/:configId` (ADMIN)
- `GET /v1/cloud/payments/options`
- `POST /v1/cloud/payments/checkout/init` (READ_WRITE+)
- `GET /v1/cloud/payments/checkout/verify/:providerRef`

## Exemple: configurer PayPal en manuel

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/payments/providers" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: gfk_xxx_or_gfc_xxx" \
  -d '{
    "provider": "paypal",
    "label": "PayPal Business",
    "mode": "live",
    "isDefault": true,
    "credentials": {
      "client_id": "PAYPAL_CLIENT_ID",
      "secret": "PAYPAL_SECRET"
    },
    "config": {
      "display_name": "Payer avec PayPal",
      "supported_currencies": ["USD", "EUR", "XOF"]
    }
  }'
```

## Exemple: init checkout

```bash
curl -X POST "https://api.noverfly.com/v1/cloud/payments/checkout/init" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: gfk_xxx_or_gfc_xxx" \
  -d '{
    "siteId": "SITE_UUID",
    "orderId": "ORDER_UUID",
    "amount": 15000,
    "currency": "XOF",
    "returnUrl": "https://example.com/payment-return",
    "provider": "paypal"
  }'
```

## Notes securite

- Ne jamais renvoyer les credentials en clair (seuls les noms de cles sont exposes).
- En cle API scopee site, la config est limitee a ce site.
- Pour la production, definir `MERCHANT_CRED_KEY` (64 hex chars) dans l'environnement.
