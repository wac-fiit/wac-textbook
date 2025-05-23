# Nasadenie do klastra kubernetes

V predchádzajúcom príklade sme si predstavili jeden zo spôsobov nasadenia našej aplikácie, pri ktorom sme využili služby typu [_Platform as a Service_][paas] v dátovom centre [Microsoft Azure][azure].
Hoci je tento spôsob nasadenia aplikácie funkčný, je vhodný len pre veľmi malé projekty. Na projektoch, kde spolupracuje niekoľko tímov a kde finálne dielo pozostáva z rôznych subsystémov, je tento spôsob nedostatočný.

V tomto cvičení si ukážeme, ako nasadiť našu aplikáciu do klastra [Kubernetes] manuálne a v ďalšej časti, ako zabezpečiť kontinuálne nasadenie aplikácie technikou [GitOps] s využitím aplikácie [Flux].

Pred samotnou prácou sa uistite, že máte zapnutú podporu Kubernetes v rámci `Docker Desktop` aplikácie, viď obrázok:

![Kubernetes v Docker for desktop](./img/060-01-docker-kubernetes.png)

### 1. Vytvorenie adresárovej štruktúry pre GitOps nasadenie aplikácie

V priečinku `${WAC_ROOT}` vytvorte nový priečinok s názvom `ambulance-gitops`. Vo VS Code pridajte tento priečinok do aktuálneho pracovného priestoru pomocou voľby menu _File->Add Folder to Workspace..._

V priečinku `${WAC_ROOT}/ambulance-gitops` vytvorte nasledujúcu adresárovú štruktúru:

```plain
ambulance-gitops
|
|- apps
|  L <pfx>-ambulance-ufe
|
|- infrastructure
|  L polyfea
|
|- components
|- clusters
    L localhost
        
```

Aplikáciu budeme nasadzovať s využitím aplikácie [kustomize] pre  riadenie konfigurácií v systéme kubernetes.  [Kustomize] je natívnou  súčasťou systému kubernetes a je tiež integrovaná do aplikácie `kubectl`. Často používanou  alternatívou k tejto aplikácii je aplikácia [Helm]. Účel použitia je pri  týchto aplikáciách rovnaký, Helm sa zameriava na techniku šablón, ktoré sú parametrizované konfiguráciou pre špecifické nasadenie. [Kustomize] šablóny odmieta, keďže nie sú priamo  nasaditeľné alebo použiteľné a pracuje na princípe prispôsobovania funkčných manifestov pomocou  čiastkových úprav - _patch_-ov - pre špecifické nasadenie. V tomto cvičení preferujeme  [Kustomize] najmä z dôvodu, že manifesty si v základnej forme zachovávajú syntaktickú a  sémantickú presnosť definovanú [Kubernetes API][k8s-api].

Štruktúra priečinkov je hierarchická. Priečinok `clusters` bude obsahovať konfiguráciu systému  pre jednotlivé špecifické klastre, väčšinu času budeme počas cvičenia pracovať s lokálnym  klastrom označeným ako `localhost`.

Priečinok `infrastructure` bude obsahovať komponenty systému, ktoré sa buď inštalujú typicky len raz počas životného cyklu klastra alebo sú zdieľané medzi tímami a sú potrebné k behu  ostatných subsystémov. Čo presne je súčasťou priečinku `infrastructure` sa môže líšiť prípad od prípadu, v zásade tam patria zdieľané služby s inými aplikáciami, ktoré na niektorých klastroch už sú predpripravené a nie je možné ich "preinštalovať" v rámci daného projektu/tímu, a na  iných klastroch ich treba predinštalovať, aby bola zabezpečená funkčnosť dodávanej aplikácie.

Priečinok `apps` obsahuje komponenty systému, ktoré projektový tím zabezpečuje pre všetky  cieľové prostredia - klastre - a má ich plne pod kontrolou.

Priečinok `components` umožňuje zdieľať konfiguráciu medzi jednotlivými variantami nasadenia, napríklad pokiaľ by náš systém mal možnosť použiť rôzne systémy DBMS, mohli by sme príslušné konfigurácie a úpravy existujúcich komponentov umiestniť do tohto priečinka. V našom prípade budeme tento priečinok primárne používať na definíciu jednotlivých verzií nášho systému.

>info:> V týchto cvičeniach používame takzvanú [_MonoRepo_](https://fluxcd.io/flux/guides/repository-structure/#monorepo) štruktúru repozitára pre priebežné nasadenie. Nie je to ale jediná možnosť, viac možností je opísaných vo [flux dokumentácii](https://fluxcd.io/flux/guides/repository-structure/).

### 2. Vytvorenie a nasadenie Deployment manifestu pre web komponent

V priečinku `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe` vytvorte súbor `deployment.yaml` s nasledujúcim obsahom (komentáre môžete vynechať):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <pfx>-ambulance-ufe-deployment      # meno deployment-u od ktorého sa odvádza meno podu
spec:
  replicas: 2 @_important_@
  selector:
    matchLabels:
        pod: <pfx>-ambulance-ufe-label    # tento riadok je len označenie štítku 
                                    # key-value pair pod=ambualnce ufe
                                    # určený pre výber podov s rovnakým štítkom
  template:                    # šablóna pre vytvorenie replík podov
    metadata:
      labels: 
        pod: <pfx>-ambulance-ufe-label    # štítky podu
    spec:
      containers:
      - name: ambulance-ufe-container # meno kontaineru - v pod-e ich môže byť viac
        image: <pfx-hub>/ambulance-ufe
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080       # port na ktorom počúva kontajner, pomenovaný ako `http`

        resources:                  # určenie kvót pre využívané fyzické prostriedky
                                    # je dôležité pre riadenie záťaže v rámci kubernetes
                                    # klastra
            requests:               
                memory: "32M"
                cpu: "0.1"
            limits:
                memory: "320M"
                cpu: "0.3"
```

Súbor `deployment.yaml` je deklaráciou nasadenia našej služby - tzv. [_workload_](https://kubernetes.io/docs/concepts/workloads/) - do klastra. Všimnite si, že požaduje nasadenie dvoch replík - čo znamená, že v klastri budú vytvorené dva _pod_-y a v rámci každého z nich bude vytvorený aj bežiaci kontajner s našou aplikáciou. Všimnite si tiež, že v šablóne pre vytvorenie podu je definovaný štítok `pod: <pfx>-ambulance-ufe-label`, ktorý je použitý v selektore pre výber podov, ktoré majú byť súčasťou tohto nasadenia. Tento štítok je použitý aj v ďalších manifestoch, ktoré budeme vytvárať. Všetky objekty (_resources_) v systéme kubernetes môžu byť označené štítkami a tiež môžu byť vyhľadávané pomocou selektorov, ktoré štítky používajú. Prípony `-label`, `-deployment` a `-container` nie sú štandardnou konvenciou, sú tu uvedené najmä pre lepšie sa zorientovanie ako jednotlivé názvy v manifeste súvisia.

Takto vytvorený manifest je funkčný a možno ho nasadiť do klastra. Predpokladáme, že máte nainštalovaný systém [kubernetes] na svojom počítači a zvolený [aktívny kontext](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#define-clusters-users-and-contexts) pre tento klaster. To, ktorý kontext je aktívny môžete zistiť vykonaním príkazu

```ps
kubectl config get-contexts
```

ktorý zobrazí výsledky v podobe:

```plain
CURRENT   NAME           CLUSTER           AUTHINFO 
*         aks            aks-cluster       user-admin
          docker-desktop docker-desktop    docker-desktop
```

kde znak `*` označuje práve aktívny kontext. Pokiaľ nie je aktívny vami požadovaný kontext, môžete ho zmeniť príkazom:

```ps
kubectl config use-context docker-desktop
```

>info:> Odporúčame nainštalovať si na svojom počítači aplikácie [kubectx], [kubens], [OpenLens][lens], respektíve [k9s](https://k9scli.io/) a naučiť sa ich používať pri práci s kubernetes klastrami. V cvičeniach sú uvedené len príkazy založené na [kubectl] nástroji, ktorý je súčasťou nainštalovaného [kubernetes] systému.

Teraz prejdite do priečinka `${WAC_ROOT}/ambulance-gitops` a vykonajte príkaz

```ps
kubectl apply -f apps/<pfx>-ambulance-ufe/deployment.yaml
```

Následne môžete vykonať (opakovane) príkaz

```ps
kubectl get pods
```

a zopakovať ho až pokiaľ neuvidíte výpis v podobe

```plain
NAME                                        READY   STATUS    RESTARTS   AGE
ambulance-ufe-deployment-7f45494d6b-qj58z   1/1     Running   0          37s
ambulance-ufe-deployment-7f45494d6b-slppc   1/1     Running   0          61s
```

>build_circle:> Status môže mať dočasne aj výpis vo forme `Container Creating`. Pokiaľ stĺpec indikuje nejakú chybu, opravte manifest `deployment.yaml` a znova ho aplikujte príkazom `kubectl apply -f apps/<pfx>-ambulance-ufe/deployment.yaml`.

Výpis indikuje, že v našom klastri máme teraz bežiace dva [pody](https://kubernetes.io/docs/concepts/workloads/pods) - zjednodušene dva virtuálne počítače pripojené k virtuálnej sieti kubernetes klastra.

### 3. Presmerovanie portov pre prístup k podom v Kubernetes a odstránenie nasadených objektov

Momentálne sú tieto pody dostupné len na virtuálnej sieti systému kubernetes. Pokiaľ k nim chceme pristúpiť, môžeme využiť funkciu [presmerovania portov](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) z lokálneho systému na cieľový pod (prípadne _service_) aktívneho klastra.

Vykonajte príkaz:

```ps
kubectl port-forward <pfx>-ambulance-ufe-deployment-<your-hash> 8081:8080
```

a následne ak otvoríme v prehliadači stránku [http://localhost:8081](http://localhost:8081), uvidíme náš zoznam čakajúcich.

Funkcia `port-forward` funguje aj pri vzdialených klastroch a je vhodná na ladenie podporných mikroslužieb, ktoré nie sú dostupné z externých sietí mimo klastra. V našom prípade bude zverejnená len služba integrujúca jednotlivé mikro-Front-End komponenty, čo je ukázané nižšie.

>info:> Nástroj [kubectl] funguje ako web klient, ktorý pristupuje k [REST API]/[Kubernetes API][k8s-api] systému kubernetes. Je možné pristupovať k tomuto API aj priamo, museli by sme ale zabezpečiť správnu autenticitu a autorizáciu našich požiadaviek, ktorú nástroj [kubectl] vykonáva automaticky na základe konfigurácie štandardne uloženej v adresári `~/.kube`.

Nakoniec odstránime nami vytvorené objekty pomocou príkazu:

```ps
kubectl delete -f apps/<pfx>-ambulance-ufe/deployment.yaml
```

### 4. Príprava manifestov pre web komponent

Teraz vytvorte súbor `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/service.yaml` s obsahom:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: <pfx>-ambulance-ufe   #  určuje aj záznam DNS v rámci virtuálne siete klastra
spec:  
  selector:
    pod: <pfx>-ambulance-ufe-label     # zhoduje sa s metadata.labels šablóny pod-u
  ports:
  - name: http
    protocol: TCP
    port: 80            # port pomenovanej služby, môže sa líšiť 
                        # od portu na ktorom počúva kontajner
    targetPort: http    # viď meno portu v deployment.yaml
```

Tento objekt v systéme kubernetes deklaruje [sieťovú službu](https://kubernetes.io/docs/concepts/services-networking/service/), ktorá zároveň implementuje jednoduchý _load balancer_ pre distribúciu HTTP požiadaviek medzi dve repliky našej webovej služby a implementuje aj [_service discovery_](https://en.wikipedia.org/wiki/Service_discovery) založený na systéme [DNS](https://en.wikipedia.org/wiki/Domain_Name_System), to znamená, že meno služby `<pfx>-ambulance-ufe` zároveň reprezentuje DNS záznam v rámci virtuálnej siete a teda naša webová služba bude interne v rámci systému kubernetes dostupná na adrese `http://<pfx>-ambulance-ufe` v rámci toho istého _namespace_ alebo na adrese `http://<pfx>-ambulance-ufe.<namespace>` z ľuboľného [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

Ďalej vytvoríme súbor, ktorý do našej obálky vloží link na náš webcomponent, `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent-link.yaml` s obsahom:

```yaml
apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: <pfx>-ambulance-ufe-link
spec:
  microFrontend: polyfea-md-shell # referencia na microfrontend, z ktorého sa načíta element
  element: polyfea-md-app # element, ktorý bude vložený na stránku
  attributes: # atribúty elementu polyfea-md-app
    - name: headline
      value: Zoznam čakajúcich <pfx>
    - name: short-headline
      value: Zoznam čakajúcich <pfx> 
    - name: supporting-text
      value: |
        Spravuje zoznam pacientov čakajúcich na vyšetrenie v ambulancii
    - name: material-icon
      value: home_health
    - name: href # presmerovanie po kliknutí na element polyfea-md-app
      value: ./<pfx>-ambulance-wl 
  priority: 10
  displayRules: # pravidlá, kde sa má element vložiť v obálke
    - anyOf:
      - context-name: applications
      - context-name: rail-content
      - context-name: drawer-content
```

>warning:> Meno elementu `<pfx>-ambulance-wl-list` musí zodpovedať komponentu, ktorý sme vytvorili predtým, pozri súbor `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.tsx`

Tento súbor je neštandardným objektom - _Custom Resource_ -  systému kubernetes. V ďalšom bode budeme vytvárať manifesty pre mikro-front-end _controller_, ktorý tieto objekty obhospodáruje. V zásade tu vykonávame registráciu mikro aplikácie - webového komponentu - do hlavnej aplikačnej obálky.

Všimnite si že používame referenciu na `microFrontend: polyfea-md-shell` a `element: polyfea-md-app`. Oba sú súčasťou riadiča, ktorý si popíšeme v bode 5.

Ďalej vytvoríme súbor, vďaka ktorému sa načíta obsah nášho webcomponentu, `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent-content.yaml` s obsahom:

```yaml
apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: <pfx>-ambulance-ufe-content
spec:
  microFrontend: <pfx>-ambulance-ufe
  element: <pfx>-ambulance-wl-list
  displayRules:
    - allOf:
      - context-name: main-content
      - path: "^(\\.?/)?<pfx>-ambulance-wl(/.*)?$"
```

Teraz nám však chýba referencia na microfrontend pre náš nový element. Vytvoríme súbor `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/microfrontend.yaml` s obsahom:

```yaml
apiVersion: polyfea.github.io/v1alpha1
kind: MicroFrontend
metadata:
  name: <pfx>-ambulance-ufe
spec:
  frontendClass: md-shell # default obsiahnutý v základe polyfea md-shell
  service: http://<pfx>-ambulance-ufe.wac-hospital # referencia na službu odkiaľ sa má načítať modul
  modulePath: build/ambulance-ufe.esm.js # cesta v rámci služby kde sa modul nachádza
```

Nakoniec vytvoríme súbor `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/kustomization.yaml`. Správa konfigurácií založená na systéme [Kustomize] využíva súbory `kustomization.yaml` ako integračný bod jednotlivých objektov deklarovaných v ostatných manifestoch. Pritom ostatné manifesty ostávajú stále použiteľné a možno ich priamo použiť ako argument aplikácie `kubectl`. Súbor `kustomization.yaml` je v zásade predpisom ako automatizovane _editovať_ originálne súbory pre nasadenie jednotlivých objektov v iných prostrediach - rozumej v iných klastroch - a tieto úpravy možno hierarchicky kumulovať. V tomto prípade ide len o vymenovanie jednotlivých manifestov a priradenie spoločných značiek.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- webcomponent-link.yaml
- webcomponent-content.yaml
- microfrontend.yaml

labels:
- pairs:
    app.kubernetes.io/component: <pfx>-ambulance-ufe
```

V tejto chvíli ešte nebudeme našu aplikáciu nasadzovať, potrebujeme najprv pripraviť manifesty pre infraštruktúru klastra. Pokiaľ ale chceme vidieť výsledok "kustomizacie" našej aplikácie, môžeme ho získať vykonaním príkazu

```yaml
kubectl kustomize ./apps/<pfx>-ambulance-ufe/
```

### 5. Príprava manifestov infraštruktúry pre nasadenie mikro-aplikácií

V tomto kroku pripravíme manifesty pre objekty našej _infraštruktúry_. V našom prípade sa teraz jedná najmä o radič mikro-aplikácií, teda akúsi aplikačnú obálku pre zobrazovanie jednotlivých web-komponentov. Pri skúmaní súčasného stavu sme nenašli implementáciu [micro-front-end systému][micro-fe], ktorý by vhodne kombinoval technológie [Web Components][webc], [Kubernetes] a [micro-Front-Ends][micro-fe]. Táto kombinácia sa pritom javí najvhodnejšia vzhľadom na súčasné trendy vývoja, keďže vytvára priestor na samostatný vývoj jednotlivých web komponentov založený na všeobecne uznávaných štandardoch, a ktoré je možno potom nasadiť deklaratívnym spôsobom v rôznych situáciách, čo priamo zodpovedá filozofii mikro služieb. Existujúce implementácie sú napríklad [bit], alebo [single-spa], ich integrácia ale väčšinou vyžaduje úzke previazanie mikro-front-end služieb.

Aby sme využili deklaratívne princípy Kubernetes API a nezávislosť tímov vyvíjajúcich jednotlivé mikro-aplikácie, bol pre potreby cvičenia vytvorený jednoduchý [_kubernetes controller_][k8s-controller], ktorý obsluhuje neštandardné objekty systému kubernetes definované pomocou [_Custom Resource Definition_][k8s-crd]. Tento _controller_ nepretržite sleduje zmeny deklarovaných objektov - _webcomponents_ - a poskytuje ich zabudovanému webovému serveru, ktorý implementuje aplikačnú obálku. Jednotlivé webové komponenty sú potom dynamicky načítavané podľa potreby a zobrazené na základe špecifikácie v týchto neštandardných objektoch. _Controller_ je implementovaný v programovacom jazyku Go (back-end) a v jazyku Typescript (front-end) a jeho zdrojové súbory sú dostupné na [https://github.com/polyfea](https://github.com/polyfea). Slúži zároveň ako motivácia a ukážka rozšíriteľnosti [kubernetes api][k8s-api], ktoré sa zameriava na deklaratívny popis želaného stavu, namiesto procedurálneho popisu, ako želaný stav dosiahnuť.

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/kustomization.yaml` s obsahom:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/polyfea/manifests//controller #?ref=v1.0.0
  - https://github.com/polyfea/manifests//md-shell # should be in the later step, we will replace it later

patches:
  - path: ./patches/service.yaml

```

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/patches/service.yaml` s obsahom:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: polyfea-controller
  namespace: polyfea
spec:
  ports:
  - name: webserver
    protocol: TCP
    port: 80
    targetPort: http
    nodePort: 30331
  type: NodePort
```

Táto úprava mení typ služby `polyfea`, ktorá je pôvodne špecifikovaná [vo vzdialenej špecifickácii](https://github.com/polyfea/manifests/blob/main/controller/service.yaml). Pôvodná špecifikácia implicitne používa typ `ClusterIP`, ktorý sprístupní službu len na internej sieti kubernetes klastra. Upravená verzia používa typ `NodePort` a nastavuje parameter `nodePort: 30331`. To znamená, že k službe `polyfea` možno pristúpiť na porte `30331` hostiteľského počítača klastra. Tento [strategic-merge patch](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#patch-using-path-strategic-merge) súbor nie je úplnou deklaráciou _Service_, obsahuje len identifikátory pre vyhľadanie konkrétneho záznamu (`kind`, `apiVersion`, `name`) a položky, ktoré majú byť upravené (`ports`, `type`).

Ďalším typom _service_, ktorý by sme mohli použiť je typ `LoadBalancer`. Konfigurácia tohto typu je závislá od poskytovateľa klastra, v prípade [Docker Desktop][docker-desktop] by bola služba dostupná na porte 80 nášho počítača. V tomto prípade ale možno použiť iba jednu službu typu `LoadBalancer` v rámci celého klastra. (V prípade klastrov v prostredí Azure alebo AWS, sa každej službe typu `LoadBalancer` priraďuje samostatná _verejná_ IP adresa).

>build_circle:> Pokiaľ používate minikube alebo iný lokálny klaster než _Docker Desktop_, tak možno budete nútený použiť prikaz `kubectl --namespace polyfea port-forward svc/polyfea-controller 30331:80` na presmerovanie portov predtým než otvoríte stránku `http:localhost:30331/fea`.

Týmto spôsobom sme vytvorili konfiguráciu založenú na vopred pripravenej konfigurácii pre systém Polyfea, ktorým budeme spravovať naše microfrontend komponenty. Viac si o tomto riadiči viete prečítať tu: [https://github.com/polyfea/polyfea/blob/main/README.md](https://github.com/polyfea/polyfea/blob/main/README.md).

>info:> Takáto priama závislosť na externých manifestoch, bez špecifikácie príslušnej verzie, je v praxi nežiadúca, keďže riskujeme, že dôjde k zmenám, ktoré narušia funkčnosť našej aplikácie. V praxi by sa tieto manifesty buď skopírovali do lokálneho repozitára, alebo sa určila konkrétna verzia (vetva/tag v GitHub), ktorá bude potom použitá v súbore `kustomization.yaml`.

### 6. Nasadenie aplikácie do lokálneho Kubernetes klastra s využitím Kustomize

V predchádzajúcich krokoch sme vytvorili deklarácie pre našu aplikáciu `ambulance-ufe` a pre infraštrukturálnu aplikáciu `polyfea`. Teraz prejdeme k deklarácii konfigurácie do špecifických prostredí - klastrov. Keďže budeme nasadzovať do nášho lokálneho kubernetes klastra, potrebujeme nasadiť obe aplikácie a vytvoriť subdoménu klastra - [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) - do ktorých ich umiestnime. Nasadenie do klastra rozdelíme do dvoch krokov:

Krok `prepare` - vytvorenie _namespace_ a nasadenie infraštruktúry klastra. Tu budeme nasadzovať služby, ktoré sú podmienkou na to, aby náš klaster bol pripravený na nasadenie samotnej aplikácie;

Krok `install` - nasadenie služieb samotnej aplikácie/aplikácií do klastra. Ak by sme mali klaster kompletne pripravený, tak stačí nasadiť tento krok.

Implicitne tiež predpokladáme, že medzi týmito krokmi existujú nejaké závislosti - napríklad objekty typu `webc` v kroku `install` predpokladajú existenciu rozšírení Kubernetes API typu `polyfea.github.io` z kroku `prepare`.

>info:> Počet krokov sa snažíme limitovať, niekedy však môže byť potrebné rozdeliť nasadenie do viacerých krokov, aby boli splnené určité predpoklady pre nasadenie našej aplikácie - typicky dostupnosť služieb, ktoré definujú spôsob ako nasadiť iné objekty alebo objekty, ku ktorým chceme limitovať prístup pre lepšiu koordináciu jednotlivých tímov.

Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../../infrastructure/polyfea
```

Teraz nam zostáva pripraviť nasadenie aplikácie `ambulance-ufe` a `polyfea-md-shell` do klastra. Vytvorte súbor `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml` s obsahom:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital  # chceme tieto objekty umiestniť do `wac-hospital` namespace
                         # tento zapis preťaží hodnoty "namespace" vo všetkých objektoch
labels:
- pairs:
    app.kubernetes.io/part-of: wac-hospital

resources:
- ../../../apps/<pfx>-ambulance-ufe

```

Overte, že je vaša konfigurácia správna vykonaním príkazu v priečinku `${WAC_ROOT}/ambulance-gitops`:

```ps
kubectl kustomize clusters/localhost/prepare
```

a následne príkaz

```ps
kubectl kustomize clusters/localhost/install
```

Pokiaľ sú vaše konfigurácie správne, zobrazí sa výpis aplikovaných YAML dokumentov, v opačnom prípade musíte opraviť chybu podľa zobrazeného výpisu.

Pred tým ako zmeny nasadíme do klastra, vytvoríme si požadovaný namespace `wac-hospital`. Tento namespace by mohol byť aj súčasťou definícii objektov pre kustomize, ale pre jednoduchosť odstraňovania a opätovného nasadenia do klastra sme sa ho rozhodli vytvoriť manuálne. Vykonajte príkaz.

```ps
  kubectl create namespace wac-hospital
```

Následne aplikujte prípravný krok príkazom v priečinku `${WAC_ROOT}/ambulance-gitops`:

```ps
  kubectl apply -k clusters/localhost/prepare
```

>build_circle:> S najväčšou pravdepodobnosťou sa pri tomto kroku objavia chyby typu
`resource mapping not found for name: ...ensure CRDs are installed first` . Táto chyba je spôsobená tým, že vytvárame nové objekty typu `WebComponent` a `MicroFrontend`, ktoré sú neštandardné a ich definícia sa ešte nestihla aplikovať v klastri. Preto je potrebné príkaz
`kubectl apply -k clusters/localhost/prepare` zopakovať. V ďalšej časti upravíme nasadenia systému polyfea tak aby sa tento problém nevyskytoval.

Je tiež možné, že pri tomto príkaze dostanete upozornenia typu: `Warning: resource ... is missing the kubectl.kubernetes.io/last-applied-configuration annotation ...`. Tieto upozornenia môžete ignorovať, `last-applied-configuration` bude implicitne vytvorená predchadzajúcim príkazom.

Následne aplikujte príkaz:

```ps
kubectl apply -k clusters/localhost/install
```

a overte, že sa novo vytvorené pody dostanú do stavu `Running` príkazom:

```ps
kubectl get pods --namespace wac-hospital
```

V prehliadači otvorte stránku [http://localhost:30331/fea](http://localhost:30331/fea), na ktorej uvidíte aplikačnú obálku s integrovanou mikro aplikáciou. Po stlačení na odkaz _Zoznam čakajúcich_ by ste mali vidieť nasledujúci výstup:

![Integrovaný zoznam čakajúcich](./img/060-02-appshell-list.png)

>build_circle:> Na niektorých systémoch nemusia byť `NodePort`-s prístupné na adrese `localhost`. Pozrite si, akým spôsobom sú prístupné na vašom systéme. Na systéme [Docker Desktop][docker-desktop] je možné použiť adresu `host.docker.internal` alebo `localhost` s portom `30331`.


Nasledujúci obrázok znázorňuje deployment a komunikačnú schému nasadenej aplikácie.

![Komunikácia medzi micro-frontend radičom a nasadeným WebComponent](./img/060-03-k8s-ufe-komunikacia.png)

### 7. Odstránenie manuálne nasadených Kubernetes objektov

Overili sme, že manuálne nasadenie našej aplikácie do Kubernetes klastra
funguje. Hoci sa môže naša štruktúra adresárov zdať komplikovaná pre potreby nasadenia jednoduchej webovej aplikácie, je dôležitým východiskom pre pridávanie ďaľších mikroslužieb v priebehu týchto cvičení.

V ďalšej časti si ukážeme, ako zabezpečiť kontinuálne nasadenie
s využitím aplikácie [FluxCD]

Predtým odstránime všetky manuálne nasadené objekty z klastra:

```ps
kubectl delete -k ./clusters/localhost/install
kubectl delete -k ./clusters/localhost/prepare
kubectl delete namespace wac-hospital
```
