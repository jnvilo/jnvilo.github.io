```
sudo apt install python3-acme python3-certbot python3-mock python3-openssl python3-pkg-resources python3-pyparsing python3-zope.interface
```

```
sudo certbot --nginx -d your_domain -d www.your_domain
```

```
sudo certbot renew --dry-run
```