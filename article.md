Cet article vous vous montrer le KRO 

Ne sortez pas les dents tout de suite, ni même la bière.
KRO C'est quoi alors ? 
un nouvel outil d'abstraction de resource très puissant Kube Resource Orchestrateur ou KRO.

La première idée derrière KRO c'est d'abstraire très facilement un groupe de resource kubernetes standard ou non (toutes les resources de divers operator sont les bienvenus aussi) derrière une nouvelle resource que l'on peut définir simplement à partir d'un fichier YAML.

Voila comment nous pouvons définir une simple application qui contiendrait un déploiement, un service et une resource ingress:
TODO

Ok ca c'est bon on savait faire vous me direz, helm fait ca depuis longtemps 
Ouai mais helm il sait pas prendre le statut d'une resource créé, pour l'utiliser en input d'une autre resource:

Ouai mais helm il sait pas séquencer la création de resource.

Ouai mais la déclaration de helm c'est pas kube native:

Ceci n'est que la pointe du KRO, il permet de faire bien plus:
- 
