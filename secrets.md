# Part 1 - Securing Secrets
## Intro
For securing secrets using Kubernetes Secrets, I followed the documentation <a href="https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/">here</a> to create the secrets and distribute them. I created 3 separate secret yaml configuration files for the secrets and passwords found in the code and config files:
- MYSQL_ROOT_PASSWORD (django-mysql-root-pass-secret.yaml)
- SECRET_KEY (django-enc-secret-key.yaml)
- RANDOM_SEED (django-random-seed.yaml)

## MYSQL_ROOT_PASSWORD
First, I searched the code for everywhere MYSQL_ROOT_PASSWORD was used.and found that referenced in 3 yaml files:
- ./GiftcardSite/k8/django-deploy.yaml
- ./db/k8/db-deployment.yaml
- ./db/k8s/db-deployment.yaml

I took the hardcoded password references in these 3 files, base64 encoded them, the added to the django-mysql-root-pass-secret.yaml file. Then, within each of the above yaml files that referenced this password, I replaced the hardcoded password in the env config with the following config which points to the django-mysql-root-pass-secret.yaml.
```
- name: MYSQL_ROOT_PASSWORD
    valueFrom:
        secretKeyRef: 
            name: mysql-root-secret
            key: password
```
Once the django-mysql-root-pass-secret.yaml was created and applied the reference to it in the over 3 yaml files, I ran the following command to create the secret in the container:
```
kubectl apply -f ./GiftcardSite/k8/django-mysql-root-pass-secret.yaml
```
this gets sourced as an environment variable when the containers are restarted and confirmed its there by running:
```
kubectl exec assignment3-django-deploy-567c568f86-zbbfg -- env | grep MYSQL_ROOT
```

## SECRET_KEY
I looked through the code for any hardcoded passwords or secret keys and found SECRET_KEY in settings.py. I followed the same approach as above to create a new secret yaml file ./GiftcardSite/k8/django-enc-secret-key.yaml and created the secret in the container:
```
kubectl apply -f ./GiftcardSite/k8/django-enc-secret-key.yaml
```
Then added the reference to it in the django-deploy.yaml file:
```
- name: SECRET_KEY
    valueFrom:
        secretKeyRef: 
            name: enc-secret-key
            key: secret-key
```
I then changed the settings.py code to remove the hardcoded SECRET_KEY value and replaced with the following line which sources the SECRET_KEY environment variable from the container:
```
SECRET_KEY = os.environ.get('SECRET_KEY')
```

## RANDOM_SEED
This was another hardcoded value in settings.py and the value is used to generate the saly used for encryption, so seemed like a value that should be protected. Followed the same approach to create the secret as the other secrets above: 
```
kubectl apply -f ./GiftcardSite/k8/django-random-seed.yaml
```
The added the reference to it in the django-deploy.yaml file:
```
- name: RANDOM_SEED
    valueFrom:
        secretKeyRef: 
            name: random-seed
            key: random-seed 
```
I then changed the settings.py code to remove the hardcoded random seed value and replaced with the following line which sources the RANDOM_KEY environment variable from the container:
```
RANDOM_SEED = base64.b64decode(os.environ.get('RANDOM_SEED'))
```

## Rebuild & Redeploy
After the above was done, I rebuilt and redeployed the pods using these commands:
```
eval $(minikube docker-env)

docker build -t nyuappsec/assign3:v0 .
docker build -t nyuappsec/assign3-proxy:v0 proxy/
docker build -t nyuappsec/assign3-db:v0 db/

kubectl apply -f db/k8
kubectl apply -f GiftcardSite/k8
kubectl apply -f proxy/k8
```

Confirmed the app still worked by relaunching with <code>
minikube service proxy-service</code>
