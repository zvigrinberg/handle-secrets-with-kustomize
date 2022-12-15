# handle-secrets-with-kustomize

## Prerequisites

- yq - YAML command line processor utility, [get it here.](https://github.com/mikefarah/yq/releases)
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

4. Verify that files were encrypted:
```shell
cat secrets/secrets.env 
user=ENC[AES256_GCM,data:FUYbxD2M,iv:YJJhMFcgFYbA4SznSlLhkboPPJTaBC5Z1i3Tkg80iws=,tag:aEO4BVdHKtdKbzBzXtbdlQ==,type:str]
password=ENC[AES256_GCM,data:jBcshtBm3H7fPg==,iv:i2mlRTLmnsKOcP04xmOpJXQPTEZ1U4hXO3Da/2tKrmU=,tag:O6s9S121WliN62VnQI+yaA==,type:str]
DBAddress=ENC[AES256_GCM,data:JEYzJSQm8LcJLno1/i5C97Y=,iv:+p3oMrUFb+MXSFhaB/IEs/emgfRArY4zcyGUFsAuiVc=,tag:r4509UvafQyx8vJCdU+esQ==,type:str]
token=ENC[AES256_GCM,data:pr8HzKC8Pp9dvLctPw==,iv:zn5Gw350zSpTxvlexuw2LpA6Ro6S1QZu5i0CrZWoccY=,tag:MRGPx3T8pW2MP/NqLnsxEw==,type:str]
sops_lastmodified=2022-12-15T13:39:39Z
sops_pgp__list_0__map_enc=-----BEGIN PGP MESSAGE-----\n\nhQEMA4Ok+CWgSAPmAQf8D+zZhRBdoFU5cpylVip+50hKPasVpjmNiYKNA6uBS5QO\nSVd03lFxEoo9sP0Y7ORpTqFAJptlQVjJwifuM1TLBZEi77R8/RRUq6r70jnIXZus\nClNKlRbIt7wqAMaxkm78LkgOyhRISmLWQvP968vaxJklPbA5aUvV6gK4OQX0R60/\nAZCPC4BeOqMp9uuw5mlmZv+q99iVWnePFzTGbfcI3oByTJ546uGcRhKC8enyMzB4\nzxUHHYhoiEogy1qYBLJQ7MLaXEZbjmrkD11pSCRkAyFWmdnhtPh9Q9tUcqeHXDej\ntAtXVGShyqEwzYtmc2QSydwEbPAxw4kUKqrwH21jDNJeAR0javq11K//ivue3KK+\nozaRfLC/6NxImLm999aloQ5d1/YX2GcVa8OJHi197eeUeWH8XLdNFPu/3oFlRd8m\nQ7Xz4whWFOstbHJ0amHBMyHwpf2qJzwwJV7+SXxpbw==\n=9uu0\n-----END PGP MESSAGE-----\n
sops_unencrypted_suffix=_unencrypted
sops_mac=ENC[AES256_GCM,data:JjYVJJ1cFiGIGWSd68SELtPObuWsxoBEAxIgtcOpLRxiYhT1MK054f+kWsF4cVoLvvuYaOdqgmg4mzF4el/n1YNsYSc75JzIInoUpLCDcKx+toobrczKmYkOgfcx/kFoH2higPsrB91GK0n7bXBSOElRdFZC01+udMHp2TCmBkk=,iv:srsigM+jdDY+yfgCh5iJSXPMvugu88+B977iOMo3thY=,tag:ht03irsorKnVlYtreKpGKg==,type:str]
sops_pgp__list_0__map_created_at=2022-12-15T13:39:38Z
sops_pgp__list_0__map_fp=2E6456AEB08601FF886147B9BD3A044912B418E2
sops_version=3.7.1

# see the fingerprint that identify the key pair used to encrypt and decrypt the content
 grep _fp secrets/secrets.env | awk -F = '{print $2}'
```
Output:
```shell
2E6456AEB08601FF886147B9BD3A044912B418E2
```
5. check kustomize integration with sopsSecretsGenerator:
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

