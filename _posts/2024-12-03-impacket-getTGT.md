---
categories: [Tools,impacket]
tags: [getTGT,impacket,impacket-getTGT,getTGT.py,kerberos,TGT]
---

## About

Impacket’s `getTGT.py` uses a valid user’s credentials to request Kerberos tickets in order to access any service or machine where that user has permissions.

Credentials do not necessarily mean that you have got your hands on to somebody's password. NTLM hashes can be used too :)

If you are using Kali linux, impacket tools should already be installed. If not it can be installed using the following command-

```shell
sudo apt install python3-impacket
```

## **Command Reference:**
### 1. With user's NTLM hashes

```shell
python3 getTGT.py lab.lcl/tony -dc-ip 10.10.10.1 -hashes :2a3de7fe356ee524cc9f3d579f2e0aa7
```
### 2. With username and password

```shell
python3 getTGT.py lab.lcl/tony:'P@$$w0rD' -dc-ip 10.10.10.1
```
> You can use DNS Name instead of IP Address for the `-dc-ip`
{: .prompt-info }


## References:

1. [GitHub Script](https://github.com/SecureAuthCorp/impacket/blob/master/examples/getTGT.py)

2. [Kerberos (II): How to attack Kerberos?
](https://www.tarlogic.com/en/blog/how-to-attack-kerberos/)
