# Typical problems and solutions

## Unable to connect
```
send_pubkey_test: no mutual signature algorithm
```
GIT_SSH_COMMAND="ssh -vv -oPubkeyAcceptedKeyTypes=+ssh-rsa" git somecommand

