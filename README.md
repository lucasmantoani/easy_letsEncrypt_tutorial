# Easy Let's Encrypt tutorial
Quick tutorial to enable HTTPS on your linux-based webserver, with Let's Encrypt.

## Renouveler un certificat Let’s Encrypt

A ce stade nous avons un certificat Let’s Encrypt installé et notre serveur bien configuré.  
Il faut savoir que la durée de vie du certificat Let’s Encrypt n’est que de 90 jours,  [3 mois](https://www.youtube.com/watch?v=xORRAhci2aU)  donc.  
Nous allons voir les commandes pour le renouveler manuellement et automatiquement.

Pour renouveler votre certificat manuellement il suffit de lancer la commande :

	./certbot-auto renew
