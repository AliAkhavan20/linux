# Kill Active ports

#### Watch port
```
netstat -ntlp | grep LISTEN
```
#### kill used port
```
sudo lsof -t -i tcp:80 -s tcp:listen | sudo xargs kill
```
