Cet article va vous montrer le(s) KRO 
=====================================

Ne sortez pas les dents tout de suite, ni même la bière.
KRO C'est quoi alors ? 
un nouvel outil d'abstraction de resource très puissant `Kube Resource Orchestrateur` ou `KRO`.

La première idée derrière KRO c'est d'abstraire très facilement un groupe de resource kubernetes standard ou non (toutes les resources de divers operator sont les bienvenus aussi) derrière une nouvelle resource que l'on peut définir simplement à partir d'un fichier YAML.

Voila comment nous pouvons définir une simple application qui contiendrait un déploiement, un service et une resource ingress:

Création d'un nouveau cluster avec `kind`:

```bash
➜  ~ kind create cluster --name demo-kro-cluster  
enabling experimental podman provider
Creating cluster "demo-kro-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.35.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-demo-kro-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-demo-kro-cluster

Thanks for using kind! 😊
```

Vérifier que le default context est le bon:
```bash
➜  ~ kubectl config get-contexts

CURRENT   NAME                    CLUSTER                  AUTHINFO                  NAMESPACE
          kind-demo               kind-demo                kind-demo                                                           
*         kind-demo-kro-cluster   kind-demo-kro-cluster    kind-demo-kro-cluster 
```

# Installation de kro
Il y a plusieurs méthodes, nous allons au plus rapide avec le helm chart:

Pour plus de détails [Kro Getting Started](https://kro.run/docs/getting-started/Installation) 

### Installation:
```bash
helm install kro oci://registry.k8s.io/kro/charts/kro \
  --namespace kro-system \
  --create-namespace
```

### Contrôlons que le helm est bien installé:
```bash
helm list -n kro-system
```

exemple de résultat:

```bash
➜  ~ helm list -n kro-system
NAME	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART     	APP VERSION
kro 	kro-system	1       	2026-03-27 11:02:22.463743612 +0100 CET	deployed	kro-v0.9.0	v0.9.0  
```

### Contrôlons que le pod tourne bien:

```bash
kubectl get pods -n kro-system
```

exemple de résultat:

```bash
➜  ~ kubectl get pods -n kro-system
NAME                   READY   STATUS    RESTARTS   AGE
kro-7cdb68bf95-mns2p   1/1     Running   0          2m1s
```

# Notre premier ResourceGraphDefinition

Notre premier ResourceGraphDefinition va déclarer une nouvelle CRD qui va s'appeler `Application` 
Cette `Application` créera 3 resources kubernetes standard:
- 1 service
- 1 deployement
- 1 ingress

Elle prendra en entrée les paramètres suivants:
- `name`: le nom de l'application
- `image`: l'adresse de l'image qui contient l'application
- `replicas`: le nombre de réplicas qu'il y aura dans le déployment
- `ingress.enabled`: pour activer ou non la création d'une resource ingress pour exposer l'application à l'extérieur du cluster

Ces paramètres seront utiles pour la création des resources, ils seront accessible via des variable de template

Elle exposera les status suivants:
- `deploymentConditions`: l'état du déploiement 
- `availableReplicas`: le nombre de réplicas


```yaml
apiVersion: kro.run/v1alpha1
kind: ResourceGraphDefinition
metadata:
  name: my-application
spec:
  # kro uses this simple schema to create your CRD schema and apply it
  # The schema defines what users can provide when they instantiate the RGD (create an instance).
  schema:
    apiVersion: v1alpha1
    kind: Application
    spec:
      # Spec fields that users can provide.
      name: string
      image: string | default="nginx"
      replicas: integer | default=3
      ingress:
        enabled: boolean | default=false
    status:
      # Fields the controller will inject into instances status.
      deploymentConditions: ${deployment.status.conditions}
      availableReplicas: ${deployment.status.availableReplicas}

  # Define the resources this API will manage.
  resources:
    - id: deployment
      template:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: ${schema.spec.name} # Use the name provided by user
        spec:
          replicas: ${schema.spec.replicas} # Use the replicas provided by user
          selector:
            matchLabels:
              app: ${schema.spec.name}
          template:
            metadata:
              labels:
                app: ${schema.spec.name}
            spec:
              containers:
                - name: ${schema.spec.name}
                  image: ${schema.spec.image} # Use the image provided by user
                  ports:
                    - containerPort: 80

    - id: service
      template:
        apiVersion: v1
        kind: Service
        metadata:
          name: ${schema.spec.name}-svc
        spec:
          selector: ${deployment.spec.selector.matchLabels} # Use the deployment selector
          ports:
            - protocol: TCP
              port: 80
              targetPort: 80

    - id: ingress
      includeWhen:
        - ${schema.spec.ingress.enabled} # Only include if the user wants to create an Ingress
      template:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: ${schema.spec.name}-ingress
        spec:
          rules:
            - http:
                paths:
                  - path: "/"
                    pathType: Prefix
                    backend:
                      service:
                        name: ${service.metadata.name} # Use the service name
                        port:
                          number: 80
```




Ok ca c'est bon on savait faire vous me direz, helm fait ca depuis longtemps 
Ouai mais helm il sait pas prendre le statut d'une resource créé, pour l'utiliser en input d'une autre resource:

Ouai mais helm il sait pas séquencer la création de resource.

Ouai mais la déclaration de helm c'est pas kube native:

Ceci n'est que la pointe du KRO, il permet de faire bien plus:
- 
