Hashicorp Vault
-------------------------------

![image](https://user-images.githubusercontent.com/92161413/230383136-59198751-5a3f-49fe-814f-ac76491d22a3.png)



O Operador external secrete integra-se ao HashiCorp Vault para gerenciamento de segredos.
O KV Secrets Engine é o único suportado por este provedor. Para outros mecanismos de segredos, consulte o Vault Generator.

Examplo:
---------------------------------


Primeiro, crie um SecretStore com um back-end de cofre. Para simplificar, usaremos uma raiz de token estática:

```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://my.vault.server:8200"
      path: "secret"
      # Version is the Vault KV secret engine version.
      # This can be either "v1" or "v2", defaults to "v2"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "vault-token"
          key: "token"
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
data:
  token: cm9vdA== # "root"
```

**NOTA:** No caso de um ClusterSecretStore, certifique-se de fornecer namespace para tokenSecretRef com o namespace do segredo que acabamos de criar.

Em seguida, crie um par k/v simples no caminho secret/foo:

```ruby
vault kv put secret/foo my-value=s3cr3t
```

Pode verificar a versão kv usando o seguinte e verificar a coluna Opções, deve indicar [versão: 2]:
```ruby
vault secrets list -detailed
```
Se você estiver usando a versão: 1, lembre-se de atualizar seu manifesto SecretStore adequadamente.

Agora crie um ExternalSecret que usa o SecretStore acima:
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: example-sync
  data:
  - secretKey: foobar
    remoteRef:
      key: foo
      property: my-value

  # metadataPolicy to fetch all the labels in JSON format
  - secretKey: tags
    remoteRef:
      metadataPolicy: Fetch 
      key: foo

  # metadataPolicy to fetch a specific label (dev) from the source secret
  - secretKey: developer
    remoteRef:
      metadataPolicy: Fetch 
      key: foo
      property: dev

---
# will create a secret with:
kind: Secret
metadata:
  name: example-sync
data:
  foobar: czNjcjN0
```
Lembre-se de que buscar os rótulos com metadataPolicy: Fetch só funciona com o mecanismo KV sercrets versão v2.

**Obtendo valores brutos**
------------------------

Você pode buscar todos os pares chave/valor para um determinado caminho se deixar remoteRef.property vazio. Isso retorna o valor secreto codificado em json para esse caminho.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  # ...
  data:
  - secretKey: foobar
    remoteRef:
      key: /dev/package.json
```

Valores alinhados
--------------------------
O Vault oferece suporte a pares chave/valor aninhados. Você pode especificar uma expressão gjson em **remoteRef.property** para obter um valor aninhado.

Dado o seguinte segredo - suponha que seu caminho seja **/dev/config:**
```ruby
{
  "foo": {
    "nested": {
      "bar": "mysecret"
    }
  }
}
```
Você pode definir o **remoteRef.property** para apontar para a chave aninhada usando uma expressão gjson.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  # ...
  data:
  - secretKey: foobar
    remoteRef:
      key: /dev/config
      property: foo.nested.bar
---
# creates a secret with:
# foobar=mysecret
```

Se você definir **remoteRef.property** como apenas foo, obterá o valor codificado em json dessa propriedade: **{"nested":{"bar":"mysecret"}}**.

Vários valores aninhados
---------------------------

Você pode extrair várias chaves de um segredo aninhado usando **dataFrom**.

Dado o seguinte segredo - assuma que seu caminho é **/dev/config**:
```ruby
{
  "foo": {
    "nested": {
      "bar": "mysecret",
      "baz": "bang"
    }
  }
}
```
Você pode definir o **remoteRef.property** para apontar para a chave aninhada usando uma expressão gjson.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  # ...
  dataFrom:
  - extract:
      key: /dev/config
      property: foo.nested
```
Isso resulta em um segredo com estes valores:
```ruby
bar=mysecret
baz=bang
```
Obtendo vários segredos
----------------------

Você pode extrair vários segredos do cofre da Hashicorp usando **dataFrom.Find**

Atualmente, **dataFrom.Find** permite que os usuários busquem nomes secretos que correspondam a um determinado padrão regexp ou busquem segredos cujas tags **custom_metadata** correspondam a um conjunto predefinido.

>**Atenção**

>A forma como o hashicorp Vault atualmente permite operações LIST é através da existência de um metadado secreto. Se você excluir o segredo, também >precisará excluir os metadados do segredo ou isso fará com que as operações de localização falhem.

Dado o seguinte segredo - assuma que seu caminho é **/dev/config**:
```ruby
{
  "foo": {
    "nested": {
      "bar": "mysecret",
      "baz": "bang"
    }
  }
}
```
Considere também que o segredo a seguir possui os seguintes **custom_metadata**:
```ruby
{
  "environment": "dev",
  "component": "app-1"
}
```
É possível encontrar este segredo por todas as seguintes possibilidades:
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  # ...
  dataFrom:
  - find: #will return every secret with 'dev' in it (including paths)
      name:
        regexp: dev
  - find: #will return every secret matching environment:dev tags from dev/ folder and beyond
      tags:
        environment: dev
```
Irá gerar um segredo com:
```ruby
{
  "dev_config":"{\"foo\":{\"nested\":{\"bar\":\"mysecret\",\"baz\":\"bang\"}}}"
}
```
Atualmente, as operações de **localização** são recursivas em uma determinada pasta do cofre, começando na definição do **provider.path**. Recomenda-se restringir o escopo da pesquisa definindo uma variável **find.path**. Isso também é útil para reduzir automaticamente os nomes de chaves secretas resultantes:
```ruby
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  # ...
  dataFrom:
  - find: #will return every secret from dev/ folder
      path: dev
      name:
        regexp: ".*"
  - find: #will return every secret matching environment:dev tags from dev/ folder
      path: dev
      tags:
        environment: dev
```
Irá gerar um segredo com:
```ruby
{
  "config":"{\"foo\": {\"nested\": {\"bar\": \"mysecret\",\"baz\": \"bang\"}}}"
}
```
Autenticação
---------------
Oferecemos suporte a cinco modos diferentes de autenticação: **baseado em token, appRole, kubernetes-native, ldap e jwt/oidc**, cada um com suas próprias compensações. Dependendo do método de autenticação, você precisa adaptar seu ambiente.

Autenticação baseada em token
----------------------------

Um token estático é armazenado em um **Kind=Secret** e é usado para autenticar com o cofre.

```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: example
spec:
  provider:
    vault:
      server: "https://vault.acme.org"
      path: "secret"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "my-secret"
          key: "vault-token"
```

**NOTA:** No caso de um **ClusterSecretStore**, certifique-se de fornecer o **namespace** em **tokenSecretRef** com o namespace em que o segredo reside.


Exemplo de autenticação AppRole
---------------------------

A autenticação AppRole lê o id secreto de um **Kind=Secret** e usa o **roleId** especificado para adquirir um token temporário para buscar segredos.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: example
spec:
  provider:
    vault:
      server: "https://vault.acme.org"
      path: "secret"
      version: "v2"
      auth:
        # VaultAppRole authenticates with Vault using the
        # App Role auth mechanism
        # https://www.vaultproject.io/docs/auth/approle
        appRole:
          # Path where the App Role authentication backend is mounted
          path: "approle"
          # RoleID configured in the App Role authentication backend
          roleId: "db02de05-fa39-4855-059b-67221c5c2f63"
          # Reference to a key in a K8 Secret that contains the App Role SecretId
          secretRef:
            name: "my-secret"
            key: "secret-id"
```
**NOTA**: No caso de um **ClusterSecretStore**, certifique-se de fornecer **namespace** em **secretRef** com o namespace onde o segredo reside.


Autenticação do Kubernetes
------------------------

**A autenticação nativa do Kubernetes** tem três opções de obtenção de credenciais para cofre:

>1.usando uma conta de serviço jwt referenciada em **serviceAccountRef**
>
>2.usando o jwt de um **Kind=Secret** referenciado pelo **secretRef**
>
>3.usando credenciais transitórias do token da conta de serviço montada no operador de segredos externos

```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: example
spec:
  provider:
    vault:
      server: "https://vault.acme.org"
      path: "secret"
      version: "v2"
      auth:
        # Authenticate against Vault using a Kubernetes ServiceAccount
        # token stored in a Secret.
        # https://www.vaultproject.io/docs/auth/kubernetes
        kubernetes:
          # Path where the Kubernetes authentication backend is mounted in Vault
          mountPath: "kubernetes"
          # A required field containing the Vault Role to assume.
          role: "demo"
          # Optional service account field containing the name
          # of a kubernetes ServiceAccount
          serviceAccountRef:
            name: "my-sa"
          # Optional secret field containing a Kubernetes ServiceAccount JWT
          #  used for authenticating with Vault
          secretRef:
            name: "my-secret"
            key: "vault"
```
**NOTA**: No caso de um **ClusterSecretStore**, certifique-se de fornecer o **namespace** em **serviceAccountRef** ou em **secretRef**, se usado.

Autenticação LDAP
-------------------
**A autenticação LDAP** usa o par nome de usuário/senha para obter um token de acesso. O nome de usuário é armazenado diretamente em um recurso **Kind=SecretStore** ou **Kind=ClusterSecretStore**, a senha é armazenada em um Kind=Secret referenciado por **secretRef**.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: example
spec:
  provider:
    vault:
      server: "https://vault.acme.org"
      path: "secret"
      version: "v2"
      auth:
        # VaultLdap authenticates with Vault using the LDAP auth mechanism
        # https://www.vaultproject.io/docs/auth/ldap
        ldap:
          # Path where the LDAP authentication backend is mounted
          path: "ldap"
          # LDAP username
          username: "username"
          secretRef:
            name: "my-secret"
            key: "ldap-password"
```
**NOTA**: No caso de um **ClusterSecretStore**, certifique-se de fornecer **namespace** em **secretRef** com o namespace onde o segredo reside.

Autenticação JWT/OIDC
-------------------------

**JWT/OIDC** usa um token **JWT** armazenado em um **Kind=Secret** e referenciado pelo **secretRef** ou um token de conta de serviço Kubernetes temporário recuperado por meio da API **TokenRequest**. Opcionalmente, um campo de função pode ser definido em um recurso **Kind=SecretStore** ou **Kind=ClusterSecretStore**.
```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: example
spec:
  provider:
    vault:
      server: "https://vault.acme.org"
      path: "secret"
      version: "v2"
      auth:
        # VaultJwt authenticates with Vault using the JWT/OIDC auth mechanism
        # https://www.vaultproject.io/docs/auth/jwt
        jwt:
          # Path where the JWT authentication backend is mounted
          path: "jwt"
          # JWT role configured in a Vault server, optional.
          role: "vault-jwt-role"

          # Retrieve JWT token from a Kubernetes secret
          secretRef:
            name: "my-secret"
            key: "jwt-token"

          # ... or retrieve a Kubernetes service account token via the `TokenRequest` API
          kubernetesServiceAccountToken:
            serviceAccountRef:
              name: "my-sa"
            # `audiences` defaults to `["vault"]` it not supplied
            audiences:
            - vault
            # `expirationSeconds` defaults to 10 minutes if not supplied
            expirationSeconds: 600
```
**NOTA**: No caso de um **ClusterSecretStore**, certifique-se de fornecer **namespace** em secretRef com o namespace onde o segredo reside.

PushSecret
------------

O Vault oferece suporte aos recursos do PushSecret, que permitem sincronizar uma determinada chave secreta do kubernetes em um segredo do cofre hashicorp. Para isso, espera-se que a chave secreta seja um objeto JSON válido.

Para usar o PushSecret, você precisa conceder permissões de criação, leitura e atualização para o caminho para o qual deseja enviar os segredos. Use-o com cuidado!

Aqui está um exemplo de como configurá-lo:
```ruby
apiVersion: v1
kind: Secret
metadata:
  name: source-secret
  namespace: default
stringData:
  source-key: "{\"foo\":\"bar\"}" # Needs to be a JSON
---
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: pushsecret-example
  namespace: default
spec:
  refreshInterval: 10s # Refresh interval for which push secret will reconcile
  secretStoreRefs: # A list of secret stores to push secrets to
    - name: vault-secretstore
      kind: SecretStore
  selector:
    secret:
      name: source-secret # Source Kubernetes secret to be pushed
  data:
    - match:
        secretKey: source-key # Source Kubernetes secret key containing the vault secret (in JSON format)
        remoteRef:
          remoteKey: vault/secret # path to vault secret. This path is appended with the vault-store path.
```

Vault Enterprise
---------------------

**Nós de Standby de Desempenho e Consistência Eventuais**

Ao usar o Vault Enterprise com nós de espera de desempenho, qualquer seguidor pode lidar com solicitações de leitura imediatamente após a autenticação do provedor. Como o Vault se torna eventualmente consistente nesse modo, essas solicitações podem falhar se o login ainda não tiver se propagado para o estado local de cada servidor.

Abaixo estão duas soluções diferentes para este cenário. Você precisará revisá-los e escolher o mais adequado para seu ambiente e configuração do Vault.

Namespaces do Vault
--------------------

Os **namespaces do Vault** são um recurso empresarial compatível com multilocação. Você pode especificar um namespace de cofre usando a propriedade **namespace** ao definir um SecretStore:
```ruby
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://my.vault.server:8200"
      # See https://www.vaultproject.io/docs/enterprise/namespaces
      namespace: "ns1"
      path: "secret"
      version: "v2"
      auth:
        # ...
```
Leia seus escritos
-----------------

O Vault 1.10.0 e posterior codifica informações no token para detectar o caso em que um servidor está atrasado. Se um servidor do Vault não tiver informações sobre o token fornecido, o Vault retornará um erro 412 para que os clientes saibam que devem tentar novamente.

Um método suportado nas versões Vault 1.7 e posteriores é utilizar o cabeçalho X-Vault-Index retornado em todas as solicitações de gravação (incluindo logins). Repassar esse cabeçalho nas solicitações subsequentes instrui o cliente do Vault a repetir a solicitação até que o servidor tenha um índice maior ou igual ao retornado com a última gravação. Obviamente, porém, isso tem um impacto no desempenho porque a leitura é bloqueada até que o estado local do seguidor seja atualizado.

Encaminhamento inconsistente
--------------------------

O Vault também oferece suporte ao proxy de solicitações inconsistentes para o líder do cluster atual para consistência imediata de leitura após gravação.

O Vault 1.10.0 e posterior oferece suporte a uma configuração de replicação que detecta quando o encaminhamento deve ocorrer e o faz de forma transparente para o cliente.

No Vault 1.7, o encaminhamento pode ser obtido definindo o cabeçalho **X-Vault-Inconsistent** para **forward-active-node**. Por padrão, esse comportamento está desabilitado e deve ser habilitado explicitamente na configuração de replicação do servidor.




























