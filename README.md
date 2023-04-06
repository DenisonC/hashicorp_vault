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









