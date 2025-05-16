# RnsrLearnDetect's Validation

Le document suivant permet, **dans Lodex**, d'effectuer des vérifications sur les rnsr trouvés via le web service [RnsrLearnDetect V2](https://services.istex.fr/attribution-dun-rnsr-a-une-affiliation-apprentissage/)

## 1ère étape : Lancement du web service rnsrLearnDetect

> [!IMPORTANT]
> Les adresses à interroger doivent être structurées sous forme de tableau. Si elles apparaissent en simple chaîne séparée par des ; par exemple , appliquer un ```.split(";")``` ou bien si elles sont structurées en matrice (un tableau contenant plusieurs tableaux), appliquez ```.flatten()``` pour obtenir un tableau simple.

Dans Lodex on crée un nouvel enrichissement appelé ```rnsrLearnDetect2```. On renseigne l'adresse du web service ```https://affiliation-rnsr.services.istex.fr/v2/affiliation/rnsr``` et on sélectionne la colonne contenant les adresses, ici appelée ```adresses```.

## 2ème étape : Lancement du web service Loterre 2xk identify

On crée un nouvel enrichissement que l'on nomme ```loterre```, on choisit comme colonne de la source ```rnsrLearnDetect2``` puis on renseigne comme URL ```https://loterre-resolvers.services.istex.fr/v1/2XK/identify```.

Ce web service va, pour chaque rnsr trouvé par le précédent web service, renvoyer des informations sur la structure ayant le rnsr en question.

## 3ème étape : Traitements et enrichissements des résultats obtenus par Loterre

On crée un enrichissement nommé ```loterreTransformed``` et dans le mode avancé on colle ce script :

```
[assign]
path = value
value = get("value.loterre") \
  .map(item => typeof item === '' \
    ? { \
        codeRNSR:            'n/a', \
        codeUniteCNRS:       'n/a', \
        sigle:               'n/a', \
        prefLabelFr:         'n/a', \
        tutellePrincipale:   'n/a', \
        tutelleSecondaire:   'n/a', \
        institutPrincipal:   'n/a', \
        institutSecondaire:  'n/a', \
        isCNRS:              false \
      } \
    : { \
        codeRNSR:            _.get(item, 'codeRNSR[0]',      'n/a'), \
        codeUniteCNRS:       _.get(item, 'codeUniteCNRS[0]', 'n/a'), \
        sigle:               _.get(item, 'sigle.xml$t',      'n/a'), \
        prefLabelFr:         _.get(item, 'prefLabel@fr',      'n/a'), \
        tutellePrincipale:   (_.get(item, 'tutellePrincipale', []).join(', ')   || 'n/a'), \
        tutelleSecondaire:   (_.get(item, 'tutelleSecondaire', []).join(', ')   || 'n/a'), \
        institutPrincipal:   (_.get(item, 'institutPrincipal', []).join(', ')   || 'n/a'), \
        institutSecondaire:  (_.get(item, 'institutSecondaire', []).join(', ') || 'n/a'), \
        isCNRS:              _.includes( \
                               _.get(item, 'tutellePrincipale', []), \
                               'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' \
                             ) \
                             || _.includes( \
                               _.get(item, 'tutelleSecondaire', []), \
                               'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' \
                             ) \
      } \
  )
```

Ce script réalise différentes opérations. Tout d'abord lorsque le web service Loterre ne trouve pas d'informations il renvoie la chaîne **"n/a"**, dans les autres cas il renvoie des objets. Pour chaque chaîne **"n/a"** on va donc créer des objets auxquels on renseigne **"n/a"** afin que tous les résultats aient la même structure.
On récupère ensuite les différentes valeurs dont on a besoin. On crée une clé ```isCnrs``` qui contiendra un booléen. On recherche si 'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' existe dans ```tutellePrincipale``` ou ```tutelleSecondaire```, si c'est le cas ```true``` sera assigné à ```isCnrs```, si ce n'est pas le cas ce sera ```false```.

On obtient donc un tableau avec des objets de ce type :
```
{
"codeRNSR":"200212721Y"
"codeUniteCNRS":"UMR8171"
"sigle":"IMAf"
"prefLabelFr":"Institut des mondes africains"
"tutellePrincipale":"CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE, UNIVERSITE PANTHEON-SORBONNE, ECOLE DES HAUTES ÉTUDES EN SCIENCES SOCIALES, INSTITUT DE RECHERCHE POUR LE DEVELOPPEMENT, AIX-MARSEILLE UNIVERSITE"
"tutelleSecondaire":"ECOLE PRATIQUE DES HAUTES ETUDES"
"institutPrincipal":"Institut des sciences humaines et sociales"
"institutSecondaire":"Institut écologie et environnement"
"isCNRS":booltrue
}
```

## 4ème étape : Vérification des rnsr

On crée un enrichissement nommé ```mergeAndDetectLoterreMatches``` et dans le mode avancé on colle ce script :

```
[assign]
path = value
value = zip(self.value.adresses,self.value.rnsrLearnDetect2).map(([addr, rnsr]) => ({ addr, rnsr })).zip(self.value.loterreTransformed).map(([a, b]) =>_.assign({}, a, b))\
    .map(item => _.assign({}, item, { \
     addr: (item.addr || '').toLowerCase(), \
    sigle: (item.sigle || '').toLowerCase(), \
    codeNumber:   (item.codeUniteCNRS || '').replace(/\D+/g, '')})).map(item => _.assign({}, item, { \
    codeMatch:  item.isCNRS === false                          \
                  ? 'Pas de code CNRS'                             \
                  : (item.addr && item.codeNumber              \
                      && item.addr.includes(item.codeNumber)    \
                      ? 'Code détecté'                                  \
                      : 'Code non détecté'),                                 \
    sigleMatch: item.sigle === 'n/a' \
                  ? 'Pas de sigle' \
                  : (item.addr && item.sigle && item.addr.includes(item.sigle) \
                      ? 'Sigle détecté' \
                      : 'Sigle non détecté') \
  })) \
  .map(item => _.assign({}, item, { \
    result:  item.codeMatch + ' & ' + item.sigleMatch}))
```

- Tout d'abord on associe chaque adresse au rnsr qui lui a été attribué par le web service avec ```zip(self.value.adresses,self.value.rnsrLearnDetect2)```
  
- Ensuite on crée une clé ```addr``` à laquelle on assigne l'adresse et une clé ```rnsr``` dans laquelle on met le rnsr.
  
- On zippe ensuite les objets contenus dans ```loterreTransformed```
  
- On met en minuscule les valeurs de ```addr``` et de ```sigle```

- On crée une clé ```sigleMatch``` qui sera remplie comme suit :
  - S'il n'y a pas de sigle, on avait reseigné "n/a" dans la clé ```sigle```. Dans ce cas on affecte la valeur ```Pas de sigle```
  - On vérifie si la valeur de ```sigle``` est contenue dans l'adresse ```addr```. Si c'est le cas on affecte ```Sigle détecté```, sinon ```Sigle non détecté```
 
- On crée une clé ```codeNumber``` à laquelle on effecte la valeur de ```codeUniteCNRS``` mais on ne garde que les nombres, pour avoir par exemple "5270" au lieu de "UMR5270", les codes labos n'étant pas toujours écrits de la même façon dans les adresses.

- On crée une clé ```codeMatch``` qui sera renseignée comme suit :
  - Si ```isCnrs``` est false alors la structure ne peut logiquement pas avoir de code unité CNRS, on renseigne donc ```Pas de code CNRS```
  - On vérifie si le code de ```codeNumber``` est présent dans l'adresse.  Si c'est le cas on affecte ```Code détecté```, sinon ```Code non détecté```
 
- On crée enfin une dernière clé intitulée ```result``` qui concatène simplement les valeurs de ```codeMatch``` et ```sigleMatch```

Au final on obtient des objets de ce type :

```
{
"addr":"université de lorraine ul, umr7118, centre national de la recherche scientifique cnrs, analyse et traitement informatique de la langue française atilf, université de lorraine, 44 av de la libération, bp 30687 54063 nancy cedex, fr"
"rnsr":"200112505T"
"codeRNSR":"200112505T"
"codeUniteCNRS":"UMR7118"
"sigle":"atilf"
"prefLabelFr":"Analyse et Traitement Informatique de la Langue Française"
"tutellePrincipale":"UNIVERSITE DE LORRAINE, CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE"
"tutelleSecondaire":"n/a"
"institutPrincipal":"Institut des sciences humaines et sociales"
"institutSecondaire":"Institut des Sciences de l'Information et de leurs Interactions"
"isCNRS":booltrue
"codeNumber":"7118"
"codeMatch":"Code détecté"
"sigleMatch":"Sigle détecté"
"result":"Code détecté & Sigle détecté"
}
```

Ici le code **7118** est bien présent dans l'adresse ainsi que le sigle de la structure **atilf**, on obtient donc dans ```result``` **"Code détecté & Sigle détecté"**

La clé ```result``` peut donc contenir 9 valeurs différentes :
+ Pas de code CNRS & Pas de sigle
+ Pas de code CNRS & Sigle détecté
+ Pas de code CNRS & Sigle non détecté
+ Code détecté & Pas de sigle
+ Code détecté & Sigle détecté
+ Code détecté & Sigle non détecté
+ Code non détecté & Pas de sigle
+ Code non détecté & Sigle détecté
+ Code non détecté & Sigle non détecté

> [!TIP]
> Afin de traiter les résultats on peut ensuite filtrer ceux-ci. Par exemple si l'on souhaite faire une colonne avec uniquement les bons rnsr trouvés on peut générer un script du type :
> ```
> [assign]
> path = value
> value = get("value.mergeAndDetectLoterreMatches").filter(obj=>obj.result==="Code détecté & Sigle détecté").map("addr")
> ```
>
> On peut ensuite réaliser d'autres filtres pour traiter les adresses n'ayant pas été validées.
