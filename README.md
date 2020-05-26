# Easy Let's Encrypt tutorial
Quick tutorial to enable HTTPS on your linux-based webserver, with Let's Encrypt.

A propos de Let’s Encrypt
Let’s encrypt est une initiative Open Source de l’ISRG qui a pour objectifs résumés de proposer à tous, gratuitement, des certificats SSL permettant de sécuriser un site web via https.
Il faut bien avouer que Let’s Encrypt est la petite révolution de ces 2 dernières années. Tout le monde s’y met. Et ce pour 2 raisons principales : Le fait que ce soit simple ET gratuit, et surtout depuis que Google pousse pour un web sécurisé.
Pour exemple, l’évolution de la recherche de « Let’s Encrypt » sur les 5 dernières années :
### Renouveler automatiquement le certificat

Maintenant nous allons voir comment automatiser le renouvellement de notre certificat avec  `CRON`.  
Personnellement je me créé un petit script que je vais placer dans le dossier  `cron.monthly`. Basique. Et « monthly » car comme vu précédemment le certificat ne sera renouvelé que dans les 30 derniers jours. Inutile donc de faire la demande toutes les semaines et encore moins tous les jours.  
Exemple de fichier  `renew-certificate.sh`  :

	#!/bin/bash

	/chemin/vers/certbot-auto renew

Vous pouvez ajouter le flag  `--quiet`  si vous ne souhaitez pas avoir de retour sur la commande.  
Dans tous les cas, pas de panique, comme nous avons ajouté notre adresse mail lors de la création du certificat, s’il y a le moindre problème, vous serez prévenu par email.

## Révoquer un certificat Let’s Encrypt

Dernier sujet : la suppression, la révocation d’un certificat. Si vous n’en n’avez plus besoin.

Nous allons utiliser la commande  `certbot-auto revoke`  en lui passant un certain nombre de paramètres : le certificat à révoquer, le chemin vers la clé privée et le chemin vers le certificat :

	./certbot-auto revoke --domain domain.tld --cert-path /etc/letsencrypt/live/domain.tld --key-path /etc/letsencrypt/live/domain.tld

Votre certificat est révoqué. Mais ce n’est pas suffisant.  
Si vous regardez vos dossiers /etc/letsencrypt/live/, /archive/ et /renewal/, vous trouverez encore les dossiers/fichiers à l’intérieur.  
Pour faire propre, nous allons tout supprimer avec la commande suivante :

	rm -rf /etc/letsencrypt/live/domain.tld/ && rm -rf /etc/letsencrypt/archive/domain.tld/ && rm /etc/letsencrypt/renewal/domain.tld.conf

### Restaurer la configuration Apache

Dernière modification, restaurer la configuration du vhost Apache dans sa configuration initiale et surtout supprimer toutes les références au SSL et aux certificats.  
Pour celaiIl suffit de se référer à la configuration de base plus haut dans cette page. Je vous laisse faire.

**Conclusion**  
J’espère que ce tuto vous sera utile et que vous en aurez appris un peu plus sur le fonctionnement de Let’s Encrypt et globalement l’intégration d’un certificat SSL pour Apache. Car pour cette dernière partie, Let’s Encrypt ou autre, c’est exactement la même chose.  
Si vous avez des remarques ou questions, n’hésitez pas à poster un commentaire.
