# Get a client cert file and save it in linux cert

## Install Keytool
```
apt install openjdk-17-jre-headless
```
## use it to get cert
```
keytool -printcert --sslserver repo.myapp.com:8443 -rfc
```
## cp cert into ca-cert folder
```
sudo apt-get install -y ca-certificates
sudo cp local-ca.crt /usr/local/share/ca-certificates
sudo update-ca-certificate
```