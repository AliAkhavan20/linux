# Ubuntu Port Redirection
### Install iptables-persistent if not already installed
```
sudo apt-get update
sudo apt-get install iptables-persistent -y
```
### Add the redirection rule
```
sudo iptables -A INPUT -p tcp --dport 443 -j REDIRECT --to-port 80
```
### Save the iptables rules
```
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```
