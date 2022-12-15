# handle-secrets-with-kustomize

## Prerequisites

- yq command line processor utility, [get it here.](https://github.com/mikefarah/yq/releases)
- kustomize, oc , or kubectl cli tool. 
- kustomize-sopssecretgenerator - a plugin that decrypts on the fly the secrets  before creating the secrets during the kustomize build process, installation instructions [here](https://github.com/goabout/kustomize-sopssecretgenerator#installation).
  This tutorial assumes using this plugin, but you can easily introduce your own plugin that will integrate with kustomize, and only change the crds in [secrets directory](./secrets) to crds of your own plugin.
Example for Crd of the current plugin:
```yaml
apiVersion: goabout.com/v1beta1
kind: SopsSecretGenerator
metadata:
  name: kustomize-secrets-generation
disableNameSuffixHash: true
envs:
  - secrets.env

```
- Gnu-Privacy guard(GPG) for creating private and public keys, and Mozilla sops for encrypting and decrypting , get [GPG here](https://gnupg.org/download/), [Mozilla sops Here](https://github.com/mozilla/sops/releases) , or install it using your
  package manager, for example:
```shell
sudo yum install gpg
sudo yum install sops
```

## Procedure 

1. Create a key pair(private and public) using GPG(with no passphrase):
```shell
gpg --batch --passphrase '' --quick-gen-key KustomizeDemo default default
```
output:
```shell
gpg: key BD3A044912B418E2 marked as ultimately trusted
gpg: revocation certificate stored as '/home/zgrinber/.gnupg/openpgp-revocs.d/2E6456AEB08601FF886147B9BD3A044912B418E2.rev'
```

2. Create a sample environment fileS to contain keys:values with secrets values:
```shell
cat > secrets/secrets.env << EOF 
user=myUser
password=myPassword
DBAddress=mySecretDBAddress
token=mySecretToken
EOF

cat > secrets/pull-secret.env << EOF
.dockerconfigjson={"auths":{"quay.io":{"username":"youruser","password":"yourpassword","auth":"eW91cnVzZXI6eW91cnBhc3N3b3Jk"}}}
EOF
```

3. Encrypt in place the files using the public key created in section 1:
```shell
sops -e -i --pgp `gpg --list-keys "KustomizeDemo" | grep pub -A1 | grep -E '^ ' | awk '{print $1}'` secrets/secrets.env
sops -e -i --pgp `gpg --list-keys "KustomizeDemo" | grep pub -A1 | grep -E '^ ' | awk '{print $1}'` secrets/pull-secret.env

```

4. check kustomize integration with sopsSecretsGenerator:
```shell
oc kustomize --enable-alpha-plugins .
```

6. Define alias to simplify kustomize build command with sops secrets plugin:
```shell
alias build-kustomize-with-secrets="oc kustomize --enable-alpha-plugins ."
```

7. Validate that secrets were decrypted correctly on the fly:
```shell
for SECRET in DBAddress password token user ; do build-kustomize-with-secrets | yq eval '.data.'$SECRET'' |  grep -v -E 'null|---' | base64 -d | awk '{print "'$SECRET'""=" $1}'  ; done
build-kustomize-with-secrets | yq eval '.data.".dockerconfigjson"' | grep -v -E 'null|---' | base64 -d
 
```
Output:
```shell
DBAddress=mySecretDBAddress
password=myPassword
token=mySecretToken
user=myUser
{"auths":{"quay.io":{"username":"youruser","password":"yourpassword","auth":"eW91cnVzZXI6eW91cnBhc3N3b3Jk"}}}
```

