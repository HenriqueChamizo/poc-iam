# 🔐 POC - Autenticação B2C com .NET + Red Hat SSO + OpenLDAP

Esta prova de conceito (POC) demonstra como realizar autenticação B2C em aplicações ASP.NET utilizando:

- **Red Hat SSO (RH-SSO)** como Identity Provider (IdP)
- **OpenLDAP** como diretório de usuários
- **OpenID Connect (OIDC)** para autenticação federada
- **ASP.NET Core (.NET 9)** como aplicação cliente
- **Docker Compose** para orquestração local

---

## 📦 Tecnologias utilizadas

| Tecnologia        | Descrição                            |
|-------------------|----------------------------------------|
| .NET 9            | Aplicação cliente                     |
| Red Hat SSO       | Identity Provider (baseado no Keycloak) |
| OpenLDAP          | Backend LDAP com os usuários autenticáveis |
| OpenID Connect    | Protocolo de autenticação             |
| Docker Compose    | Orquestração local dos serviços       |

---

## 🐳 Docker Compose

```yaml
version: "3.9"

services:
  keycloak:
    image: registry.redhat.io/rh-sso-7/sso74-openshift-rhel8
    platform: linux/amd64
    container_name: redhat-sso
    environment:
      SSO_ADMIN_USERNAME: admin
      SSO_ADMIN_PASSWORD: admin
      SSO_REALM: b2c-poc
    ports:
      - "8080:8080"

  openldap:
    image: osixia/openldap:1.5.0
    container_name: openldap
    environment:
      - LDAP_ORGANISATION=MinhaEmpresa
      - LDAP_DOMAIN=empresa.local
      - LDAP_ADMIN_PASSWORD=admin
    ports:
      - "389:389"
```

---

## 🚀 Execução

### ✅ 1. Suba os containers

```bash
docker compose up -d
```

---

### ✅ 2. Configure o Realm no RH-SSO

Acesse o painel admin:

> http://localhost:8080/auth  
> Usuário: `admin`  
> Senha: `admin`

1. Crie ou edite o Realm `b2c-poc`
2. Configure a federação de usuários:

#### 🧩 User Federation com OpenLDAP

- **Vendor**: Other
- **Connection URL**: `ldap://openldap:389`
- **Bind DN**: `cn=admin,dc=empresa,dc=local`
- **Bind Credential**: `admin`
- **Users DN**: `ou=users,dc=empresa,dc=local` *(crie via .ldif)*
- **UUID LDAP attribute**: `uid` ou `entryUUID` ✅
- **Edit mode**: `READ_ONLY` (ou `IMPORT`)

> Após salvar, clique em:
> - ✅ **Test Connection**
> - ✅ **Test Authentication**
> - ✅ **Synchronize all users**

---

### ✅ 3. Popular o OpenLDAP com usuários

Crie um arquivo `users.ldif`:

```ldif
dn: ou=users,dc=empresa,dc=local
objectClass: organizationalUnit
ou: users

dn: uid=joao,ou=users,dc=empresa,dc=local
objectClass: inetOrgPerson
cn: João Silva
sn: Silva
uid: joao
userPassword: joao123
```

Execute:

```bash
docker cp users.ldif openldap:/tmp/users.ldif
docker exec -it openldap ldapadd -x -D "cn=admin,dc=empresa,dc=local" -w admin -f /tmp/users.ldif
```

---

### ✅ 4. Configure sua aplicação .NET

No `Program.cs` da sua aplicação .NET:

```csharp
AppContext.SetSwitch("Microsoft.IdentityModel.Logging.IdentityModelEventSource.ShowPII", true);

builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "http://localhost:8080/auth/realms/b2c-poc";
    options.ClientId = "dotnet-app";
    options.ClientSecret = "COPIE_O_SECRET_DO_CLIENT_AQUI";
    options.ResponseType = "code";
    options.RequireHttpsMetadata = false;
    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;

    options.Scope.Add("openid");
    options.Scope.Add("profile");

    options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
    {
        NameClaimType = "preferred_username",
        RoleClaimType = "roles"
    };
});
```

---

### ✅ 5. Teste o login

1. Acesse sua aplicação .NET: `http://localhost:5126`
2. Você será redirecionado para o login do RH-SSO
3. Faça login com um usuário importado do LDAP (ex: `joao / joao123`)
4. Após login, será redirecionado e autenticado com token OIDC

---

## ✅ Recursos implementados

- Autenticação federada via RH-SSO
- Backend LDAP com OpenLDAP
- OpenID Connect com .NET
- Criação de usuários via `.ldif`
- Integração via User Federation

---

## 📚 Referências

- [OpenLDAP Docker](https://github.com/osixia/docker-openldap)
- [Red Hat SSO](https://access.redhat.com/products/red-hat-single-sign-on)
- [Keycloak LDAP Integration](https://www.keycloak.org/docs/latest/server_admin/#_ldap)

---