# Lab 15 – Cloud Security and Encryption

Encrypt and decrypt a disposable local file with `age`, inspect file permissions, and discuss key management. This is a hands-on cryptography exercise, not a substitute for AWS KMS, Azure Key Vault, Google Cloud KMS, or a security review.

## 1. Create a private lab workspace and sample data

```bash
umask 077
mkdir -p ~/devops-labs/lab15
cd ~/devops-labs/lab15
printf 'This document contains only sample lab data.\n' > sample.txt
chmod 600 sample.txt
ls -l sample.txt
```

## 2. Generate a key and encrypt/decrypt the file

The private key must remain confidential. This disposable lab key is not a production key.

```bash
sudo apt-get update
sudo apt-get install -y age
age-keygen -o key.txt
chmod 600 key.txt
age-keygen -y key.txt > recipient.txt
AGE_RECIPIENT=$(cat recipient.txt)
age --recipient "$AGE_RECIPIENT" --output sample.txt.age sample.txt
file sample.txt.age
age --decrypt --identity key.txt --output recovered.txt sample.txt.age
cmp sample.txt recovered.txt
sha256sum sample.txt recovered.txt
```

The matching hashes confirm that this exercise recovered the original bytes. Encryption does not provide key rotation, identity-based access control, audit logging, or protection if the private key is exposed. Discuss envelope encryption and why production keys belong in a managed KMS/HSM boundary rather than application source.

## 3. Remove plaintext and clean up

```bash
rm -f sample.txt recovered.txt sample.txt.age key.txt recipient.txt
rmdir ~/devops-labs/lab15
```
