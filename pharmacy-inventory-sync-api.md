---
layout: default
title: API de synchronisation des stocks pharmacie
---

[English](/en/pharmacy-inventory-sync-api.html)

# API de synchronisation des stocks pharmacie

Transmettez les stocks et les prix de votre système d'inventaire vers Medelu.

**Method:** `POST` only  
**URL:** `https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync`  
**Auth:** HMAC-SHA256 per inventory system (`verify_jwt` is `false`; no Supabase JWT required)

Medelu fournit hors bande :

- le `hmac_secret` pour votre système d'inventaire
- le nom de votre système d'inventaire pour le header `X-Inventory-System` (case sensitive ; valeur attribuée par Medelu ; à utiliser exactement telle que fournie)
- les codes CNS de pharmacie activés pour votre intégration sur la plateforme Medelu

---

## Headers requis

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Inventory-System` | Nom de votre système d'inventaire tel qu'attribué par Medelu (doit correspondre exactement, case sensitive) |
| `X-Inventory-Signature` | `timestamp=<unix_seconds>,signature=<hex_hmac>`. `timestamp` correspond au moment de préparation de la requête (maintenant), pas à la dernière modification des stocks dans votre système. |

---

## Request body

**When to set `timestamptz`:** utilisez l'heure à laquelle vous **préparez et envoyez cette requête API**, pas celle à laquelle les stocks ou les prix ont été modifiés pour la dernière fois dans votre système d'inventaire. Le `timestamp` du header de signature et le `timestamptz` du body doivent refléter ce même instant (à ±1 seconde près). En raison de la fenêtre de 5 minutes, un timestamp provenant d'un export d'inventaire plus ancien sera rejeté.

JSON object :

```json
{
  "pharmacy_code": "811111-11",
  "timestamptz": "2026-06-26T14:30:00Z",
  "products": [
    {"cefip": "0144099", "price": 9.99, "stock": 6, "tva": 3.00},
    {"cefip": "0104906", "price": 12.45, "stock": 56, "tva": 17.00}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `pharmacy_code` | string | Code CNS de la pharmacie au Luxembourg (`pharmacies.cns_code`) |
| `timestamptz` | string | Heure ISO 8601 de **préparation de cette requête API** (même instant que le `timestamp` de la signature, ±1 s). **Ne pas** utiliser la date de dernière modification des stocks dans votre système.
| `products` | array | Une ou plusieurs lignes produit |
| `products[].cefip` | string | Code produit CEFIP (les codes à 6 chiffres reçoivent un `0` en tête côté serveur) |
| `products[].price` | number | Prix actuel |
| `products[].stock` | integer | Quantité en stock (nombre entier) |
| `products[].tva` | number | Taux de TVA |

---

## Signing (HMAC-SHA256)

Définissez `timestamp` et `timestamptz` au moment où vous construisez la requête (juste avant la signature et l'envoi). Ne réutilisez pas de timestamps provenant de votre base d'inventaire ou de fichiers d'export.

1. Choisissez `timestamp` = **now**, en **integer Unix seconds** au moment de la préparation de la requête (pas en millisecondes, pas l'heure de dernière modification de l'inventaire).
2. Définissez `timestamptz` dans le JSON body au même instant que `timestamp` (±1 seconde).
3. Sérialisez le JSON body en **une seule string** (les octets exacts que vous enverrez dans le HTTP body).
4. Construisez le signed payload :

   ```
   signed_payload = "<timestamp>" + "." + <raw JSON body>
   ```

5. Calculez :

   ```
   signature = HMAC_SHA256(hmac_secret, signed_payload) → lowercase hex
   ```

6. Envoyez le header :

   ```
   X-Inventory-Signature: timestamp=<timestamp>,signature=<signature>
   ```

### Timestamp rules

| Check | Rule |
|-------|------|
| Header `timestamp` format | Chiffres uniquement, **integer Unix seconds** au moment de la préparation (pas de millisecondes, p. ex. valeurs ≥ `1000000000000`) |
| Request validity window | Le header `timestamp` doit être à **5 minutes (300 secondes)** de l'heure serveur Medelu, dans le passé comme dans le futur |
| Body vs header | `timestamptz` doit être défini à la préparation de la requête et correspondre au header `timestamp` à **±1 seconde** près |

### 5-minute request window (replay protection)

Chaque requête inclut un `timestamp` dans `X-Inventory-Signature`. Il s'agit de l'heure à laquelle la requête a été **signée et envoyée**, pas de la dernière modification de vos données d'inventaire. Medelu le compare à l'heure serveur à l'arrivée de la requête.

- **Allowed:** `timestamp` au plus **5 minutes plus ancien** ou **5 minutes plus récent** que l'heure serveur (300 secondes dans chaque sens).
- **Rejected:** si le timestamp est à plus de 5 minutes dans le passé ou dans le futur, l'API renvoie **`400`** avec le message `Request timestamp outside allowed window`.
- **Implication:** choisissez `timestamp` et `timestamptz` immédiatement avant la signature. Calculez le HMAC et envoyez la requête HTTP tout de suite. Ne mettez pas en file d'attente, ne signez pas par lot et ne réessayez pas plus tard avec le même signed payload.
- **Implication:** vous ne pouvez pas réutiliser un request body et une signature déjà signés ; chaque sync doit être signée à nouveau avec un timestamp actuel.
- **Common mistake:** utiliser le timestamp « last updated » ou d'export de votre système d'inventaire échouera si cet instant date de plus de 5 minutes. Horodatez toujours au moment de l'envoi.

La fenêtre de 5 minutes ne s'applique qu'au header `timestamp`. Le champ `timestamptz` du body n'est pas comparé directement à l'heure serveur. Il doit seulement correspondre au header `timestamp` à ±1 seconde près, et les deux doivent être définis à la préparation de la requête.

**Important:** le HMAC est calculé sur les **exact raw request body bytes**. Les espaces, l'ordre des clés et le formatage des nombres doivent être identiques entre la signature et l'envoi. Un JSON pretty-printed avec des espacements différents fera échouer la vérification de signature.

---

## Responses

Toutes les réponses sont en JSON avec `Content-Type: application/json`.

### Success (`200`)

Lorsqu'au moins un produit est mis à jour :

```json
{
  "success": true,
  "message": "Inventory synced successfully",
  "processed_products": 2
}
```

Lorsque tous les CEFIP de la request payload sont inconnus de Medelu (ignorés silencieusement) ou que le tableau `products` est vide après filtrage :

```json
{
  "success": true,
  "message": "No products to sync",
  "processed_products": 0
}
```

### Error status codes

Chaque ligne correspond à une valeur exacte de `message` renvoyée par l'API.

| HTTP | `message` | When |
|------|-----------|------|
| `400` | `timestamp must be unix seconds (integer, no decimals, not milliseconds)` | Le `timestamp` de `X-Inventory-Signature` n'est pas une seconde Unix entière valide |
| `400` | `Request timestamp outside allowed window` | Le header `timestamp` est à plus de 5 minutes avant ou après l'heure serveur Medelu (souvent dû à un ancien timestamp d'inventaire au lieu de l'heure de préparation) |
| `400` | `Invalid JSON body` | Le request body n'est pas du JSON valide |
| `400` | `Invalid request format. Required: pharmacy_code, timestamptz, products array` | JSON parsé mais champs requis manquants |
| `400` | `timestamptz does not match signature timestamp` | Le `timestamptz` du body ne correspond pas au header `timestamp` à ±1 seconde près |
| `400` | `Invalid product payload` | Un ou plusieurs champs produit invalides ; la réponse inclut un string array `errors` |
| `401` | `unauthorized` | Headers `X-Inventory-System` ou `X-Inventory-Signature` manquants ; signature header mal formé ; système d'inventaire inconnu, désactivé ou mal configuré |
| `401` | `unauthorized: invalid signature` | La signature HMAC ne correspond pas au body et au timestamp |
| `403` | `Pharmacy not authorized for this inventory system` | Le `pharmacy_code` existe mais n'est pas lié à votre système d'inventaire côté Medelu |
| `404` | `Pharmacy with provided CNS code not found` | Aucune pharmacie avec ce `pharmacy_code` |
| `405` | `Method not allowed` | La méthode HTTP n'est pas `POST` |
| `500` | `Error validating products` | Échec serveur lors de la vérification des codes CEFIP |
| `500` | `Error updating inventory` | Échec serveur lors de l'écriture des données d'inventaire |
| `500` | `Internal server error` | Erreur serveur inattendue |

Example `400` with product validation errors :

```json
{
  "success": false,
  "message": "Invalid product payload",
  "errors": [
    "products[0].stock must be a valid whole number"
  ]
}
```

---

## Examples

Remplacez `YOUR_HMAC_SECRET` et `YOUR_INVENTORY_SYSTEM_NAME` par les valeurs communiquées par Medelu.

`timestamp` et `timestamptz` sont définis à **maintenant** lors de l'exécution (heure de préparation de la requête), pas à la date de dernière modification de l'inventaire.

<div class="code-tabs" data-code-tabs>
  <div class="code-tabs__bar" role="tablist" aria-label="Code examples">
    <button type="button" class="code-tabs__tab" role="tab" aria-selected="true" data-tab-target="curl">curl (macOS / Linux)</button>
    <button type="button" class="code-tabs__tab" role="tab" aria-selected="false" data-tab-target="nodejs">Node.js</button>
  </div>

  <div class="code-tabs__panel" role="tabpanel" data-tab-panel="curl">
    <p>On Linux, replace <code>date -u -r "$TIMESTAMP"</code> with <code>date -u -d "@$TIMESTAMP"</code>.</p>
{% highlight bash linenos %}
SECRET='YOUR_HMAC_SECRET'
INVENTORY_SYSTEM='YOUR_INVENTORY_SYSTEM_NAME'
TIMESTAMP=$(date +%s)
TIMESTAMPTZ=$(date -u -r "$TIMESTAMP" '+%Y-%m-%dT%H:%M:%SZ')
BODY=$(printf '{"pharmacy_code":"811111-11","timestamptz":"%s","products":[{"cefip":"0144099","price":9.99,"stock":6,"tva":3.00},{"cefip":"0104906","price":12.45,"stock":56,"tva":17.00}]}' "$TIMESTAMPTZ")
HMAC_SIGNATURE=$(printf '%s.%s' "$TIMESTAMP" "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -i --request POST 'https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync' \
  --header "X-Inventory-System: ${INVENTORY_SYSTEM}" \
  --header "X-Inventory-Signature: timestamp=${TIMESTAMP},signature=${HMAC_SIGNATURE}" \
  --header 'Content-Type: application/json' \
  --data "$BODY"
{% endhighlight %}
  </div>

  <div class="code-tabs__panel" role="tabpanel" data-tab-panel="nodejs" hidden>
    <p>Uses the built-in <code>crypto</code> module. <code>bodyString</code> must be the exact string sent as the request body.</p>
{% highlight javascript linenos %}
import crypto from "node:crypto";
// or const crypto = require("node:crypto");

const HMAC_SECRET = process.env.INVENTORY_HMAC_SECRET; // shared out of band
const INVENTORY_SYSTEM = process.env.INVENTORY_SYSTEM_NAME; // assigned by Medelu
const URL =
  "https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync";

function signInventoryRequest(secret, timestamp, bodyString) {
  const signedPayload = `${timestamp}.${bodyString}`;
  return crypto
    .createHmac("sha256", secret)
    .update(signedPayload, "utf8")
    .digest("hex");
}

async function syncInventory() {
  const timestamp = Math.floor(Date.now() / 1000); // now, when preparing the request
  const timestamptz = new Date(timestamp * 1000).toISOString(); // same instant as timestamp

  const body = {
    pharmacy_code: "811111-11",
    timestamptz,
    products: [
      { cefip: "0144099", price: 9.99, stock: 6, tva: 3.0 },
      { cefip: "0104906", price: 12.45, stock: 56, tva: 17.0 },
    ],
  };

  // Stable JSON: no extra spaces; same string for HMAC and fetch body
  const bodyString = JSON.stringify(body);
  const signature = signInventoryRequest(HMAC_SECRET, timestamp, bodyString);

  const response = await fetch(URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Inventory-System": INVENTORY_SYSTEM,
      "X-Inventory-Signature": `timestamp=${timestamp},signature=${signature}`,
    },
    body: bodyString,
  });

  const result = await response.json();
  console.log(response.status, result);
}

syncInventory().catch(console.error);
{% endhighlight %}
  </div>
</div>
