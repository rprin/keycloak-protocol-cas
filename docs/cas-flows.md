# Flux d'authentification CAS — keycloak-protocol-cas

Documentation des flux implémentés par le plugin (CAS 1.0 / 2.0 / 3.0 + SAML 1.1), avec les paramètres attendus à chaque étape. Tous les détails ci-dessous sont issus du code source (`src/main/java/org/keycloak/protocol/cas/`).

## Vue d'ensemble

Base URL :

```
https://<host>/realms/{realm}/protocol/cas
```

### Endpoints

| Chemin | Méthode | Classe | Rôle |
|---|---|---|---|
| `/login` | GET | `AuthorizationEndpoint` | Démarre l'authentification, émet un ticket `ST-` |
| `/validate` | GET | `ValidateEndpoint` | Validation CAS 1.0 (réponse texte) |
| `/serviceValidate` | GET | `ServiceValidateEndpoint` | Validation CAS 2.0 (XML/JSON) |
| `/p3/serviceValidate` | GET | `ServiceValidateEndpoint` | Validation CAS 3.0 (alias, même classe) |
| `/proxyValidate` | GET | `ProxyValidateEndpoint` | Validation proxy (accepte `ST-` et `PT-`) |
| `/p3/proxyValidate` | GET | `ProxyValidateEndpoint` | Alias CAS 3.0 |
| `/proxy` | GET | `ProxyEndpoint` | Émission de ticket proxy `PT-` |
| `/samlValidate` | POST | `SamlValidateEndpoint` | Validation SAML 1.1 (SOAP) |
| `/logout` | GET | `LogoutEndpoint` | Déconnexion navigateur (front-channel) |

### Identification du client

Le protocole CAS ne transporte **pas de client ID** : le client est résolu à chaque requête en cherchant le premier client du realm avec `protocol=cas` dont les redirect URIs configurées correspondent au paramètre `service` (ou `TARGET`). Conséquences :

- deux clients CAS ne doivent pas partager de redirect URI ;
- un client désactivé → erreur `400 CLIENT_DISABLED` ;
- aucun client correspondant → erreur `400 CLIENT_NOT_FOUND`.

### Tickets

| Préfixe | Nom | Usage unique ? |
|---|---|---|
| `ST-` | Service Ticket | oui (consommé à la validation) |
| `PT-` | Proxy Ticket | oui |
| `PGT-` | Proxy Granting Ticket | non (réutilisable via `/proxy`) |
| `PGTIOU-` | Proxy Granting Ticket IOU | identifiant retourné au callback `pgtUrl` |

Format interne d'un ticket : `{préfixe}{uuid}--{hex(userSessionId)}--{clientUUID}` (le séparateur `.` est aussi accepté en lecture pour compatibilité). Expiration = `accessCodeLifespan` du realm.

### Paramètres communs

| Paramètre | Où utilisé | Requis | Description |
|---|---|---|---|
| `service` | `/login`, `/validate`, `/serviceValidate`, `/logout` | oui (sauf `/logout`) | URL du SP ; validée contre les redirect URIs du client |
| `TARGET` | `/login`, `/samlValidate` | en lieu et place de `service` | Variante SAML 1.1 ; sur `/login`, la réponse utilise `SAMLart=` au lieu de `ticket=` |
| `ticket` | `/validate`, `/serviceValidate`, `/proxyValidate` | oui | Ticket `ST-` (ou `PT-` pour `/proxyValidate`) |
| `pgtUrl` | `/validate`, `/serviceValidate`, `/proxyValidate` | non | Callback proxy : Keycloak appelle `GET {pgtUrl}?pgtIou=...&pgtId=...` et inclut le `PGTIOU-` dans la réponse |
| `renew` | `/login`, tous les `*Validate` | non (flag de présence) | Force une authentification interactive ; à la validation, échoue si la session était issue d'une SSO passive |
| `gateway` | `/login` | non (flag de présence) | Login passif : sans session valide, redirige vers `service` **sans** ticket au lieu d'afficher le formulaire |
| `format` | `/serviceValidate`, `/proxyValidate`, `/proxy` | non | `XML` (défaut) ou `JSON` (insensible à la casse) ; toute autre valeur → `400 INVALID_REQUEST` |
| `targetService` | `/proxy` | oui | Service cible du ticket proxy |
| `pgt` | `/proxy` | oui | Ticket `PGT-` |

Vérifications communes à tous les endpoints : SSL requis si le realm l'impose (`403`), realm actif (`403`), puis résolution du client.

Codes d'erreur CAS possibles dans les réponses de validation (`CASErrorCode`) : `INVALID_REQUEST`, `INVALID_TICKET_SPEC`, `UNAUTHORIZED_SERVICE_PROXY`, `INVALID_PROXY_CALLBACK`, `PROXY_CALLBACK_ERROR`, `INVALID_TICKET`, `INVALID_SERVICE`, `INTERNAL_ERROR`.

---

## Flux 1 — Login (émission de ticket)

```
Navigateur / SP                Keycloak (protocol/cas)
      |
      |  GET /login?service=https://sp/callback[&renew][&gateway]
      |  (SAML 1.1: ...?TARGET=https://sp/callback)
      |------------------------------------------->
      |          1. checkSsl / checkRealm
      |          2. résolution du client (protocol=cas, service ∈ redirect URIs)
      |          3. création d'une auth session -> formulaire de login Keycloak
      |<--------- 302 /realms/{realm}/login?execution=... (flux standard)
      |
      |--- POST identifiants (formulaire Keycloak) ------------>
      |          4. succes -> émission ST-{uuid}--{sess}--{client}
      |<--------- 302 https://sp/callback?ticket=ST-...
      |           (SAML 1.1: ?SAMLart=ST-...)
```

Comportements particuliers :

- `service` manquant → `400` (paramètre manquant) ; client introuvable/désactivé → `400`.
- `renew` présent : l'auth session exige une re-authentification interactive (`requireReauthentication`).
- `gateway` présent sans session valide : `302` vers `service` **sans** ticket (le SP doit traiter l'absence de ticket comme « utilisateur anonyme »).

## Flux 2 — Validation CAS 1.0 (`/validate`)

```
SP   GET /validate?service=https://sp/callback&ticket=ST-...[&pgtUrl=...][&renew]
     --------------------------------------------------------------->
Keycloak  vérifie SSL/realm/client/ticket (usage unique, expiration, même client)
SP   <----- 200 text/plain "yes\n{username}\n"      (succès)
     <----- 400 text/plain "no\n"                   (échec)
```

Réponse brute : `yes\n<utilisateur>\n` en cas de succès, `no\n` sinon. C'est le seul endpoint qui répond en `text/plain`.

## Flux 3 — Validation CAS 2.0 / 3.0 (`/serviceValidate`, `/p3/serviceValidate`)

```
SP   GET /serviceValidate?service=...&ticket=ST-...[&format=XML|JSON][&pgtUrl=...][&renew]
     ---------------------------------------------------------------------->
Keycloak  1. vérifie le ticket (ST- unique, non expiré, client cohérent)
           2. si pgtUrl: appelle GET {pgtUrl}?pgtIou=PGTIOU-...&pgtId=PGT-... (doit répondre 200)
           3. collecte les attributs via les protocol mappers du client
SP   <----- 200 XML ou JSON (selon format, défaut XML)
```

Réponse de succès (XML) :

```xml
<cas:serviceResponse xmlns:cas="http://www.openlabs.org/protocols/cas">
  <cas:authenticationSuccess>
    <cas:user>jdoe</cas:user>
    <cas:proxyGrantingTicket>PGTIOU-...</cas:proxyGrantingTicket>  <!-- si pgtUrl fourni -->
    <cas:attributes>
      <email>jdoe@example.com</email>
      <!-- un élément par mapper configuré sur le client -->
    </cas:attributes>
  </cas:authenticationSuccess>
</cas:serviceResponse>
```

Réponse d'échec (HTTP 400/403) :

```xml
<cas:serviceResponse xmlns:cas="http://www.openlabs.org/protocols/cas">
  <cas:authenticationFailure code="INVALID_TICKET">Code not valid</cas:authenticationFailure>
</cas:serviceResponse>
```

Avec `format=JSON`, la même structure est sérialisée sous `{"serviceResponse": {...}}` (valeurs nulles omises). Les attributs proviennent des mappers de la famille `org.keycloak.protocol.cas.mappers.*` configurés sur le client (user attribute, user property, rôles, claims hardcodés…).

## Flux 4 — Validation proxy (`/proxyValidate`, `/p3/proxyValidate`)

```
SP   GET /proxyValidate?ticket=ST-...|PT-...[&pgtUrl=...][&renew]
     ------------------------------------------------------------>
SP   <----- 200 serviceResponse (même forme que /serviceValidate)
```

Particularité : contrairement aux autres endpoints de validation, **aucun paramètre `service` n'est lu ni vérifié** ici — la cohérence se fait uniquement entre le ticket et le client de session. Accepte indifféremment un `ST-` ou un `PT-`.

## Flux 5 — Émission de ticket proxy (`/proxy`)

```
Proxy CAS   GET /proxy?targetService=https://autre-sp/&pgt=PGT-...
            ----------------------------------------------------->
Keycloak    vérifie le PGT (réutilisable, non consommé)
Proxy CAS   <----- 200 <cas:proxySuccess><cas:proxyTicket>PT-...</cas:proxyTicket></cas:proxySuccess>
              (ou <cas:proxyFailure code="..."> si échec)
```

## Flux 6 — SAML 1.1 (`/samlValidate`)

```
SP   POST /samlValidate?TARGET=https://sp/callback
     Header: Content-Type: text/xml;charset=utf-8
             SOAPAction: http://www.oasis-open.org/committees/security
     Body:   enveloppe SOAP contenant
             samlp:RequestAbstractService
               samlp:AssertionArtifact = ST-...
     ---------------------------------------------------------------->
SP   <----- 200 SOAP SAML 1.1 Response
             - succes: Assertion avec user + attributes
             - echec:  Status (code + raison)
```

Notes :

- le header `SOAPAction` est obligatoire (`http://www.oasis-open.org/committees/security`), sinon `400` ;
- les erreurs reviennent en HTTP 200 avec un statut SAML dans le corps (comportement SOAP) ;
- côté login, demander avec `TARGET=` au lieu de `service=` fait renvoyer le ticket dans `SAMLart=` plutôt que `ticket=`.

## Flux 7 — Logout front-channel (`/logout`)

Requête : `GET /logout?service=<url>` — le paramètre `service` est **optionnel**.

```
Navigateur   GET /logout?service=https://sp/callback
             ---------------------------------------------------------->
Keycloak     1. resolution de la cible de redirection (service -> client CAS)
             2. recherche d'une session active via le cookie d'identite
                ├── aucune session  → 302 service (si cible) / 400 (sinon)
                └── session active  → 3. notes sur la session
                                      4. browserLogout:
                                         - backchannel logout de chaque client session
                                           (POST SLO CAS, voir flux 8)
                                         - expiration des cookies d'identite
                                         5. CASLoginProtocol.finishBrowserLogout:
                                            302 service (si cible) / page info
SP           <----- 302 https://sp/callback          (si service resolu)
            <----- page info "Logout successful"     (sinon)
```

### Étape 1 — Résolution de la cible de redirection

`LogoutEndpoint.checkClient(service)` cherche le premier client `protocol=cas` dont les redirect URIs acceptent `service` (via `RedirectUtils.verifyRedirectUri`) :

- correspondance trouvée → cible = la **valeur exacte de `service`** telle que fournie (rendue absolue si elle était relative) ;
- aucune correspondance → **pas d'erreur, pas de cible** : contrairement à `/login` et `/serviceValidate` (qui renvoient `400`), `/logout` ignore silencieusement un `service` non reconnu ;
- particularité : un client **désactivé** est quand même accepté ici (pas de vérification `isEnabled()`, contrairement aux autres endpoints).

`verifyRedirectUri` peut rejeter un `service` en apparence valide dans ces cas : URI non parsable ; query string contenant `ticket` ou `SAMLart` (protection contre l'injection de paramètres) ; aucune redirect URI configurée ou aucun match (exact ou wildcard `*`) ; schéma non autorisé (ex. `http` alors que seul `https:` est configuré) ; motif de chemin « unsafe » (`../` encodé ou non) bloquant l'usage du wildcard.

### Étape 2 — Recherche de la session active (cookie d'identité)

`AuthenticationManager.authenticateIdentityCookie(...)` ne renvoie une session que si **toutes** ces conditions sont réunies :

- le cookie `KEYCLOAK_IDENTITY` est présent et son jeton passe la vérification (signature, realm émetteur, type) ;
- la session référencée existe encore **et** est valide (pas expirée par le délai d'inactivité ni par la durée de vie maximale du realm) ;
- l'utilisateur est valide (compte actif…).

#### Cas A — Aucune session active

- cible résolue → `302` vers la cible, émis directement par `LogoutEndpoint` (aucun SLO envoyé) ;
- pas de cible → `400 FAILED_LOGOUT`.
- Effet de bord : si la session existait mais avait **expiré**, Keycloak la nettoie pendant cette vérification — ce nettoyage déclenche les **backchannel logouts** (donc les POST SLO CAS, flux 8) avant de répondre.

#### Cas B — Session active

1. Notes posées sur la session : `KEYCLOAK_LOGOUT_PROTOCOL=cas` et, si cible résolue, `CAS_LOGOUT_REDIRECT_URI=<cible>` ;
2. `AuthenticationManager.browserLogout(...)` :
   - **Session issue d'un broker IdP** : si la session a été créée par un identity provider et que l'IdP initiateur ne correspond pas, Keycloak délègue au broker (`keycloakInitiatedBrowserLogout`) ; si le broker renvoie une réponse (typiquement un redirect vers le SLO de l'IdP), elle est retournée **immédiatement** — l'étape 3 est contournée et le redirect `service` CAS est ignoré ;
   - sinon → `finishBrowserLogout` :
     a. backchannel logout de chaque client session de la session utilisateur (pour un client CAS : POST SLO si un service ticket a été émis — flux 8) ;
     b. frontchannel logout pour les clients ayant l'option *Front-channel logout* activée → **non implémenté pour CAS** (`RuntimeException` → HTTP 500) : ne pas activer cette option sur un client CAS ;
     c. expiration des cookies d'identité et « remember me » ;
     d. appel de `CASLoginProtocol.finishBrowserLogout(...)` qui lit la note `CAS_LOGOUT_REDIRECT_URI` :
        - note présente → `302` vers la cible ;
        - note absente → page d'information « Logout successful ».

### Tableau récapitulatif

| Session active | `service` résolu | Résultat |
|---|---|---|
| oui | oui | POST SLO + expiration des cookies, puis `302 service` |
| oui | non / absent | POST SLO + expiration des cookies, puis page « Logout successful » |
| non | oui | `302 service` direct (aucun SLO) |
| non | non / absent | `400 FAILED_LOGOUT` |

Cas particuliers : session brokerisée (retour anticipé possible vers le SLO de l'IdP, redirect `service` ignoré) ; option *Front-channel logout* activée sur un client CAS (erreur 500).

## Flux 8 — Single Logout back-channel (SLO)

Au teardown d'une session utilisateur (déconnexion, expiration, invalidation), Keycloak notifie les SP concernés :

```
Keycloak  pour chaque client session CAS ayant eu un ST valide:
           POST {redirectUri du client}
           Content-Type: application/x-www-form-urlencoded
           logoutRequest=<samlp:LogoutRequest ...>
             saml:NameID       = @NOT_USED@
             samlp:SessionIndex = ST-...   (le dernier service ticket)
           (suit un redirect 302 si l'URL gagne un slash final)
```

Conditions et limites :

- la notification n'est envoyée que si `serviceValidate` a été appelé au moins une fois (c'est ce qui stocke le service ticket dans la client session) ;
- la cible est la redirect URI de la client session (la dernière `service` validée) ;
- malgré le format SAML, ce message est défini par la spec CAS SLO et n'a aucun lien avec SAML 2.0 ECP ;
- `frontchannelLogout` (redirect de déconnexion navigateur vers le SP) **n'est pas implémenté** (lève `RuntimeException`).

## Cycle de vie d'un ticket

```
/login (succes)          /serviceValidate             /proxy
     |                         |                          |
     v                         v                          v
ST-{uuid}--{sess}--{client}   ST consomme (supprime      PT-{uuid}--{sess}--{client}
     |                        du store one-shot)            |
     |                         |                            |
     +--- expire apres realm.accessCodeLifespan ------------+
     |
     +--- si pgtUrl a ete fourni: PGT-{uuid} (REUTILISABLE) --+
                                                              v
                                                     /proxy?pgt=PGT-... (n'importe combien de fois)
```

- un `ST-` est **à usage unique** : la première validation le supprime du store ; une seconde tentative → `INVALID_TICKET` ;
- un `PGT-` reste valable jusqu'à expiration et peut servir à émettre plusieurs `PT-` ;
- `renew` à la validation : si le ticket provient d'une session SSO non interactive → `INVALID_TICKET` (« Interactive authentication was requested but not performed ») ;
- un ticket liant un client différent de celui résolu par `service` → `INVALID_SERVICE`.
