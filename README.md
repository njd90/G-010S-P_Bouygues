<p align="left"> <img src="https://komarev.com/ghpvc/?username=njd90&label=Profile%20views&color=0e75b6&style=flat" alt="njd90" /> </p>


# G-010S-P sur infra Bouygues Offre Must

## Matériels nécessaires :  
Un convertisseur Fibre vers RJ45 (MC220L de Tp-Link)
Un ONU G-010S-P  

## Récupération des informations :  

La majorité des informations nécessaires sont disponibles sur les étiquettes de l’ONT externe. 
- le Hardware version (HW)
- le numero de série (SN)
- l'IMIE

Cependant, il manque le Software Version. Pour cela, il faut se connecter à l'interface web de l'ONT, cependant Bouygues l'a bloquée. Voici la procédure pour tout de même se connecter : 
- retirer fibre et cable RJ45 de l'ONT
- débrancher le cable d'alim 
- appuyer sur le bouton reset puis rebrancher l'alim et maintenir appuyé le bouton reset jusqu'à ce que la led LOS clignote rouge.
- brancher un cable RJ45.
- après quelques minutes, la LED lan va s'allumer et clignoter
- Sur sa machine, configurer l'adresse IP : 192.168.100.2 (masque : 255.255.255.0)
- se connecter en web à 192.168.100.1
	Login : telecomadmin
	Mot de passe : admintelecom
- Sur la page « Device Information », vous trouverez le « Software Version » mais également les autres informations des étiquettes.

![Image](../main/HG8010Hv3_Bouygues.png?raw=true)

## Passage du G-010S-P en Carlitoxx V1
### Informations pour se connecter en ssh au firmware d'origine :
> Adresse IP: 192.168.1.10  
> Login: ONTUSER  
> Password: SUGAR2A041  

Dans un premier temps, il faut utiliser l'ONU avec le MC220L, brancher le MC220L directement sur un PC en fixant une ip static de type 192.168.1.X (différente de 192.168.1.10).

### Backup l'ONU G-010S-P avant flash : 
Se connecter en ssh à l'ONU  
```
ssh ONTUSER@192.168.1.10
```

Backup avec la commande dd 
```
dd if=/dev/mtd0 of=/tmp/mtd0.bin
dd if=/dev/mtd1 of=/tmp/mtd1.bin
dd if=/dev/mtd2 of=/tmp/mtd2.bin
dd if=/dev/mtd3 of=/tmp/mtd3.bin
dd if=/dev/mtd4 of=/tmp/mtd4.bin
dd if=/dev/mtd5 of=/tmp/mtd5.bin
```

Transfert des backup
```
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd0.bin mtd0.bin
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd1.bin mtd1.bin
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd2.bin mtd2.bin
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd3.bin mtd3.bin
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd4.bin mtd4.bin
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/mtd5.bin mtd5.bin
```

Bakcup des variables : 
```
fw_printenv > /tmp/fw_printenv.backup
uci show > /tmp/uci_show.backup

scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/fw_printenv.backup .
scp -o KexAlgorithms=diffie-hellman-group1-sha1 ONTUSER@192.168.1.10:/tmp/uci_show.backup .
```


### Flash du G-010S-P : 

Recupérez les images de la carlitoxx v1 : [Carlitoxx v1](https://github.com/njd90/G-010S-P_Bouygues/raw/main/CarlitoxV1.zip)

Transférer les images sur l'ONU 
```
scp -o KexAlgorithms=diffie-hellman-group1-sha1 mtd2.bin ONTUSER@192.168.1.10:/tmp/
scp -o KexAlgorithms=diffie-hellman-group1-sha1 mtd5.bin ONTUSER@192.168.1.10:/tmp/
```

Se connecter à l'ONU en SSH
```
ssh ONTUSER@192.168.1.10
```

Charger les nouvelles images : 
```
mtd -e image0 write /tmp/mtd2.bin image0
mtd -e linux write /tmp/mtd5.bin linux
```
*** sur un G-010S-P image0 correspond à mtd2 et linux mtd3 ***

Renseigner les 2 variables : 
```
fw_setenv ont_serial XXXXXXXXX
fw_setenv target oem-generic
```
Où ont_serial [XXXXXXXXX] correspond au numero de série de l'ONT Bouygues commencant par HWTC

Redémarrer sur l'image Carlitoxx:
```
fw_setenv committed_image 0
reboot
```
## Configuration de l'ONU

### Configuration de Carlitoxx v1

Une fois l'ONU redémarré via un test ping -t -w 30 192.168.1.10

Aller sur l'interface web de celui-ci http://192.168.1.10  
Le login est : root  
Pas de mot de passe  

Lors de votre première connexion, il vous sera demandé de renseigner un password. Ce mot de passe sera utilisé en SSH et en HTTP par la suite.  

L'utilisateur par defaut en SSH est : root.

SSH sur l'ONU
```
ssh root@192.168.1.10
```

Editer le fichier 
```
vi /etc/init.d/sys.sh
```

Editez la partie oem-generic qui correspond au paramétrage fait à l'étape précédente (fw_setenv target oem-generic)

Remplacer les lignes du fichier  
uci set sys.mib.vendor_id='ZM\0\0'  
uci set sys.mib.ont_version='SFP-P05\0\0\0\0\0\0\0'  
uci set sys.mib.equipment_id='GPONSTICK\0\0\0\0\0\0\0'  

par  

uci set sys.mib.vendor_id=HWTC (4 premières lettres du SN)  
uci set sys.mib.ont_version=XXXX (Hardware version)  
uci set sys.mib.equipment_id=HWTCXXXXXXXX (SN)  

![Image](../main/conf_sys_sh_var.png?raw=true)


Ensuite, il faut renseigner certaines variables :  
fw_envset ont_serial HWTCXXXXXXXX (SN)  
fw_setenv image0_version V3XXXXXXXXXXX (Software version)  
Optionnel : fw_setenv image1_version V3XXXXXXXXXXX (Software version)  


### Test de l'ONU dans le MC220L

Il est maintenant temps de connecter la fibre et de vérifier que la configuration de l'ONU permet de se connecter à l'OLT, d'obtenir les VLAN et de pouvoir obtenir un trafic montant et descendant.  

Se connecter en SSH sur l'ONU
```
ssh root@192.168.1.10
```

On vérifie l'inscription sur l’arbre GPON :
```watch -n 1 onu ploamsg```

Vous devez constater le passage O5
> errorcode=0 curr_state=5 previous_state=4 elapsed_msec=30428

curr_state doit être à 5

Pour vérifier que le vlan 100 est bien transmis, il suffit d'exécuter la commande :
```gtop ```

puis c et v 

Vous devez obtenir :  
![Image](../main/gtop_vlan.png?raw=true)

Si vous êtes arrivé jusqu'ici, normalement votre ONU est correctement configuré.

Il ne vous reste plus qu'à configurer votre routeur pour obtenir une adresse IP.

## Liens utiles : 
https://lafibre.info/remplacer-bbox/remplacer-lont-externe-de-loffre-must-par-un-onu-sfp/msg867098/#msg867098

https://github.com/Berzerker/google-fiber-2gbps-bypass

https://www.dslreports.com/forum/r32230041-Internet-Bypassing-the-HH3K-up-to-2-5Gbps-using-a-BCM57810S-NIC

https://forum.openwrt.org/t/support-ma5671a-sfp-gpon/48042/33

https://lafibre.info/remplacer-livebox/guide-de-connexion-fibre-directement-sur-un-routeur-voire-meme-en-2gbps/msg832904/#msg832904
