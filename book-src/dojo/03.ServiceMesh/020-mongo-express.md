# (Voliteľné - samostatná práca) Nasadenie Mongo Express

Pri vývoji aplikácie je vhodné mať možnosť sledovať stav databázy. Pre túto potrebu je možné použiť nástroj [Mongo Express][mongoexpress], ktorý je možné nasadiť do kubernetes klastra. V tejto chvíli už máte všetky potrebné poznatky, aby ste to dokázali samostatne. Vyskúšajte nasadiť Mongo Express do kubernetes klastra na základe informácií, ktoré už máte k dispozícii tak, aby bola táto aplikácia obslúžená na ceste `/mongo-express` a následne porovnajte výsledok s tu uvedeným postupom, ktorý obsahuje aj nasadenie prístupu k aplikácii formou mikro frontend aplikácie.

V adresári `${WAC_ROOT}/ambulance-gitops/apps/mongo-express` vytvorte súbor `deployment.yaml` s obsahom:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:  
  name: &PODNAME mongo-express
  annotations: 
spec:
  replicas: 1  
  selector:
    matchLabels:
      pod: *PODNAME
  template:
    metadata:
      labels: 
        pod: *PODNAME
    spec:
      containers:
      - image: mongo-express
        name: mongo-express
        env:
        - name: ME_CONFIG_SITE_BASEURL @_important_@
          value: /mongo-express/ @_important_@
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:  
            secretKeyRef: 
              name: mongodb-auth @_important_@
              key: username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:  
            secretKeyRef: 
              name: mongodb-auth @_important_@
              key: password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-connection @_important_@
              key: host
        # authentication and authorization at cluster level
        - name: ME_CONFIG_BASICAUTH_USERNAME
          value: "" @_important_@
        - name: ME_CONFIG_BASICAUTH_PASSWORD
          value: "" @_important_@
        - name: ME_CONFIG_BASICAUTH
          value: "false"
        ports:
        - name: http
          containerPort: 8081
        resources:
          limits:
            cpu: '1'
            memory: '512M'
          requests:
            cpu: '0.01'
            memory: '128M'
```

Všimnite si, že sa odkazujeme na secret `mongodb-auth` a configmap `mongodb-connection`. Tieto objekty sme už prevzali z konfigurácie webapi služby. Tu využijeme fakt, že sme pri týchto objektoch potlačili generovanie mena s hashom. Nastavením premenných prostredia `ME_CONFIG_BASICAUTH_...` na prázdny reťazec sme zároveň potlačili autentifikáciu pri prístupe na túto službu. Predpokladáme, že autentifikácia a autorizácia bude zabezpečená na úrovni kubernetes klastra - viď nasledujúce kapitoly. Premennou prostredia `ME_CONFIG_SITE_BASEURL` sme nastavili cestu pre [&lt;base&gt; element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base), čo zabezpečí správne načítavanie zdrojov tejto aplikácie.

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/service.yaml` s obsahom:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: &SVCNAME mongo-express
spec:
  ports:
  - name: http
    protocol: TCP
    port: 8081
    targetPort: 8081
  selector:
    pod: *SVCNAME
```

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/http-route.yaml`, ktorý bude smerovať požiadavky na ceste `/mongo-express` na službu `mongo-express`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mongo-express
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mongo-express
      backendRefs:
        - group: ""
          kind: Service
          name: mongo-express
          port: 8081
```

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/webcomponent-link.yaml` s obsahom:

```yaml
apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: mongo-express-ufe-link
spec:
  microFrontend: polyfea-md-shell
  element: polyfea-md-app
  attributes: 
    - name: headline
      value: Mongo Express
    - name: short-headline
      value: UI Access to local Mongo DB
    - name: material-icon
      value: database
    - name: href
      value: ./mongo-express
  priority: 10
  displayRules:
    - anyOf:
      - context-name: applications
      - context-name: rail-content
      - context-name: drawer-content
```

Ďalej súbor `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/webcomponent-content.yaml`:

```yaml
apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: mongo-express-ufe-content
spec:
  element: iframe
  attributes:
    - name: src
      value: /mongo-express
  displayRules:
    - allOf:
      - context-name: main-content
      - path: "^(\\.?/)?mongo-express(/.*)?$"
  style:
    - name: width
      value: 100%
    - name: height
      value: 100%
```

Táto konfigurácia sprístupní aplikáciu [MongoExpress] ako súčasť nášho mikro front-end rozhrania. Samotná aplikácia potom bude dostupná pomocou `iframe` elementu s nastaveným atribútom `src="/mongo-express"`.

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/kustomization.yaml` s obsahom:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- webcomponent-link.yaml
- webcomponent-content.yaml
- http-route.yaml

commonLabels:
  app.kubernetes.io/component: mongo-express
```

Upravte súbor `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml`

```yaml
...
resources:
- ../../../apps/pfx-ambulance-ufe
- ../../../apps/pfx-ambulance-webapi
- ../../../apps/mongo-express @_add_@
...
```

Otvorte príkazové okno v adresári `${WAC_ROOT}/ambulance-gitops` a overte správnosť Vašej konfigurácie

```ps
kubectl kustomize clusters/localhost/install
```

Archivujte zmeny do git repozitára a odovzdajte ich do vzdialeného repozitára.

```ps
git add .
git commit -m "Mongo Express deployment"
git push
```

Po aplikovaní zmien službou [FluxCD][flux] prejdite na stránku [http://localhost/fea](http://localhost/fea) a overte, že sa Vám zobrazuje aj aplikácia [Mongo Express][mongoexpress].
