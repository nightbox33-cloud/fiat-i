# Auditoría FIAT (fiatdao/fiat) — resultado

**Objeto:** `src/` completo del repo `nightbox33-cloud/fiat-i` (head `084693a`).
**Integridad:** `diff -rq` contra `fiatdao/fiat@main` → **0 diferencias** en `src/` (no hay backdoor inyectado; todo lo que sigue es código upstream real, byte a byte).
**Método:** lectura de las 46 unidades monetarias (core, 7 adaptadores de vault, 4 subastas, 9 guards, 17 actions, PRBProxy) + ejecución real. No hubo `forge` disponible, así que compilé los `.sol` del repo con `solc-js 0.8.13` y ejecuté las trazas en un EVM real (`@ethereumjs/vm 6.5.0`), sin `deal`, sin mocks de token, sin atajos: despliegue de `Codex`, `Collybus`, `Vault20`, `Publican`, `Limes`, `Aer`, `SurplusAuction`, `DebtAuction`, `CollateralAuction`, `Tenebrae`, `FIAT`, `Moneta`. Scripts: `/home/user/.audit/t3.js`, `t5.js`, `t6.js`, `t8.js`.

---

# 1. CRÍTICO — `Tenebrae.skipAuction()` no comprueba `live == 0`: bucle `liquidate → skipAuction` que acuña FIAT sin respaldo desde el crédito de cualquiera

## 1.1 El defecto concreto (no es una opinión sobre diseño, es una asimetría en el bytecode)

`src/core/Tenebrae.sol`:

```solidity
// l.241  — shutdown-only en la documentación ("Has to be performed before global debt is fixed")
function skipAuction(address vault, uint256 auctionId) external override {
    if (debt != 0) revert Tenebrae__skipAuction_debtNotZero();     // <-- única guarda
    ...
// l.270
function offsetPosition(address vault, uint256 tokenId, address user) external override {
    if (debt != 0) revert Tenebrae__offsetPosition_debtNotZero();  // <-- única guarda
```

`debt` **sólo** se escribe en `fixGlobalDebt()` (l.324), y `fixGlobalDebt()` empieza con `if (live != 0) revert Tenebrae__fixGlobalDebt_stillLive();` (l.320). Es decir: `debt == 0` **siempre** mientras el sistema está vivo ⇒ `debt != 0` **no** es una guarda de estado, y `skipAuction`/`offsetPosition` quedan ejecutables en cualquier momento.

Prueba de que la omisión es un descuido y no un diseño: los dos hermanos de la misma fase sí tienen la guarda —
`closePosition` l.307 `if (live != 0) revert Tenebrae__closePosition_stillLive();` y `fixGlobalDebt` l.320. No existe ni el error `Tenebrae__skipAuction_stillLive()` ni `..._offsetPosition_stillLive()` en el contrato (l.90-105).

Ninguna de las dos funciones tiene `checkCaller`, ni usa `msg.sender` como permiso: **cualquiera**.

## 1.2 La cadena monetaria

```
Limes.liquidate(vault, tokenId, position, keeper)            [permissionless, l.169]
  └─ codex.confiscateCollateralAndDebt(...)                  → el colateral de la posición pasa al balance de la subasta
  └─ aer.queueDebt(due)
  └─ CollateralAuction.startAuction(..., keeper)             [checkCaller = Limes]
        └─ tip = flatTip + wmul(debt, feeTip)
           codex.createUnbackedDebt(aer, keeper, tip)        [CollateralAuction.sol l.315-321]
                                                             → credit[keeper] += tip, unbackedDebt[aer] += tip, globalDebt += tip
Tenebrae.skipAuction(vault, auctionId)                       [permissionless, l.241]
  └─ codex.createUnbackedDebt(aer, aer, debt_)               l.249
  └─ collateralAuction.cancelAuction(auctionId)              l.250 → devuelve el colateral a Tenebrae y resta debtOnAuction en Limes
  └─ codex.confiscateCollateralAndDebt(..., +collateralToSell, +normalDebt)   l.253
        → la posición RECUPERA el colateral y una deuda de debt_/rate (¡con el recargo de liquidación 1,1×!)
        → unbackedDebt[aer] vuelve a su sitio, pero **globalDebt NO se reduce**
Moneta.exit(attacker, credit)                                [permissionless] → FIAT real, 1:1
```

Dos consecuencias, ambas verificadas en EVM:

1. **La posición queda MÁS líquidable que antes** (deuda × 1,1 por ciclo, mismo colateral) ⇒ el bucle no se agota y no cuesta nada: el atacante no compra nada en la subasta, sólo la cancela.
2. **Cada ciclo regala `tip` al `keeper` que el atacante eligió** y lo paga con `createUnbackedDebt` (emisión sin respaldo). `keeper` es un parámetro de `Limes.liquidate` elegido por el llamante.

`skipAuction` es la única pieza que hace el bucle gratis: sin ella, tras liquidar, la posición queda sin colateral y sin deuda (no reintentable) y recuperar el colateral exige pagarlo en la subasta. Con la guarda `live == 0` que le falta, el bucle es imposible: `Limes.liquidate` exige `limes.live == 1` (l.175) y `Tenebrae.lock()` pone `limes.live = 0`.

## 1.3 PoC A — atacante con posición propia (140 oro de colateral, 100 FIAT prestados)

Parámetros del despliegue de referencia (los que usan los propios tests del repo: `flatTip = 100e18`, `feeTip = 0.02e18`, `liquidationRatio 1.3`, `liquidationPenalty 1.1`, `maxDebtOnAuction 25000e18`, spot = 1).

```
start   : position(coll=140, debt=100.00) FIAT=100  | globalDebt=1100   FIAT.supply=100
cycle 0 : auction#1 debt=125.2711 | skipAuction OK | FIAT 100        -> 202.505 (+102.505)
cycle 1 : auction#2 debt=137.7982 | skipAuction OK | FIAT 202.505    -> 305.261 (+102.756)
cycle 2 : auction#3 debt=151.5781 | skipAuction OK | FIAT 305.261    -> 408.293 (+103.032)
cycle 3 : auction#4 debt=166.7359 | skipAuction OK | FIAT 408.293    -> 511.628 (+103.335)
cycle 4 : auction#5 debt=183.4094 | skipAuction OK | FIAT 511.628    -> 615.296 (+103.668)
cycle 5 : auction#6 debt=201.7504 | skipAuction OK | FIAT 615.296    -> 719.331 (+104.035)
printed : position(coll=140, debt=201.75) FIAT=719.33 | globalDebt=2838.59 FIAT.supply=719.33
final   : liquida una vez más y RECOMPRA su propio colateral en la subasta con el dinero impreso
bought  : debt=0   FIAT=673.77  credit=10  codex collateral balance=140
after vault.exit -> attacker ERC20 gold = 140
```

**Balance del atacante: pone 140 de oro, sale con 140 de oro + 673,77 FIAT y deuda 0.** 619,33 FIAT acuñados de la nada, y el colateral sigue siendo suyo. El FIAT emitido se cobra contra el colateral del tercero (`bob`, 2000 de oro) al que además le han subido la deuda.

## 1.4 PoC B — atacante SIN capital y SIN posición: 0 → 355,12 de oro del tercero

`eve` no tiene colateral, ni posición, ni rol. Sólo cicla la posición de `bob` (2000 oro / 1000 deuda) con `keeper = eve`:

```
cycle 0: eve FIAT=140.09 | bob coll=2000 debt=2004.33 | globalDebt=4148.75  | supply=240.09
cycle 1: eve FIAT=284.18 | bob coll=2000 debt=2204.76 | globalDebt=6497.61  | supply=384.18
cycle 2: eve FIAT=432.69 | bob coll=2000 debt=2425.24 | globalDebt=9071.35  | supply=532.69
```

Después el shutdown normal (`lock`, `offsetPosition`, `fixGlobalDebt`), `eve` convierte su FIAT fabricado en crédito y redime:

```
redemptionPrice = 0.8207246236723267 oro/FIAT
eve redeem of fabricated FIAT -> OK
*** eve received GOLD: 355.1166184084765 (started with 0) ***
bob gold left in codex = 0 | bob credit = 1000   (sus 1000 de crédito sólo compran 820 de oro, no 2000)
```

**eve: 0 → 355,12 oro. bob: 2000 de colateral → 820 de valor de rescate.** Delta positivo, fondos de un tercero, cero privilegios, 6 transacciones.

## 1.5 PoC C — sin capital, 8 ciclos (escala)

```
eve (sin capital, sin rol) termina con FIAT: 1258.4260137829372
bob coll=2000 (intacto) pero deuda inflada a 3905.87
globalDebt: 2100 -> 26184.06   FIAT.supply: 1358.43
```

`globalDebt` es el **denominador** de `Tenebrae.redemptionPrice` (`wdiv(sub(collateral,lost), wmul(debt, WAD))`, l.217-221) y el valor que fija `fixGlobalDebt` (l.324). Por eso cada ciclo, además de imprimir FIAT, **licúa el precio de rescate de todos los tenedores de FIAT** (con 3 ciclos ya cae a 0,82; con 8, el denominador es 12× el real). Además cada `skipAuction` deja `credit[aer] > 0`, lo que hace insatisfacible la precondición `codex.credit(aer) == 0` de `fixGlobalDebt` salvo que el guardián drene el superávit (`Aer.transferCredit`/`settleDebtWithSurplus`), y ese drenaje consume el `unbackedDebt` que `redeem` necesita ⇒ el shutdown queda en un estado donde **no se puede rescatar**.

## 1.6 Veredicto

- **¿Vulnerable ahora? SÍ.** `skipAuction` (l.241) y `offsetPosition` (l.270) admiten llamada de cualquiera con `live == 1`; probado en EVM.
- **¿Drena a terceros? SÍ.** 355,12 de oro del colateral de `bob` a `eve` (PoC B), y 619,33 FIAT sin respaldo en el PoC A cuyo único respaldo es el mismo pool de colateral.
- **¿Delta > 0 para el atacante? SÍ**, con capital cero (PoC B/C).
- **Direcciones afectadas:** `Tenebrae`, `Limes`, `CollateralAuction`/`NoLossCollateralAuction`, `Aer`, `Codex`, `Moneta`, todos desplegados y cableados entre sí (si no estuvieran cableados, el propio shutdown no funcionaría: `cancelAuction` exige que la subasta tenga a `Tenebrae` como caller, y `redeem` exige `aer` y `codex`).
- **Condición necesaria para la emisión:** `flatTip > 0 || feeTip > 0` en el `CollateralAuction` de ese vault. Los tests del repo configuran `100 ether` y `0.02 ether` (`CollateralAuction.t.sol:477-478, 528-529`; `NoLossCollateralAuction.t.sol:463-464, 514-515`). **Si ambos fuesen 0**, el bucle sigue siendo explotable en su otra mitad: cada ciclo multiplica por 1,1 la deuda de la víctima, mete `debt_` de deuda fantasma en `globalDebt` (verificado: +2048,7 por el primer ciclo) y hunde el precio de rescate de todos los FIAT.
- **Hecho que mataría la afirmación si fuese al revés:** que `Tenebrae.skipAuction` revirtiera con `live == 1`. No lo hace: es el único punto de la cadena que lo impide y está ausente.

---

# 2. ALTO — `Tenebrae.offsetPosition()` ejecutable en vivo: cierre forzado y congelación del colateral de CUALQUIER posición

Mismo defecto de guarda (l.270), explotado por un tercero arbitrario (`eve`) sobre una posición ajena y sana:

```
[live] codex.live=1 tenebrae.live=1
[victim] collateral=140 debt=100
[A] tenebrae.offsetPosition(victim) by eve while LIVE -> SUCCESS
    victim position after: collateral=40 debt=0
    tenebrae codex collateral balance  = 100        # 100 de oro secuestrados y congelados
    aer unbackedDebt                   = 100
    tenebrae.normalDebtByTokenId       = 100
    tenebrae.lostCollateral            = 0
    redemptionPrice now                = 0          # mientras live no hay rescate posible
```

Efecto: cualquier dirección puede cerrar por la fuerza la posición de cualquiera, en cualquier momento, al precio de contado y sin penalización; el colateral va a `codex.balances[vault][tokenId][tenebrae]`, donde queda **congelado** (sólo sale por `redeem`, que exige `debt != 0`, es decir, shutdown completado), y la deuda se convierte en `unbackedDebt[aer]`. La víctima pierde el control de su posición (no puede reponer colateral ni cancelar la deuda) y, si es solvente, pierde el upside; se le acredita `normalDebtByTokenId` para un rescate que aún no puede ejecutar.

- **¿Vulnerable ahora? SÍ** (probado). **¿Drena a terceros? NO directamente:** el atacante no puede retirar el colateral confiscado — no tiene crédito y `redemptionPrice` es 0 mientras el sistema está vivo (probado: `redeem` revierte). **Hecho que mata la calificación de crítico:** el atacante no obtiene ni un token por esta vía; es DoS/confiscación, no robo.
- Se convierte en robo sólo combinado con #1 (que le da el crédito con el que rescatar el botín), y así está explotado en el PoC B.

---

# 3. Rutas evaluadas y descartadas (con el hecho concreto que las mata)

| Ruta | Qué se comprobó | Hecho que mata la afirmación |
|---|---|---|
| `Flash.flashLoan` (mint de FIAT, ERC3156) | `createUnbackedDebt(Flash,Flash,amount)` → `moneta.exit` acuña al receiver → `transferFrom(receiver,Flash)` → `moneta.enter` → `settleUnbackedDebt` | El neto es cero obligatorio: sin el `transferFrom` de la devolución la tx revierte; `moneta.exit` exige `delegates[Flash][Moneta]`, que el constructor fija, y `Moneta` sólo mueve su propio crédito (`Codex.transferCredit` ve `msg.sender == Moneta`). No queda FIAT ni crédito en el receiver. |
| `Flash.creditFlashLoan` | Crédito al receiver, `settleUnbackedDebt(amount)` al final | El callback debe devolver el crédito (`transferCredit(receiver,Flash,amount)` desde el propio receiver) y `settleUnbackedDebt` lo consume entero; si no, revierte por underflow en `unbackedDebt[Flash]`. |
| `LeverSPTActions.onFlashLoan` / `onCreditFlashLoan` (no validan `initiator`) | `flash.flashLoan` es permissionless y `data` es libre | Toda la autoridad que necesitan viene de `delegates[position][ACTION]`, que el **propio proxy** concede y revoca en la misma tx (`codex.grantDelegate(self)` l.214 / `revokeDelegate(self)` l.234). Una llamada directa al action escribe sobre el balance/crédito del **propio action** (que no tiene inventario) y sobre `position` sólo si ésta delegó en el action. Sin delegado, `Codex__modifyCollateralAndDebt_notAllowedSender`. |
| `VaultActions.modifyCollateralAndDebt` público | Se ejecuta bajo `delegatecall` del PRBProxy: `address(this)` = proxy | Llamado directamente, `address(this)` es el action: mueve el colateral/crédito del action, no el de nadie. A través del proxy exige `proxy.execute`, que PRBProxy restringe a `owner`/envoy. |
| `VaultSY.unwrap(bondId,to,amount)` regala el resto (`< principalFloor`) | `if (sub(bond.principal, share) < principalFloor) share = bond.principal;` | El regalo sólo se activa cuando el que quema posee prácticamente todo el supply del bond (si no, `_burn` revierte por saldo), y el excedente está acotado por `principalFloor` (parámetro de gobernanza). No hay camino para que un tercero se lleve principal ajeno. |
| `VaultSY` decimales | `unwrap` devuelve unidades 1e18 (`amount*principal/totalSupply`) | Correcto sólo si `underlierScale == 1e18`; el underlier vivo es `maDAI` (18 decimales). Con un underlier de 6 decimales el adapter sería inconsistente, pero eso es "si listan otro activo" y está fuera. |
| `SurplusAuction.startAuction(creditToSell, bid)` con `bid > 0` | `closeAuction` quema `auctions[id].bid` del balance del contrato, que sólo acumula `bid - initialBid` | `Aer.startSurplusAuction` pasa siempre `bid = 0` (`surplusAuction.startAuction(surplusAuctionSellSize, 0)`), y `startAuction` es `checkCaller` (sólo Aer). No alcanzable. |
| `SurplusAuction.submitBid` / `closeAuction` (permissionless) | El pagador paga al postor anterior y sólo el incremento al contrato; el crédito se entrega al ganador y se quema el token | Es Flap de DSS línea a línea; el contrato siempre retiene exactamente `bid` (con el bid inicial 0) y el crédito se escrow-ea en `startAuction`. Sin desvío de fondos de terceros. |
| `DebtAuction.submitBid/closeAuction/cancelAuction` | Pago de crédito al postor anterior, `settleAuctionedDebt(min(bid,debtOnAuction))` | El flujo es Flop de DSS; `closeAuction` sólo transfiere tokens que el contrato ya tiene (dotación de tesorería) y no toca crédito ajeno. |
| `NoLossCollateralAuction.takeCollateral` permite `owe > debt` | "el usuario pagaría una prima al recipient" | La prima va íntegra a `aer` (`transferCredit(msg.sender, aer, owe)`), no al atacante; el precio tiene suelo `floorPrice = debt/collateralToSell` y `done` se activa justo en el suelo ⇒ nunca vende por debajo de la deuda. |
| `NoLossCollateralAuction.status()` (`price = max(floor, calculator.price)`) | Suelo dinámico invariante por compra parcial (`debt/coll` se conserva) | Verificado algebraicamente: comprar `owe/price` deja la misma relación; no hay ventana para comprar bajo el suelo. |
| `CollateralAuction.status/takeCollateral` | `done` si `elapsed > maxAuctionDuration || price/startPrice < maxDiscount` | Requisitos de temporizador/calculadora del propio vault; no hay parámetro del llamante que los mueva. |
| `Limes.liquidate` (`deltaCollateral = mul(collateral, deltaNormalDebt) / normalDebt` dentro de `unchecked`) | `mul` no comprueba overflow | Haría falta `collateral * deltaNormalDebt > 2^256` (~1e77 unidades con 18 decimales); ningún vault vivo alcanza eso. |
| `Codex.wmul` con `int256` (redondeo hacia cero al devolver deuda) | `deltaDebt = wmul(rate, deltaNormalDebt)` trunca | Diferencia < 1 wei por operación; sin camino para extraer valor significativo ni para cruzar un mínimo vivo (`debtFloor`, `auctionDebtFloor`, quórum). |
| `CodexGuard.setDebtCeiling(vault)` (permissionless) | Sube el techo del vault sólo hasta `min(debt + maxDelta, maxDebtCeiling)`, respeta `delay` desde `lastInc`, y ajusta el techo global con el mismo delta | El incremento global queda acotado por el mismo delta ⇒ no se puede prestar por encima del techo global; y `setDebtCeilingAdjuster` es `isSenatus`. |
| Resto de guards (`Aer`, `Auction`, `Limes`, `Collybus`, `Publican`, `Vault`) | Mapa de modificadores | Todos los setters son `isGuardian`/`isSenatus`/`isDelayed`; el único método permissionless es `Delayed.execute(target,data,eta)`, que exige un `keccak256(abi.encode(target,data,eta))` previamente encolado con `eta >= now + delay`. |
| `PRBProxy` como trampolín de llamadas arbitrarias | `execute` / `setPermission` / `transferOwnership` | `execute` exige `msg.sender == owner` o `permissions[msg.sender][target][selector]`; no hay `fallback`. No sirve para llamar a `Codex` en nombre de un proxy ajeno. |
| `Moneta.enter/exit` y `FIAT` | `enter` quema FIAT del que llama y mueve crédito **propio** de Moneta; `exit` exige `delegates[user][Moneta]` | No hay función que acuñe FIAT sin quemar crédito del llamante (`exit` transfiere el crédito de `msg.sender` a Moneta antes de `fiat.mint`). La única emisión sin crédito previo es la de #1. |
| `Aer.settleAuctionedDebt` / `unqueueDebt` (permissionless) | `settleUnbackedDebt` consume crédito **de Aer** y `debtOnAuction` | Sólo puede quemar el crédito que Aer ya tiene y hasta `debtOnAuction`; no mueve un token a favor del llamante. |
| `Tenebrae.redeem` (reparto pro-rata) | `redemptionPrice` por (vault, tokenId) con denominador `globalDebt` global | No es explotable por sí solo: quien redime aporta crédito 1:1 y recibe `credit*price`; el desequilibrio sólo aparece si alguien infla `globalDebt` — que es exactamente lo que hace #1. |
| `VaultFC.redeemAndExit` (`uint88(cTokensOwed)`) | Cast truncante a uint88 | Haría falta `cTokensOwed >= 2^88` (~3e26 en unidades de 8 decimales) y en ese caso el atacante recibiría **menos**, no más. |

---

# 4. Resumen

- **1 crítico real:** `Tenebrae.skipAuction()` sin guarda de estado ⇒ bucle `Limes.liquidate → Tenebrae.skipAuction` permissionless que (a) paga el `tip` de keeper al atacante en cada vuelta con `createUnbackedDebt`, (b) devuelve el colateral a la posición con la deuda multiplicada por la penalización, dejándola igual de líquidable, y (c) infla `globalDebt` (denominador del rescate) sin activo que lo respalde. PoC: 0 → 355,12 de oro de un tercero; 619,33 FIAT impresos con 140 de colateral propio; `globalDebt` ×12 en 8 ciclos.
- **1 alto con el mismo origen:** `Tenebrae.offsetPosition()` en vivo ⇒ confiscación y congelación del colateral de cualquier posición por cualquiera, sin beneficio directo para el llamante.
- **Arreglo:** añadir a `skipAuction` y a `offsetPosition` la misma guarda que ya tienen `closePosition` y `fixGlobalDebt` (`if (live != 0) revert ..._stillLive();`), y no usar `debt != 0` como sustituto de estado.
