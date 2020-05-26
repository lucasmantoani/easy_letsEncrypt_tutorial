# Easy Let's Encrypt tutorial
Quick tutorial to enable HTTPS on your linux-based webserver, with Let's Encrypt.

A propos de Let’s Encrypt
Let’s encrypt est une initiative Open Source de l’ISRG qui a pour objectifs résumés de proposer à tous, gratuitement, des certificats SSL permettant de sécuriser un site web via https.
Il faut bien avouer que Let’s Encrypt est la petite révolution de ces 2 dernières années. Tout le monde s’y met. Et ce pour 2 raisons principales : Le fait que ce soit simple ET gratuit, et surtout depuis que Google pousse pour un web sécurisé.
Pour exemple, l’évolution de la recherche de « Let’s Encrypt » sur les 5 dernières années :

Evolution recherche Let's Encrypt

Contexte
Nous allons donc voir comment déployer Let’s Encrypt sur notre serveur Apache, mais d’une façon particulière.
En effet, le logiciel Certbot que nous allons utiliser propose divers plugins et assistants, dont pour Apache, qui permettent d’installer automatiquement les certificats, de modifier les configurations d’Apache, etc… Tout cela automatiquement.
Cela conviendra parfaitement au plus grand nombre, il faut bien le dire.
Mais personnellement, j’aime bien : 1/ savoir ce qu’il se passe, 2/ savoir comment cela fonctionne, 3/ garder la main sur le paramétrage.
Et par dessus tout je déteste qu’un logiciel quel qu’il soit, fasse sa vit seul, modifie mes configurations, redémarre les services, etc… D’autant plus sur un serveur en production.
En le faisant manuellement, au moins, en cas de problème, je sais où chercher et où intervenir.

Pour ce tuto je pars du principe que nous avons une configuration Apache classique, et qu’il n’y pas de cas particuliers.
Je suppose aussi que nous avons plusieurs sites hébergés sur notre serveur et qu’un ou plusieurs d’entre eux vont passer du http au https.

Enfin, notez que ce tutoriel est réalisé sur une Debian 7 et Apache 2.2. Cela fonctionne exactement de la même façon sur une Debian 8 et Apache 2.4.

Prérequis
Avoir installé un serveur Apache 2 et qu’il soit fonctionnel avec les mods SSL et rewrite activés.
Pour vérifier s’ils sont actifs exécutez :

root@g:~# apachectl -M
Loaded Modules:
 ...
 rewrite_module (shared)
 ssl_module (shared)
 ...
Syntax OK
Vous voyez en gras les 2 modules chargés.
Si ce n’est pas le cas, exécutez en fonction :

root@g:~# a2enmod ssl 
Enabling module ssl.
See /usr/share/doc/apache2.2-common/README.Debian.gz on how to configure SSL and create self-signed certificates.
To activate the new configuration, you need to run:
  service apache2 restart

root@g:~# a2enmod rewrite 
Enabling module rewrite.
To activate the new configuration, you need to run:
  service apache2 restart

root@g:~# service apache2 restart 
[ ok ] Restarting web server: apache2 ... waiting .
N’oubliez pas de redémarrer Apache une fois que les modules sont mis à jour.

Méthodologie
La mise en place va se faire en 6 grandes étapes :

Installer le programme Certbot qui va générer les certificats
Créer un premier certificat Let’s Encrypt
Modifier la configuration d’un vhost Apache pour y intégrer les certificats
Vérifier la configuration ainsi que la qualité du paramétrage SSL
Apprendre à renouveler un certificat
Apprendre à révoquer un certificat
Installation de Certbot
Pour commencer, on télécharge Certbot et on le rend exécutable :

wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
C’est un simple fichier, stockez le à un endroit approprié. Dans /opt par exemple.
Exécutez-le une première fois « à blanc » pour qu’il télécharge ses dépendances et s’installe.

./certbot-auto
Lorsque ce sera terminé, il vous proposera de commencer le travail de création. Quittez. Nous y reviendrons plus tard.

Les fichiers principaux sont installés. Tout se trouve dans /etc/letsencrypt.
Dans se dossier on trouvera 3 dossiers de stockage pour les certificats :
archive : dossier de base ou sont stockés les certificats
live : dossier des certificats actifs. Des liens symboliques vers le dossier archive.
renewal : dossier qui contient les informations de renouvellement des certificats.

Création du certificat Let’s Encrypt
Lancez la commande suivante en veillant à remplacer les valeurs par les bonnes informations :

./certbot-auto certonly --webroot --webroot-path /srv/www/domain.tld/ --domain domain.tld --domain www.domain.tld --email mon@email.com
Explications :

certonly : on demande la création du certificat uniquement.
--webroot : on utilise le plugin webroot qui se contente d’ajouter des fichiers dans le dossier défini via --webroot-path.
--webroot-path : le chemin de votre « DocumentRoot » Apache. Certbot placera ses fichiers dans $DocumentRoot/.well-known/ pour les tests et vérifications
--domain : le nom de domaine à certifier.
--email : l’adresse qui recevra les notifications de Let’s Encrypt. Principalement pour rappeler de renouveler le certificat le moment venu.

Options de création
Au niveau des paramètres de création, l’essentiel est là. Je vous invite à consulter la doc pour voir les autres options disponibles.
On pourrait éventuellement durcir le certificat en le passant en 4096 bits (2048 bits par défaut) en ajoutant le flag --rsa-key-size 4096 mais c’est un peu disproportionné. C’est vous qui voyez.

Pour revenir sur le flag --domain, attention ! Let’s encrypt ne gère pas à ce jour le Wildcard. Il faut donc penser impérativement à déclarer le domaine ET le sous-domaine « www ». Si vous êtes dans une configuration classique évidemment où votre site est accessible depuis http://domain.tld et http://www.domain.tld.

Les certificats obtenus
Si vous cherchez vos certificats, ils se trouvent dans les dossiers : /etc/letsencrypt/live/domain.tld/
Vous avez pour chaque certificat 4 fichiers :

privkey.pem : La clé privée de votre certificat. A garder confidentielle en toutes circonstances et à ne communiquer à personne quel que soit le prétexte. Vous êtes prévenus !
cert.pem : Le certificat serveur et à préciser pour les versions d’Apache < 2.4.8. Ce qui est notre cas ici.
chain.pem : Les autres certificats, SAUF le certificat serveur. Par exemple les certificats intermédiaires. Là encore pour les versions d’Apache < 2.4.8.
fullchain.pem : Logiquement, l’ensemble des certificats. La concaténation du cert.pem et du chain.pem. A utiliser cette fois-ci pour les versions d’Apache >= 2.4.8.

Le cas des certificats pour différents sous-domaines
Pour une gestion simplifiée de vos certificats, je vous conseille de ne pas intégrer tous vos sous-domaines au même certificat.
Si vous gérez plusieurs sites différents sur des sous-domaines différents, et donc sur des vhost différents : par exemple blog.domain.com et forum.domain.com, à ce moment là vous créer 2 certificats distincts. Comme cela, en cas de modification, déplacement ou suppression de l’un de vos sites, tout est cloisonné et il n’y a pas d’impacts sur les autres certificats.

Intégrer le certificat Let’s Encrypt à Apache
Maintenant que nous avons le certificat, il ne reste plus qu’à l’intégrer au vhost Apache et ainsi servir du contenu en https.
Tout d’abord il faut éditer le vhost afin de déclarer tout le paramétrage nécessaire.
Je suppose ici que nous sommes dans un cas classique où notre site est accessible à la fois depuis domain.com et www.domain.com. Le grand classique.

Modification du vhost
Pour l’instant le vhost ressemble à quelque chose comme ceci :

<VirtualHost *:80>

    ServerName domain.tld
    ServerAlias www.domain.tld
    DocumentRoot /path/to/files

    LogLevel warn
    ErrorLog ${APACHE_LOG_DIR}/domain.tld-error.log
    CustomLog ${APACHE_LOG_DIR}/domain.tld-access.log combined

    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^www\. [NC]
    RewriteRule ^(.*)$ http://www.%{HTTP_HOST}$1 [R=301,L]

    <Directory /path/to/files>
        Options -Indexes -MultiViews
        AllowOverride all
        Order allow,deny
        allow from all
    </Directory>

</VirtualHost>
Ici, le détail de cette configuration importe peu. L’important est de voir que le site est accessible à la fois depuis domain.tld et www.domain.tld, et que si quelqu’un demande http://domain.tld : il y aura une redirection 301 automatique vers le sous-domaine http://www.domain.com.
Nous allons reproduire la même chose en version https. Je vous propose la nouvelle configuration suivante :

<VirtualHost *:80>

    ServerName domain.tld
    ServerAlias www.domain.tld

    RewriteEngine on
    RewriteCond %{HTTPS} !on
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}

</VirtualHost>

<VirtualHost *:443>

    ServerName domain.tld
    ServerAlias www.domain.tld

    DocumentRoot /path/to/files/www.domain.tld

    <Directory /path/to/files>
        Options -Indexes
        AllowOverride all
        Order allow,deny
        allow from all
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/domain.tld/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/domain.tld/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/domain.tld/chain.pem
    SSLProtocol all -SSLv2 -SSLv3
    SSLHonorCipherOrder on
    SSLCompression off
    SSLOptions +StrictRequire
    SSLCipherSuite ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    LogLevel warn
    ErrorLog ${APACHE_LOG_DIR}/www.domain.tld-error.log
    CustomLog ${APACHE_LOG_DIR}/www.domain.tld-access.log combined

</VirtualHost>
On voit maintenant que l’on a 2 vhost dans un vhost ! Un pour le port 80 et le http (<VirtualHost *:80>) et un autre pour le port 443 et le https (<VirtualHost *:443>).
Le premier vhost du port 80 ne fait rien d’autre qu’une redirection automatique de l’http vers le https. SI jamais quelqu’un le demandait.
Mis à part cela, il n’y a rien. Pas de documentRoot, pas de logs, etc… C’est un choix. Etant donné que je souhaite que 100% du trafic passe sur https, toutes les informations complémentaires sont inutiles, à commencer pour les logs.

Ensuite il y a la déclaration du vhost https. Comme vous pouvez le voir, la première partie est identique à la configuration initiale, et il n’y a pas de raison qu’elle soit différente. Par contre, en deuxième partie, il y a toutes les déclarations SSL :

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/domain.tld/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/domain.tld/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/domain.tld/chain.pem
    SSLProtocol all -SSLv2 -SSLv3
    SSLHonorCipherOrder on
    SSLCompression off
    SSLOptions +StrictRequire
    SSLCipherSuite ECDHE-RSA-AES128-GCM-SHA256 [...]
    # Note: It’s also recommended to enable HTTP Strict Transport Security (HSTS)
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Explications :

SSLEngine on : Activation du module SSL
SSLCertificateFile : chemin vers le fichier cert.pem
SSLCertificateKeyFile : chemin vers le fichier privkey.pem
SSLCertificateChainFile : chemin vers le fichier chain.pem
SSLProtocol : Active le support de toutes les versions SSL mais SAUF SSLv2 et v3 qui ne sont plus suffisamment sécurisées.
SSLHonorCipherOrder : Quand activé, le serveur force ses préférences de protocoles et non le navigateur du client.
SSLCompression : Active la compression SSL
SSLOptions +StrictRequire : Exige une connexion SSL stricte
SSLCipherSuite : Liste des algorithmes de chiffrement disponibles
Header always set Strict-Transport-Security : ou HSTS. Indique aux navigateurs que seul le https est supporté et le force en définissant une durée d’application (longue). Objectif : éviter les attaques « Man in the middle ».

Vérification de la configuration du vhost
Reste maintenant à tester notre configuration :

apachectl configtest
Logiquement devrait vous retourner un « Syntax OK ». Si c’est le cas, il faut reload/restart Apache pour qu’il prenne en compte les modifications.

service apache2 restart
Et là, vous pouvez avoir l’erreur suivante :

[....] Reloading web server config: apache2[Sat Oct 08 11:17:49 2016] [warn] _default_ VirtualHost overlap on port 443, the first has precedence.
... waiting [Sat Oct 08 11:17:49 2016] [warn] _default_ VirtualHost overlap on port 443, the first has precedence
ok
Nous venons d’ajouter un nouveau vhost écoutant sur le port 443 mais nous avons aussi le vhost « default-ssl » qui est déjà présent. Donc 2 vhost, 2 sites pour le même port ça ne va pas. Il faut donner des précisions à Apache pour qu’il sache quoi servir.
Etant dans un contexte de serveurs basés sur le nom, il faut ajouter cette instruction au fichier ports.conf, qui ressemble à ceci par défaut :

<IfModule mod_ssl.c>
    # If you add NameVirtualHost *:443 here, you will also have to change
    # the VirtualHost statement in /etc/apache2/sites-available/default-ssl
    # to <VirtualHost *:443>
    # Server Name Indication for SSL named virtual hosts is currently not
    # supported by MSIE on Windows XP.
    Listen 443
</IfModule>
On va lui ajouter l’instruction NameVirtualHost *:443 ce qui nous donnera :

<IfModule mod_ssl.c>
    # If you add NameVirtualHost *:443 here, you will also have to change
    # the VirtualHost statement in /etc/apache2/sites-available/default-ssl
    # to <VirtualHost *:443>
    # Server Name Indication for SSL named virtual hosts is currently not
    # supported by MSIE on Windows XP.
    Listen 443
    NameVirtualHost *:443
</IfModule>
Vous noterez que les choses sont biens faites, regardez les commentaires, on nous indique que si on ajoute NameVirtualHost *:443 ici il faut également modifier le vhost default-ssl pour remplacer <VirtualHost _default_:443> par <VirtualHost *:443>.

On édite le fichier /etc/apache2/sites-available/default-ssl sur les 2 premières lignes qui sont :

IfModule mod_ssl.c>
<VirtualHost _default_:443>
...
Et qui deviennent alors :

IfModule mod_ssl.c>
<VirtualHost *:443>
...
On restart Apache une dernière fois et il ne devrait plus y avoir de soucis. Rendez-vous sur votre site, et vous pourrez vérifier que tout fonctionne bien avec https activé et fonctionnel. Et surtout sans alerte.

Vérification de la sécurité
Dernier point à voir sur la configuration : la qualité de sécurisation de notre connexion SSL.
En effet, en fonction des paramètres définis dans notre vhost, entre autre, il se peut que la sécurisation ne soit pas optimale. Il y a donc potentiellement un risque d’usurpation d’identité ou encore d’attaque de type « Man in the middle ».
Avec les paramètres définis plus haut, il n’y a pas de soucis. Vous devriez obtenir la note « A+ ». Il s’agit des paramètres par défaut de Let’s Encrypt. Mais cette note n’est pas sans contrepartie.

Commencez par tester votre site sur SSL Server Test : https://www.ssllabs.com/ssltest/

Si vous avez bien suivi ce tuto, vous devriez avoir le fameux « A+ » :

Note A+ SSL server test

Maintenant tout est affaire de compromis.
Plus vous durcirez vos règles, plus vous obtiendrez une meilleure note, plus vous perdrez en compatibilité avec les anciens navigateurs. IE8 sur XP par exemple.
Donc regardez bien vos stats de visites au préalable pour connaitre les configurations de vos visiteurs et adaptez en fonction.
La sécurité c’est bien, mais si vous perdez vos visiteurs ce n’est pas une solution non plus.

Renouveler un certificat Let’s Encrypt
A ce stade nous avons un certificat Let’s Encrypt installé et notre serveur bien configuré.
Il faut savoir que la durée de vie du certificat Let’s Encrypt n’est que de 90 jours, 3 mois donc.
Nous allons voir les commandes pour le renouveler manuellement et automatiquement.

Pour renouveler votre certificat manuellement il suffit de lancer la commande :

./certbot-auto renew
Cette commande interroge les serveurs Let’s encrypt et ces derniers vont déterminer si votre certificat a besoin d’être renouvelé ou non.
Notez que tant que vous ne serez pas dans les 30 derniers jours avant la date d’expiration, Let’s encrypt ne le renouvellera pas. Mais si besoin vous pouvez forcer malgré tout le renouvellement en ajoutant le flag --force-renewal.
Lors du renouvellement vous pouvez également demander la modification de la force de la clé RSA. Si vous avez créé votre certificat en 2048 bits, vous pouvez le passer en 4096, et inversement. Pour cela vous ajouterez le flag --rsa-key-size 4096.
La commande serait alors :

./certbot-auto renew --rsa-key-size 4096 --force-renewal
Ce sont là les options principales. Il y en a d’autres disponibles en fonction de vos besoins, là encore je vous invite à consulter la documentation.

Pour terminer sur le renouvellement il est important de signaler que la commande ./certbot-auto renew renouvelle l’ensemble des certificats Let’s Encrypt disponibles sur la machine.
Pour le moment il n’est pas possible de renouveler un domaine en particulier. Cela sera probablement disponible dans le futur.

Renouveler automatiquement le certificat
Maintenant nous allons voir comment automatiser le renouvellement de notre certificat avec CRON.
Personnellement je me créé un petit script que je vais placer dans le dossier cron.monthly. Basique. Et « monthly » car comme vu précédemment le certificat ne sera renouvelé que dans les 30 derniers jours. Inutile donc de faire la demande toutes les semaines et encore moins tous les jours.
Exemple de fichier renew-certificate.sh :

#!/bin/bash

/chemin/vers/certbot-auto renew
Vous pouvez ajouter le flag --quiet si vous ne souhaitez pas avoir de retour sur la commande.
Dans tous les cas, pas de panique, comme nous avons ajouté notre adresse mail lors de la création du certificat, s’il y a le moindre problème, vous serez prévenu par email.

Révoquer un certificat Let’s Encrypt
Dernier sujet : la suppression, la révocation d’un certificat. Si vous n’en n’avez plus besoin.

Nous allons utiliser la commande certbot-auto revoke en lui passant un certain nombre de paramètres : le certificat à révoquer, le chemin vers la clé privée et le chemin vers le certificat :

./certbot-auto revoke --domain domain.tld --cert-path /etc/letsencrypt/live/domain.tld --key-path /etc/letsencrypt/live/domain.tld
Votre certificat est révoqué. Mais ce n’est pas suffisant.
Si vous regardez vos dossiers /etc/letsencrypt/live/, /archive/ et /renewal/, vous trouverez encore les dossiers/fichiers à l’intérieur.
Pour faire propre, nous allons tout supprimer avec la commande suivante :

rm -rf /etc/letsencrypt/live/domain.tld/ && rm -rf /etc/letsencrypt/archive/domain.tld/ && rm /etc/letsencrypt/renewal/domain.tld.conf
Restaurer la configuration Apache
Dernière modification, restaurer la configuration du vhost Apache dans sa configuration initiale et surtout supprimer toutes les références au SSL et aux certificats.
Pour celaiIl suffit de se référer à la configuration de base plus haut dans cette page. Je vous laisse faire.

Conclusion
J’espère que ce tuto vous sera utile et que vous en aurez appris un peu plus sur le fonctionnement de Let’s Encrypt et globalement l’intégration d’un certificat SSL pour Apache. Car pour cette dernière partie, Let’s Encrypt ou autre, c’est exactement la même chose.
Si vous avez des remarques ou questions, n’hésitez pas à poster un commentaire.
