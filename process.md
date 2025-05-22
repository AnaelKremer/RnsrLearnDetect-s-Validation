# RnsrLearnDetect's Validation

Le document suivant permet, **dans Lodex**, d'effectuer des vérifications sur les rnsr trouvés via le web service [RnsrLearnDetect V2](https://services.istex.fr/attribution-dun-rnsr-a-une-affiliation-apprentissage/)

## 1ère étape : Lancement du web service rnsrLearnDetect

> [!IMPORTANT]
> Les adresses à interroger doivent être structurées sous forme de tableau. Si elles apparaissent en simple chaîne séparée par des ; par exemple , appliquer un ```.split(";")``` ou bien si elles sont structurées en matrice (un tableau contenant plusieurs tableaux), appliquez ```.flatten()``` pour obtenir un tableau simple.

Dans Lodex on crée un nouvel enrichissement appelé ```rnsrLearnDetect2```. On renseigne l'adresse du web service ```https://affiliation-rnsr.services.istex.fr/v2/affiliation/rnsr``` et on sélectionne la colonne contenant les adresses, ici appelée ```adresses```.

## 2ème étape : Lancement du web service Loterre 2xk expand

On crée un nouvel enrichissement que l'on nomme ```loterre```, on choisit comme colonne de la source ```rnsrLearnDetect2``` puis on renseigne comme URL ```https://loterre-resolvers.services.istex.fr/v1/2XK/expand```.

Ce web service va, pour chaque rnsr trouvé par le précédent web service, renvoyer des informations sur la structure ayant le rnsr en question.

## 3ème étape : Traitements et enrichissements des résultats obtenus par Loterre

On crée un enrichissement nommé ```loterreTransformed``` et dans le mode avancé on colle ce script :

```
[assign]
path = value
value = get('value.loterre').castArray()\
  .map(item => typeof item === 'string' \
    ? { \
        codeRNSR:            'Non trouvé', \
        codeUniteCNRS:       'Non trouvé', \
        sigle:               'Non trouvé', \
        prefLabelFr:         'Non trouvé', \
        altLabel:            ['Non trouvé'], \
        tutellePrincipale:   ['Non trouvé'], \
        tutelleSecondaire:   ['Non trouvé'], \
        institutPrincipal:   ['Non trouvé'], \
        institutSecondaire:  ['Non trouvé'], \
        isCNRS:              'Non trouvé' \
      } \
    :  { \
        codeRNSR:            _.chain(item.wdt$P3016).castArray().map('xml$t').pull('Non trouvé').uniq().toString().value(), \
        codeUniteCNRS:       _.get(item, 'wdt$P4550.xml$t', 'Non CNRS'), \
        sigle:               _.get(item, 'wdt$P1813.xml$t', 'Pas de Sigle'), \
        prefLabelFr:         _.chain(item).get('skos$prefLabel').castArray().filter(obj => (obj.xml$lang === 'fr')).map('xml$t').toString().value(), \
        altLabel:            _.chain(item) \
                                .get('skos$altLabel').castArray().map('xml$t') \
                                .xor([ \
                                    _.get(item, 'wdt$P1813.xml$t', ''), \
                                    _.get(item, 'wdt$P4550.xml$t', '') \
                                ]) \
                                .compact() \
                            .value(), \
        tutellePrincipale:   _.chain(item).get( 'inist$tutellePrincipale').castArray().map('xml$t').value(), \
        tutelleSecondaire:   _.chain(item).get( 'inist$tutelleSecondaire').castArray().map('xml$t').value(), \
        institutPrincipal:   _.chain(item).get( 'inist$institutPrincipal').castArray().map('xml$t').value(), \
        institutSecondaire:  _.chain(item).get( 'inist$institutSecondaire').castArray().map('xml$t').value(), \
        isCNRS:              _.includes( \
                               _.chain(item).get( 'inist$tutellePrincipale').castArray().map('xml$t').value(), \
                               'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' \
                             ) \
                             || _.includes( \
                               _.chain(item).get( 'inist$tutelleSecondaire').castArray().map('xml$t').value(), \
                               'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' \
                             ) ? 'CNRS' : 'Non CNRS'\
      } \
  ).castArray()
```

Ce script réalise différentes opérations. Tout d'abord lorsque le web service Loterre ne trouve pas d'informations il renvoie la chaîne **"n/a"**, dans les autres cas il renvoie des objets. Pour chaque chaîne **"n/a"** on va donc créer des objets auxquels on renseigne **"Non trouvé"**, sous forme de chaîne ou de tableau afin que tous les résultats aient la même structure.

On récupère ensuite les différentes valeurs dont on a besoin, certaines nécessitent des traitements.

 - ```codeRNSR``` peut apparaître sous forme de tableau de 2 éléments dont l'un est **"Non trouvé"** (donnée d'origine de Loterre, aucun rapport avec les "Non trouvé" du bloc précédent), le cas échéant on le retire pour ne garder que le RSNR.

 - ```altLabel``` contient des autres codes ou noms de laboratoires, mais peut également contenir le sigle ou le codeUniteCNRS déjà présents dans les clés du même nom. Pour éviter de fausser les tests de l'étape 4, on retire ces valeurs si elles sont déjà présentes dans ```sigle``` et/ou dans ```codeUniteCNRS```.
   
 - ```isCnrs``` où l'on cherche si 'CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE' existe dans ```tutellePrincipale``` ou ```tutelleSecondaire```, si c'est le cas **CNRS** sera assigné, si non ce sera **Non CNRS**.

On obtient donc un tableau avec des objets de ce type :
```
{
"codeRNSR":"200212721Y"
"codeUniteCNRS":"UMR8171"
"sigle":"IMAf"
"altLabel":[]
"prefLabelFr":"Institut des mondes africains"
"tutellePrincipale":"CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE, UNIVERSITE PANTHEON-SORBONNE, ECOLE DES HAUTES ÉTUDES EN SCIENCES SOCIALES, INSTITUT DE RECHERCHE POUR LE DEVELOPPEMENT, AIX-MARSEILLE UNIVERSITE"
"tutelleSecondaire":"ECOLE PRATIQUE DES HAUTES ETUDES"
"institutPrincipal":"Institut des sciences humaines et sociales"
"institutSecondaire":"Institut écologie et environnement"
"isCNRS":"CNRS"
}
```

## 4ème étape : Vérification des rnsr

On crée un enrichissement nommé ```mergeAndDetectLoterreMatches``` et dans le mode avancé on colle ce script :

```
[assign]
path = value
value = zip(self.value.adressesconcatFrance, self.value.rnsrLearnDetect2) \
  .map(([addr, rnsrDetect]) => ({ addr, rnsrDetect })) \
  .zip(self.value.loterreTransformed) \
  .map(([a, b]) => _.assign({}, a, b)) \
  .map(item => _.assign({}, item, { \
    addr:            _.chain(item.addr).toLower().deburr().value(), \
    sigle:           _.chain(item.sigle).toLower().deburr().value(), \
    codeNumber:      _.chain(item.codeUniteCNRS) \
                       .thru(val => val === 'Non trouvé' || val === 'Non CNRS' ? val : val.replace(/\D+/g, '')) \
                       .value(), \
    prefLabelFr:     item.prefLabelFr, \
    tutellePrincipale: item.tutellePrincipale \
  })) \
  .map(item => _.assign({}, item, { \
    codeMatch:      item.codeNumber === 'Non CNRS' \
                       ? null \
                       : item.codeNumber === 'Non trouvé' \
                         ? null \
                         : (item.codeNumber && item.addr.includes(item.codeNumber) \
                             ? 'Code CNRS détecté' \
                             : null), \
    sigleMatch:     item.sigle === null \
                       ? 'Pas de sigle' \
                       : (item.addr.includes(item.sigle) \
                           ? 'Sigle détecté' \
                           : null), \
    prefLabelMatch: _.chain(item.prefLabelFr) \
                       .toLower().deburr() \
                       .thru(pref => pref && item.addr.includes(pref) ? 'Intitulé exact trouvé' : null) \
                       .value(), \
    altLabelMatch:  _.chain(item.altLabel) \
                       .castArray() \
                       .without('Non trouvé') \
                       .map(val => /\d.*\d/.test(val) ? val.replace(/\D+/g, '') : val) \
                       .uniq() \
                       .filter(v => v && item.addr.includes(v)) \
                       .value(), \
    wordsMatch:     _.chain(item.prefLabelFr || '') \
                       .toLower() \
                       .deburr() \
                       .words() \
                       .filter(w => w.length >= 4 && w !== 'laboratoire' && w !== 'institut') \
                       .filter(w => item.addr.includes(w)) \
                       .value() \
  })) \
  .map(item => ({ \
    result: item.rnsrDetect === 'n/a' \
      ? 'RNSR non détecté' \
      : (item.rnsrDetect !== 'n/a' && item.codeRNSR === 'Non trouvé') \
        ? 'RNSR non répertorié dans Loterre' \
        : _.chain([item.codeMatch, item.sigleMatch, item.prefLabelMatch]) \
            .filter(v => v) \
            .join(' & ') \
            .thru(base => \
              _.size(item.altLabelMatch) > 0 \
                ? (_.size(base) > 0 \
                    ? base + ' & ' + _.size(item.altLabelMatch) + ' altLabel(s) trouvé(s)' \
                    : _.size(item.altLabelMatch) + ' altLabel(s) trouvé(s)') \
                : (_.size(base) > 0 \
                    ? base \
                    : 'Aucun match') \
            ) \
            .thru(str => \
              _.size(item.wordsMatch) > 0 \
                ? (str === 'Aucun match' \
                    ? _.size(item.wordsMatch) + ' mot(s) trouvé(s)' \
                    : str + ' & ' + _.size(item.wordsMatch) + ' mot(s) trouvé(s)') \
                : str \
            ) \
            .value(), \
    addr:               item.addr, \
    prefLabel:          item.prefLabelFr, \
    sigle:              item.sigle, \
    codeUniteCNRS:      item.codeUniteCNRS, \
    altLabel:           item.altLabel, \
    altLabelMatch:      item.altLabelMatch, \
    wordsMatch:         item.wordsMatch, \
    prefLabelMatch:     _.toString(item.prefLabelMatch), \
    tutellePrincipale:  item.tutellePrincipale, \
    tutelleSecondaire:  item.tutelleSecondaire, \
    institutPrincipal:  item.institutPrincipal, \
    institutSecondaire: item.institutSecondaire, \
    isCNRS :            item.isCNRS \
  }))
```

- Tout d'abord on associe chaque adresse au rnsr qui lui a été attribué par le web service avec ```zip(self.value.adresses,self.value.rnsrLearnDetect2)```
  
- Ensuite on crée une clé ```addr``` à laquelle on assigne l'adresse et une clé ```rnsrDetect``` dans laquelle on met le rnsr.
  
- On zippe ensuite les objets contenus dans ```loterreTransformed```
  
- On met en minuscule les valeurs de ```addr``` et de ```sigle```

- On crée une clé ```codeNumber``` à laquelle on effecte la valeur de ```codeUniteCNRS```. Si la valeur est **Non trouvé** ou **Non CNRS**, on la garde. Sinon, on retire les caractères alphabétiques pour avoir par exemple "5270" au lieu de "UMR5270", les codes labos n'étant pas toujours écrits de la même façon dans les adresses.

- On crée une clé ```sigleMatch``` qui sera remplie comme suit :
  - S'il n'y a pas de sigle, on avait reseigné "n/a" dans la clé ```sigle```. Dans ce cas on affecte la valeur ```Pas de sigle```
  - On vérifie si la valeur de ```sigle``` est contenue dans l'adresse ```addr```. Si c'est le cas on affecte ```Sigle détecté```, sinon ```Sigle non détecté```

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
"isCNRS":true
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
