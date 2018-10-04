Configurar LetsEncrypt para el tile server
==========================================

Configuración de SSL para el servidor https://tile.azr.es

Manual completo aquí: https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-16-04


```shell
# Obtenemos repositorio, actualizamos e instalamos
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-apache

# Configuramos LetsEncryps para tile.azr.es
sudo certbot --apache -d tile.azr.es
```

En principio el certificado se renueva automaticamente vía cron, diariamente. Para comprobarlo puedes utilizar este comando:

```shell
sudo certbot renew --dry-run
```

Esto te modificar el archivo de configuración de apache 000-default.conf, el archivo de configuración lo puedes encontrar en el mismo repositorio.