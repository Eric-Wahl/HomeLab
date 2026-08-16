# Authentik -CT 113

`192.168.1.13` · pve-nas · LXC (script community-scripts) URL : `https://auth.ndd.xyz`

Fournisseur d'identité OIDC pour Immich, Nextcloud et Outline.

## Structure Authentik

Trois objets par service :

- **Provider** — la configuration OAuth2/OIDC (client ID, secret, redirect URIs)
- **Application** — l'entrée visible par l'utilisateur, porte le _slug_
- **Flow** — le parcours d'authentification (mot de passe, 2FA…)

Le **slug de l'application** apparaît dans l'URL de discovery. Il doit correspondre exactement à ce que le service attend.

## URLs

Discovery, propre à chaque application :

```
https://auth.ndd.xyz/application/o/<slug>/.well-known/openid-configuration
```

Endpoints génériques, communs à tous :

```
Authorize : https://auth.ndd.xyz/application/o/authorize/
Token     : https://auth.ndd.xyz/application/o/token/
Userinfo  : https://auth.ndd.xyz/application/o/userinfo/
JWKS      : https://auth.ndd.xyz/application/o/<slug>/jwks/
Logout    : https://auth.ndd.xyz/application/o/<slug>/end-session/
```

Asymétrie à retenir : authorize / token / userinfo sont génériques, discovery / jwks / logout sont scopés par slug.

## Reverse proxy

NPM, hôte `auth.ndd.xyz` :

- Scheme `**https**`, `192.168.1.13:9443` (Authentik sert son propre TLS)
- Websockets activé
- Certificat wildcard, Force SSL, HTTP/2
- Advanced : `proxy_ssl_verify off;`

## Configuration commune des providers

Pour les trois : **Applications → Providers → Create → OAuth2/OpenID Provider**

| Champ              | Valeur                                            |
| ------------------ | ------------------------------------------------- |
| Authorization flow | `default-provider-authorization-implicit-consent` |
| Client type        | Confidential                                      |
| Signing Key        | certificat par défaut                             |

Le consentement implicite évite la page « autoriser cette application ? » à chaque connexion.

Noter le **Client ID** et le **Client Secret** générés — le secret n'est plus affiché intégralement ensuite.

---

## Immich

**Provider** — `Provider for Immich`

Redirect URIs, mode Strict :

```
https://immich.ndd.xyz/auth/login
https://immich.ndd.xyz/user-settings
app.immich:///oauth-callback
```

La troisième est indispensable au fonctionnement des applications mobiles iOS et Android.

**Application** — nom `Immich`, slug `immich`

**Côté Immich** — Administration → Settings → OAuth :

| Champ               | Valeur                                                                       |
| ------------------- | ---------------------------------------------------------------------------- |
| Issuer URL          | `https://auth.ndd.xyz/application/o/immich/.well-known/openid-configuration` |
| Client ID / Secret  | depuis le provider                                                           |
| Scope               | `openid email profile`                                                       |
| Signing algorithm   | `RS256`                                                                      |
| Storage label claim | `preferred_username`                                                         |
| Auto Register       | activé                                                                       |
| Auto Launch         | désactivé                                                                    |

Laisser la connexion par mot de passe active tant que l'OIDC n'est pas validé.

L'e-mail de l'utilisateur Authentik doit correspondre à celui du compte Immich existant. Sinon Auto Register crée un compte vide au lieu d'ouvrir la bibliothèque migrée.

---

## Nextcloud

Nécessite l'application `user_oidc` côté Nextcloud (voir doc Nextcloud).

**Provider** — `Provider for Nextcloud`

Redirect URI, mode Strict :

```
https://nextcloud.ndd.xyz/apps/user_oidc/code
```

**Application** — nom `Nextcloud`, slug `nextcloud`

**Côté Nextcloud** :

```
docker exec --user www-data nextcloud-aio-nextcloud php occ user_oidc:provider Authentik \
  --clientid="<client id>" \
  --clientsecret="<client secret>" \
  --discoveryuri="https://auth.ndd.xyz/application/o/nextcloud/.well-known/openid-configuration" \
  --scope="openid email profile" \
  --mapping-uid=preferred_username
```

Vérifier `provider-1-uniqueUid = 0`, sinon les comptes sont nommés avec un hash.

---

## Outline

**Provider** — `Provider for Outline`

Redirect URI, mode Strict :

```
https://wiki.ndd.xyz/auth/oidc.callback
```

Point avec un **point**, pas un slash : `oidc.callback`.

**Application** — nom `Outline`, slug `outline`

**Côté Outline** — `/opt/outline/.env` :

```
URL=https://wiki.ndd.xyz
OIDC_CLIENT_ID=<client id>
OIDC_CLIENT_SECRET=<client secret>
OIDC_AUTH_URI=https://auth.ndd.xyz/application/o/authorize/
OIDC_TOKEN_URI=https://auth.ndd.xyz/application/o/token/
OIDC_USERINFO_URI=https://auth.ndd.xyz/application/o/userinfo/
OIDC_LOGOUT_URI=https://auth.ndd.xyz/application/o/outline/end-session/
OIDC_USERNAME_CLAIM=preferred_username
OIDC_DISPLAY_NAME=Authentik
OIDC_SCOPES=openid profile email
```

Commenter **toutes** les lignes `SLACK_*` non vides, y compris `SLACK_VERIFICATION_TOKEN` et `SLACK_MESSAGE_ACTIONS` : Outline refuse de démarrer avec `SLACK_APP_ID cannot be used without SLACK_CLIENT_ID`.

```
systemctl restart outline
journalctl -u outline -n 20 --no-pager
```

Chercher `OIDC plugin registered` et l'absence de `Environment configuration is invalid`.

Le premier utilisateur à se connecter devient administrateur du workspace — se connecter avec son propre compte, pas `akadmin`.

---

## Utilisateurs

**Directory → Users → Create**

Ne pas utiliser `akadmin` pour les connexions applicatives : c'est le compte de service d'Authentik, et son `preferred_username` produit des comptes nommés `akadmin` dans les applications.

Champs importants :

- **Username** — devient l'identifiant dans Immich, Nextcloud et Outline via `preferred_username`
- **Email** — doit correspondre aux comptes existants pour que la liaison se fasse au lieu d'une création

Mot de passe : bouton dédié sur la fiche utilisateur après création.

---

## 2FA (TOTP) — à mettre en place

Authentik gère le 2FA par étapes de flow, pas par simple interrupteur.

**1. Étape de configuration**

Flows and Stages → Stages → Create → **Authenticator TOTP Setup Stage**

- Name : `totp-setup`
- Configuration flow : `default-authenticator-totp-setup`

**2. Étape de validation**

Stages → Create → **Authenticator Validation Stage**

- Name : `mfa-validation`
- Device classes : TOTP
- Not configured action : **Configure**, en pointant vers `totp-setup`
- Last validation threshold : `seconds=0` (à chaque connexion) ou `hours=24`

**3. Liaison au flow de connexion**

Flows → `default-authentication-flow` → onglet **Stage Bindings** → Bind existing stage

- Stage : `mfa-validation`
- Order : `30`

Avec « Not configured action = Configure », l'enrôlement se fait à la connexion suivante, sans e-mail. Compatible Authy, Aegis, Google Authenticator, 1Password.

Tester dans un second navigateur avant de fermer sa session admin.

---

## Sauvegarde

Authentik stocke tout en base PostgreSQL — providers, applications, flows, utilisateurs. Sans dump, une perte du conteneur signifie tout reconfigurer.

Identifier le conteneur et la base :

```
docker ps | grep -i postgres
docker exec <conteneur> env | grep -i postgres
```

Puis un dump quotidien vers un emplacement répliqué, sur le modèle d'Immich et Nextcloud.

---

## Points d'attention

- **Slug = URL.** Une application créée avec un slug différent de celui attendu donne un 404 sur la discovery, sans autre indice.
