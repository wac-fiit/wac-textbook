# Manuálne nasadenie aplikácie v Microsoft Azure

Vývoj v krátkych cykloch a rýchla spätná väzba sú podstatou agilného vývoja. Možnosť práce s reálnou aplikáciou - namiesto PowerPoint prezentácie - a možnosť pristúpiť k aplikácii v ľubovoľnom čase, umožňuje získať relevantnú spätnú väzbu. Naším cieľom je preto čo najskôr túto aplikáciu nasadiť do verejného dátového centra.

Najjednoduchší spôsob ako nasadiť aplikáciu, je manuálne prekopírovať súbory na miesto v cloude. To je vhodné napr. v prípade, keď nemáme k dispozícii kontinuálnu integráciu alebo vyvíjame aplikáciu lokálne. Ďalšou, čoraz častejšie využívanou možnosťou, je kontajnerizácia aplikácie a následné nasadenie takéhoto kontajnera. Tento spôsob si ukážeme v nasledujúcich krokoch. V ďalšej kapitole si potom nakonfigurujeme automatizovaný spôsob kontinuálneho nasadenia aplikácie.

### 1. Aktivácia Microsoft Azure účtu

Prejdite na stránku [Visual Studio Dev Essentials][vs-essentials], pripojte sa k službám a aktivujte si voľný účet k _Microsoft Azure_ službám. Pri aktivácii budete požiadaný o údaje o kreditnej karte, nebudú ale uplatnené voči nej žiadne poplatky, pokiaľ sa tak explicitne nerozhodnete. Pri cvičeniach budeme používať len služby, ktoré sú zdarma.

![Voľné služby Azure](./img/042-01-AzureFree.png)

Po aktivácii budete presmerovaný na stránku [Azure Portal][azure-portal], na ktorej môžete spravovať svoje prostriedky vo verejnom dátovom centre _Microsoft Azure_.

### 2. Nasadenie aplikácie na Microsoft Azure pomocou Docker kontajnera

V portáli zvoľte v ľavom paneli položku _+ Create a resource_ a zvoľte _Web App_.

Vybete novú skupinu prostriedkov - _Resource Group - Create New_ - a pomenujte ju `WebInCloud-Dojo`.

Skupina prostriedkov združuje prostriedky patriace k jednej logickej aplikácii a umožňuje nakladať s viacerými prostriedkami ako s jedným celkom (zrušenie, vytvorenie, presun medzi účtami, riadenie prístupu, a podobne).

Do poľa _Name_ zadajte meno aplikácie, napr. `<pfx>-ambulance`. Toto meno bude použité ako názov servera, na ktorom bude vaša aplikácia vystavená, preto musí byť názov jedinečný.
  
V rámci sekcie _Instance Details_:

* V riadku _Publish_ vyberte _Docker Container_
* Ako _Operating System_ nastavte _Linux_.
* V riadku _Region_ zvoľte `West Europe` (dátové centrum v Holandsku).

V sekcii _Pricing plan_ zvoľte nový _Linux plan_ (_Create new_). Pomenujte ho `WebInCloud-DojoServicePlan` a zmeňte štandardné nastavenie v poli _Pricing Plan_ na hodnotu _Free F1_.

![Vytvorenie prostriedku typu Web App](./img/042-01-CreateWebApp.png)

Stlačte na tlačidlo _Next : Container_ a upravte voľbu _Image Source_ na hodnotu `Other container registries` a do políčka _Image and tag_ vložte hodnotu `<your-account>/ambulance-ufe:latest`

![Nastavenie docker obrazu pre Azure We Applikáciu](./img/042-02-CreateWebAppDocker.png)

Teraz stlačte na tlačidlo _Review + create_, skontrolujte nastavenia webovej aplikácie a potvrďte voľbou _Create_.

Po chvíli sa v portáli vo vašich notifikáciách zobrazí nová správa _Deployment succeeded_. Zvoľte voľbu _Pin to Dashboard_ (voliteľné) a potom _Go to Resource_. V prehľade webovej aplikácie (_Overview_) vidíte odkaz -_URL_- na novú aplikáciu. Otvorte tento odkaz v novom okne.

![Prehľad o prostriedku Web App](./img/042-03-WebAppOverview.png)

V prehliadači by ste mali vidieť známy zoznam čakajúcich pacientov, tentoraz obslúžený z  dátového centra Microsoft Azure.

Naša aplikácia je nasadená. Ako bolo spomenuté vyššie, toto je najzákladnejší manuálny spôsob nasadenia
aplikácie na cloud. V ďalších krokoch si ukážeme, ako tento proces automatizovať.

>$apple:> V prípade, že máte procesor s arm64 architektúrou, je možné, že bude v tejto chvíli Vaša aplikácia nefunkčná z dôvodu nekompatibility architektúry procesorov. Pokračujte do ďalšej kapitoly, kde bude upravený predpis priebežnej integrácie tak, aby vytváral obraz pre rôzne platformy a architektúry procesora.
