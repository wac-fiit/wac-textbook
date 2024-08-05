# Generovanie kostry obslužného kódu Web API

Podobne ako pri generovaní kódu klienta v predchádzajúcom cvičení, aj pri generovaní kostry obslužného kódu API využijeme nástroje OpenAPI. Rozdiel v tomto prípade spočíva najmä v tom, že v prípade servera generátor nevie určiť požadovanú aplikačnú logiku a preto vygeneruje len základný skeleton, ktorého aktuálnu funkcionalitu musíme doimplementovať.

Na generovanie kódu využijeme generátor [go-gin-server](https://openapi-generator.tech/docs/generators/go-gin-server). Hoci by bola funkcionalita generovaná týmto generátorom pre potreby cvičenia postačujúca, ukážeme si ako upraviť šablóny generátora tak, aby generoval abstraktné typy a aby sme dosiahli opakované generovanie kódu pri prípadnej zmene API špecifikácie bez nutnosti manuálneho prepisovania už existujúceho kódu.

### 1. Príprava na Generovanie Kódu

Vytvorte súbor `${WAC_ROOT}/ambulance-webapi/scripts/run.ps1` s nasledujúcim obsahom:

```ps
param (
    $command
)

if (-not $command)  {
    $command = "start"
}

$ProjectRoot = "${PSScriptRoot}/.."

$env:AMBULANCE_API_ENVIRONMENT="Development"
$env:AMBULANCE_API_PORT="8080"  

switch ($command) {
    "start" {
        go run ${ProjectRoot}/cmd/ambulance-api-service
    }
    "openapi" {
        docker run --rm -ti -v ${ProjectRoot}:/local openapitools/openapi-generator-cli generate -c /local/scripts/generator-cfg.yaml 
    }
    default {
        throw "Unknown command: $command"
    }
}
```

V tomto projekte nebudeme používať `openapi-generator-cli` prostredníctvom npm balíčka, ale budeme priamo využívať jeho implementáciu dodanú ako softvérový obraz. Skript `run.ps1` budeme ďalej rozširovať za účelom automatizácie často vykonávaných príkazov počas vývoja. Teraz vytvorte súbor `${WAC_ROOT}/ambulance-webapi/scripts/generator-cfg.yaml` s nasledujúcim obsahom:

```yaml
generatorName: go-gin-server
outputDir: /local
inputSpec: /local/api/ambulance-wl.openapi.yaml
enablePostProcessFile: true
additionalProperties:
  apiPath: internal/ambulance_wl
  packageName: ambulance_wl
  interfaceOnlyt: true
```

Vytvorte súbor `${WAC_ROOT}/ambulance-webapi/.openapi-generator-ignore` s obsahom:

```text
main.go
go.mod
Dockerfile
README.md
api/openapi.yaml
```

Týmto predpisom zakážeme generovanie súborov, ktoré nechceme vygenerovať.

### 2. Spustenie Generátora

Uložte súbory a v priečinku `${WAC_ROOT}/ambulance-webapi` vykonajte príkaz

```ps
./scripts/run.ps1 openapi
```

Po jeho ukončení sa v priečinku `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl` objavia súbory, ktoré obsahujú obslužné rutiny pre spracovanie požiadaviek prichádzajúcich na API. Napríklad v súbore `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/api_ambulance_waiting_list.go` sa nachádza predpis rozhrania, ktoré je potrebné naimplementovať.

```go
...
type AmbulanceWaitingListAPI interface {
    // CreateWaitingListEntry Post /api/waiting-list/:ambulanceId/entrieso
    // Saves new entry into waiting list 
     CreateWaitingListEntry(c *gin.Context)
...
}
```

>info:> Adresár s menom `internal` zabraňuje prístupu k definíciám v ňom obsiahnutých typov z iných modulov.  _Package_, ktorého implementácia je v priečinku, ktorého ľubovoľný nadradený priečinok sa volá internal, poskytuje svoje zverejnené typy len v rámci modulu, v ktorom je implementovaný.

Ďalej sa pozrieme na súbor `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/routers.go`. V tomto súbore sú zadefinované cesty, ktoré náš web server bude obsluhovať. V princípe ide iba o mapovanie cesty z našej OpenAPI špecifikácie na volanie metódy zadefinovanej rozhraním v súbore ktorý sme si popísali vyššie.

Naše implementácie budeme podsúvať pomocou štruktúri `ApiHandleFunctions`:

```go
type ApiHandleFunctions struct {

	// Routes for the AmbulanceConditionsAPI part of the API
	AmbulanceConditionsAPI AmbulanceConditionsAPI
	// Routes for the AmbulanceWaitingListAPI part of the API
	AmbulanceWaitingListAPI AmbulanceWaitingListAPI
}
```

Táto štruktúra očakáva implementácie oboch našich rozhraní, ktoré sa využijú pri vyvolávaní jednotlivých metód nášho web servera. Štruktúra vstupuje do funkcie na vytvorenie inštancie routera a to buď samotne alebo s už nainicializovanou inštanciou gin enginu:

```go
func NewRouter(handleFunctions ApiHandleFunctions) *gin.Engine
func NewRouterWithGinEngine(router *gin.Engine, handleFunctions ApiHandleFunctions) *gin.Engine
```


### 3. Vytvorenie základnej implementácie

Keďže generátor generuje iba obálku našeho api vytvorenie implementácie ostáva na nás. Na začiatok si vytvoríme iba jednoduchú implementáciu, ktorá bude vraciat HTTP Status kód 501 Not Implemented (neimplementované).

Vytvorte súbor `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/api_ambulance_conditions_impl.go`:

```go
package ambulance_wl

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type implAmbulanceConditionsAPI struct {
}

func NewAmbulanceConditionsApi() AmbulanceConditionsAPI {
	return &implAmbulanceConditionsAPI{}
}

func (o implAmbulanceConditionsAPI) GetConditions(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}
```

Dôležité je si všimnúť, že v jazyku GO sú rozhrania implementované implicitne.
Vytvorili sme si jednoduchý typ `implAmbulanceConditionsAPI`. Implementáciou funkcie `GetConditions` sme implementovali rozhranie `AmbulanceConditionsAPI`. Teraz by sme mohli využiť štruktúru `implAmbulanceConditionsAPI` ako implementáciu tohoto rozhrania. Keďže je pre nás však dôležitá aj kvalita kódu, máme to pripravené tak aby došlo k validácií či daná štruktúra naozaj implementuje naše rozhranie. Táto štruktúra je schválne zadefinovaná malým písmenom na začiatku aby sme zabezpečili že nebude exportovaná mimo nášho balíku. To znamená že nevieme vytvoriť jej inštanciu priamo mimo balíku `ambulance_wl`. Jej inštanciu budeme vytvárať pomocou metódy `NewAmbulanceConditionsApi`. Metóda vracia štruktúru, ktorá musí implementovať rozhranie `AmbulanceConditionsAPI`. V tejto metóde len jednoducho vrátime našu štruktúru, ktorá už toto rozhranie implementuje. Ak by naša štruktúra toto rozhranie neimplementovala tento kód by sa nedal skompilovať.

>homework:> Skúste zakomentovať metódu `GetConditions`. Pokiaľ používaťe VSCode rozšírenie pre GO mali by ste vidiet chybovú hlášku.

Rovnaký princíp využijeme pre vytvorenie `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/api_ambulance_waiting_list_impl.go`:

```go
package ambulance_wl

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type implAmbulanceWaitingListAPI struct {
}

func NewAmbulanceWaitingListApi() AmbulanceWaitingListAPI {
	return &implAmbulanceWaitingListAPI{}
}

func (o implAmbulanceWaitingListAPI) CreateWaitingListEntry(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}

func (o implAmbulanceWaitingListAPI) DeleteWaitingListEntry(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}

func (o implAmbulanceWaitingListAPI) GetWaitingListEntries(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}

func (o implAmbulanceWaitingListAPI) GetWaitingListEntry(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}

func (o implAmbulanceWaitingListAPI) UpdateWaitingListEntry(c *gin.Context) {
	c.AbortWithStatus(http.StatusNotImplemented)
}
```

Náš web server bude teraz schopný obsluhvať metódy, ktoré sme zadefinovali v OpenApi špecifikácie. No naša implementácia zatiaľ vracia iba status kód 501. Skutočnú implementáciu budeme robiť až v nasledujúcich kapitolách. Teraz potrebujeme ešte nastaviť aby gin engine používal náš vygenerovaný router.

Upravte obsah súboru  `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go`: 

```go
package main

import (
    ...
  "github.com/<github_id>/ambulance-webapi/internal/ambulance_wl" @_add_@
)

func main() {
  ...
  // request routings
  handleFunctions := &ambulance_wl.ApiHandleFunctions{ @_add_@
    AmbulanceConditionsAPI:  ambulance_wl.NewAmbulanceConditionsApi(), @_add_@
    AmbulanceWaitingListAPI: ambulance_wl.NewAmbulanceWaitingListApi(), @_add_@
  } @_add_@
  ambulance_wl.NewRouterWithGinEngine(engine, *handleFunctions) @_add_@
  engine.GET("/openapi", api.HandleOpenApi)
  engine.Run(":" + port)
}
```

V priečinku `${WAC_ROOT}/ambulance-webapi` vykonajte príkaz:

```ps
go run cmd/ambulance-api-service/main.go
```

a potom v druhom termináli vykonajte príkaz:

```ps
curl -v http://localhost:8080/api/waiting-list/bobulova/condition
```

Výsledok by mal byť podobný tomuto:

```text
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /api/waiting-list/bobulova/condition HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 501 Not Implemented   @_implemented_@
< Date: Fri, 11 Aug 2023 15:26:48 GMT
< Content-Length: 0
< 
* Connection #0 to host localhost left intact
```

Náš server síce vracia len chybové hlásenie `501 Not Implemented`, ale to je v poriadku, pretože sme vytvorili len kostru servera, ktorý ešte nie je implementovaný. V nasledujúcich krokoch sa budeme venovať implementácii servera. Môžeme ale bez problémov doplniť špecifikáciu a nanovo vygenerovať kostru servera bez toho, aby sme si prepisovali existujúci kód. Navyše, kód nebude kompilovateľný pokiaľ nebudeme mať implementované všetky operácie nášho API.

### 4. Archivácia a Pokročilé Možnosti OpenAPI Generátora

Archivujte zmeny v git repozitári. V priečinku `${WAC_ROOT}/ambulance-webapi` vykonajte príkazy:

```ps
git add .
git commit -m "Kostra servera"
git push
```

Generovanie obslužného kódu môže byť ešte viac rozšírené a prispôsobené pomocou šablón v OpenAPI generátore. Šablóny umožňujú definovať, ako bude vygenerovaný kód vyzerať, a tým poskytujú veľkú flexibilitu pri tvorbe API. Napríklad, môžeme pridať parametre ako vstupné argumenty do metód, čo nám pomôže eliminovať opakujúce sa bloky kódu. Takto je možné šablóny navrhnúť tak, aby generovali efektívnejší a čitateľnejší kód.

Použitím šablón môžeme špecifikovať generovanie kódu pre rôzne jazyky, kde každá šablóna definuje základnú štruktúru a implementáciu pre konkrétny cieľový jazyk. Okrem základných šablón je možné vytvárať aj vlastné šablóny, ktoré môžu byť upravené na mieru konkrétnym požiadavkám projektu.

Aj keď sme sa rozhodli pre jednoduchšiu implementáciu kvôli prehľadnosti a ľahšiemu pochopeniu postupu, OpenAPI generátor ponúka pokročilejšie techniky a možnosti prispôsobenia. Pre viac informácií a príklady týchto techník môžete navštíviť [dokumentáciu](https://openapi-generator.tech/docs/templating/) a [repozitár](https://github.com/OpenAPITools/openapi-generator/tree/v7.0.0-beta/modules/openapi-generator/src/main/resources) generátorov pre openapi-generator-cli, kde sú k dispozícii rôzne šablóny a príklady ich použitia.
