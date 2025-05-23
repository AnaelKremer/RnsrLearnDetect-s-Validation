# RnsrLearnDetect's Validation

Le document suivant permet, **dans Lodex**, d'effectuer des vérifications sur les rnsr trouvés via le web service [RnsrLearnDetect V2](https://services.istex.fr/attribution-dun-rnsr-a-une-affiliation-apprentissage/)

## 1ère étape : Lancement du web service rnsrLearnDetect

> [!IMPORTANT]
> Les adresses à interroger doivent être structurées sous forme de tableau. Si elles apparaissent en simple chaîne séparée par des ; par exemple , appliquer un ```.split(";")``` ou bien si elles sont structurées en matrice (un tableau contenant plusieurs tableaux), appliquez ```.flatten()``` pour obtenir un tableau simple.
> 
> Même si votre jeu de données n'a qu'une seule adresse par ligne, appliquez castArray() sur les adresses et aussi sur les rnsr détectés afin d'avoir des tableaux. Autrement, la fonction ```.zip()``` de l'étape 4 ne fonctionnera pas.

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
  .zip(self.value.loterreExpandTransformed) \
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
    sigleMatch:     item.addr.includes(item.sigle) \
                           ? 'Sigle détecté' \
                           : null, \
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
    wordsMatch:     _.chain(item.prefLabelFr) \
                       .toLower() \
                       .deburr() \
                       .words() \
                       .filter(w => w.length >= 4 && w !== 'laboratoire' && w !== 'institut') \
                       .filter(w => item.addr.includes(w)) \
                       .value(), \
    trimMatch:      _.chain(item.prefLabelFr) \
                        .toLower() \
                        .deburr() \
                        .words() \
                        .filter(w => w.length >= 4 && w !== 'laboratoire' && w !== 'institut') \
                        .map(w => _.trim(w.slice(0, 3))) \
                        .filter(abr => item.addr.includes(abr)) \
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
            .thru(result => \
              _.size(item.altLabelMatch) > 0 \
                ? (_.size(result) > 0 \
                    ? result + ' & ' + _.size(item.altLabelMatch) + ' altLabel(s) trouvé(s)' \
                    : _.size(item.altLabelMatch) + ' altLabel(s) trouvé(s)') \
                : (_.size(result) > 0 \
                    ? result \
                    : 'Aucun match') \
            ) \
            .thru(result => \
              _.size(item.wordsMatch) > 0 \
                ? (result === 'Aucun match' \
                    ? _.size(item.wordsMatch) + ' mot(s) trouvé(s)' \
                    : result + ' & ' + _.size(item.wordsMatch) + ' mot(s) trouvé(s)') \
                : result) \
            .thru(result=>_.size(item.trimMatch)>0 \
              ? (result==='Aucun match' \
                  ? _.size(item.trimMatch)+' abréviation(s) trouvée(s)' \
                  : result+' & '+_.size(item.trimMatch)+' abréviation(s) trouvée(s)') \
              : result) \
            .value(), \
    addr:               item.addr, \
    prefLabel:          item.prefLabelFr, \
    sigle:              item.sigle, \
    codeUniteCNRS:      item.codeUniteCNRS, \
    altLabel:           item.altLabel, \
    altLabelMatch:      item.altLabelMatch, \
    wordsMatch:         item.wordsMatch, \
    trimMatch:          item.trimMatch, \
    prefLabelMatch:     _.toString(item.prefLabelMatch), \
    tutellePrincipale:  _.compact(item.tutellePrincipale), \
    tutelleSecondaire:  _.compact(item.tutelleSecondaire), \
    institutPrincipal:  _.compact(item.institutPrincipal), \
    institutSecondaire: _.compact(item.institutSecondaire), \
    isCNRS :            item.isCNRS \
  }))
```

- Tout d'abord on associe chaque adresse au rnsr qui lui a été attribué par le web service avec ```zip(self.value.adresses,self.value.rnsrLearnDetect2)```
  
- Ensuite on crée une clé ```addr``` à laquelle on assigne l'adresse et une clé ```rnsrDetect``` dans laquelle on met le rnsr.
  
- On zippe ensuite les objets contenus dans ```loterreTransformed```
  
- On met en minuscule les valeurs de ```addr``` et de ```sigle```

- On crée une clé ```codeNumber``` à laquelle on effecte la valeur de ```codeUniteCNRS```. Si la valeur est **Non trouvé** ou **Non CNRS**, on la garde. Sinon, on retire les caractères alphabétiques pour avoir par exemple "5270" au lieu de "UMR5270", les codes labos n'étant pas toujours écrits de la même façon dans les adresses.

- On crée une clé ```sigleMatch``` qui vérifie si la valeur de ```sigle``` est contenue dans l'adresse ```addr```. Si c'est le cas on affecte **Sigle détecté**, sinon **null**.

- On crée une clé ```codeMatch``` qui sera renseignée comme suit :
  - Si la valeur de ```codeNumber``` est **Non CNRS** ou **Non trouvé** on renvoie **null**.
  - Sinon, si le code de ```codeNumber``` est présent dans l'adresse on renvoie **Code CNRS détecté** sinon **null**.

- On crée une clé ```prefLabelMatch``` où l'on prend l'intitulé de la structure, que l'on normalise, puis on cherche si la chaîne exacte est dans ```addr```. Si c'est le cas on renvoie **Intitulé exact trouvé** sinon **null**
 
- On crée une clé ```altLabelMatch``` dans laquelle on met les appellations alternatives. On retire **Non trouvé** s'il était présent, on capture toutes les valeurs qui contiennent au moins 2 nombres et pour lesquelles on enlève les caractères alphabétiques (le sigle **L2C** est conservé, mais **UMR5221** devient **5221**). On dédoublonne puis on teste si chaque élément du tableau est inclus dans ```addr```, on renvoie dans un tableau les éléments qui apparaissent effectivement dans l'adresse.

- On crée une clé ```wordsMatch``` où l'on prend l'intitulé de la structure, que l'on normalise, puis découpe en mots. On retire ensuite tous les mots de moins de 4 caractères de sorte à enlever les mots vides, on retire également **laboratoire** et **institut** qui sont trop génériques et créeront des faux positifs. On vérifie enfin si chaque mot est inclus dans ```addr``` et renvoie un tableau avec les mots trouvés.

- On crée enfin une dernière clé intitulée ```trimMatch``` qui réalise les mêmes transformations de ```wordsMatch```, mais ici on va abréger chaque mot à ses 3 premiers caractères avant de chercher leur présence dans l'adresse. Cela permet par exemple pour l'adresse **univ toulouse, umr evol & divers biol, cnrs, france** et d'après le prefLabel **Évolution et Diversité Biologique** de renvoyer un tableau de 3 bonnes réponses : **["evo","div","bio"]**.


- On génère ensuite un nouvel objet où l'on restitue tous les résultats ainsi que les informations de Loterre. On crée une nouvelle clé intitulée ```result``` dans laquelle on va verbaliser et synthétiser les résultats.
  
  - Si ```rnsrDetect``` contient **n/a**, cela signifie que le web service n'avait pas trouvé de RNSR, on renvoie donc **RNSR non détecté**
  - Si ```rnsrDetect``` ne contient pas **n/a** (donc contient un RNSR) et que ```codeRNSR``` contient **Non trouvé**, cela signifie que le RNSR détécté n'a pas été retrouvé dans Loterre, c'est le cas pour d'anciennes adresses associées à des laboratoires fermés par exemple. Dans ce cas on renvoie **RNSR non répertorié dans Loterre**.
  - Ensuite on génère un tableau contenant les réponses de ```codeMatch```, ```sigleMatch``` et ```prefLabelMatch```, on retire les **null* donc les résultats négatifs, et on joint les valeurs. Ce qui donne par exemple **"Code CNRS détecté & Sigle détécté"**.
    - On post-traite ensuite cette chaîne, on calcule en parallèle combien de réponses contient ```altLabelMatch```. S'il y a au moins une réponse, on capture le nombre, lui ajoute la chaîne **altLabel(s) trouvé(s)** et on insère cette chaîne dans la chaîne de ```result```. Cela donne par exemple **Sigle trouvé & 1 altLabel(s) trouvé(s)**. Si le tableau ```altLabelMatch``` est vide et que la chaîne ```result``` est vide également, on n'a trouvé aucun résultat, on renvoie alors **Aucun match**.
    - On post-traite maintenant cette chaîne de la même façon avec ```wordsMatch```. Si ```result``` contient **Aucun match** mais que le tableau contient des mots, on renvoie **x mot(s) trouvé(s)**; si ```result``` contient autre chose et que le tableau contient des mots, on ajoute ** & x mot(s) trouvé(s)**; enfin si ```result``` contient **Aucun match** et que la tableau est vide, on conserve **Aucun match**.
    - Enfin on post-traite la chaîne obtenue de la même façon en traitant maintenant ```trimMatch``` en plus. Si ```result``` contient **Aucun match** mais que le tableau contient des abréviations, on renvoie **x abréviation(s) trouvée(s)**; si ```result``` contient autre chose et que le tableau contient des abréviations, on ajoute ** & x abréviation(s) trouvée(s)**; enfin si ```result``` contient **Aucun match** et que la tableau est vide, on conserve **Aucun match**.


A l'issue de tous ces traitements on obtient des objets de ce type :

```
{
"result":"Code CNRS détecté & 3 abréviation(s) trouvée(s)"
"addr":"univ toulouse 3, umr evolut & divers biol 5174, f-31062 toulouse, france"
"prefLabel":"Évolution et Diversité Biologique"
"sigle":"edb"
"codeUniteCNRS":"UMR5174"
"altLabel":[]
"altLabelMatch":[]
"wordsMatch":[]
"trimMatch":[
  "evo",
  "div",
  "bio"]
"prefLabelMatch":""
"tutellePrincipale":[
  "UNIVERSITE TOULOUSE - PAUL SABATIER",
  "CENTRE NATIONAL DE LA RECHERCHE SCIENTIFIQUE",
  "INSTITUT DE RECHERCHE POUR LE DEVELOPPEMENT"]
"tutelleSecondaire":[]
"institutPrincipal":["Institut écologie et environnement"]
"institutSecondaire":["Institut des sciences biologiques"]
"isCNRS":"CNRS"
}
```
